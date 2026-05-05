# Credit counters внутри PCIe Flow Control

**Credit counters** — это счетчики, через которые PCIe контролирует, сколько TLP определенного типа можно отправить, не переполнив приемный буфер соседнего PCIe-порта.

Главная идея:

```
Receiver имеет ограниченные buffers.
Receiver выдает credits.
Transmitter тратит credits при отправке TLP.
Receiver возвращает credits, когда освобождает buffers.
```

В FPGA/Vivado credit counters важны в основном для понимания:

```
почему PCIe core иногда не принимает новые TLP от user logic;
почему DMA throughput ниже ожидаемого;
какой тип traffic стал bottleneck;
как debug-ить backpressure на RQ/CQ/CC/RC или AXI4-Stream интерфейсах.
```

AMD/Xilinx PCIe integrated blocks выводят flow-control information через специальные `cfg_fc_*` signals; эти сигналы показывают состояние credit pools для Posted, Non-Posted и Completion traffic. ([AMD PG054](https://docs.amd.com/r/en-US/pg054-7series-pcie/Flow-Control-Credit-Information?utm_source=chatgpt.com))

---

# 1. Что именно считает credit counter

Credit counter считает не “байты на линии” и не “количество TLP вообще”, а доступное место в приемном буфере для конкретного класса TLP.

PCIe делит flow control на шесть основных credit pools:

```
PH    — Posted Header
PD    — Posted Data

NPH   — Non-Posted Header
NPD   — Non-Posted Data

CplH  — Completion Header
CplD  — Completion Data
```

AMD PG213/PG346 описывают отдельные `cfg_fc_ph`, `cfg_fc_pd`, `cfg_fc_nph`, `cfg_fc_npd`, `cfg_fc_cplh`, `cfg_fc_cpld` outputs для этих credit classes; выбор того, какую информацию вывести, делается через `cfg_fc_sel[2:0]`. ([AMD PG213](https://docs.amd.com/r/en-US/pg213-pcie4-ultrascale-plus/Configuration-Flow-Control-Interface?utm_source=chatgpt.com), [AMD PG346](https://docs.amd.com/r/en-US/pg346-cpm-pcie/Configuration-Flow-Control-Interface?utm_source=chatgpt.com))

---

# 2. Почему шесть счетчиков, а не один

PCIe traffic бывает разного типа:

```
Posted traffic       — например Memory Write
Non-Posted traffic   — например Memory Read Request
Completion traffic   — например Completion with Data
```

И каждый TLP имеет:

```
header
optional payload
```

Поэтому нужен отдельный учет:

```
места под headers;
места под payload data.
```

Например:

```
Memory Write 64 bytes:
    требует Posted Header credit
    требует Posted Data credits

Memory Read Request:
    требует Non-Posted Header credit
    обычно не требует Non-Posted Data credit

Completion with Data 128 bytes:
    требует Completion Header credit
    требует Completion Data credits
```

---

# 3. Header credit

**Header credit** — это место для одного TLP header.

AMD documentation указывает, что один Header Credit соответствует 3-DWORD или 4-DWORD TLP header. ([AMD PG054](https://docs.amd.com/r/en-US/pg054-7series-pcie/Receiver-Flow-Control-Credits-Available?utm_source=chatgpt.com))

Практически:

```
один TLP обычно тратит 1 header credit
```

Даже если header 3DW или 4DW, это все равно один header credit соответствующего типа.

Примеры:

```
Memory Write TLP:
    - 1 PH

Memory Read Request:
    - 1 NPH

Completion with Data:
    - 1 CplH
```

---

# 4. Data credit

**Data credit** — это место для payload.

AMD documentation указывает, что один Data Credit соответствует 16 bytes payload. ([AMD PG054](https://docs.amd.com/r/en-US/pg054-7series-pcie/Receiver-Flow-Control-Credits-Available?utm_source=chatgpt.com))

Формула:

```
Data credits = ceil(payload_bytes / 16)
```

Примеры:

```
Payload 16 bytes  -> 1 data credit
Payload 32 bytes  -> 2 data credits
Payload 64 bytes  -> 4 data credits
Payload 128 bytes -> 8 data credits
Payload 256 bytes -> 16 data credits
Payload 512 bytes -> 32 data credits
```

Если payload отсутствует:

```
Data credits = 0
```

---

# 5. Пример credit consumption для Memory Write

Memory Write — это Posted TLP.

Например, FPGA DMA отправляет Memory Write на 256 bytes в host memory.

Credit consumption:

```
PH = 1
PD = ceil(256 / 16) = 16
```

То есть:

```
Memory Write 256B -> 1 PH + 16 PD
```

Если Posted Data credits закончились, DMA write path не сможет продолжать отправлять payload-carrying Memory Write TLP, даже если сам PCIe link находится в `L0`.

---

# 6. Пример credit consumption для Memory Read Request

Memory Read Request — это Non-Posted TLP без payload.

Например, FPGA DMA хочет прочитать 512 bytes из host memory.

Сам request обычно тратит:

```
NPH = 1
NPD = 0
```

Но потом host вернет Completion with Data.

Эти Completion TLP уже будут потреблять receive-side Completion buffer resources на стороне FPGA/core:

```
CplH
CplD
```

То есть DMA read ограничивается не только тем, сколько read requests можно отправить, но и тем, насколько хорошо FPGA/core может принять и обработать completions.

---

# 7. Пример credit consumption для Completion with Data

Если FPGA отвечает на Memory Read из host BAR и возвращает 32 bytes:

```
Completion with Data 32B
```

Credit consumption на transmit side относительно receiver credits:

```
CplH = 1
CplD = ceil(32 / 16) = 2
```

То есть:

```
Completion 32B -> 1 CplH + 2 CplD
```

Если Completion credits недоступны, core не должен отправлять Completion TLP до появления credits.

---

# 8. Posted / Non-Posted / Completion в контексте FPGA

Для FPGA Endpoint это удобно связать с направлениями:

```
Host -> FPGA BAR write:
    FPGA принимает Posted traffic

Host -> FPGA BAR read:
    FPGA принимает Non-Posted request
    FPGA отправляет Completion

FPGA -> Host DMA write:
    FPGA отправляет Posted traffic

FPGA -> Host DMA read:
    FPGA отправляет Non-Posted request
    FPGA принимает Completion
```

То есть разные datapath внутри FPGA нагружают разные credit pools.

---

# 9. Tx credits и Rx credits

Важно различать два смысла credit counters.

## Tx-side view

Передатчик хранит счетчики credits, которые ему выдал receiver.

```
"Сколько я еще могу отправить соседу?"
```

Например, FPGA Endpoint хочет отправить Memory Write в Root Port.

Ему нужны Tx Posted credits, выданные Root Port.

## Rx-side view

Receiver имеет свои buffers и сообщает партнеру, сколько места доступно.

```
"Сколько я могу принять от соседа?"
```

AMD PG054 указывает, что integrated block может предоставлять user application информацию о Transaction Layer transmit and receive buffer credit pools, включая current space available, credit limit и consumed information для Posted, Non-Posted и Completion pools. ([AMD PG054](https://docs.amd.com/r/en-US/pg054-7series-pcie/Flow-Control-Credit-Information?utm_source=chatgpt.com))

---

# 10. Limit / consumed / available

Внутри flow control удобно думать о трех величинах:

```
limit       — сколько credits выдано / максимальная граница;
consumed    — сколько credits уже потрачено на отправленные TLP;
available   — сколько еще можно отправить.
```

Упрощенная логика:

```
available = limit - consumed
```

На практике счетчики могут быть modulo counters, иметь scaling и IP-specific encoding, поэтому напрямую считать по этой формуле в RTL обычно не нужно. Но концептуально именно это происходит.

---

# 11. Credit initialization

Когда PCIe link поднимается, стороны должны обменяться начальными flow-control значениями.

Идея:

```
Receiver сообщает transmitter-у initial credits.
Transmitter инициализирует свои credit counters.
После этого transmitter может отправлять TLP только в пределах этих credits.
```

AMD performance paper описывает, что credit-based flow control устраняет packet discard из-за overflow приемного буфера. ([AMD Performance PDF](https://docs.amd.com/api/khub/documents/4uw~7uS2eK5x7lKbNdxWlw/content?utm_source=chatgpt.com))

Для FPGA-разработчика важный вывод:

```
до завершения link / data-link initialization нельзя считать credit state готовым для нормального traffic.
```

---

# 12. Credit update

Receiver возвращает credits, когда освобождает buffer space.

Упрощенно:

```
1. Receiver принял TLP.
2. TLP занял место в buffer.
3. Receiver обработал / передал TLP дальше.
4. Buffer освободился.
5. Receiver сообщает transmitter-у новые credits.
```

Это не обязательно происходит мгновенно после приема TLP.

Поэтому даже на хорошем link может быть временная нехватка credits, особенно при burst traffic.

---

# 13. Почему credit counter может стать bottleneck

Если receiver медленно освобождает buffer, transmitter видит мало available credits.

Итог:

```
TLP отправляются с паузами;
AXI4-Stream tready падает;
DMA throughput падает;
pipeline не держит максимальную скорость.
```

Это может происходить даже при:

```
LTSSM = L0
Gen3/Gen4 speed correct
width correct
lspci видит устройство
BAR работает
```

То есть credit bottleneck — это уже не link-training проблема.

---

# 14. Credit counters и AXI4-Stream backpressure

В Vivado PCIe IP user interface часто основан на AXI4-Stream.

Если core не может принять TLP от user logic, он может снять `tready`.

Ситуация:

```
user logic:
    tvalid = 1

PCIe core:
    tready = 0
```

Одна из возможных причин:

```
недостаточно credits для отправки TLP данного типа
```

Но `tready = 0` не всегда означает именно credits. Это может быть также:

```
reset;
link not ready;
internal buffer full;
configuration not complete;
protocol block busy;
user interface throttling.
```

Поэтому credit counters полезны как дополнительный диагностический сигнал.

---

# 15. `cfg_fc_sel`

Во многих AMD/Xilinx PCIe IP flow-control information multiplexed.

То есть один набор outputs может показывать разные параметры в зависимости от `cfg_fc_sel`.

Например:

```
cfg_fc_sel = выбрать:
    available credits
    limit
    consumed
    received credits
    другие internal FC views
```

Точная расшифровка зависит от конкретного PCIe IP: 7-series, UltraScale, UltraScale+, Versal CPM/PL PCIe могут иметь разные таблицы и кодировки. AMD PG213/PG346 прямо указывают, что flow-control information выбирается через `cfg_fc_sel[2:0]`. ([AMD PG213](https://docs.amd.com/r/en-US/pg213-pcie4-ultrascale-plus/Configuration-Flow-Control-Interface?utm_source=chatgpt.com), [AMD PG346](https://docs.amd.com/r/en-US/pg346-cpm-pcie/Configuration-Flow-Control-Interface?utm_source=chatgpt.com))

Практическое правило:

```
не интерпретируй cfg_fc_* без таблицы именно для своего IP.
```

---

# 16. Типовые `cfg_fc_*` сигналы

Можно встретить сигналы:

```
cfg_fc_ph
cfg_fc_pd
cfg_fc_nph
cfg_fc_npd
cfg_fc_cplh
cfg_fc_cpld
cfg_fc_sel
```

Смысл по названиям:

```
ph   -> Posted Header
pd   -> Posted Data
nph  -> Non-Posted Header
npd  -> Non-Posted Data
cplh -> Completion Header
cpld -> Completion Data
```

В UltraScale+ PG213 эти outputs описаны как Posted/Non-Posted/Completion Header/Data Flow Control Credits, а `cfg_fc_sel` выбирает тип flow-control information, выводимой core. ([AMD PG213](https://docs.amd.com/r/en-US/pg213-pcie4-ultrascale-plus/Configuration-Flow-Control-Interface?utm_source=chatgpt.com))

---

# 17. Scale signals

В некоторых PCIe IP рядом с counters есть scale outputs:

```
cfg_fc_pd_scale
cfg_fc_nph_scale
cfg_fc_npd_scale
cfg_fc_cplh_scale
cfg_fc_cpld_scale
```

Они нужны, потому что credit value может быть масштабирован или закодирован.

AMD PG346 описывает `*_scale` outputs как scale для соответствующего credit number, например Posted Data, Non-Posted Header/Data и Completion Header/Data. ([AMD PG346](https://docs.amd.com/r/en-US/pg346-cpm-pcie/Configuration-Flow-Control-Interface?utm_source=chatgpt.com))

Практическое правило:

```
если есть scale signal, не воспринимай raw cfg_fc_* как простое число credits без учета scale.
```

---

# 18. Что смотреть в ILA

Для debug credit counters полезно вывести:

```
cfg_fc_sel
cfg_fc_ph
cfg_fc_pd
cfg_fc_nph
cfg_fc_npd
cfg_fc_cplh
cfg_fc_cpld

RQ/CC/CQ/RC valid/ready
DMA request valid/ready
DMA payload size
current_speed
negotiated_width
link_up / dl_up
reset/user_ready
```

Важно смотреть вместе:

```
credit counters + tvalid/tready + тип TLP
```

Сами counters без знания того, какой traffic идет, малоинформативны.

---

# 19. Как debug-ить DMA write через credits

DMA write из FPGA в host memory — это Posted Memory Write.

Главные credit pools:

```
PH
PD
```

Если DMA write тормозит, смотреть:

```
достаточно ли PH;
достаточно ли PD;
какой payload size;
как часто RQ/AXIS tready падает;
нет ли backpressure от PCIe core;
не слишком ли маленькие bursts;
не ограничен ли MPS;
нет ли reset/link events.
```

Пример:

```
DMA отправляет 256B MWr:
    нужно 1 PH + 16 PD

Если PD часто около 0:
    Posted Data credits могут ограничивать write throughput.
```

---

# 20. Как debug-ить DMA read через credits

DMA read из host memory сложнее.

FPGA отправляет Memory Read Requests:

```
NPH обычно основной credit для request
```

Затем получает Completions:

```
CplH / CplD pressure на приемной стороне
```

Смотреть:

```
NPH credits для request generation;
количество outstanding reads;
tag availability;
RC/completion path ready;
completion buffering;
MRRS;
split completions;
AXI/stream sink backpressure.
```

Если NPH мало или request scheduler держит мало outstanding запросов, read throughput будет низким.

---

# 21. Почему Memory Read Request почти не грузит data credits

Memory Read Request обычно не имеет payload.

Поэтому он тратит:

```
1 NPH
0 NPD
```

Но он создает будущую нагрузку на Completion traffic.

То есть ошибка в мышлении:

```
"Read request маленький, значит read throughput всегда легко получить."
```

На самом деле read performance зависит от:

```
outstanding request depth;
completion return latency;
completion buffering;
CplD bandwidth;
tag management;
MRRS;
host behavior.
```

---

# 22. Completion traffic для BAR reads

Когда host читает FPGA BAR, FPGA отправляет Completion.

Если host делает много read requests, FPGA completion path должен выдерживать:

```
CplH credits;
CplD credits;
CC stream bandwidth;
user logic read latency;
completion formatting.
```

Для register access это редко bottleneck.

Но для больших host reads через BAR или memory-mapped windows completion traffic может стать важным.

---

# 23. Posted writes могут “забивать” datapath

Memory Writes — posted, поэтому они не требуют Completion.

Это хорошо для throughput.

Но большой поток posted writes может создать pressure на:

```
Posted receive buffers;
BAR/AXI write path;
DMA receive path;
internal FIFOs;
CDC buffers.
```

PCIe flow control защищает от overflow на link level, но твоя user logic должна корректно принимать backpressure от PCIe core.

---

# 24. Почему credits не равны FIFO occupancy в твоем RTL

Credit counter внутри PCIe core связан с PCIe receive/transmit buffer space.

Он не равен напрямую:

```
occupancy твоего AXI FIFO;
occupancy DMA FIFO;
числу слов в user stream;
числу descriptors;
числу outstanding commands.
```

Между ними есть protocol logic.

Например:

```
PCIe core имеет credits,
но твой DMA FIFO пустой -> нечего отправлять.

Твой DMA FIFO полный,
но PCIe core не имеет credits -> отправка заблокирована.
```

Нужно смотреть оба уровня.

---

# 25. Credit counter saturation

Иногда counter может выглядеть “постоянно высоким”.

Это может означать:

```
traffic данного типа почти не идет;
receiver имеет много buffer space;
этот pool не является bottleneck;
значение масштабировано;
pool фактически неограничен для данного сценария;
ты смотришь не тот cfg_fc_sel режим.
```

Поэтому высокий credit counter сам по себе не говорит, что throughput хороший.

---

# 26. Credit counter near zero

Если counter часто около нуля, это может быть признаком bottleneck.

Но нужно уточнить:

```
какой именно pool около нуля;
какой traffic идет;
есть ли tready drops;
есть ли gaps между TLP;
не является ли это нормальным при burst;
не ограничивает ли другой ресурс.
```

Например:

```
PD near zero при DMA write
```

может указывать на Posted Data credit pressure.

А:

```
NPH near zero при DMA read
```

может ограничивать количество read requests in flight.

---

# 27. Credits и MPS

**MPS** — Max Payload Size.

Если MPS больше, один Memory Write TLP может нести больше payload.

Credit consumption по data credits:

```
payload_bytes / 16
```

Например:

```
MPS 128B -> 8 data credits per full TLP
MPS 256B -> 16 data credits per full TLP
MPS 512B -> 32 data credits per full TLP
```

Более крупный payload обычно уменьшает overhead на byte, но требует больше data credits на один TLP.

Детальный throughput impact лучше оставить для будущей заметки **throughput limits**.

---

# 28. Credits и MRRS

**MRRS** — Max Read Request Size.

Он влияет на размер Memory Read Request, который requester может отправить.

Сам request обычно тратит один NPH, но размер запроса влияет на то, сколько completion data вернется.

Например:

```
MRRS 512B:
    один read request может вернуть до 512B completions
```

Completion может прийти несколькими Completion TLP, в зависимости от completer и boundary rules.

Credit counters здесь важны на обоих концах:

```
NPH — чтобы отправить requests;
CplH/CplD — чтобы принять returning completions.
```

---

# 29. Credits и outstanding requests

Для read throughput мало иметь один read request за раз.

Нужно держать несколько requests “in flight”.

Но каждый outstanding request занимает ресурсы:

```
tag;
request context;
NPH credit на отправку;
completion buffer capacity;
downstream processing capacity.
```

Если NPH credits или tags заканчиваются, read request scheduler должен остановиться.

Это может быть нормальным и ожидаемым ограничением.

---

# 30. Credits и tags — разные вещи

Не путать:

```
credit
```

и

```
tag
```

Credit отвечает:

```
можно ли отправить TLP с точки зрения buffer space receiver-а
```

Tag отвечает:

```
как сопоставить Completion с исходным Non-Posted request
```

Для DMA read нужны оба ресурса.

Можно иметь credits, но не иметь свободных tags.

Можно иметь tags, но не иметь NPH credits.

---

# 31. Credit counters и split completions

Большой Memory Read может вернуться несколькими Completion TLP.

Например:

```
FPGA запросила 1024B read
host вернул несколько completions
```

Каждый Completion TLP потребляет:

```
1 CplHceil(payload / 16) CplD
```

На receive side FPGA/core должен быть готов принять burst completions.

Это уже напрямую связано с будущей темой **buffering implication**.

---

# 32. Что происходит, если credits недостаточно

Если недостаточно credits, transmitter не должен отправлять TLP данного типа.

В пользовательском интерфейсе это может проявиться как:

```
tready = 0;
scheduler stops issuing requests;
DMA pauses;
request queue stops draining;
gaps появляются между TLP.
```

Это корректное поведение.

Ошибка начинается тогда, когда user logic:

```
игнорирует tready;
теряет TLP;
не удерживает payload stable;
нарушает AXI4-Stream handshake;
не умеет продолжить после паузы.
```

---

# 33. Credit counters и AXI handshake correctness

Если PCIe core снимает `tready`, user logic обязана удерживать:

```
tvalid;
tdata;
tkeep;
tlast;
tuser;
```

до успешного handshake.

Правило AXI4-Stream:

```
transfer happens when tvalid && tready
```

Если user logic меняет TLP while `tvalid=1` and `tready=0`, это уже не flow-control проблема, а ошибка RTL.

---

# 34. Почему flow control может выглядеть как “рандомные паузы”

Credits возвращаются не непрерывно каждый такт, а через protocol updates и внутреннюю обработку receiver-а.

Поэтому traffic может выглядеть bursty:

```
несколько TLP подрядпаузаснова несколько TLP
```

Это особенно заметно при:

```
маленьких buffers;маленьких credit pools;большом round-trip latency;switch/retimer topology;host memory latency;burst completions.
```

---

# 35. Credit counters и switch topology

Если между Root Port и FPGA есть PCIe switch, flow control работает на каждом link отдельно:

```
Root Port <-> SwitchSwitch <-> FPGA Endpoint
```

FPGA видит credits своего непосредственного link partner — switch downstream/upstream port, а не обязательно конечного host memory subsystem.

Это важно:

```
credit availability на одном hop не гарантирует отсутствие bottleneck дальше по пути
```

---

# 36. Credit counters и Virtual Channels

В базовых FPGA Endpoint designs обычно используется VC0.

Но PCIe flow control conceptually per Virtual Channel.

AMD PG346 имеет `cfg_fc_vc_sel`, который выбирает Virtual Channel для flow-control information, например VC0 или VC1. ([AMD PG346](https://docs.amd.com/r/en-US/pg346-cpm-pcie/Configuration-Flow-Control-Interface?utm_source=chatgpt.com))

Практически для большинства обычных FPGA PCIe проектов:

```
сначала думай про VC0
```

но знать про per-VC nature полезно для продвинутых случаев.

---

# 37. Credit counters после link-up

Сразу после link-up credit counters могут иметь initial values.

AMD PG054 receiver credits section описывает values credits available immediately after `user_lnk_up` assertion and before reception of any TLP; если space для категории exhausted, соответствующий credit available signal показывает zero, а credits возвращаются к initial values после draining TLPs. ([AMD PG054](https://docs.amd.com/r/en-US/pg054-7series-pcie/Receiver-Flow-Control-Credits-Available?utm_source=chatgpt.com))

Практический вывод:

```
снимать credit snapshot полезно после link ready, но до запуска traffic.
```

Это дает baseline.

---

# 38. Как построить простой credit monitor

Для debug можно сделать register block, который показывает:

```
min observed PHmin observed PDmin observed NPHmin observed NPDmin observed CplHmin observed CplDcurrent selected valuestready drop countersTLP sent counters by typecompletion received counters
```

Полезнее всего не только текущее значение, но и minimum за период.

Например:

```
PD current = 80PD minimum = 0
```

Это говорит, что во время теста PD credits полностью исчерпывались, даже если сейчас они восстановились.

---

# 39. Пример debug register map

```
0x00 FC_STATUS0x04 FC_SEL0x08 FC_PH_CUR0x0C FC_PD_CUR0x10 FC_NPH_CUR0x14 FC_NPD_CUR0x18 FC_CPLH_CUR0x1C FC_CPLD_CUR0x20 FC_PH_MIN0x24 FC_PD_MIN0x28 FC_NPH_MIN0x2C FC_NPD_MIN0x30 FC_CPLH_MIN0x34 FC_CPLD_MIN0x40 RQ_TREADY_DROP_CNT0x44 CC_TREADY_DROP_CNT0x48 DMA_WR_TLP_CNT0x4C DMA_RD_REQ_CNT0x50 DMA_CPL_CNT
```

Так можно debug-ить не только через ILA, но и через software.

---

# 40. Как интерпретировать min counters

Примеры:

```
PD_MIN = 0 during DMA write:    Posted Data credits вероятно ограничивали write burst.NPH_MIN = 0 during DMA read:    read request issue мог упираться в Non-Posted Header credits.CPLD_MIN low during heavy inbound read completions:    completion buffering / receive path стоит проверить.PH_MIN high, PD_MIN low:    ограничивает payload buffer, не header buffer.
```

Но всегда проверять вместе с:

```
tready drops;DMA state;payload size;link speed/width;MPS/MRRS;outstanding depth.
```

---

# 41. Частые ошибки понимания credit counters

## Ошибка 1

```
Credit counter = свободное место в моем user FIFO.
```

Нет. Это PCIe flow-control resource, не напрямую твой RTL FIFO.

---

## Ошибка 2

```
Если credits есть, throughput должен быть максимальным.
```

Нет. Могут ограничивать DMA engine, AXI backpressure, MPS/MRRS, host memory, driver, IOMMU.

---

## Ошибка 3

```
Если tready = 0, значит credits закончились.
```

Не обязательно. Это только одна из возможных причин.

---

## Ошибка 4

```
Memory Read Request требует data credits.
```

Обычно нет payload, значит data credits не тратятся на сам request. Но completions потом создают Completion Data pressure.

---

## Ошибка 5

```
Credits одинаковы на всем PCIe path.
```

Нет. Flow control link-local, особенно важно при switch topology.

---

## Ошибка 6

```
Можно игнорировать scale/cfg_fc_sel и читать raw cfg_fc_* как абсолютные credits.
```

Нужно смотреть Product Guide именно своего PCIe IP.

---

# 42. Практическая таблица TLP → credits

|TLP type|Traffic class|Header credit|Data credit|
|---|---|---|---|
|Memory Write with payload|Posted|PH = 1|PD = ceil(payload/16)|
|Memory Read Request|Non-Posted|NPH = 1|обычно NPD = 0|
|Configuration Read|Non-Posted|NPH = 1|обычно NPD = 0|
|Configuration Write|Non-Posted|NPH = 1|NPD зависит от payload|
|Completion without Data|Completion|CplH = 1|CplD = 0|
|Completion with Data|Completion|CplH = 1|CplD = ceil(payload/16)|

---

# 43. Checklist для credit counter debug

```
1. Link в L0 и Data Link init completed?2. Какой traffic сейчас идет: MWr, MRd, Cpl?3. Какие credit pools должны использоваться?4. Какой cfg_fc_sel выбран?5. Учитываются ли scale signals?6. Какие current credit values?7. Какие minimum credit values за тест?8. Есть ли tready drops?9. На каком stream падает ready: RQ, CC, CQ, RC?10. Какой payload size?11. Какие MPS/MRRS?12. Сколько outstanding read requests?13. Хватает ли tags?14. Есть ли backpressure после PCIe core?15. Есть ли host/software/IOMMU ограничения?
```

---

# 44. Главное резюме

**Credit counters** — это инструмент учета PCIe flow-control resources.

Главные правила:

```
1. Есть шесть основных credit pools: PH, PD, NPH, NPD, CplH, CplD.2. Header credit обычно тратится по 1 на TLP.3. Data credit соответствует 16 bytes payload.4. Memory Write тратит Posted credits.5. Memory Read Request тратит Non-Posted Header credits.6. Completion with Data тратит Completion Header/Data credits.7. Credits link-local: они относятся к соседнему PCIe port.8. Нехватка credits проявляется как throttling/backpressure.9. `cfg_fc_*` нужно интерпретировать по Product Guide конкретного IP.10. Credit counters полезны только вместе с TLP type, valid/ready и DMA state.
```

Короткая формула:

```
Header credits отвечают: “сколько TLP headers можно принять?”Data credits отвечают: “сколько payload bytes можно принять?”
```

Практическая формула:

```
Memory Write N bytes:    1 PH + ceil(N/16) PDMemory Read Request:    1 NPHCompletion N bytes:    1 CplH + ceil(N/16) CplD
```