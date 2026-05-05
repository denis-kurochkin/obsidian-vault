# Throughput limits внутри PCIe Flow Control

**Throughput limits** — это тема о том, почему реальная скорость PCIe-передачи почти всегда ниже красивой цифры вида:

```
PCIe Gen3 x4
PCIe Gen4 x8
PCIe Gen5 x16
```

Link speed и link width дают только **верхнюю физическую границу**. Реальный throughput дополнительно ограничивают:

```
TLP overhead;
MPS;
MRRS;
credits;
outstanding requests;
completion latency;
DMA scheduler;
buffering;
AXI/AXIS backpressure;
host/root complex;
driver;
IOMMU/cache;
clock domain crossings;
reset/recovery events.
```

Главная мысль:

> PCIe throughput — это не только “какая generation и сколько lanes”.  
> Это результат всей цепочки: link → protocol → credits → DMA → buffers → host memory → software.

---

# 1. Теоретическая скорость link

Первый уровень — физическая пропускная способность link.

Например:

```
Gen1: 2.5 GT/s
Gen2: 5.0 GT/s
Gen3: 8.0 GT/s
Gen4: 16.0 GT/s
Gen5: 32.0 GT/s
```

PCI-SIG указывает, что PCIe 3.0 использует 8.0 GT/s и заменяет 8b/10b на более эффективное 128b/130b encoding; в PCIe 2.0 при 5.0 GT/s 8b/10b дает 20% overhead, а PCIe 3.0 за счет 128b/130b достигает примерно вдвое большей delivered bandwidth.

Упрощенная полезная таблица на одну lane, one direction:

|PCIe Gen|Raw rate|Encoding|Примерная полезная скорость на lane|
|---|---|---|---|
|Gen1|2.5 GT/s|8b/10b|~250 MB/s|
|Gen2|5.0 GT/s|8b/10b|~500 MB/s|
|Gen3|8.0 GT/s|128b/130b|~985 MB/s|
|Gen4|16.0 GT/s|128b/130b|~1.97 GB/s|
|Gen5|32.0 GT/s|128b/130b|~3.94 GB/s|

Для `x4` умножаем примерно на 4:

```
Gen3 x4 ≈ 3.94 GB/s per direction
Gen4 x4 ≈ 7.88 GB/s per direction
Gen5 x4 ≈ 15.75 GB/s per direction
```

Это еще не application throughput. Это верхняя граница после line encoding, до TLP/protocol/application overhead.

---

# 2. Direction matters

PCIe full-duplex.

Это значит:

```
TX direction имеет свою bandwidth
RX direction имеет свою bandwidth
```

Например:

```
Gen3 x4 ≈ 3.94 GB/s FPGA -> host
Gen3 x4 ≈ 3.94 GB/s host -> FPGA
```

одновременно, теоретически.

Но реальная система может быть асимметричной:

```
DMA write быстрее DMA read;
host-to-card быстрее card-to-host;
read completions хуже держат pipeline;
одна сторона чаще дает backpressure.
```

---

# 3. Negotiated speed/width — первая проверка

Перед любым throughput debug нужно проверить фактические:

```
current speed
negotiated width
```

Vivado IP может быть настроен на Gen4 x8, но реально link может подняться как:

```
Gen3 x4
Gen2 x1
Gen1 x4
```

AMD configuration status interface предоставляет сигналы для current speed и negotiated link width; negotiated width валиден, когда link status показывает завершенную Data Link initialization.

Если negotiated width/speed ниже ожидаемых, performance debug начинается не с credits или DMA, а с:

```
LTSSM / Recovery;
equalization;
slot/root port capability;
BIOS settings;
lane mapping;
signal integrity;
Vivado PCIe IP configuration.
```

---

# 4. Link bandwidth ≠ payload throughput

Даже если link поднялся как Gen3 x4, application не получает все 3.94 GB/s как payload.

Причины:

```
TLP headers;
DLLP;
LCRC;
framing;
idle/control symbols;
packet alignment;
completion headers;
read request overhead;
software/driver overhead;
DMA descriptor overhead.
```

Поэтому лучше различать:

```
raw line rate;
encoded data rate;
PCIe packet throughput;
payload throughput;
application-level throughput.
```

---

# 5. TLP overhead

Каждый TLP несет header.

Упрощенно:

```
payload полезен;
header overhead неизбежен.
```

Если payload маленький, overhead большой.

Например:

```
Memory Write 16B:
    header заметная доля packet

Memory Write 256B:
    header намного меньшая доля packet
```

Поэтому крупные TLP обычно эффективнее, чем мелкие.

---

# 6. MPS — Max Payload Size

**MPS** задает максимальный payload одного TLP.

Типовые значения:

```
128B
256B
512B
1024B
```

AMD configuration status interface описывает Max Payload Size как поле Device Control register; receiver должен уметь принимать TLP такого размера, а transmitter не должен генерировать TLP payload больше установленного MPS.

Практический смысл:

```
чем больше MPS,
тем меньше header overhead на byte payload.
```

---

# 7. MPS example

Допустим, DMA write отправляет 4096 bytes.

При MPS = 128B:

```
4096 / 128 = 32 TLP
```

При MPS = 256B:

```
4096 / 256 = 16 TLP
```

При MPS = 512B:

```
4096 / 512 = 8 TLP
```

Меньше TLP значит:

```
меньше headers;
меньше PH credit pressure;
меньше arbitration overhead;
лучше efficiency.
```

Но больше MPS требует:

```
больше payload buffering;
больше burst handling;
больше PD/CplD credits на один TLP.
```

---

# 8. MPS не всегда можно просто увеличить

MPS выбирается не только FPGA.

Он зависит от:

```
capability FPGA Endpoint;
Root Complex;
switches;
BIOS/UEFI;
OS;
driver;
PCIe hierarchy;
Device Control register.
```

Устройство может рекламировать поддержку 512B, но система может установить 128B.

AMD PG054 по 7-series PCIe описывает advertised MPS в Device Capability register и то, что значение Device Control register выводится в user application через configuration output.

Практически:

```
проверяй фактический MPS в config/status или lspci,а не только настройку Vivado IP.
```

---

# 9. MRRS — Max Read Request Size

**MRRS** задает максимальный размер Memory Read Request, который Requester может отправить.

AMD PG213 указывает, что Max Read Request Size берется из Device Control register bits и ограничивает размер Read Requests, которые logic может генерировать как Requester.

Практический смысл:

```
MRRS влияет на DMA read request granularity.
```

Больше MRRS может означать:

```
меньше read requests;
меньше request header overhead;
больше data per outstanding request.
```

Но:

```
completions могут быть split;
нужен completion buffering;
нужны tags/context;
больше in-flight data.
```

---

# 10. MPS vs MRRS

Коротко:

```
MPS  — максимум payload TLP, который можно отправлять/принимать.
MRRS — максимум размера Memory Read Request как Requester.
```

Для DMA write важнее MPS.

Для DMA read важны оба:

```
MRRS определяет размер read request;
MPS/host/completer behavior влияют на размер completion TLP;
completion buffering определяет, выдержит ли FPGA returning data.
```

---

# 11. Posted write efficiency

DMA write из FPGA в host memory обычно использует Memory Write TLP.

Это Posted traffic.

Плюсы:

```
не нужно ждать Completion;
легче держать pipeline;
обычно проще достичь высокой скорости.
```

Ограничения:

```
PH/PD credits;
MPS;
payload aggregation;
RQ/AXIS backpressure;
DMA source rate;
host memory write acceptance;
descriptor engine;
AXI/stream width.
```

Если источник данных не может обеспечить непрерывный поток, link будет простаивать независимо от credits.

---

# 12. Read throughput обычно сложнее write throughput

DMA read из host memory требует:

```
выдать Memory Read Requests;
держать несколько outstanding requests;
получить Completions;
сопоставить tags;
буферизовать completion payload;
выдать data downstream.
```

То есть read performance зависит от round-trip latency.

Если в каждый момент есть только один outstanding read, throughput будет очень низким.

---

# 13. Bandwidth-delay intuition

Для read throughput нужно покрыть latency количеством данных in flight.

Идея:

```
required_in_flight_bytes ≈ target_bandwidth * round_trip_latency
```

Пример:

```
target = 4 GB/s
round_trip_latency = 1 us

required_in_flight ≈ 4 KB
```

Если у тебя in-flight только 512B, pipeline будет простаивать.

Это не точный расчет PCIe, но очень полезная инженерная интуиция.

---

# 14. Outstanding requests

Для высокого read throughput DMA должен держать несколько read requests in flight.

Ограничители:

```
NPH credits;
free tags;
request context table;
MRRS;
completion buffer space;
host/root complex;
DMA scheduler policy.
```

Если free tags закончились — новые reads нельзя выдавать, даже если credits есть.

Если completion buffer почти full — новые reads выдавать опасно, даже если tags есть.

---

# 15. Tags как throughput resource

Для каждого Non-Posted request нужен tag, чтобы связать Completion с исходным request.

Если доступно мало tags:

```
мало outstanding reads;
плохое latency hiding;
низкий read throughput.
```

Versal PL PCIe4 features упоминают built-in Initiator Read Request/Completion Tag Manager и поддержку большого числа outstanding initiator read request transactions.

Практический вывод:

```
для read performance tags так же важны, как credits и MRRS.
```

---

# 16. Credits как throughput limit

Credit counters могут ограничивать скорость.

Например:

```
DMA write:
    PD credits заканчиваются -> write stalls

DMA read:
    NPH credits заканчиваются -> read requests stalls

Completion receive:
    completion buffering/backpressure -> read data stalls
```

Credit starvation может выглядеть как:

```
tready периодически падает;
TLP идут пачками с паузами;
DMA scheduler часто idle;
link utilization низкий.
```

---

# 17. Credits не единственный bottleneck

Если credits не заканчиваются, throughput все равно может быть низким.

Другие причины:

```
маленькие TLP;
маленький MPS;
маленький MRRS;
мало outstanding requests;
плохая descriptor supply;
AXI backpressure;
CDC FIFO почти full/empty;
host memory latency;
IOMMU;
cache effects;
driver overhead;
interrupt overhead;
test software не способен читать/писать быстро.
```

Поэтому credits — важный, но не единственный performance counter.

---

# 18. Buffering limit

Даже если PCIe link и credits позволяют, throughput ограничит внутренняя буферизация.

Примеры:

```
payload FIFO слишком маленький;completion FIFO часто almost_full;descriptor FIFO empty;stream source FIFO empty;output sink FIFO full;async FIFO между clock domains не выдерживает burst.
```

Типичный симптом:

```
PCIe core мог бы работать быстрее,но user logic постоянно создает backpressure.
```

---

# 19. AXI/AXI4-Stream interface width

PCIe IP user-side datapath имеет конечную ширину и частоту.

Пример:

```
128-bit @ 250 MHz = 4 GB/s theoretical bus payload rate256-bit @ 250 MHz = 8 GB/s512-bit @ 250 MHz = 16 GB/s
```

Если link Gen4 x8 теоретически может больше, но user interface только 128-bit @ 250 MHz, throughput ограничится user interface.

AMD Versal PL PCIe4 features указывают, что AXI4-Stream interface к customer logic может иметь ширины 64/128/256/512 bits.

---

# 20. User clock and backpressure

Даже широкая шина не гарантирует throughput.

Нужно:

```
держать tvalid почти постоянно;получать tready почти постоянно;не создавать bubbles между TLP;иметь enough buffering;не ломать packet boundary.
```

Если каждый TLP separated by gaps, effective throughput падает.

---

# 21. Payload aggregation limit

Если source генерирует маленькие chunks, DMA должен агрегировать их в большие Memory Write TLP.

Плохо:

```
каждые 16B отправляются отдельным Memory Write TLP
```

Лучше:

```
собрать 128B/256B/512B payload и отправить одним TLP
```

Ограничения:

```
latency;packet boundary;flush policy;buffer depth;alignment;MPS.
```

---

# 22. Alignment

Misalignment может ухудшать efficiency.

Возможные проблемы:

```
TLP пересекает boundary;byte enables становятся сложнее;payload splitting;меньше эффективный размер packet;больше completions;AXI burst split;host memory page boundary.
```

AMD QDMA performance guidance рекомендует для optimal QDMA streaming performance выравнивать packet buffers descriptor ring как минимум на 256 bytes; также есть рекомендация ограничивать outstanding descriptor fetch до менее 8 KB.

Практический смысл:

```
alignment и descriptor organization могут реально влиять на performance,даже если link speed правильный.
```

---

# 23. 4 KB boundary

PCIe Memory Read/Write requests имеют ограничения, связанные с 4 KB boundary.

Практически DMA packetizer часто должен split-ить transfers так, чтобы не нарушать boundary rules.

Симптом:

```
большой transfer дробится на большее число TLP,чем ожидалось.
```

Это увеличивает overhead и может снизить throughput.

---

# 24. Descriptor overhead

DMA не просто гонит данные.

Он также:

```
читает descriptors;обновляет status;пишет completion records;генерирует interrupts;обрабатывает rings/queues.
```

Если descriptors маленькие или software часто вмешивается, overhead растет.

Для маленьких transfers throughput часто ограничен не PCIe bandwidth, а per-transfer overhead.

---

# 25. Small transfer problem

Маленькие transfers почти всегда неэффективны.

Например:

```
4 KB transfer
```

может быть намного менее эффективен, чем:

```
4 MB transfer
```

Причины:

```
descriptor overhead;software syscall overhead;interrupt overhead;cache/IOMMU effects;недостаточно времени, чтобы pipeline заполнился;TLP overhead относительно payload.
```

Для throughput-тестов нужно использовать достаточно большие transfers и warm-up.

---

# 26. Interrupt overhead

Если DMA генерирует interrupt на каждый маленький transfer, throughput падает.

Лучше:

```
batch completions;polling для high-rate tests;interrupt coalescing;larger descriptors;larger buffers.
```

Interrupt latency и CPU overhead легко становятся bottleneck.

---

# 27. Host memory subsystem

PCIe throughput зависит не только от FPGA.

Host side может ограничивать:

```
CPU/root complex;memory controller;NUMA topology;IOMMU;chipset;PCIe switch;driver;OS scheduler;cache coherency;DMA mapping;page pinning;thermal/power limits.
```

Если FPGA подключена к одному NUMA node, а memory buffer выделен на другом, throughput может быть хуже.

---

# 28. IOMMU effects

IOMMU может добавлять overhead и менять поведение DMA.

Симптомы:

```
низкий throughput;разные результаты с разными drivers/settings;малые transfers особенно медленные;больше latency на DMA mapping.
```

Это не значит, что IOMMU нужно всегда отключать. Но при performance debug нужно знать, включен ли он и как driver делает mappings.

---

# 29. Cache effects

Для host-to-FPGA или FPGA-to-host tests нужно понимать:

```
CPU читает cached memory или uncached;DMA buffer coherent или non-coherent;есть ли cache flush/invalidate;test меряет PCIe или memory copy/cache speed.
```

Иногда benchmark показывает скорость CPU/cache, а не реальный PCIe throughput.

---

# 30. Driver/test software limit

Плохой benchmark может ограничивать throughput.

Примеры:

```
слишком маленькие buffers;один transfer at a time;blocking API;лишние memcpy;interrupt на каждый transfer;нет queue depth;не pinned memory;неправильный NUMA node.
```

Перед debug FPGA полезно проверить known-good driver/example throughput для этого IP.

AMD QDMA performance guidance как раз разделяет performance debug по направлениям и режимам и дает рекомендации по descriptor/queue behavior.

---

# 31. Direction-specific limits

## FPGA → Host DMA write

Обычно ограничивают:

```
PD credits;MPS;payload aggregation;source stream rate;RQ tready;host memory write acceptance;descriptor supply.
```

## Host → FPGA DMA read

Обычно ограничивают:

```
NPH credits;tags;MRRS;completion latency;completion buffering;RC path;sink readiness.
```

## Host BAR reads/writes

Обычно ограничивают:

```
software overhead;completion latency;BAR logic;AXI-lite/register path;small transaction overhead.
```

BAR не стоит использовать как high-throughput data path, если есть DMA.

---

# 32. BAR access throughput

BAR register access обычно медленный по сравнению с DMA.

Почему:

```
small transactions;CPU MMIO overhead;read requires Completion;ordering/serialization effects;driver/software overhead.
```

BAR хорош для:

```
control registers;status;small configuration;doorbells;debug.
```

Для data plane лучше DMA.

---

# 33. Completion timeout risk

Если FPGA принимает Memory Read Request от host, но долго не отправляет Completion, возможен completion timeout.

Это уже не только performance, а functional reliability.

Поэтому BAR read path должен быть:

```
маленьким;быстрым;приоритетным;не зависящим от перегруженной DMA data path.
```

Для долгих операций лучше:

```
host пишет command;FPGA выполняет;host poll-ит status;потом читает result.
```

---

# 34. Arbitration limits

Если разные источники конкурируют за PCIe transmit path:

```
DMA writes;DMA read requests;BAR completions;MSI/MSI-X;messages;
```

arbiter может ограничить throughput или создать starvation.

Хороший arbiter учитывает:

```
completion priority;credits;available payload;tags;fairness;latency-sensitive traffic;bulk traffic.
```

Плохой arbiter может дать высокий bulk throughput, но сломать BAR reads или interrupts.

---

# 35. Completion priority

Completion traffic должен иметь гарантированный forward progress.

Если DMA write stream постоянно занимает transmit path, а Completion на host BAR read ждет слишком долго, host может получить timeout.

Практическое правило:

```
bulk posted traffic не должен starve completion traffic.
```

---

# 36. Backpressure chain

Throughput часто падает из-за цепочки backpressure.

Пример:

```
application sink full    -> completion FIFO full    -> RC path backpressure    -> read scheduler stops    -> no outstanding reads    -> PCIe link idle    -> throughput low
```

Или:

```
source stream FIFO empty    -> DMA packetizer idle    -> RQ stream idle    -> link idle
```

Нужно искать самый ранний узел, который создает паузу.

---

# 37. CDC limits

Если PCIe user_clk и application_clk разные, async FIFO может стать throughput limit.

Например:

```
PCIe side может передавать 8 GB/sapplication side может принять 5 GB/s
```

FIFO временно сгладит burst, но средняя скорость ограничится 5 GB/s.

Если application side регулярно останавливается, FIFO depth должна покрывать stall или scheduler должен throttle-ить upstream.

---

# 38. FPGA resource/timing limits

Иногда throughput ограничивает сама FPGA-логика:

```
packetizer не держит частоту;wide AXIS path плохо timing closes;BRAM/URAM bandwidth недостаточен;AXI interconnect создает arbitration delays;DDR controller не держит bandwidth;DMA engine имеет bubbles;scheduler не pipelined.
```

В Vivado это проявляется как:

```
низкая достижимая user_clk;много register slices;tready bubbles;очереди не успевают drain/fill.
```

---

# 39. DDR/HBM bandwidth inside FPGA

Если PCIe DMA идет через external DDR/HBM, итоговая скорость ограничена не только PCIe.

Например:

```
PCIe Gen4 x8 может ~15.75 GB/sно DDR path реально дает 8 GB/s
```

Тогда PCIe будет простаивать.

Проверять нужно:

```
AXI DDR bandwidth;burst length;alignment;bank conflicts;interconnect arbitration;read/write mix;clock ratio;cache/buffer policy.
```

---

# 40. Measuring throughput correctly

Для корректного измерения:

```
использовать большие buffers;делать warm-up;мерить отдельно H2C и C2H;проверять negotiated speed/width;фиксировать MPS/MRRS;исключать memcpy из измерения;учитывать CPU affinity/NUMA;смотреть ILA/counters;сравнивать с known-good example.
```

Не смешивать:

```
скорость PCIe DMAскорость user-space copyскорость записи на дискскорость обработки application logic
```

---

# 41. Useful counters in FPGA

Для throughput debug полезны:

```
TLP sent count;payload bytes sent;payload bytes received;RQ valid && ready cycles;RC valid && ready cycles;CQ valid && ready cycles;CC valid && ready cycles;tready low counters;FIFO min/max levels;credit min counters;outstanding read count;outstanding read bytes;tag allocation stalls;descriptor empty stalls;completion FIFO full stalls;source FIFO empty stalls;sink FIFO full stalls.
```

Главное — считать не только bytes, но и причины простоя.

---

# 42. Stall reason counters

Очень полезно сделать counters:

```
stall_no_credit;stall_no_tag;stall_no_descriptor;stall_payload_fifo_empty;stall_completion_fifo_full;stall_axis_not_ready;stall_link_not_ready;stall_reset;stall_alignment_split;stall_mps_split;
```

Тогда performance debug становится не гаданием, а анализом.

---

# 43. ILA signals for throughput

В ILA смотреть:

```
RQ/RC/CQ/CC valid/ready;tlast/tkeep;credit counters;MPS/MRRS config;current speed/width;DMA scheduler state;outstanding read count;completion FIFO level;payload FIFO level;descriptor FIFO level;tag free count;arbiter grant;reset/link events.
```

Смотреть нужно одновременно поток и причину пауз.

---

# 44. Common throughput patterns

## Pattern 1

```
RQ valid часто 1, RQ ready часто 0
```

Возможные причины:

```
credits;core backpressure;link/config not ready;internal PCIe IP limits.
```

## Pattern 2

```
RQ valid часто 0
```

Возможные причины:

```
DMA source не дает data;descriptor FIFO empty;scheduler idle;payload aggregation ждет data.
```

## Pattern 3

```
RC valid приходит burst-ами, completion FIFO full
```

Вероятно:

```
completion sink слишком медленный;read scheduler выдает слишком много in-flight data;нужен threshold.
```

## Pattern 4

```
credits не заканчиваются, но throughput низкий
```

Вероятно:

```
не credits bottleneck;искать DMA/software/AXI/host limit.
```

---

# 45. Theoretical efficiency example: Memory Write

Допустим, payload:

```
MPS = 256B
```

Упрощенно overhead на TLP header меньше, чем при 128B.

Для 4096B:

```
MPS 128B -> 32 TLPMPS 256B -> 16 TLPMPS 512B -> 8 TLP
```

То есть увеличение MPS уменьшает packet count.

Но если payload source не может собирать 256B/512B непрерывно, реальная эффективность не вырастет.

---

# 46. Theoretical efficiency example: Memory Read

Для read throughput:

```
Memory Read Request header отправляется FPGACompletion headers + data возвращаются от host
```

Если MRRS маленький, нужно больше requests.

Если completions дробятся на маленькие packets, overhead растет.

Идея:

```
большой MRRS помогает уменьшить request overhead,но требует больше completion buffering и outstanding tracking.
```

---

# 47. Why Gen4 x8 may not double Gen3 x8 in practice

Даже если link bandwidth удвоился:

```
Gen3 x8 -> Gen4 x8
```

application throughput может не удвоиться, если bottleneck был не link.

Например:

```
DMA engine max = 6 GB/sDDR path max = 8 GB/sdriver/test max = 5 GB/scompletion buffering слишком малMPS/MRRS не изменилисьoutstanding reads недостаточны
```

Тогда faster link просто даст больше idle time.

---

# 48. Performance debug order

Хороший порядок:

```
1. Проверить link speed/width.2. Проверить Data Link stable, нет постоянного Recovery.3. Проверить MPS/MRRS.4. Проверить direction: write или read.5. Проверить DMA transfer size.6. Проверить payload size/TLP count.7. Проверить valid/ready utilization.8. Проверить credit min counters.9. Проверить outstanding requests/tags.10. Проверить FIFO levels and backpressure.11. Проверить descriptor supply.12. Проверить host driver/IOMMU/NUMA.13. Проверить stress/cold/warm behavior.
```

---

# 49. Practical expected result mindset

Не стоит ожидать ровно теоретический максимум.

Например, для Gen3 x4 теоретический one-direction encoded payload ceiling около 3.94 GB/s. Реальная DMA может быть:

```
3.2–3.6 GB/s — хорошо для многих систем;2.0 GB/s — надо смотреть bottleneck;500 MB/s — вероятно серьезный configuration/software/architecture limit.
```

Конкретные цифры зависят от IP, host, direction, MPS/MRRS, driver и test methodology.

---

# 50. Throughput vs latency

Оптимизация throughput часто ухудшает latency.

Например:

```
больше aggregation buffer -> выше throughput, больше latency;больше outstanding reads -> выше throughput, сложнее buffering;interrupt coalescing -> выше throughput, выше notification latency;large DMA buffers -> выше throughput, хуже responsiveness.
```

Для control plane и data plane могут быть разные настройки.

---

# 51. Control plane vs data plane

BAR/register access — control plane.

DMA streams — data plane.

Нельзя оптимизировать их одинаково.

Control plane:

```
низкая latency;простая логика;надежный completion path;маленькие transfers.
```

Data plane:

```
большие buffers;batching;outstanding requests;aggregation;throughput.
```

Если data plane забивает control plane, это архитектурная ошибка.

---

# 52. Common mistakes

## Ошибка 1

```
Считать theoretical Gen x lanes скоростью DMA.
```

Нет. Это только верхняя граница.

---

## Ошибка 2

```
Мерить throughput на маленьких transfers.
```

Малые transfers показывают overhead, а не bandwidth.

---

## Ошибка 3

```
Игнорировать negotiated width/speed.
```

Если link поднялся x1 вместо x4, все остальное вторично.

---

## Ошибка 4

```
Оптимизировать credits, когда source FIFO пустой.
```

Сначала найти настоящий stall reason.

---

## Ошибка 5

```
Выдать много read requests без completion buffering.
```

Можно получить backpressure и throughput collapse.

---

## Ошибка 6

```
Не проверять MPS/MRRS в runtime.
```

Vivado setting и actual OS-configured value могут отличаться.

---

## Ошибка 7

```
Использовать BAR/MMIO как high-throughput path.
```

Для больших данных нужен DMA.

---

## Ошибка 8

```
Не отделять host/software bottleneck от FPGA bottleneck.
```

Нужно сравнивать с example design и смотреть counters.

---

# 53. Practical checklist

```
Link:    current_speed    negotiated_width    recovery events    link_down_seenConfig:    MPS    MRRS    Bus Master Enable    Memory Space EnableDMA:    transfer size    descriptor depth    outstanding read count    tags    payload aggregation    completion buffer levelFlow control:    PH/PD/NPH/CplH/CplD minimums    tready low counters    stall reason countersFPGA:    AXIS utilization    FIFO levels    DDR/BRAM bandwidth    CDC FIFO behavior    scheduler bubblesHost:    driver mode    IOMMU    NUMA    pinned memory    CPU overhead    benchmark method
```

---

# 54. Главное резюме

Throughput limits в PCIe FPGA design складываются из нескольких уровней.

Главные правила:

```
1. Link speed/width — только верхняя граница.2. MPS уменьшает overhead для write/completion payload.3. MRRS влияет на read request granularity.4. DMA read требует outstanding requests, tags и completion buffering.5. Credits могут throttle-ить TLP transmission.6. Buffers сглаживают bursts, но не исправляют средний throughput mismatch.7. AXI/AXIS valid/ready utilization показывает реальные bubbles.8. Host/software/IOMMU/NUMA могут быть bottleneck.9. BAR/MMIO не заменяет DMA data path.10. Хороший performance debug требует counters причин простоя.
```

Короткая формула:

```
PCIe throughput =    min(        link bandwidth,        TLP efficiency,        credit availability,        outstanding depth,        buffer bandwidth,        user logic bandwidth,        host memory/software bandwidth    )
```

Практическая формула для debug:

```
Сначала докажи speed/width.Потом найди, где появляются bubbles.Потом оптимизируй именно этот bottleneck.
```