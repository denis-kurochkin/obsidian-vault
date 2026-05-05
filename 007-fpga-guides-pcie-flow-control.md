**PCIe flow control** — это механизм, который не дает передатчику отправить больше TLP-пакетов, чем приемник способен принять и буферизовать.

В PCIe это особенно важно, потому что link может быть очень быстрым, а принимающая сторона имеет ограниченные буферы. Поэтому PCIe использует **credit-based flow control**: приемник сообщает передатчику, сколько у него есть свободного места для разных типов TLP. AMD описывает Transaction Layer как уровень, который управляет TLP buffer space через credit-based flow control.

Короткая идея:

```
Receiver говорит:
    "У меня есть место для N таких пакетов"

Transmitter отправляет только если:
    credits достаточно
```

---

# 1. Зачем нужен PCIe flow control

PCIe link работает full-duplex и может передавать много traffic:

```
Receiver говорит:
    "У меня есть место для N таких пакетов"

Transmitter отправляет только если:
    credits достаточно
```

Если бы передатчик просто слал пакеты без ограничений, приемник мог бы переполнить internal buffers.

Flow control решает задачу:

```
не переполнить receive buffers на следующем hop
```

Важно: PCIe flow control работает **между соседними PCIe-портами**, то есть link-local.

Например:

```
Root Port <-> FPGA Endpoint
```

или:

```
Root Port <-> Switch <-> FPGA Endpoint
```

В случае switch flow control работает отдельно на каждом link segment.

---

# 2. Что такое credit

**Credit** — это разрешение отправить определенный объем traffic определенного типа.

Упрощенно:

```
1 credit = место в приемном буфере
```

Но PCIe делит credits по типам traffic.

Это нужно потому, что разные TLP имеют разную важность и разные требования к буферизации.

---

# 3. Основные типы traffic

PCIe TLP делятся на три основные категории:

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

Например, DMA write из FPGA в host memory — это posted traffic.

## Non-Posted

Non-Posted transaction требует Completion.

Типичные примеры:

```
Memory Read
Configuration Read
Configuration Write
```

Например, DMA read request из FPGA к host memory — это non-posted traffic.

## Completion

Completion — это ответ на non-posted request.

Например:

```
Memory Read Request
        |
        v
Completion with Data
```

---

# 4. Header credits и Data credits

Для каждого типа traffic PCIe отдельно учитывает:

```
Header credits
Data credits
```

То есть появляются шесть основных credit pools:

```
PH    — Posted Header
PD    — Posted Data

NPH   — Non-Posted Header
NPD   — Non-Posted Data

CplH  — Completion Header
CplD  — Completion Data
```

AMD PG054 указывает, что PCIe integrated block предоставляет информацию о credit pools для Posted, Non-Posted и Completion traffic, включая limit/consumed/current credit state.

Практический смысл:

```
Header credit нужен для самого TLP header.
Data credit нужен для payload.
```

Например, Memory Write with payload требует:

```
Posted Header credit
Posted Data credits
```

А Memory Read Request без payload требует в основном:

```
Non-Posted Header credit
```

---

# 5. Почему credits делятся на header/data

TLP может быть:

```
только header
```

или:

```
header + payload
```

Memory Read Request обычно не несет payload, но требует Completion позже.

Memory Write несет payload.

Completion может быть:

```
Completion without Data
Completion with Data
```

Поэтому PCIe отдельно контролирует:

```
сколько packet headers можно принять;
сколько payload data можно принять.
```

AMD документация по receiver flow-control credits указывает, что один Header Credit соответствует 3- или 4-DWORD TLP header, а один Data Credit соответствует 16 bytes payload.

---

# 6. Где flow control находится в PCIe stack

Flow control относится к Transaction Layer / Data Link interaction.

Упрощенно:

```
Transaction Layer:
    хочет отправить TLP

Flow Control:
    проверяет, есть ли credits

Data Link Layer:
    доставляет TLP через link

Physical Layer:
    передает bits по lanes
```

То есть flow control не про LTSSM напрямую и не про equalization. Это уже protocol-level ограничение после того, как link работает.

---

# 7. Что видит FPGA-разработчик в Vivado

В Vivado PCIe IP flow control обычно проявляется через:

```
cfg_fc_* signals
RQ flow control status
available credits
TLP interface backpressure
AXI4-Stream tready deassertion
DMA throughput limitations
```

Например, в UltraScale+ PCIe4 есть Configuration Flow Control Interface с сигналами для Posted, Non-Posted и Completion credits, а выбор того, какую credit-информацию вывести, делается через `cfg_fc_sel[2:0]`.

---

# 8. Почему это важно для DMA

Для DMA flow control особенно важен.

## DMA write

FPGA отправляет:

```
Memory Write TLP
```

Это Posted traffic.

Зависит от:

```
PH credits
PD credits
```

Если Posted Data credits мало, DMA write throughput может упасть.

## DMA read

FPGA отправляет:

```
Memory Read Request
```

Это Non-Posted traffic.

Затем FPGA получает:

```
Completion with Data
```

То есть DMA read зависит от:

```
NPH credits на отправку read requests
CplH/CplD buffering на прием completions
tag/outstanding request capacity
host/root complex behavior
```

Поэтому DMA read часто сложнее оптимизировать, чем DMA write.

---

# 9. Credits и backpressure

Когда credits заканчиваются, PCIe core не должен отправлять соответствующий TLP.

На FPGA user side это может выглядеть как:

```
AXI4-Stream tready = 0
```

или как невозможность scheduler-а отправить новые requests.

AMD описывает в UltraScale+ PCIe4 feature overview, что flow control status interface на RQ interface показывает доступный transmit credit, чтобы user application могла планировать requests и избегать блокировки internal pipeline из-за нехватки credits от link partner.

Практическая мысль:

```
PCIe core может быть link-up,но user logic не может отправлять TLP,потому что credits временно недоступны.
```

---

# 10. Flow control и buffering в FPGA

Flow control напрямую связан с буферами.

Для хорошего FPGA PCIe design нужны буферы:

```
на входе DMA;
на выходе DMA;
для pending read requests;
для received completions;
для BAR request/completion path;
для stream backpressure;
для CDC между PCIe user_clk и application clocks.
```

Если буферов мало, будут симптомы:

```
низкий throughput;
частый backpressure;
tready часто падает;
DMA не держит line rate;
read requests не выдаются достаточно глубоко;
completions приходят burst-ами и давят user logic.
```

---

# 11. Flow control и throughput

Flow control может ограничить throughput даже при хорошем physical link.

Например:

```
Link: Gen3 x4
Но throughput низкий
```

Причины могут быть не в LTSSM, а в:

```
маленький MPS;
маленький MRRS;
мало outstanding read requests;
заканчиваются credits;
маленькие DMA bursts;
недостаточные buffers;
AXI/AXIS backpressure;
host/root complex limitations;
driver/IOMMU/cache effects.
```

Это важное разделение:

```
Link speed/width показывает потенциальную пропускную способность.
Flow control и buffering определяют, насколько хорошо она используется.
```

---

# 12. Completion credits и read performance

Read performance часто ограничивается тем, как хорошо requester держит pipeline.

Для DMA read нужно:

```
отправить много Memory Read Requests;
держать достаточно outstanding requests;
принимать Completions;
не переполнить completion buffers;
сопоставлять completions с tags;
отдавать данные дальше без backpressure.
```

Если completion path плохо буферизован, PCIe core или user logic может начать тормозить прием completions.

Это может привести к падению throughput или даже protocol-level проблемам, если логика неправильно обрабатывает backpressure.

---

# 13. Selective flow control для Non-Posted requests

В AMD UltraScale+ PCIe4 есть механизм selective flow control for Non-Posted requests на completer interface: core позволяет user logic управлять потоком Non-Posted requests через credit-based mechanism, при этом Posted transactions должны продолжать доставляться. Это нужно, потому что PCIe specification требует, чтобы posted traffic не блокировался только потому, что user logic временно не готова принимать Non-Posted requests.

Практический смысл для FPGA:

```
Если BAR/read-request path не готов,
нужно корректно тормозить Non-Posted traffic,
не ломая обработку Posted writes.
```

---

# 14. Flow control и AXI4-Stream

Если PCIe IP работает через AXI4-Stream TLP interface, flow control часто проявляется как ready/valid.

Например:

```
user logic хочет отправить TLP
s_axis_tvalid = 1
PCIe core не готов
s_axis_tready = 0
```

Причины `tready = 0` могут быть разные:

```
нет credits;
внутренний buffer full;
link не готов;
reset;
configuration не завершена;
core временно блокирует traffic.
```

Поэтому при debug важно не просто видеть `tready = 0`, а понимать, на каком уровне причина.

---

# 15. Что flow control не решает

PCIe flow control не исправляет:

```
неправильный TLP format;
ошибочный BAR decode;
отсутствующий Completion;
битый DMA descriptor;
ошибочный CDC внутри FPGA;
неверный reset sequencing;
плохую signal integrity;
LTSSM проблемы.
```

Flow control только гарантирует, что packet не будет отправлен, если на следующем hop нет buffer space для этого типа packet.

---

# 16. Как думать при debug throughput

Если link уже поднялся и устройство видно, но throughput ниже ожиданий, полезный порядок:

```
1. Проверить negotiated speed/width.
2. Проверить MPS/MRRS.
3. Проверить размер DMA bursts.
4. Проверить outstanding read requests.
5. Проверить AXI/AXIS backpressure.
6. Проверить credit-related signals.
7. Проверить host/root complex limitations.
8. Проверить IOMMU/cache/driver overhead.
9. Проверить buffering и CDC внутри FPGA.
```

Flow control — один из центральных пунктов в середине этого списка.

---

# 17. Главное резюме

PCIe flow control — это credit-based механизм, который защищает приемные буферы от overflow.

Главные идеи:

```
1. Receiver выдает credits.
2. Transmitter тратит credits при отправке TLP.
3. Credits делятся на Posted, Non-Posted и Completion.
4. Для каждого типа есть Header и Data credits.
5. DMA write в основном зависит от Posted credits.
6. DMA read зависит от Non-Posted requests и Completion handling.
7. Нехватка credits вызывает backpressure.
8. Backpressure может ограничить throughput даже при хорошем link speed.
9. В Vivado flow control виден через cfg_fc/status/RQ/CQ/AXIS-ready signals.
10. Flow control не заменяет правильный TLP, DMA, BAR, CDC и reset design.
```

Короткая формула:

```
PCIe credits = “сколько еще TLP данного типа приемник готов принять”
```

Еще короче:

```
LTSSM поднимает link.Flow control не дает переполнить buffers.DMA architecture определяет, насколько хорошо используются credits.
```