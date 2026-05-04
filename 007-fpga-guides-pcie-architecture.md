# PCIe architecture: layers + LTSSM + link bring-up

## Контекст FPGA / Vivado / High-Speed Interface

**PCI Express** — это высокоскоростной последовательный интерфейс, который в FPGA обычно реализуется не “с нуля”, а через встроенный PCIe hard block / integrated block плюс пользовательская логика вокруг него. В Vivado для AMD/Xilinx FPGA PCIe IP может работать, например, как **Endpoint** или **Root Port**, а пользовательская логика обычно взаимодействует с ним через AXI4-Stream TLP interface, AXI Bridge, XDMA/QDMA или другой wrapper, в зависимости от выбранного IP. Для UltraScale+ AMD описывает PCIe integrated block как высокоскоростной serial interconnect IP, поддерживающий Endpoint/Root Port modes, разные link widths/speeds и AXI4-Stream интерфейс transaction layer packets.

---

# 1. Общая архитектура PCIe

PCIe удобно понимать как стек уровней:

```
Software / Driver / OS
        |
Configuration Space / BARs / DMA model
        |
Transaction Layer
        |
Data Link Layer
        |
Physical Layer
        |
GT Transceivers / PCB / Connector / Host
```

В FPGA-разработке нас обычно интересуют две стороны:

```
1. PCIe protocol side:
   LTSSM, link training, TLP, DLLP, flow control, configuration space

2. FPGA user side:
   AXI4-Stream, AXI-MM bridge, DMA, BAR logic, interrupts, CDC, reset
```

То есть PCIe — это не просто “быстрые differential lanes”. Это protocol stack, где physical link, надежность передачи, transaction semantics и software enumeration должны сойтись вместе.

---

# 2. Transaction Layer

**Transaction Layer** — верхний уровень PCIe protocol stack.

Он работает с **TLP**, то есть **Transaction Layer Packets**.

Примеры TLP:

```
Memory Read
Memory Write
Completion
Configuration Read
Configuration Write
Message
```

AMD в документации описывает Transaction Layer как верхний слой PCIe architecture, который принимает, буферизует и распространяет TLP; TLP используются для memory, I/O, configuration и message transactions, а сам слой управляет ordering rules и credit-based flow control.

В FPGA это обычно тот уровень, который ближе всего к твоей пользовательской логике.

Например, если ты работаешь с PCIe Endpoint, то host может обращаться к твоему FPGA через BAR:

```
Host CPU / Root Complex
        |
PCIe Memory Write TLP
        |
FPGA PCIe IP
        |
BAR decode / AXI / user logic
```

Или наоборот, FPGA может делать DMA-запись в память host:

```
FPGA DMA engine
        |
Memory Write TLP
        |
PCIe link
        |
Host memory
```

---

# 3. Data Link Layer

**Data Link Layer** находится между Transaction Layer и Physical Layer.

Его задача — сделать обмен TLP между двумя соседними PCIe components надежным на уровне конкретного link.

Упрощенно:

```
Transaction Layer:
    "вот TLP, его надо отправить"

Data Link Layer:
    "я обеспечу надежную передачу этого TLP через link"

Physical Layer:
    "я превращу это в symbols/bits на serial lanes"
```

AMD описывает Data Link Layer как промежуточный слой между Transaction и Physical Layer, отвечающий за надежный механизм обмена TLP между двумя компонентами на link; он также занимается error detection/recovery, initialization services и DLLP, которые несут power management, flow control и TLP acknowledgments.

Здесь появляются важные термины:

```
DLLP — Data Link Layer Packet
ACK/NAK
Replay
Flow Control credits
Link-level reliability
```

Для FPGA-разработчика это обычно скрыто внутри PCIe hard IP, но при debug может всплывать через ошибки link, replay, malformed TLP, credit starvation и т.д.

---

# 4. Physical Layer

**Physical Layer** отвечает за реальный link:

```
serial lanes
GT transceivers
encoding/scrambling
lane polarity
lane reversal
link speed
link width
electrical behavior
training sequences
```

AMD описывает Physical Layer как слой, который связывает Data Link Layer с signaling technology, делится на logical и electrical sub-blocks; logical sub-block выполняет framing/de-framing TLP/DLLP и содержит LTSSM, а data exchange идет через serial lines одного или нескольких GT transceivers.

Для FPGA это очень важная часть, потому что здесь уже появляются аппаратные вопросы:

```
REFCLK
GT quad placement
lane mapping
AC coupling
polarity inversion
reset sequencing
signal integrity
equalization
board losses
connector quality
```

Если link не поднимается, проблема часто находится именно здесь или на границе reset/clocking/GT.

---

# 5. Что такое LTSSM

**LTSSM** — это **Link Training and Status State Machine**.

Это state machine внутри Physical Layer, которая отвечает за:

```
link initialization
receiver detect
lane training
speed negotiation
width negotiation
equalization
transition to L0
recovery
link maintenance
```

AMD прямо указывает, что LTSSM реализован в logical sub-block Physical Layer и handles link initialization, training и maintenance.

Главная цель LTSSM:

```
перевести PCIe link из reset/no-link состояния в рабочее состояние L0
```

`L0` — это нормальное активное состояние link, в котором уже возможен обычный обмен TLP.

---

# 6. Упрощенная картина LTSSM

Типовой путь link training можно представить так:

```
Detect
  |
Polling
  |
Configuration
  |
L0
```

Более полный набор состояний может включать:

```
Detect
Polling
Configuration
L0
Recovery
Disabled
Hot Reset
Loopback
L0s / L1 power states
```

Для первого bring-up важнее всего понимать:

```
если link дошел до L0 — physical/link training в целом прошел;
если link застрял до L0 — нужно смотреть LTSSM state, refclk, reset, lanes, SI, host/root complex.
```

---

# 7. Что значит “link up”

В PCIe есть несколько уровней “оно работает”.

```
1. GT clocks/resets стабильны
2. LTSSM дошел до L0
3. Link negotiated expected speed/width
4. Host увидел устройство
5. Enumeration прошла
6. BAR назначены
7. Driver загрузился
8. Memory read/write работают
9. DMA работает
10. Interrupts работают
```

Очень частая ошибка — считать, что если LTSSM = L0, то “PCIe полностью работает”.

На самом деле `L0` означает, что link training прошел. Но дальше еще нужны configuration space, BARs, driver, command register bits, bus mastering, memory space enable и корректная user logic.

AMD debug checklist прямо предлагает: если IP не виден в `lspci`, проверить LTSSM state на `L0`; а если memory read/write fails, проверять Memory Space Enable и Bus Master Enable в command register.

---

# 8. Link bring-up в FPGA

Упрощенный bring-up PCIe Endpoint в FPGA выглядит так:

```
1. FPGA configured
2. PCIe REFCLK present and stable
3. PCIe reset released correctly
4. GT / PHY initialization done
5. LTSSM starts training
6. Receiver detect sees link partner
7. Link negotiates width and speed
8. LTSSM reaches L0
9. Root Complex enumerates Endpoint
10. Host assigns bus/device/function and BARs
11. OS/driver enables memory/bus mastering
12. User logic starts real transactions
```

В Vivado/AMD flow полезно отличать:

```
link training problem
enumeration problem
driver problem
DMA/user logic problem
```

Они выглядят похоже “PCIe не работает”, но debug у них разный.

---

# 9. FPGA PCIe Endpoint: что внутри IP, а что делаешь ты

Обычно PCIe IP берет на себя:

```
Physical Layer
Data Link Layer
большую часть Transaction Layer
LTSSM
configuration space mechanics
flow control
TLP framing/parsing
error handling на уровне IP
```

Твоя логика обычно отвечает за:

```
BAR behavior
register map
AXI/AXIS integration
DMA engine или подключение XDMA/QDMA
interrupt generation
buffering
CDC между PCIe user clock и остальной логикой
reset sequencing
application protocol
```

Если используешь **XDMA/QDMA**, то часть DMA-протокола уже делает готовый subsystem.

Если используешь “raw” integrated block, придется лучше понимать TLP.

---

# 10. Endpoint и Root Port

В PCIe есть роли:

```
Root Complex / Root Port — обычно CPU/SoC/host side
Endpoint — устройство, например FPGA card
Switch — PCIe коммутатор между ними
```

В типичном FPGA add-in-card сценарии FPGA — это **Endpoint**.

В embedded-системах FPGA/SoC иногда может быть **Root Port**, если она сама управляет Endpoint-устройствами.

Для твоего контекста с SoC и FPGA важно сразу разделять:

```
FPGA как Endpoint:    host/SoC перечисляет FPGAFPGA как Root Port:    FPGA/SoC перечисляет внешние PCIe устройства
```

Архитектура logic, reset, software и debug в этих случаях отличаются.

---

# 11. Configuration Space, BAR и enumeration

После того как link достиг `L0`, Root Complex начинает **enumeration**.

Он читает configuration space Endpoint-устройства:

```
Vendor IDDevice IDClass CodeCapabilitiesBAR sizeMSI/MSI-X capabilityPCIe capabilitylink capability/status
```

BAR — это окно адресного пространства, через которое host обращается к устройству.

Например:

```
Host writes address 0xA000_1000        |Root Complex формирует PCIe Memory Write TLP        |FPGA Endpoint получает TLP        |BAR hit        |user register write
```

Если link в L0, но устройство не видно в OS, проблема может быть в reset timing, FPGA configuration time, PERST#, LTSSM, configuration space или host enumeration.

---

# 12. Почему FPGA configuration time важен

Для PCIe add-in card FPGA должна успеть сконфигурироваться и поднять PCIe Endpoint к моменту, когда host начинает enumeration.

Если FPGA конфигурируется слишком долго, host может “не увидеть” Endpoint.

Для этого у Xilinx/AMD есть темы вроде:

```
Tandem PCIefast configurationconfiguration time budgetingPERST# timing
```

Это отдельная большая тема, но при bring-up ее нельзя игнорировать.

---

# 13. Link width и link speed

PCIe link характеризуется:

```
width: x1, x2, x4, x8, x16speed: Gen1, Gen2, Gen3, Gen4, Gen5...
```

Например:

```
Gen2 x4Gen3 x8Gen4 x4
```

В Vivado IP ты задаешь максимальные capabilities, но фактический link может подняться ниже:

```
ожидали Gen3 x4получили Gen1 x1
```

Это значит, что link в принципе поднялся, но training/negotiation выбрали меньшую скорость или ширину.

AMD debug checklist рекомендует при debug смотреть, в какую negotiated width и speed реально поднялся link, например через Vivado ILA/configuration status signals.

---

# 14. Link bring-up debug: что смотреть первым

При первом bring-up полезно идти сверху вниз:

```
1. Есть ли REFCLK?2. Правильно ли подключен PERST# / reset?3. Совпадает ли lane mapping?4. Правильный ли PCIe IP mode: Endpoint/Root Port?5. Доходит ли LTSSM до L0?6. Какая negotiated width/speed?7. Видно ли устройство в lspci?8. Назначены ли BAR?9. Включены ли Memory Space Enable / Bus Master Enable?10. Работают ли basic reads/writes?11. Работает ли DMA?
```

Для AMD/Vivado link debug можно включить PCIe Link Debug: в Versal PCIe Integrated Block Vivado может сохранять LTSSM state transitions, доступные в Hardware Manager.

---

# 15. Если LTSSM не доходит до L0

Если LTSSM застрял до `L0`, это обычно не проблема драйвера.

Возможные причины:

```
нет или плохой REFCLKPERST# не отпущеннеправильная reset sequenceнеправильный lane mappingперепутана polarityпроблема AC couplingплохой signal integrityне тот PCIe modehost slot неактивенпроблема питанияневерная GT/quad placementнесовместимость speed/equalization
```

AMD PCIe debug checklist для link training issues предлагает проверять LTSSM graph, использовать protocol analyzer/oscilloscope, проверять refclk jitter, power integrity, channel loss и Eye Scan plots.

---

# 16. Если LTSSM = L0, но `lspci` не видит устройство

Это уже другой класс проблемы.

Возможные причины:

```
FPGA подняла link слишком поздно для host enumerationhost не сделал rescanconfiguration space некорректенPERST#/fundamental reset timing неправильныйdevice disabled в BIOS/UEFIошибка mode Endpoint/Root Portне тот slot/topology
```

Практически часто помогают:

```
warm reboot hostPCIe rescan в Linuxпроверка LTSSM = L0проверка negotiated speed/widthпроверка configuration/status signals в ILA
```

AMD checklist тоже предлагает сначала проверить `lspci`, затем warm reboot/re-run `lspci`, а если IP всё еще не обнаружен — проверить LTSSM на `L0`.

---

# 17. Если устройство видно, но обмен не работает

Если устройство есть в `lspci`, но memory read/write или DMA не работают, link training уже не главный подозреваемый.

Дальше проверяются:

```
BAR mappingMemory Space EnableBus Master EnableAXI/AXIS handshakeDMA descriptor setupcompletion handlinginterruptscache coherency / address translationIOMMUCDC внутри FPGAreset user logic
```

Для Endpoint с DMA особенно важно, включен ли **Bus Master Enable**, иначе Endpoint может быть виден, но не сможет инициировать DMA в host memory. AMD debug checklist отдельно указывает проверять Memory Space Enable и Bus Master Enable при failing memory read/write.

---

# 18. В контексте Vivado: что обычно важно

Для PCIe IP в Vivado на первом этапе особенно важны:

```
device/package supportGT quad locationREFCLK pin / MGTREFCLKlane pinsPERST# inputuser clock outputreset outputs/statuslink up / phy link upltssm statenegotiated speed/widthAXI4-Stream или AXI bridge interfaceexample design
```

При первом запуске лучше не начинать сразу со сложной application logic.

Лучше порядок такой:

```
1. Сгенерировать PCIe IP.2. Запустить example design.3. Проверить link training.4. Проверить lspci.5. Проверить basic BAR access.6. Потом добавлять свою user logic.
```

AMD debug materials также рекомендуют проверять проблему на example design и сравнивать LTSSM transition в simulation/example design с Vivado ILA на железе.

---

# 19. Главное резюме

PCIe в FPGA нужно понимать слоями:

```
Physical Layer:    lanes, GT, REFCLK, LTSSM, equalization, link width/speedData Link Layer:    reliable link-level exchange, DLLP, ACK/NAK, flow controlTransaction Layer:    TLP, Memory Read/Write, Completion, Configuration, BAR, DMA
```

**LTSSM** — это state machine, которая поднимает и обслуживает physical link. Для первого bring-up главный признак успеха на низком уровне — переход в `L0`.

Но полный успех PCIe — это не только `L0`:

```
L0+ enumeration+ BAR assignment+ driver+ memory/bus mastering enable+ working user logic / DMA
```

Короткая формула:

```
LTSSM = link живойEnumeration = host увидел устройствоBAR/DMA = protocol и user logic реально работают
```