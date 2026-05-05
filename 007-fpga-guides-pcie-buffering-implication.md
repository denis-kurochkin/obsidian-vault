# Buffering implication внутри PCIe Flow Control

**Buffering implication** — это тема о том, какие буферы нужны вокруг PCIe core и почему PCIe flow control напрямую влияет на архитектуру FIFO, DMA scheduler, AXI4-Stream backpressure и throughput.

В предыдущей заметке про **credit counters** основной вопрос был:

```
сколько TLP определенного типа можно отправить?
```

В этой заметке вопрос другой:

```
где хранить данные, requests и completions,пока PCIe link, host или user logic временно не готовы?
```

Главная идея:

> PCIe credits защищают буферы соседнего PCIe-порта.  
> Но они не защищают автоматически твои внутренние FPGA-буферы, DMA queues, stream FIFOs и CDC paths.

---

# 1. Почему buffering важен в PCIe

PCIe traffic почти всегда bursty.

Даже если средний throughput умеренный, данные приходят и уходят пачками:

```
несколько TLP подряд
пауза
снова несколько TLP
```

Причины:

```
credit updates приходят не каждый такт;
host memory latency переменная;
DMA работает burst-ами;
completions могут возвращаться пачками;
AXI/AXIS downstream может давать backpressure;
PCIe core имеет внутренние очереди;
software/driver работает descriptor-ами.
```

Поэтому PCIe design без достаточной буферизации может быть функционально корректным, но медленным или нестабильным под нагрузкой.

---

# 2. Credits защищают не твой FIFO

Важно не путать:

```
PCIe credits
```

и:

```
твои FPGA FIFOs
```

PCIe credits говорят:

```
у соседнего PCIe port есть место принять TLP
```

Они не говорят:

```
у твоего DMA FIFO есть место;
твой AXI stream sink готов;
твой packet parser не переполнен;
твоя application clock domain успевает обработать данные.
```

Поэтому внутри FPGA нужны собственные буферы и собственный flow control.

---

# 3. Где обычно нужны буферы

В PCIe FPGA design буферы обычно нужны в нескольких местах:

```
1. Перед transmit path в PCIe core.
2. После receive path из PCIe core.
3. В completion receive path.
4. В DMA descriptor path.
5. В DMA payload path.
6. Между PCIe user clock и application clocks.
7. Между packet layer и register/DMA logic.
8. Перед external stream source/sink.
9. Для interrupt/status/event queues.
```

Не существует одного универсального “PCIe FIFO”. Буферизация должна соответствовать типу traffic.

---

# 4. Внутренние буферы PCIe IP

PCIe IP сам имеет внутренние буферы.

Например, AMD Versal PL PCIe4 feature list указывает, что для Transaction Layer Packet buffering используется UltraRAM, включая replay buffer, Received Posted Transaction FIFO и Received Completion Transaction FIFO. Для Versal PL PCIe4 эти FIFO могут быть, например, 16/32 KB для received posted transactions и 32/64 KB для received completions в зависимости от configuration.

Практический вывод:

```
PCIe core уже буферизует часть traffic,
но user logic все равно должна уметь принимать backpressure
и иметь собственные buffers под свою архитектуру.
```

---

# 5. Почему внутреннего buffering core недостаточно

PCIe core не знает твою application semantics.

Он не знает:

```
где границы твоих packets;
сколько latency допустимо;
можно ли дропать данные;
сколько descriptors должно быть in-flight;
какой stream приоритетнее;
что делать при reset downstream;
какой clock domain медленнее;
нужно ли сохранить packet atomicity.
```

Поэтому даже если core имеет receive FIFO, твой design должен решить:

```
куда положить TLP/data дальше;что делать, если downstream занят;как не потерять packet;когда останавливать upstream;когда выдавать credits/ready.
```

---

# 6. Буферизация и AXI4-Stream handshake

На PCIe user interface часто используется AXI4-Stream.

Главное правило:

```
transfer happens when tvalid && tready
```

Если downstream не готов, он снимает `tready`.

Источник должен удерживать:

```
tvalid
tdata
tkeep
tlast
tuser
```

до handshake.

Буфер нужен, чтобы не потерять данные, когда:

```
PCIe core выдает TLP,
а твой parser/DMA/application временно не готов.
```

Или наоборот:

```
твоя DMA logic готова отправлять,
а PCIe core временно снял tready.
```

---

# 7. Буферы на transmit side

Transmit side — это путь из FPGA в PCIe link.

Примеры:

```
FPGA DMA write -> PCIe Memory Write TLP
FPGA DMA read request -> PCIe Memory Read Request
FPGA BAR completion -> Completion TLP
FPGA interrupt -> MSI/MSI-X message
```

Буферизация нужна перед PCIe core, потому что core может временно остановить прием TLP.

Причины:

```
не хватает credits;
core internal buffer full;
link временно не готов;
user reset/link status;
flow-control throttling;
priority/arbitration между streams.
```

---

# 8. Tx FIFO: что хранить

Tx FIFO может хранить:

```
готовый TLP;
payload data для будущего TLP;
descriptor/context;
packet metadata;
address/length command;
stream packet до packetizer.
```

Выбор зависит от архитектуры.

Например:

```
stream data -> packetizer -> PCIe core
```

может требовать FIFO перед packetizer.

А:

```
DMA descriptors -> scheduler -> packetizer -> PCIe core
```

требует очереди descriptors и payload buffers.

---

# 9. Не всегда нужно буферизовать готовые TLP

Иногда лучше хранить не готовый TLP, а команду:

```
address
length
type
tag
pointer to payload buffer
```

И формировать TLP только когда PCIe core готов.

Плюсы:

```
меньше ширина FIFO;легче менять MPS/MRRS splitting;проще scheduling;меньше риска держать огромные TLP в регистрах.
```

Минусы:

```
меньше ширина FIFO;
легче менять MPS/MRRS splitting;
проще scheduling;
меньше риска держать огромные TLP в регистрах.
```

---

# 10. Буферы на receive side

Receive side — это путь из PCIe core в FPGA logic.

Примеры:

```
Host Memory Write to BAR
Host Memory Read Request to BAR
Host Completion for FPGA DMA Read
Configuration/Message-related traffic
```

Буферизация нужна, потому что incoming TLP может прийти burst-ом, а user logic может обрабатывать медленнее.

Если receive path не буферизован, будет постоянный backpressure к PCIe core.

---

# 11. Posted receive buffering

Posted traffic обычно включает Memory Writes.

Например:

```
Host пишет блок данных в FPGA BAR/window
```

Это может прийти burst-ом.

Нужно решить:

```
куда складывать payload;
как обрабатывать address;
можно ли принимать out-of-order;
что делать при full;
есть ли packet boundary;
можно ли тормозить host через flow control;
какой latency до application logic.
```

PCIe core может иметь internal posted FIFO, но если твоя logic не drain-ит его достаточно быстро, backpressure поднимется выше.

---

# 12. Non-Posted receive buffering

Non-Posted traffic — это requests, на которые нужно ответить.

Например:

```
Host Memory Read from FPGA BAR
```

FPGA должна:

```
принять request;
сохранить request context;
прочитать данные из register/memory;
сформировать Completion;
отправить Completion.
```

Для этого нужны буферы не только под сам request, но и под context:

```
tag;
requester ID;
address;
byte enables;
length;
attributes;
BAR id;
completion status.
```

---

# 13. Selective Non-Posted buffering

В AMD UltraScale+ PCIe4 core есть selective flow control для Non-Posted requests на Completer Request interface. User logic сообщает доступность buffer space через `pcie_cq_np_req[1:0]`; core доставляет Non-Posted request только когда внутренний NP credit count больше нуля, но продолжает доставлять Posted requests даже если Non-Posted delivery приостановлена.

Практический смысл:

```
если твой BAR read/completion path временно занят,
ты можешь притормозить Non-Posted requests,
не блокируя Posted writes.
```

Это очень важная архитектурная точка: разные типы traffic не должны бездумно сидеть в одном общем FIFO, если это создает head-of-line blocking.

---

# 14. Почему Posted нельзя просто блокировать из-за Non-Posted

PCIe specification требует, чтобы completer request interface продолжал доставлять Posted transactions даже когда user logic не готова принимать Non-Posted transactions. AMD PG213 прямо описывает этот случай и реализует отдельный credit mechanism для Non-Posted requests, чтобы не влиять на Posted traffic.

Практический вывод:

```
receive architecture должна разделять Posted и Non-Posted пути
или хотя бы гарантировать, что NP backpressure не блокирует posted progress.
```

---

# 15. Head-of-line blocking

**Head-of-line blocking** — ситуация, когда один packet в начале очереди блокирует packets за ним.

Пример плохой архитектуры:

```
единый RX FIFO:
    [Non-Posted Read Request, Posted Write, Posted Write, Posted Write]
```

Если Non-Posted request не может быть обработан, потому что completion path занят, весь FIFO стоит. Posted writes за ним не проходят, хотя их можно было бы принять.

Лучше:

```
separate path for Posted writes
separate path for Non-Posted requests
separate completion generation path
```

---

# 16. Completion buffering

Completion buffering особенно важен для DMA read.

Когда FPGA делает DMA read из host memory:

```
FPGA sends Memory Read Requests
Host returns Completions with Data
```

Completions могут приходить:

```
burst-ами;
неравномерно;
с разным размером;
несколькими TLP на один request;
с задержкой;
иногда не в том темпе, в котором application готова их принять.
```

Поэтому completion receive path должен иметь достаточный buffer.

---

# 17. Почему completion bursts опасны

Допустим, DMA выдала много read requests.

Некоторое время completions не приходят.

Потом host возвращает много completions подряд:

```
CplD, CplD, CplD, CplD, CplD...
```

Если downstream stream sink не готов, нужно где-то хранить returning data.

Если буфера мало:

```
RC path начинает давать backpressure;
core buffers заполняются;
request scheduler должен остановиться;
throughput падает;
в плохой архитектуре возможен deadlock-like stall.
```

---

# 18. DMA read требует два вида buffering

Для DMA read нужны:

```
1. Request-side buffering:
   descriptors, tags, outstanding context.

2. Completion-side buffering:
   returned data, completion metadata, reorder/context handling.
```

Нельзя оптимизировать только request side.

Если выдать много requests, но не иметь buffer для completions, design сам себя перегрузит.

---

# 19. Outstanding data threshold

QDMA documentation описывает performance control для H2C Stream Engine: можно остановить выдачу новых read requests в PCIe RQ/RC, если количество outstanding data на PCIe side превышает программируемый threshold `H2C_DATA_THRESH`; это предотвращает перегрузку user side и по умолчанию отключено.

Смысл для собственной архитектуры:

```
scheduler должен знать не только “есть ли tags/credits”,
но и “хватит ли downstream buffer для возвращающихся completions”.
```

---

# 20. Completion buffer sizing intuition

Для completion buffer нужно учитывать:

```
максимальное число outstanding read requests;
MRRS;
возможное splitting completions;
latency до downstream consumer;
burstiness host/root complex;
ширину user interface;
clock domain crossing;
application backpressure.
```

Упрощенная верхняя оценка:

```
completion_buffer >= max_outstanding_read_bytes
```

Но это грубо и часто слишком много.

Практически используют threshold/scheduler:

```
не выдавать больше read requests,
чем completion path способен принять.
```

---

# 21. Request context buffer

Для каждого outstanding read request нужен context.

Например:

```
tag;
target buffer address;
remaining byte count;
expected completion status;
request ID / queue ID;
stream packet metadata;
ordering information.
```

Completion приходит с tag, и DMA должна понять:

```
куда положить данные;
какому descriptor они соответствуют;
закрыт ли request;
нужно ли отправить completion event software.
```

Это отдельный буфер, не payload FIFO.

---

# 22. Payload buffer

Payload buffer хранит сами данные.

Для DMA write:

```
данные, которые FPGA отправляет в host
```

Для DMA read:

```
данные, которые host вернул в FPGA
```

Payload buffer может быть:

```
BRAM FIFO;
URAM FIFO;
external DDR;
AXI stream FIFO;
packet buffer;
ring buffer.
```

Выбор зависит от throughput, latency и размера bursts.

---

# 23. Descriptor buffer

DMA обычно работает descriptor-ами.

Descriptor buffer нужен, чтобы:

```
хранить команды transfer;
развязать software и hardware;
держать pipeline заполненным;
не простаивать из-за задержек чтения descriptor;
разрешить несколько transfers in flight.
```

Если descriptor buffer маленький, PCIe link может простаивать даже при свободных credits.

---

# 24. Buffering для DMA write

DMA write из FPGA в host memory:

```
FPGA data source -> buffer -> packetizer -> PCIe Memory Write
```

Нужны буферы:

```
input stream FIFO;
payload aggregation buffer;
descriptor/context FIFO;
TLP staging buffer;
CDC FIFO, если source clock != PCIe user_clk.
```

Цель:

```
формировать достаточно крупные, непрерывные Memory Write TLP
и переживать моменты, когда PCIe core снимает ready.
```

---

# 25. Payload aggregation

Если источник данных выдает маленькие куски, а PCIe MPS позволяет большие payload, полезно агрегировать.

Например:

```
source дает 16B chunks
MPS = 256B
```

Можно собрать 16 chunks в один 256B Memory Write.

Плюсы:

```
меньше header overhead;
лучше efficiency;
меньше PH pressure;
лучше throughput.
```

Минусы:

```
больше latency;
нужен aggregation buffer;
нужно правильно обрабатывать packet boundary/flush.
```

---

# 26. Buffering для DMA read

DMA read из host memory:

```
read descriptor
    |
issue Memory Read Requests
    |
receive Completions
    |
buffer returned data
    |
deliver to user logic
```

Нужны буферы:

```
descriptor FIFO;
request context/tag table;
completion metadata FIFO;
completion payload FIFO;
output stream FIFO;
outstanding byte counter;
threshold logic.
```

Ключевой риск:

```
выдать больше read requests, чем completion path способен принять.
```

---

# 27. Buffering для BAR reads

BAR read — это Non-Posted request от host.

FPGA должна быстро вернуть Completion.

Для простых register reads достаточно:

```
маленький request context buffer;
register read mux;
completion output buffer.
```

Для BAR reads в memory/window:

```
нужен read request queue;
memory access pipeline;
completion data buffer;
split completion handling;
backpressure на NP requests.
```

Если read data приходит из медленного clock domain, обязательно нужна staging/buffering логика.

---

# 28. Buffering для BAR writes

BAR write — Posted traffic.

Простой register write:

```
TLP -> decode -> write register
```

Но если host пишет burst в memory window:

```
TLP payload -> posted write FIFO -> memory/stream sink
```

Если sink не готов, posted write FIFO должен принять burst или backpressure пойдет в PCIe core.

---

# 29. Buffering и CDC

PCIe user clock часто не совпадает с application clocks.

Например:

```
pcie_user_clk = 250 MHz
processing_clk = 200 MHz
adc_clk = 156.25 MHz
```

Тогда между PCIe и application нужны CDC mechanisms:

```
async FIFO для streams;
handshake для registers/control;
dual-clock FIFO для payload;
reset-aware CDC для status.
```

Нельзя просто протянуть AXI4-Stream signals через clock domains.

---

# 30. Async FIFO на границе PCIe/application

Типичная структура:

```
PCIe user_clk domain
        |
        v
async FIFO
        |
        v
application_clk domain
```

Для DMA write source:

```
application stream -> async FIFO -> PCIe packetizer
```

Для DMA read sink:

```
PCIe completion data -> async FIFO -> application stream
```

При этом FIFO depth должна учитывать не только средние частоты, но и burstiness completions/posted writes.

---

# 31. Buffer depth: что влияет

Размер буфера зависит от:

```
PCIe link speed/width;
user interface width/frequency;
MPS/MRRS;
burst length;
maximum stall time downstream;
number of outstanding requests;
host latency;
clock domain mismatch;
packet boundary requirements;
политика drop/backpressure;
ресурсы BRAM/URAM/DDR.
```

Формально идеального универсального числа нет.

Но плохой признак:

```
FIFO depth выбрана “на глаз” без worst-case burst/stall оценки.
```

---

# 32. Простая оценка для posted write FIFO

Если host может прислать burst:

```
B bytes
```

а downstream может быть занят:

```
Tstall
```

то FIFO должен покрыть хотя бы:

```
bytes_arriving_during_stall - bytes_drained_during_stall
```

Если downstream совсем не читает во время stall:

```
FIFO >= maximum burst to absorb
```

Плюс margin.

---

# 33. Простая оценка для completion FIFO

Если DMA разрешает:

```
N outstanding read bytes
```

то completion side потенциально должен быть готов принять до `N` bytes returned data, если scheduler не ограничивает выдачу requests.

Более практично:

```
completion FIFO depth + downstream drain rate
задают threshold для outstanding read bytes.
```

То есть:

```
если completion FIFO почти полный,
request scheduler должен перестать выдавать новые read requests.
```

---

# 34. Threshold вместо огромного FIFO

Не всегда нужно делать огромный FIFO.

Можно использовать threshold control:

```
если completion_fifo_level > HIGH_WATERMARK:
    stop issuing new read requests

если completion_fifo_level < LOW_WATERMARK:
    resume issuing read requests
```

Это дешевле, чем хранить весь worst-case объем.

Но нужно учитывать latency:

```
после stop еще могут прийти completions по уже outstanding requests.
```

Поэтому high watermark должен оставлять запас.

---

# 35. High watermark и in-flight data

Правильная логика:

```
available_completion_space > possible_returning_data
```

Перед выдачей нового read request нужно подумать:

```
если этот request вернется completion-ами,
куда они поместятся?
```

Иначе можно получить:

```
scheduler продолжал выдавать read requests;
completion FIFO переполнился;
RC path backpressure;
throughput collapse.
```

---

# 36. Skid buffers

**Skid buffer** — небольшой буфер на ready/valid interface.

Он полезен, когда:

```
нужно разорвать длинный ready path;
нужно удержать один-два beats при внезапном backpressure;
нужно улучшить timing;
нужно сделать interface проще для synthesis.
```

PCIe user side часто имеет широкие datapaths:

```
128-bit
256-bit
512-bit
```

и высокие частоты. AMD Versal PL PCIe4 поддерживает AXI4-Stream datapath widths 64/128/256/512 bits к customer logic.

На таких интерфейсах skid/register slices часто важны для timing closure.

---

# 37. Register slice vs FIFO

Register slice:

```
маленькая latency;
улучшает timing;
не дает большой burst absorption.
```

FIFO:

```
больше buffering;
может пересекать clock domain;
может сглаживать bursts;
дороже по ресурсам.
```

Обычно нужны оба:

```
PCIe core -> register slice -> parser -> FIFO -> application
```

---

# 38. Packet boundary

PCIe TLP и user packets имеют границы.

Буфер должен сохранять:

```
tlast;
tkeep;
tuser;
metadata;
byte enables;
address association;
descriptor association.
```

Если FIFO хранит только `tdata`, но не хранит sideband, packet corrupt.

Для AXI4-Stream PCIe path обычно нужно буферизовать:

```
{tuser, tlast, tkeep, tdata}
```

---

# 39. Partial TLP problem

Плохо, если архитектура может принять начало TLP, но не иметь места для конца.

Например:

```
принимаем 512B TLPFIFO осталось 64B
```

Нужно либо:

```
гарантировать space для whole packet before accepting;
или поддерживать beat-by-beat backpressure корректно;
или иметь packet buffer достаточного размера;
или не начинать передачу downstream без места.
```

Для AXI stream это особенно важно, если downstream не допускает разрыв packet.

---

# 40. Store-and-forward vs cut-through

Два подхода:

## Store-and-forward

```
сначала принять весь packet в buffer,
потом обрабатывать/отправлять.
```

Плюсы:

```
проще packet-level decisions;
легче проверять boundaries;
можно drop/retry на packet level;
удобно для variable latency.
```

Минусы:

```
больше latency;
больше buffer memory.
```

## Cut-through

```
начать передавать packet дальше до полного приема.
```

Плюсы:

```
ниже latency;меньше buffering.
```

Минусы:

```
сложнее backpressure;сложнее error handling;нужно гарантировать downstream readiness.
```

---

# 41. Deadlock-like stalls

PCIe flow control и внутренние FIFOs могут создать “зависание”, если потоки зависят друг от друга.

Пример плохой ситуации:

```
completion FIFO full    -> RC path stopped    -> DMA read completions не принимаются    -> DMA engine ждет completions, чтобы освободить descriptors    -> descriptors не освобождаются    -> software/engine не двигается
```

Или:

```
Non-Posted request accepted    -> needs Completion    -> Completion output FIFO full    -> request context held    -> NP request queue fills
```

Решение:

```
separate buffers;reserved space for completions;thresholds;no circular dependencies;clear priority rules;reset/recovery path.
```

---

# 42. Completion path должен иметь приоритет

Completion traffic важен для forward progress.

Если система приняла Non-Posted request, она должна уметь вернуть Completion.

Поэтому completion generation path нельзя полностью заблокировать менее важным traffic.

Практическая идея:

```
reserve buffer space for completions;дать completion path высокий приоритет;не смешивать completion output с бесконечным posted write stream без arbitration.
```

---

# 43. Separate queues by traffic class

Хорошая архитектура часто разделяет:

```
posted write queue;non-posted request queue;completion output queue;DMA write queue;DMA read request queue;DMA read completion queue;interrupt/message queue.
```

Это уменьшает head-of-line blocking.

Единый общий FIFO проще, но часто хуже для performance и debug.

---

# 44. Arbitration

Если несколько источников хотят отправлять TLP:

```
DMA Memory Writes;DMA Memory Read Requests;BAR Completions;MSI/MSI-X;internal messages;
```

нужен arbiter.

Arbiter должен учитывать:

```
credits;completion priority;fairness;starvation;available payload data;available tags;link readiness;packet boundaries.
```

---

# 45. Completion starvation

Плохая ситуация:

```
DMA write stream постоянно занимает transmit path;BAR read Completion ждет;host read timeout.
```

Даже если bandwidth хороший, system-level behavior плохой.

Поэтому completion traffic обычно должен иметь приоритет над bulk posted writes.

---

# 46. Buffering and timeout behavior

Host может ожидать Completion на Memory Read.

Если FPGA приняла request, но долго не отвечает, возможны:

```
software hang;PCIe completion timeout;AER error;driver failure;system instability.
```

Поэтому BAR read path должен быть:

```
простым;быстрым;с приоритетом;с маленькой и предсказуемой latency.
```

Для медленных операций лучше не делать blocking BAR read. Лучше:

```
write command register;poll status register;read result later.
```

---

# 47. Buffering and reset

Reset может ломать буферы.

Нужно определить:

```
что происходит с pending requests;очищаются ли FIFOs;что происходит с outstanding DMA reads;как сообщается error software;когда можно снова выдавать requests;можно ли сбросить user logic при активном PCIe link.
```

Если reset очищает completion context table, но completions еще могут вернуться, это опасно.

Нужен recovery protocol:

```
quiesce traffic;stop issuing requests;wait for outstanding completions or timeout;reset buffers;restart cleanly.
```

---

# 48. Buffering and link down

Если link падает:

```
outstanding requests могут не завершиться;Tx queues могут содержать неотправленные TLP;Rx buffers могут иметь partial state;DMA descriptors могут остаться pending.
```

Нужно иметь политику:

```
flush;error flag;driver notification;DMA abort;reinitialization;link recovery handling.
```

Иначе после link recovery design может продолжить со старым inconsistent state.

---

# 49. Buffering and software-visible status

Хорошая практика — иметь counters/registers:

```
tx_fifo_level;rx_fifo_level;completion_fifo_level;descriptor_fifo_level;max_fifo_level_seen;overflow_seen;underflow_seen;backpressure_counter;tready_low_counter;dropped_packet_counter;outstanding_read_bytes;outstanding_read_count;
```

Это помогает debug-ить без постоянной ILA.

---

# 50. What to put in ILA

Для buffering debug полезно смотреть:

```
AXI4-Stream valid/ready/last/user;FIFO level;FIFO almost_full/almost_empty;credit counters;RQ/RC/CQ/CC traffic;DMA scheduler state;outstanding request count;outstanding byte count;tag allocation;completion FIFO level;BAR request/response state;reset/link status.
```

Главное — смотреть не только один интерфейс, а цепочку:

```
source -> FIFO -> packetizer -> PCIe core
```

или:

```
PCIe core -> parser -> FIFO -> application
```

---

# 51. Типовые симптомы недостаточной буферизации

```
низкий throughput;tready часто падает;DMA идет рывками;BAR reads иногда зависают;completion FIFO overflow;posted writes вызывают backpressure;DMA read работает хуже DMA write;Gen3/Gen4 link не используется полностью;работает на малых packets, ломается на больших;работает в simulation, но не держит stress test.
```

---

# 52. Частые ошибки

## Ошибка 1

```
Считать, что PCIe core buffers достаточно.
```

Core buffers не заменяют application-level buffering.

---

## Ошибка 2

```
Смешать Posted, Non-Posted и Completion в один FIFO без анализа.
```

Это может создать head-of-line blocking.

---

## Ошибка 3

```
Выдавать много DMA read requests без completion-side buffer.
```

Можно перегрузить returning completion path.

---

## Ошибка 4

```
Не хранить metadata вместе с data.
```

Packet boundary, tag, address, byte enables и descriptor association теряются.

---

## Ошибка 5

```
Игнорировать tready.
```

Это ломает AXI4-Stream protocol.

---

## Ошибка 6

```
Не иметь high watermark для outstanding read data.
```

Scheduler может выдать больше read requests, чем completion path выдержит.

---

## Ошибка 7

```
BAR read path сделать через медленную/занятую application logic.
```

Host может получить timeout.

---

## Ошибка 8

```
Не учитывать reset/link down при pending buffers.
```

После recovery state может стать inconsistent.

---

# 53. Практическая таблица: traffic → buffer

|Traffic|Где нужен buffer|Главный риск|
|---|---|---|
|DMA write|input stream FIFO, payload aggregation, Tx staging|PCIe core backpressure, small TLPs|
|DMA read request|descriptor FIFO, tag/context table|мало outstanding или перегруз completions|
|DMA read completion|completion FIFO, reorder/context buffer|burst completions, downstream stall|
|BAR write|posted write FIFO / register staging|posted burst pressure|
|BAR read|NP request queue + completion buffer|host timeout, blocked CC path|
|MSI/MSI-X|small event queue|interrupt lost or delayed|
|CDC boundary|async FIFO|clock mismatch, burst absorption|

---

# 54. Checklist для buffering architecture

```
1. Какие traffic types есть: MWr, MRd, Cpl, BAR, MSI?2. Где границы clock domains?3. Какие paths могут давать backpressure?4. Где packet boundary должен сохраняться?5. Какие metadata идут вместе с data?6. Сколько outstanding read requests разрешено?7. Сколько returning completion data может прийти?8. Есть ли high/low watermarks?9. Какие FIFOs разделены по traffic class?10. Есть ли отдельный completion path?11. Есть ли skid buffers на широких ready/valid paths?12. Что происходит при reset?13. Что происходит при link down?14. Есть ли debug counters уровней FIFO?15. Есть ли stress test на bursts/backpressure?
```

---

# 55. Главное резюме

Buffering в PCIe FPGA design — это не “поставить FIFO где-нибудь”.

Это архитектурная часть flow control.

Главные правила:

```
1. PCIe credits защищают buffers link partner-а, не твои RTL FIFOs.2. Внутренние buffers PCIe IP не заменяют application buffering.3. Posted, Non-Posted и Completion traffic лучше разделять.4. DMA read требует request-side и completion-side buffering.5. Completion path должен иметь гарантированный forward progress.6. Backpressure должен быть корректно обработан через ready/valid.7. Metadata нужно буферизовать вместе с data.8. CDC между PCIe и application clocks требует async FIFO/CDC protocol.9. Scheduler должен учитывать available buffer space, а не только credits/tags.10. Reset/link-down должны очищать или корректно завершать pending state.
```

Короткая формула:

```
Credits говорят: “можно ли отправить TLP?”Buffers отвечают: “куда положить TLP/data/context, пока все остальные не готовы?”
```

Практическая формула для DMA read:

```
не выдавай read requests быстрее,чем completion path способен принять возвращающиеся данные.
```