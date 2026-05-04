# PCIe Layers: Transaction Layer / Data Link Layer / Physical Layer

Внутри блока **PCIe architecture** удобно начинать именно с уровней, потому что почти любая PCIe-проблема в FPGA сводится к вопросу:

```
это проблема TLP / Transaction Layer?
это проблема надежной доставки / Data Link Layer?
это проблема физического линка / Physical Layer?
это проблема user logic вокруг PCIe IP?
```

PCIe — это не просто высокоскоростная serial line. Это стек протоколов:

```
Transaction Layer
        |
Data Link Layer
        |
Physical Layer
        |
GT / PCB / Connector / Link Partner
```

В Vivado большая часть этого стека реализована внутри PCIe Integrated Block / PCIe IP. Пользовательская логика обычно видит не “сырые линии”, а интерфейс уровня TLP: например AXI4-Stream Request/Completion streams, AXI bridge, XDMA/QDMA или другой wrapper. В современных AMD/Xilinx PCIe blocks AXI4-Stream interface к customer logic может иметь разные ширины datapath, а в Versal PL PCIe4 описаны отдельные Initiator/Target Request/Completion streams.

---

# 1. Общая картина слоев

Упрощенно:

```
Transaction Layer:
    что нужно сделать?
    Memory Read, Memory Write, Completion, Config, Message

Data Link Layer:
    как надежно доставить packet соседнему PCIe устройству?
    ACK/NAK, replay, LCRC, DLLP

Physical Layer:
    как передать bits/symbols по lanes?
    SerDes, encoding, scrambling, equalization, lane training
```

Еще короче:

```
Transaction Layer — смысл операции
Data Link Layer   — надежность на одном link
Physical Layer    — электрическая/serial передача
```

---

# 2. Важное разделение: end-to-end vs link-local

PCIe transaction может идти через несколько устройств:

```
Root Complex -> Switch -> Endpoint
```

Transaction Layer думает в терминах конечной операции:

```
host читает BAR FPGA
FPGA делает DMA write в host memory
host пишет configuration register
```

Data Link Layer работает только между двумя соседними PCIe компонентами:

```
Root Port <-> SwitchSwitch    <-> Endpoint
```

То есть Data Link Layer не “знает” весь путь. Он обеспечивает надежную доставку на конкретном физическом link. AMD описывает Data Link Layer как промежуточный слой между Transaction и Physical Layer, который обеспечивает надежный exchange между двумя компонентами на link, включая TLP exchange, error detection/recovery, initialization services и DLLP.

---

# 3. Важное для FPGA-разработчика

В FPGA ты редко реализуешь все PCIe layers вручную.

Обычно PCIe IP делает:

```
Physical Layer
Data Link Layer
часть или почти весь Transaction Layer
configuration handling
LTSSM
flow control machinery
error handling
```

А твоя логика делает:

```
BAR register map
DMA/control logic
AXI/AXIS integration
interrupt generation
buffering
CDC к остальным clock domains
reset sequencing
application protocol
```

Если используешь XDMA/QDMA, то еще больше Transaction Layer details скрыто готовым subsystem.

Если используешь raw AXI4-Stream TLP interface, нужно понимать TLP гораздо лучше.

---

# 4. Transaction Layer: что это такое

**Transaction Layer** — верхний слой PCIe protocol stack.

Он формирует и принимает **TLP**:

```
TLP = Transaction Layer Packet
```

TLP — это packet, который описывает операцию:

```
Memory Read
Memory Write
Completion
Configuration Read
Configuration Write
Message
```

Если host пишет в BAR FPGA, это приходит как Memory Write TLP.

Если host читает BAR FPGA, это приходит как Memory Read TLP, а FPGA должна вернуть Completion TLP.

---

# 5. Transaction Layer в FPGA Endpoint

Для FPGA Endpoint типовые transaction scenarios:

```
Host -> FPGA:
    Configuration Read/Write
    Memory Write to BAR
    Memory Read from BAR

FPGA -> Host:
    Completion for host read
    DMA Memory Write
    DMA Memory Read Request
    Interrupt Message
```

Вид с точки зрения FPGA:

```
PCIe IP получает TLP
        |
        v
user-side interface
        |
        v
BAR logic / DMA / application logic
```

В raw TLP-интерфейсе пользовательская логика должна понимать format packets. AMD документация по 7-series PCIe AXI4-Stream interface прямо говорит, что packets, отправляемые в core, должны соответствовать TLP formatting rules PCIe Base Specification, а user application отвечает за корректность packets; core не проверяет, что packet корректно сформирован, и malformed TLP может быть передан.

---

# 6. TLP: из чего состоит packet

Упрощенно TLP состоит из:

```
Header
Optional data payload
Optional digest / ECRC
```

Header содержит смысл операции:

```
тип TLP
адрес
length
requester/completer information
tag
byte enables
attributes
traffic class
```

Для Memory Write TLP обычно есть payload.

Для Memory Read Request payload обычно нет: это запрос на чтение.

Для Completion with Data payload содержит возвращаемые данные.

AMD PG054 показывает, что header TLP может занимать 3 или 4 DWORD в зависимости от TLP format/type, и что packet byte order на AXI4-Stream interface должен соответствовать PCIe Base Specification.

---

# 7. Posted, Non-Posted, Completion

PCIe transactions часто делят на три большие группы:

```
Posted
Non-Posted
Completion
```

## Posted

Posted transaction не требует Completion.

Типичный пример:

```
Memory Write
```

Host отправил Memory Write в BAR FPGA — отдельный Completion от FPGA обычно не нужен.

Смысл:

```
отправил и не ждешь ответа на уровне TLP completion
```

## Non-Posted

Non-Posted transaction требует Completion.

Типичный пример:

```
Memory Read
Configuration Read
Configuration Write
```

Host сделал Memory Read из BAR FPGA — FPGA должна вернуть Completion.

## Completion

Completion — ответ на Non-Posted request.

Например:

```
Memory Read Request
        |
        v
Completion with Data
```

---

# 8. Почему это важно

Если FPGA получает Memory Read TLP, она не может просто “принять его и забыть”.

Она должна вернуть Completion.

Иначе host/driver может зависнуть, получить timeout или ошибку.

Пример:

```
Host reads FPGA BAR register
        |
        v
FPGA receives Memory Read Request
        |
        v
FPGA must return Completion with Data
```

Для Memory Write такого обязательного Completion нет.

---

# 9. Requester и Completer

В PCIe есть роли на уровне transaction:

```
Requester — тот, кто инициирует request
Completer — тот, кто отвечает на request
```

Host читает BAR FPGA:

```
FPGA = Completer
Host/Root Complex = Requester
```

FPGA DMA читает host memory:

```
FPGA = Requester
Host/Root Complex = Completer
```

Один и тот же Endpoint может быть и Completer, и Requester.

---

# 10. Completer path в FPGA

Когда host обращается к BAR:

```
Host -> FPGA Memory Read/Write
```

FPGA работает как **Completer**.

В AMD UltraScale+/Versal-style interfaces это часто выражается через отдельные streams:

```
CQ — Completer Request
CC — Completer Completion
```

Идея:

```
CQ: host request пришел в FPGA
CC: FPGA отправляет completion назад
```

В QDMA документации AMD CQ/CC описываются как modules, которые receive and process TLP requests from remote PCIe agent, а BAR information используется, чтобы определить, куда направить request.

---

# 11. Requester path в FPGA

Когда FPGA сама инициирует transaction к host memory:

```
FPGA -> Host Memory Read/Write
```

FPGA работает как **Requester**.

Обычно это DMA path.

В AMD-style интерфейсах это часто:

```
RQ — Requester Request
RC — Requester Completion
```

Идея:

```
RQ: FPGA отправляет request
RC: FPGA получает completion на свои read requests
```

Для DMA write completions обычно нет, потому что Memory Write — posted.

Для DMA read completion будет, потому что Memory Read — non-posted.

---

# 12. BAR access как Transaction Layer пример

Допустим, host пишет register FPGA:

```
CPU store to mapped BAR address
        |
Root Complex creates Memory Write TLP
        |
PCIe link
        |
FPGA PCIe IP receives TLP
        |
BAR hit
        |
AXI/logic register write
```

Если host читает register FPGA:

```
CPU load from mapped BAR address
        |
Root Complex creates Memory Read TLP
        |
FPGA receives request
        |
user logic provides data
        |
FPGA sends Completion with Data
        |
CPU receives read result
```

Это Transaction Layer behavior.

---

# 13. Configuration transactions

До обычных BAR accesses host делает enumeration.

Он читает configuration space:

```
Vendor ID
Device ID
Class Code
BAR sizes
Capabilities
MSI/MSI-X capability
PCIe capability
```

Configuration Read/Write TLP — это тоже Transaction Layer.

После enumeration host назначает BAR addresses и включает нужные command bits.

Для FPGA logic важно знать, что устройство может быть link-up, но host еще не включил Memory Space Enable или Bus Master Enable. AMD configuration status interface описывает Function Status bits, включая Memory Space Enable и Bus Master Enable, которые используются для enable requests/completions from host logic.

---

# 14. Max Payload Size и Max Read Request Size

На Transaction Layer важны параметры:

```
MPS  = Max Payload Size
MRRS = Max Read Request Size
```

MPS ограничивает payload size TLP.

MRRS ограничивает размер Memory Read Request, который Requester может генерировать.

AMD configuration status interface выводит Max_Payload_Size и Max_Read_Request_Size из Device Control register; документация указывает, что receiver должен handle TLPs as large as configured MPS, а transmitter не должен генерировать TLP payload больше установленного значения.

В этой заметке не будем глубоко разбирать throughput, потому что это лучше вынести в будущую тему про flow control, credits и performance.

---

# 15. Tag и outstanding reads

Для Non-Posted Reads важны tags.

Упрощенно:

```
FPGA отправила Memory Read Request с tag = 5
позже пришел Completion с tag = 5
FPGA понимает, к какому запросу относится ответ
```

Если DMA делает много outstanding read requests, ему нужно хранить context по tags.

В некоторых AMD PCIe blocks встроены managers для read request/completion tags; например Versal PL PCIe4 features упоминают built-in Initiator Read Request/Completion Tag Manager и поддержку большого числа outstanding initiator read request transactions.

---

# 16. Ordering: почему порядок важен

Transaction Layer также связан с ordering rules.

Простая идея:

```
не все PCIe transactions обязаны завершаться строго в порядке отправки
```

Некоторые requests/completions могут приходить с разной задержкой.

Для FPGA DMA это важно:

```
нельзя считать, что completion для read request A всегда придет раньше B,если protocol/ordering это не гарантирует
```

Подробное ordering лучше разбирать отдельно вместе с TLP basics и DMA architecture.

---

# 17. Data Link Layer: что это такое

**Data Link Layer** — слой между Transaction Layer и Physical Layer.

Он отвечает за надежную передачу TLP через один PCIe link.

Его задачи:

```
принимать TLP от Transaction Layer
добавлять link-level служебную информацию
передавать через Physical Layer
получать подтверждения
обнаруживать ошибки
запускать replay при необходимости
обрабатывать DLLP
```

AMD формулирует Data Link Layer как слой, который обеспечивает reliable mechanism for exchange между двумя компонентами на link, включая data exchange TLPs, error detection/recovery, initialization services и generation/consumption DLLP.

---

# 18. TLP и DLLP

На Data Link Layer есть два важных типа packet:

```
TLP  — Transaction Layer Packet
DLLP — Data Link Layer Packet
```

TLP несет transaction:

```
Memory Read
Memory Write
Completion
Configuration
Message
```

DLLP несет служебную link-level информацию:

```
ACK / NAK
flow control updates
power management information
other link-level control
```

AMD документация указывает, что DLLP передают информацию между Data Link Layers двух напрямую соединенных компонентов; DLLP convey power management, flow control и TLP acknowledgments.

---

# 19. Data Link Layer как “надежная доставка”

Упрощенный путь отправки TLP:

```
Transaction Layer создал TLP
        |
        v
Data Link Layer добавил sequence/LCRC/replay context
        |
        v
Physical Layer отправил по lanes
```

На приемной стороне:

```
Physical Layer принял packet
        |
        v
Data Link Layer проверил link-level integrity
        |
        v
Transaction Layer получил TLP
```

Если обнаружена ошибка, Data Link Layer может использовать NAK/replay mechanism.

---

# 20. ACK/NAK и replay

PCIe link не просто “шлет packet и надеется”.

Data Link Layer использует механизм подтверждений.

Упрощенно:

```
Sender отправил TLP
Receiver проверил
Receiver отправил ACK или NAK через DLLP
Sender удаляет TLP из replay buffer или переотправляет
```

На уровне FPGA-разработчика это обычно скрыто внутри PCIe IP.

Но это объясняет, почему внутри PCIe IP есть replay buffers и почему link errors могут снижать throughput.

AMD Versal PL PCIe4 features, например, упоминают UltraRAM buffering для TLP, включая 32 KB Replay Buffer.

---

# 21. LCRC

Data Link Layer добавляет link-level CRC.

Идея:

```
TLP защищается на конкретном link
receiver проверяет, что packet не поврежден
```

Если ошибка обнаружена, packet может быть переотправлен.

Это link-local reliability.

Это не то же самое, что end-to-end application checksum.

---

# 22. Почему Data Link Layer не виден пользователю напрямую

В Vivado PCIe IP пользовательская логика обычно не формирует DLLP и не управляет ACK/NAK вручную.

Ты видишь:

```
TLP streams
configuration/status signals
flow-control related status
link status
errors
```

Но не строишь DLLP сам.

Поэтому для обычной FPGA application logic Data Link Layer — это скорее важный debug/concept layer, чем прямой RTL-интерфейс.

---

# 23. Flow control находится между Transaction и Data Link

PCIe использует credit-based flow control.

В этой заметке глубоко не разбираем credits, потому что это отдельная будущая тема.

Но важно понимать место:

```
Transaction Layer хочет отправлять TLP
Data Link / protocol machinery следит, есть ли credits/buffer space
если credits нет — отправка ограничивается
```

AMD configuration status interface, например, описывает credit-based control для delivery Non-Posted requests across interface: core временно останавливает доставку Non-Posted requests в user logic, когда internal credit count равен нулю, но продолжает доставлять Posted TLPs.

---

# 24. Physical Layer: что это такое

**Physical Layer** — нижний слой PCIe.

Он отвечает за физическую передачу bits/symbols через lanes:

```
TX/RX differential pairs
GT transceivers
serialization/deserialization
clock recovery
encoding/scrambling
lane alignment
equalization
polarity
link training
```

В FPGA это связано с MGT/GTY/GTH/GTYP transceivers, REFCLK, GT placement, lane pins и board-level signal integrity.

---

# 25. Physical Layer в AMD/Vivado PCIe IP

В AMD PCIe documentation Physical Layer описывается как слой, который связывает Data Link Layer с signaling technology и состоит из logical и electrical sub-blocks; logical sub-block делает framing/de-framing TLP/DLLP и содержит LTSSM, а serial exchange идет через GT transceivers.

Практический смысл:

```
часть Physical Layer — protocol logic внутри PCIe hard block
часть Physical Layer — GT transceivers
часть Physical Layer — плата, connector, host/device link partner
```

---

# 26. Что Physical Layer делает с packet

Упрощенно:

```
Data Link Layer передает packet вниз
Physical Layer делает framing/encoding/scrambling
GT превращает parallel data в serial stream
serial lanes передают signal
приемник делает обратное преобразование
```

На этом уровне уже работают:

```
lane deskew
lane reversal support
polarity inversion
equalization
receiver detection
electrical idle
speed change
```

Часть этих возможностей зависит от конкретного FPGA family/IP.

---

# 27. Link width и speed — Physical Layer результат

Physical Layer отвечает за negotiated:

```
link width: x1, x2, x4, x8, x16
link speed: Gen1, Gen2, Gen3, Gen4, Gen5...
```

В Vivado/AMD IP можно задавать максимальные capabilities, но фактический результат определяется training и link partner.

Например:

```
configured capability: Gen3 x4
actual negotiated:     Gen1 x1
```

Это уже не Transaction Layer проблема. Это lower-layer/link-training вопрос.

AMD configuration status interface имеет signals для negotiated width и current speed; negotiated width valid when link status shows DL initialization complete, а current speed кодирует 2.5/5.0/8.0/16.0/32.0 GT/s.

---

# 28. LTSSM живет в Physical Layer

LTSSM — это часть Physical Layer logical sub-block.

Он отвечает за:

```
Detect
Polling
Configuration
L0
Recovery
power/link states
```

Подробно LTSSM ты планируешь разобрать отдельно, поэтому здесь только граница:

```
LTSSM отвечает за состояние link
Transaction Layer отвечает за TLP операции
Data Link Layer отвечает за надежность packet exchange
```

Если LTSSM не дошел до L0, Transaction Layer traffic нормально работать не будет.

---

# 29. LinkUp и Data Link initialization

Полезно различать:

```
Physical link training
Data Link initialization
Transaction traffic
```

AMD configuration status interface показывает `phy_link_status` с состояниями вроде no receivers detected, link training in progress, link up with DL initialization in progress, and link up with DL initialization completed.

То есть “link up” может быть промежуточным понятием.

Для полноценной работы хочется:

```
Physical link trained
Data Link initialization completed
Configuration/enumeration completed
```

---

# 30. Как TLP проходит через слои

Пример: FPGA DMA Memory Write в host memory.

```
User DMA logic
    |
    v
Transaction Layer:
    формирует Memory Write TLP
    address + byte count + payload
    |
    v
Data Link Layer:
    добавляет link-level reliability context
    sequence / LCRC / replay handling
    |
    v
Physical Layer:
    передает packet через lanes
    |
    v
Root Complex receives
```

Memory Write — posted transaction, поэтому Completion TLP обычно не возвращается.

---

# 31. Пример: Host читает FPGA BAR

```
Host CPU load from BAR
    |
    v
Root Complex создает Memory Read Request TLP
    |
    v
Physical/Data Link доставляют TLP до FPGA
    |
    v
FPGA Transaction Layer получает request
    |
    v
User BAR logic готовит read data
    |
    v
FPGA sends Completion with Data TLP
```

Здесь участвуют все уровни, но ошибка может быть на разных местах:

```
нет link — Physical/LTSSM
link есть, но packet corrupted/replay storm — Data Link / SI
request пришел, completion не отправлен — Transaction/user logic
completion malformed — Transaction/user logic
```

---

# 32. Где слои видны в Vivado

В Vivado PCIe IP layers напрямую не выглядят как три отдельных RTL-модуля, но их следы есть:

```
Physical Layer:    ltssm_state    phy_link_up/down    negotiated speed/width    GT status    refclk/reset signalsData Link Layer:    link active / DL initialization    replay/error/status signals    flow control related statusTransaction Layer:    AXI4-Stream TLP interfaces    CQ/CC/RQ/RC    BAR hit    cfg/status    max payload/read request
```

AMD configuration status interface прямо связывает link-down status с Physical Layer LTSSM и отдельно показывает link status including DL initialization completed.

---

# 33. AXI4-Stream и TLP

В raw PCIe IP integration пользовательская логика часто работает через AXI4-Stream.

Это не “обычный поток байтов”, а поток TLP.

Например, на transmit side:

```
s_axis_* или RQ/CC stream        |        vPCIe core transmits TLP
```

На receive side:

```
PCIe core receives TLP        |        vm_axis_* или CQ/RC stream
```

AMD PG054 описывает TLP format on AXI4-Stream interface, включая byte order и размещение bytes/DWORD на datapath.

---

# 34. 7-series vs UltraScale+/Versal interface style

В старых/простых PCIe cores можно встретить более общий TX/RX AXI4-Stream interface.

В UltraScale+/Versal-style integrated blocks часто используются четыре логически разделенных stream:

```
CQ — Completer RequestCC — Completer CompletionRQ — Requester RequestRC — Requester Completion
```

Это удобно, потому что разделяет роли:

```
host requests into FPGAFPGA completions to hostFPGA requests to hosthost completions into FPGA
```

Versal PL PCIe4 features explicitly mention AXI4-Stream interface to customer logic with four independent Initiator/Target Request/Completion streams.

---

# 35. Layer mapping для CQ/CC/RQ/RC

Упрощенная таблица:

|Stream|Смысл|Типичная роль FPGA|
|---|---|---|
|CQ|Requests from link to FPGA as Completer|Host читает/пишет BAR|
|CC|Completions from FPGA as Completer|FPGA отвечает на host reads|
|RQ|Requests from FPGA as Requester|DMA read/write requests|
|RC|Completions received by FPGA as Requester|DMA read completions|

Это Transaction Layer view.

Data Link и Physical layers под этим уже скрыты внутри IP.

---

# 36. Что значит malformed TLP

Malformed TLP — это packet, который нарушает expected TLP format/rules.

Например:

```
неверный headerнедопустимое lengthнеправильное сочетание fieldspayload не соответствует declared lengthunsupported request format
```

Если пользовательская логика сама формирует TLP, это становится ее ответственностью.

AMD PG054 предупреждает, что user application отвечает за validity packets на transaction interface, а core не проверяет, что packet correctly formed, что может привести к malformed TLP transmission.

---

# 37. Почему с XDMA/QDMA проще

XDMA/QDMA скрывают много TLP деталей.

Например, вместо ручной генерации Memory Read/Write TLP пользователь работает с:

```
AXI memory-mapped interfaceAXI4-Stream interfacedescriptor ringDMA control registersinterrupts
```

PCIe subsystem сам делает request/completion machinery.

Но понимание layers все равно нужно для debug:

```
link не поднялся — Physical/LTSSMустройство не видно — config/enumerationDMA не идет — Transaction/DMA/driver/IOMMU/creditsreplay/errors — Data Link/SI
```

---

# 38. Layer-specific debug мышление

Если проблема:

```
LTSSM не L0negotiated x1 вместо x4Gen1 вместо Gen3link периодически падает в Recovery
```

Сначала думать про Physical Layer / LTSSM / board / GT.

Если проблема:

```
replay count растетLCRC errorslink нестабилен под нагрузкой
```

Думать про Data Link symptoms, но root cause часто physical signal integrity.

Если проблема:

```
BAR read не возвращает данныеDMA read completions не сходятсяhost видит Unsupported Requestmalformed TLP
```

Думать про Transaction Layer / user logic / DMA / driver.

---

# 39. Не путать Data Link Layer и “data payload”

Название **Data Link Layer** не означает “слой пользовательских данных”.

Payload пользовательских данных находится внутри TLP, то есть относится к Transaction Layer packet.

Data Link Layer — это link-level reliability/control layer.

То есть:

```
TLP payload          -> Transaction LayerACK/NAK/replay/LCRC  -> Data Link LayerSerDes/lane/equalize -> Physical Layer
```

---

# 40. Не путать Physical Layer и “все, что связано с FPGA pins”

Physical Layer включает GT/pins/lanes, но для FPGA bring-up на нижнем уровне также важны:

```
REFCLKPERST#power railsGT reference clockingconstraints / pin planningboard routingconnectorhost slot behavior
```

Часть этого находится вне PCIe protocol layer model, но влияет именно на Physical Layer symptoms.

---

# 41. Где configuration space находится в этой модели

Configuration transactions — это TLP на Transaction Layer.

Configuration space itself — это набор registers/capabilities, через которые host узнает устройство.

В FPGA IP часть configuration space реализована внутри PCIe core.

User logic может видеть status/config signals:

```
Memory Space EnableBus Master EnableMax Payload SizeMax Read Request Sizepower statelink status
```

AMD configuration status interface перечисляет такие signals, включая command register bits, Max Payload, Max Read Request, link status, width и speed.

---

# 42. Где BAR находится в этой модели

BAR — это часть configuration/Transaction Layer semantics.

Host назначает BAR address при enumeration.

Когда host обращается к этому address range, Root Complex генерирует Memory Read/Write TLP.

FPGA PCIe IP определяет BAR hit и передает request в user logic.

То есть BAR — это мост между:

```
host address map        |PCIe Transaction Layer        |FPGA register/memory map
```

---

# 43. Где interrupts находятся в этой модели

PCIe interrupts — это тоже protocol-level events.

В modern PCIe обычно используют:

```
MSIMSI-X
```

Они передаются как PCIe messages / memory-write-like mechanisms, в зависимости от типа.

Для FPGA это обычно выглядит как interrupt request interface к PCIe IP или DMA subsystem.

Подробно interrupts лучше вынести в отдельную тему после BAR/DMA.

---

# 44. Layer boundaries на примере ошибки

Допустим, Linux видит устройство в `lspci`, BAR назначен, но чтение BAR зависает.

Это значит:

```
Physical Layer: скорее всего достаточно работает, раз устройство перечисленоData Link Layer: базовая доставка работаетTransaction Layer / user logic: вероятный источник проблемы
```

Проверять:

```
приходит ли Memory Read request в CQ/RX streamформируется ли Completionправильный ли Completion length/status/tagне заблокирован ли CC/TX streamвключен ли Memory Space Enableне сброшена ли user logic
```

---

# 45. Еще пример ошибки

Устройство не видно в `lspci`, а LTSSM не доходит до L0.

Это почти не имеет отношения к BAR/TLP/DMA.

Сначала проверять:

```
REFCLKPERST#lane mappingGT resetполярность laneslink partnerpowersignal integrityVivado PCIe IP config
```

Это Physical Layer / link training problem.

Подробно это лучше оставить для следующей заметки про **link training sequence**.

---

# 46. Еще пример ошибки

Устройство видно, DMA работает на малой скорости, но под нагрузкой throughput падает или появляются replay/errors.

Возможные уровни:

```
Data Link Layer:    replay, ACK/NAK, link-level errorsPhysical Layer:    SI проблемы, equalization, channel lossTransaction/flow control:    credits, buffer limitations, MPS/MRRS, outstanding requests
```

Здесь уже важно разделять Data Link reliability и Transaction-level throughput.

Эту тему логично разобрать позже в **PCIe flow control credits**.

---

# 47. Практическая таблица уровней

|Уровень|Основные объекты|FPGA/Vivado признаки|
|---|---|---|
|Transaction Layer|TLP, BAR, Completion, DMA, Config|CQ/CC/RQ/RC, AXIS TLP, cfg/status, BAR hit|
|Data Link Layer|DLLP, ACK/NAK, replay, LCRC, credits|DL init, replay/error symptoms, flow-control related status|
|Physical Layer|lanes, GT, LTSSM, speed, width|LTSSM, link_up, negotiated width/speed, GT status|

---

# 48. Что нужно запомнить

Главная модель:

```
Transaction Layer:    формирует meaning операцииData Link Layer:    обеспечивает reliable packet delivery на одном linkPhysical Layer:    передает packet по serial lanes и поднимает/поддерживает link
```

Для FPGA-разработчика:

```
если работаешь с BAR/DMA/TLP — ты на Transaction Layer sideесли видишь replay/LCRC/credit симптомы — смотри Data Link/flow controlесли link не L0 или width/speed неправильные — смотри Physical/LTSSM/GT/board
```

Короткая формула:

```
TLP говорит, что сделать.DLLP помогает надежно довезти TLP по link.Physical Layer превращает это в serial traffic по lanes.
```

---

# 49. Мини-checklist для чтения PCIe IP signals

Когда смотришь PCIe IP в Vivado или ILA, полезно группировать signals:

```
Physical:    ltssm_state    phy_link_down    phy_link_status    negotiated_width    current_speed    gt_reset_doneData Link:    dl initialization status    replay/error indicators    flow-control statusTransaction:    CQ/CC/RQ/RC valid/ready    BAR hit    cfg command bits    max_payload    max_read_request    MSI/MSI-X status
```

Так debug становится не хаотичным, а layered.

---

# 50. Главное резюме

**Transaction Layer** — это уровень операций: Memory Read/Write, Completion, Configuration, Message, BAR, DMA.

**Data Link Layer** — это надежность на одном PCIe link: DLLP, ACK/NAK, replay, error detection, flow-control служебные сообщения.

**Physical Layer** — это serial link: lanes, GT, LTSSM, speed/width, equalization, electrical behavior.

В Vivado пользовательская логика чаще всего соприкасается с PCIe на Transaction Layer через AXI4-Stream TLP streams, AXI bridge или DMA subsystem. Data Link и Physical layers в основном скрыты внутри IP, но именно они часто определяют, поднимается ли link, держится ли он под нагрузкой и почему negotiated speed/width не совпадает с ожиданием.