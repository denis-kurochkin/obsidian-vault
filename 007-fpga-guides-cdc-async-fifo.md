# Async FIFO внутри CDC

**Async FIFO** — это FIFO-буфер, у которого запись и чтение работают от разных clock domains.

Он используется, когда нужно безопасно передать **поток данных** между двумя несинхронными или независимо работающими clock domains.

Типичный случай:

```
clk_wr domain                          clk_rd domain

data_wr -----> async FIFO -----> data_rd
wr_en  ----->              -----> rd_en
full   <-----              -----> empty
```

Например:

```
ADC clock domain       -> system clock domain
SFP/Aurora domain      -> user logic domain
PCIe user clock        -> internal processing clock
AXI-Stream clk A       -> AXI-Stream clk B
UART RX clock enable   -> system clock
```

Главная идея async FIFO:

> данные физически хранятся в памяти FIFO, а между clock domains пересекаются не сами data-биты, а управляющая информация о позициях записи и чтения.

---

# 1. Зачем нужен async FIFO

Async FIFO нужен тогда, когда между clock domains передается не одиночный flag и не редкое событие, а **последовательность слов данных**.

Например:

```
clk_a = 156.25 MHz
clk_b = 100 MHz

source каждый несколько тактов выдает data word
destination забирает data word в своем темпе
```

Без FIFO пришлось бы строить сложный handshake на каждое слово. Это возможно, но неудобно и плохо масштабируется для потоков.

Async FIFO решает сразу несколько задач:

```
1. безопасный CDC;
2. временная буферизация данных;
3. decoupling между producer и consumer;
4. компенсация burst-передач;
5. согласование разных частот чтения и записи;
6. поддержка flow control через full/empty.
```

---

# 2. Чем async FIFO отличается от обычного FIFO

Обычный synchronous FIFO:

```
один clock для записи и чтения
```

```
clk
 |
 +--> write logic
 |
 +--> read logic
```

Async FIFO:

```
один clock для записи
другой clock для чтения
```

```
wr_clk domain               rd_clk domain

write pointer               read pointer
write enable                read enable
full flag                   empty flag
```

То есть async FIFO — это не просто “массив памяти с двумя портами”. Это CDC-структура, внутри которой есть:

```
dual-port memory
write pointer
read pointer
Gray-code pointer crossing
full/empty generation
synchronizer chains
```

---

# 3. Где async FIFO находится в архитектуре

Типичная структура проекта:

```
source module
    |
    | data + valid / wr_en
    v
async FIFO
    |
    | data + valid / empty
    v
destination module
```

Например:

```
Aurora RX user_clk domain
        |
        v
async FIFO
        |
        v
system processing clk domain
```

Или:

```
sensor_clk domain
        |
        v
async FIFO
        |
        v
AXI-Stream clk domain
```

Важно: async FIFO является **границей clock domains**. После FIFO данные уже находятся в домене чтения.

---

# 4. Базовые сигналы async FIFO

Упрощенный интерфейс async FIFO:

```verilog
input  wire         wr_clk,
input  wire         wr_rst,

input  wire         wr_en,
input  wire [W-1:0] din,
output wire         full,

input  wire         rd_clk,
input  wire         rd_rst,

input  wire         rd_en,
output wire [W-1:0] dout,
output wire         empty
```

Сторона записи:

```
wr_clk
din
wr_en
full
```

Сторона чтения:

```
rd_clk
dout
rd_en
empty
```

Правило записи:

```verilog
if (wr_en && !full)
    FIFO writes din;
```

Правило чтения:

```verilog
if (rd_en && !empty)
    FIFO reads next word;
```

---

# 5. Producer и consumer

В async FIFO обычно есть две стороны:

```
producer — пишет данные в FIFO
consumer — читает данные из FIFO
```

Producer работает в `wr_clk`.

Consumer работает в `rd_clk`.

Producer должен уважать `full`.

Consumer должен уважать `empty`.

Плохой вариант:

```verilog
assign wr_en = data_valid;
```

если не учитывается `full`.

Правильнее:

```verilog
assign wr_en = data_valid && !fifo_full;
```

Плохой вариант на стороне чтения:

```verilog
assign rd_en = 1'b1;
```

если не учитывается `empty`.

Правильнее:

```verilog
assign rd_en = consumer_ready && !fifo_empty;
```

---

# 6. Что происходит при записи

В write domain FIFO хранит write pointer.

```
wr_ptr указывает, куда будет записано следующее слово
```

Когда выполняется запись:

```
wr_en && !full
```

FIFO делает:

```
memory[wr_ptr] <= din
wr_ptr <= wr_ptr + 1
```

Это происходит только в `wr_clk` domain.

---

# 7. Что происходит при чтении

В read domain FIFO хранит read pointer.

```
rd_ptr указывает, откуда будет прочитано следующее слово
```

Когда выполняется чтение:

```
rd_en && !empty
```

FIFO делает:

```
dout <= memory[rd_ptr]
rd_ptr <= rd_ptr + 1
```

Это происходит только в `rd_clk` domain.

---

# 8. Главная CDC-проблема внутри async FIFO

Чтобы понять, пустой FIFO или полный, одной стороне нужно знать положение pointer другой стороны.

Write domain должен знать read pointer:

```
чтобы определить full
```

Read domain должен знать write pointer:

```
чтобы определить empty
```

Но pointers находятся в разных clock domains.

```
wr_ptr живет в wr_clk domain
rd_ptr живет в rd_clk domain
```

Нельзя просто передать binary pointer из одного домена в другой побитно.

Например:

```
binary pointer: 0111 -> 1000
```

Меняются сразу 4 бита.

При побитной синхронизации принимающий домен может временно увидеть мусорное значение:

```
0000
1111
1010
0011
```

Это может привести к неправильному `full` или `empty`.

---

# 9. Почему используется Gray code

В async FIFO pointers обычно передают между доменами в **Gray code**.

Свойство Gray code:

```
при переходе к следующему значению меняется только один бит
```

Например для 3 бит:

```
binary   gray
000      000
001      001
010      011
011      010
100      110
101      111
110      101
111      100
```

Переходы Gray code:

```
000 -> 001  меняется 1 бит
001 -> 011  меняется 1 бит
011 -> 010  меняется 1 бит
010 -> 110  меняется 1 бит
```

Это удобно для CDC: даже если принимающий домен захватит pointer около момента изменения, неопределенность будет только между старым и новым соседним значением, а не произвольным мусором.

---

# 10. Binary pointer и Gray pointer

Внутри каждого домена обычно есть два представления pointer:

```
binary pointer — удобно увеличивать на 1
gray pointer   — удобно передавать через CDC
```

Write domain:

```
wr_ptr_bin
wr_ptr_gray
```

Read domain:

```
rd_ptr_bin
rd_ptr_gray
```

Передача между доменами:

```
wr_ptr_gray -> synchronizer -> rd_clk domain
rd_ptr_gray -> synchronizer -> wr_clk domain
```

Структура:

```
wr_clk domain                                  rd_clk domain

wr_ptr_bin
    |
    v
wr_ptr_gray  ---- CDC sync ---->  wr_ptr_gray_sync_rd

rd_ptr_gray_sync_wr  <---- CDC sync ----  rd_ptr_gray
                                            ^
                                            |
                                      rd_ptr_bin
```

---

# 11. Как формируется empty

`empty` формируется в read domain.

Read side сравнивает:

```
текущий read pointer
синхронизированный write pointer
```

Если read pointer догнал write pointer, значит читать нечего.

Упрощенно:

```
empty = (rd_ptr_gray_next == wr_ptr_gray_sync_rd)
```

Смысл:

```
если следующий read pointer равен известному write pointer,
FIFO пустой или станет пустым после чтения
```

`empty` должен использоваться только в `rd_clk` domain.

---

# 12. Как формируется full

`full` формируется в write domain.

Write side сравнивает:

```
следующий write pointer
синхронизированный read pointer
```

Для circular buffer нужно отличать состояние:

```
wr_ptr == rd_ptr
```

Оно может означать:

```
FIFO empty
```

или

```
FIFO full
```

Поэтому pointers обычно делают на один бит шире адреса памяти.

Например, если FIFO depth = 16:

```
адрес памяти: 4 бита
pointer:      5 бит
```

Дополнительный старший бит помогает отличить полный буфер от пустого.

Для full обычно проверяется, что write pointer “догнал” read pointer с учетом wrap-around.

Упрощенный смысл:

```
full = next write position would collide with read position
```

`full` должен использоваться только в `wr_clk` domain.

---

# 13. Почему flags имеют задержку

`full` и `empty` не обновляются мгновенно относительно другой стороны FIFO.

Причина: pointer другой стороны проходит через synchronizer.

Например:

```
write side записала данные
wr_ptr изменился
wr_ptr_gray прошел CDC в rd_clk domain
только потом read side увидит, что FIFO уже не empty
```

То есть после записи read side может увидеть `empty = 1` еще несколько тактов `rd_clk`.

Это нормально.

То же самое для `full`:

```
read side прочитала данные
rd_ptr изменился
rd_ptr_gray прошел CDC в wr_clk domain
только потом write side увидит, что место освободилось
```

Поэтому `full` может оставаться активным еще несколько тактов после чтения.

Это тоже нормально.

---

# 14. Async FIFO не гарантирует нулевую latency

Async FIFO добавляет задержку.

Latency зависит от:

```
режима чтения;
глубины internal pipeline;
частот wr_clk и rd_clk;
состояния empty/full;
настроек FIFO IP;
используется ли First Word Fall Through;
есть ли output register.
```

Обычная ситуация:

```
данные записали в wr_clk domain
через несколько тактов rd_clk они стали доступны на чтение
```

Это не проблема, если архитектура рассчитана на потоковую передачу.

Если нужна передача одиночной команды с подтверждением, иногда handshake проще и понятнее, чем FIFO.

---

# 15. Standard FIFO mode и First Word Fall Through

У FIFO часто есть два режима чтения:

```
standard read mode
First Word Fall Through / FWFT
```

## Standard mode

В standard mode `dout` обновляется после `rd_en`.

Типовая логика:

```
если !empty:
    подать rd_en
    на следующем такте получить dout
```

Условно:

```
rd_en -> dout valid later
```

## FWFT mode

В FWFT mode первое доступное слово “само появляется” на `dout`, когда FIFO не пустой.

Тогда:

```
!empty означает, что dout уже valid
```

Условно:

```
!empty -> dout already valid
rd_en  -> consume current word
```

FWFT часто удобен для потоковых интерфейсов, потому что `empty` можно трактовать как обратный `valid`.

---

# 16. Async FIFO и valid/ready

Для streaming-интерфейсов FIFO часто оборачивают в valid/ready.

Сторона записи:

```
s_valid
s_ready
s_data
```

Связь с FIFO:

```verilog
assign s_ready = !fifo_full;
assign fifo_wr_en = s_valid && s_ready;
assign fifo_din = s_data;
```

Сторона чтения:

```
m_valid
m_ready
m_data
```

Для FWFT FIFO:

```verilog
assign m_valid = !fifo_empty;
assign m_data  = fifo_dout;
assign fifo_rd_en = m_valid && m_ready;
```

Это превращает async FIFO в CDC-мост для streaming data.

---

# 17. Async FIFO и AXI-Stream

Для AXI-Stream CDC обычно используют async FIFO с сигналами:

```
tdata
tvalid
tready
tlast
tkeep
tuser
```

Запись в FIFO:

```
когда s_axis_tvalid && s_axis_tready
```

Чтение из FIFO:

```
когда m_axis_tvalid && m_axis_tready
```

В FIFO обычно кладут не только `tdata`, но и sideband-сигналы:

```
{tuser, tlast, tkeep, tdata}
```

Например:

```
FIFO width = DATA_WIDTH + KEEP_WIDTH + 1 + USER_WIDTH
```

Для AXI-Stream в Vivado удобнее использовать готовые IP:

```
AXI4-Stream Data FIFO
AXIS Clock Converter
FIFO Generator
xpm_fifo_async
```

Если задача простая и без полного AXIS-интерфейса, `xpm_fifo_async` часто достаточно.

---

# 18. Выбор глубины FIFO

Глубина FIFO — один из главных архитектурных параметров.

Она зависит от:

```
1. разницы средних скоростей записи и чтения;
2. burst length на входе;
3. пауз на стороне чтения;
4. допустимой latency;
5. доступных ресурсов FPGA;
6. требований к backpressure;
7. поведения при overflow/underflow.
```

Пример 1: запись медленнее чтения.

```
wr_clk domain выдает слово раз в 10 тактов
rd_clk domain почти всегда готов читать
```

Глубина может быть небольшой:

```
16 / 32 words
```

Пример 2: запись идет burst-ами.

```
source может быстро записать 256 слов,
а consumer иногда занят
```

Глубина должна покрывать burst и паузы consumer:

```
512 / 1024 words или больше
```

Пример 3: средняя скорость записи выше средней скорости чтения.

```
input average rate > output average rate
```

Тогда FIFO рано или поздно переполнится независимо от глубины.

FIFO не исправляет постоянный mismatch throughput. Он только временно буферизует.

---

# 19. Простая оценка глубины FIFO

Для грубой оценки можно использовать идею:

```
required depth >= max_written_during_stall - read_during_stall + margin
```

Например:

```
source пишет 1 слово каждый такт 100 MHz
destination может остановиться на 2 us
```

За 2 us source запишет:

```
100 MHz * 2 us = 200 слов
```

Минимальная глубина с запасом:

```
256 или 512 слов
```

Если destination во время этой паузы вообще не читает, FIFO должен вместить весь burst.

Если читает частично, можно вычесть прочитанные слова.

---

# 20. Almost full и almost empty

Во многих FIFO есть дополнительные flags:

```
almost_full
almost_empty
prog_full
prog_empty
data_count
```

Они нужны, чтобы реагировать заранее.

Например, `full` может прийти слишком поздно для внешнего источника, которому нужно несколько тактов, чтобы остановиться.

Тогда используют `almost_full`.

Пример:

```verilog
assign source_ready = !fifo_almost_full;
```

А не:

```verilog
assign source_ready = !fifo_full;
```

Это полезно, если upstream имеет pipeline latency.

---

# 21. Overflow и underflow

**Overflow** — попытка записать в полный FIFO.

```
wr_en = 1 при full = 1
```

**Underflow** — попытка читать из пустого FIFO.

```
rd_en = 1 при empty = 1
```

В хорошем дизайне эти ситуации не должны происходить.

Минимальные защиты:

```verilog
assign fifo_wr_en = src_valid && !fifo_full;
assign fifo_rd_en = dst_ready && !fifo_empty;
```

Но в сложной системе лучше явно отслеживать ошибки:

```verilog
always @(posedge wr_clk) begin
    if (wr_en && full)
        fifo_overflow <= 1'b1;
end

always @(posedge rd_clk) begin
    if (rd_en && empty)
        fifo_underflow <= 1'b1;
end
```

Эти флаги полезны при debug в железе.

---

# 22. Data count в async FIFO

Некоторые FIFO дают счетчики:

```
wr_data_count
rd_data_count
```

Но в async FIFO к ним нужно относиться аккуратно.

`wr_data_count` обычно корректен в `wr_clk` domain.

`rd_data_count` обычно корректен в `rd_clk` domain.

Они могут иметь задержку и не обязаны быть абсолютно мгновенным “истинным количеством слов” с точки зрения обоих доменов.

Правило:

```
wr_data_count использовать только в write domain
rd_data_count использовать только в read domain
```

Для точного междоменного управления лучше использовать flags или отдельный protocol.

---

# 23. Resource choice: LUTRAM, BRAM, URAM

FIFO может быть реализован на разных ресурсах FPGA:

```
distributed RAM / LUTRAM
Block RAM / BRAM
UltraRAM / URAM
registers
```

Примерный выбор:

|FIFO|Обычно подходит|
|---|---|
|маленький, до десятков слов|LUTRAM или registers|
|средний, сотни/тысячи слов|BRAM|
|очень большой буфер|URAM, если доступно|
|очень широкий data bus|BRAM/URAM или несколько RAM параллельно|
|низкая latency, малая глубина|LUTRAM/registers|

В Xilinx/Vivado это часто задается параметром типа:

```
FIFO_MEMORY_TYPE
```

Например:

```
"auto"
"block"
"distributed"
"ultra"
```

Обычно для начала удобно использовать `"auto"`, а потом смотреть utilization и timing.

---

# 24. Xilinx XPM: xpm_fifo_async

В Vivado практичный способ сделать async FIFO — использовать `xpm_fifo_async`.

Это параметризованный Xilinx-модуль.

Упрощенный пример wrapper:

```verilog
module cdc_async_fifo #(
    parameter integer DATA_WIDTH = 32,
    parameter integer FIFO_DEPTH = 1024
)(
    input  wire                  wr_clk,
    input  wire                  wr_rst,
    input  wire                  wr_en,
    input  wire [DATA_WIDTH-1:0] din,
    output wire                  full,
    output wire                  almost_full,

    input  wire                  rd_clk,
    input  wire                  rd_rst,
    input  wire                  rd_en,
    output wire [DATA_WIDTH-1:0] dout,
    output wire                  empty,
    output wire                  almost_empty
);

    xpm_fifo_async #(
        .FIFO_MEMORY_TYPE    ("auto"),
        .FIFO_WRITE_DEPTH    (FIFO_DEPTH),
        .WRITE_DATA_WIDTH    (DATA_WIDTH),
        .READ_DATA_WIDTH     (DATA_WIDTH),

        .READ_MODE           ("fwft"),
        .FIFO_READ_LATENCY   (0),

        .CDC_SYNC_STAGES     (2),
        .RELATED_CLOCKS      (0),

        .USE_ADV_FEATURES    ("0707"),

        .WR_DATA_COUNT_WIDTH (1),
        .RD_DATA_COUNT_WIDTH (1),

        .DOUT_RESET_VALUE    ("0"),
        .ECC_MODE            ("no_ecc"),
        .FULL_RESET_VALUE    (0),
        .PROG_FULL_THRESH    (10),
        .PROG_EMPTY_THRESH   (10),
        .WAKEUP_TIME         (0)
    ) u_xpm_fifo_async (
        .wr_clk        (wr_clk),
        .rst           (wr_rst),
        .wr_en         (wr_en),
        .din           (din),
        .full          (full),
        .almost_full   (almost_full),

        .rd_clk        (rd_clk),
        .rd_en         (rd_en),
        .dout          (dout),
        .empty         (empty),
        .almost_empty  (almost_empty),

        .sleep         (1'b0),
        .injectsbiterr (1'b0),
        .injectdbiterr (1'b0),

        .overflow      (),
        .underflow     (),
        .wr_ack        (),
        .valid         (),
        .data_valid    (),

        .prog_full     (),
        .prog_empty    (),

        .wr_data_count (),
        .rd_data_count (),

        .sbiterr       (),
        .dbiterr       ()
    );

endmodule
```

Это не единственно возможная конфигурация. В реальном проекте параметры нужно подбирать под режим чтения, глубину, использование flags и требования к latency.

---

# 25. Важные параметры xpm_fifo_async

## `FIFO_WRITE_DEPTH`

Глубина FIFO по write side.

Обычно степень двойки:

```
16, 32, 64, 512, 1024, 2048
```

Для async FIFO лучше использовать глубины, которые нормально поддерживаются IP.

---

## `WRITE_DATA_WIDTH` и `READ_DATA_WIDTH`

Ширина записи и чтения.

Можно делать одинаковую ширину:

```
write 32 bitread  32 bit
```

Некоторые FIFO поддерживают width conversion:

```
write 32 bitread  64 bit
```

или:

```
write 64 bitread  8 bit
```

Но width conversion усложняет понимание количества слов, latency и sideband-сигналов. Для CDC-начала лучше использовать одинаковую ширину.

---

## `READ_MODE`

Часто варианты:

```
"std"
"fwft"
```

`"fwft"` удобен для valid/ready.

`"std"` удобен, когда логика явно управляет read latency.

---

## `FIFO_READ_LATENCY`

Для `"std"` обычно задает latency output path.

Для `"fwft"` часто используют `0`.

Нужно сверяться с выбранным режимом.

---

## `CDC_SYNC_STAGES`

Количество synchronizer stages для pointer crossing.

Обычно:

```
2
```

Для повышенной надежности:

```
3 или больше
```

Глубоко в MTBF здесь не уходим, потому что это отдельная тема.

---

## `RELATED_CLOCKS`

Если clocks связаны, можно указывать соответствующий режим.

Для действительно asynchronous/unrelated clocks обычно:

```
.RELATED_CLOCKS(0)
```

В большинстве CDC-сценариев между независимыми доменами это безопасная настройка.

---

## `USE_ADV_FEATURES`

В XPM этот параметр включает или отключает дополнительные порты:

```
almost_full
almost_empty
data_count
overflow
underflow
valid
wr_ack
```

Конкретное значение зависит от того, какие сигналы нужны. Его удобно настраивать через template в Vivado.

---

# 26. Где взять правильный template в Vivado

В Vivado удобно открыть:

```
Language Templates
    -> Verilog
        -> Xilinx Parameterized Macros
            -> FIFO
                -> xpm_fifo_async
```

Или использовать IP Catalog:

```
FIFO Generator
AXI4-Stream Data FIFO
AXIS Clock Converter
```

Практический подход:

```
1. Взять XPM template из Vivado.
2. Настроить параметры.
3. Обернуть в свой wrapper.
4. В проекте использовать только wrapper, а не раскидывать XPM-инстансы по всему RTL.
```

Wrapper дает единый стиль и упрощает замену реализации.

---

# 27. FIFO Generator IP или XPM FIFO

В Vivado есть несколько вариантов.

## FIFO Generator IP

Плюсы:

```
GUI-настройка;
понятные параметры;
можно быстро сгенерировать;
хорошо подходит для стандартных FIFO.
```

Минусы:

```
появляется IP-файл;
сложнее переносить между проектами;
требует управления generated sources;
менее удобно для чистого RTL-репозитория.
```

## XPM FIFO

Плюсы:

```
чистый RTL instantiation;
хорошо подходит для git;
легко параметризовать;
не нужно генерировать отдельный IP;
удобно для reusable modules.
```

Минусы:

```
нужно аккуратно задать параметры;
template выглядит громоздко;
для новичка меньше визуальной подсказки, чем в IP GUI.
```

## AXIS Clock Converter / AXI-Stream Data FIFO

Плюсы:

```
готовый AXI-Stream интерфейс;
обрабатывает tvalid/tready/tlast/tkeep;
удобно в Block Design.
```

Минусы:

```
более тяжелая интеграция;
не всегда удобно в чистом RTL;
иногда избыточно для простой задачи.
```

---

# 28. Рекомендуемый практический выбор

Для обычного RTL-проекта:

```
single data stream без AXIS:
    xpm_fifo_async через свой wrapper

AXI-Stream между clock domains:
    AXIS Clock Converter или AXI4-Stream Data FIFO
    либо свой wrapper на xpm_fifo_async с tdata/tlast/tkeep/tuser

маленький control/event crossing:
    не FIFO, а synchronizer/handshake

большой burst/data buffer:
    xpm_fifo_async или FIFO Generator на BRAM/URAM
```

---

# 29. Async FIFO как CDC boundary

Хороший стиль — явно показывать FIFO как границу доменов.

```verilog
// clk_a domain signals
wire        rx_valid_a;
wire [31:0] rx_data_a;
wire        rx_ready_a;

// clk_b domain signals
wire        rx_valid_b;
wire [31:0] rx_data_b;
wire        rx_ready_b;

// CDC boundary
axis_async_fifo u_rx_cdc_fifo (
    .s_axis_aclk   (clk_a),
    .s_axis_tvalid (rx_valid_a),
    .s_axis_tready (rx_ready_a),
    .s_axis_tdata  (rx_data_a),

    .m_axis_aclk   (clk_b),
    .m_axis_tvalid (rx_valid_b),
    .m_axis_tready (rx_ready_b),
    .m_axis_tdata  (rx_data_b)
);
```

По коду сразу видно:

```
до FIFO — clk_a domain
после FIFO — clk_b domain
```

Это сильно упрощает debug и review.

---

# 30. Нельзя читать `full` в read domain

`full` принадлежит write domain.

Плохо:

```verilog
always @(posedge rd_clk) begin
    if (fifo_full) begin
        ...
    end
end
```

Почему плохо:

```
fifo_full сформирован в wr_clk domain
rd_clk logic не должна напрямую использовать этот сигнал
```

Правильно:

```
full используется только на стороне записи
empty используется только на стороне чтения
```

Если read side нужно знать “FIFO почти полный”, это уже отдельный status crossing или нужно использовать правильный флаг в нужном домене, если IP его предоставляет.

---

# 31. Нельзя читать `empty` в write domain

Аналогично:

```
empty принадлежит read domain
```

Плохо:

```verilog
always @(posedge wr_clk) begin
    if (!fifo_empty) begin
        ...
    end
end
```

Write side должен ориентироваться на:

```
full
almost_full
wr_data_count
```

Read side должен ориентироваться на:

```
empty
almost_empty
rd_data_count
```

---

# 32. Нельзя подавать wr_en без учета full

Плохой вариант:

```verilog
assign fifo_wr_en = src_valid;
```

При `full = 1` будет overflow.

Правильно:

```verilog
assign src_ready  = !fifo_full;
assign fifo_wr_en = src_valid && src_ready;
```

Если source не умеет останавливаться, нужен либо FIFO достаточной глубины, либо отдельная политика потери данных, например:

```
drop new data
drop old data
set overflow flag
packet discard
backpressure upstream
```

Это должно быть архитектурным решением, а не случайным поведением.

---

# 33. Нельзя подавать rd_en без учета empty

Плохой вариант:

```verilog
assign fifo_rd_en = dst_ready;
```

Правильно:

```verilog
assign dst_valid  = !fifo_empty;
assign fifo_rd_en = dst_ready && dst_valid;
```

Для standard FIFO mode `dst_valid` может быть не просто `!empty`, потому что есть read latency. Для FWFT это проще.

---

# 34. Async FIFO не заменяет packet logic

FIFO хранит слова. Он не всегда знает, где начинается и заканчивается пакет.

Для packet stream нужно передавать sideband:

```
data
last
keep
user
error
channel id
```

Например:

```verilog
assign fifo_din = {
    s_axis_tlast,
    s_axis_tkeep,
    s_axis_tdata
};

assign {
    m_axis_tlast,
    m_axis_tkeep,
    m_axis_tdata
} = fifo_dout;
```

Если потерять `tlast`, destination side не сможет восстановить границы пакета.

---

# 35. Width conversion в FIFO

Иногда удобно писать одной шириной, а читать другой.

Например:

```
write side:  8 bit
read side:  32 bit
```

Или:

```
write side:  64 bit
read side:  16 bit
```

Это полезно для согласования интерфейсов, но добавляет нюансы:

```
1. порядок байтов;
2. когда появляется valid output;
3. как обрабатывается последний неполный word;
4. как передавать keep/last;
5. как считать глубину;
6. как интерпретировать data_count.
```

Для packet-based stream width conversion лучше делать осознанно, особенно если есть `tkeep` и `tlast`.

---

# 36. Packet overflow policy

Если FIFO используется для пакетов, нужно решить, что делать при переполнении.

Варианты:

```
1. остановить upstream через backpressure;
2. отбросить новое слово;
3. отбросить весь текущий packet;
4. отбросить старые данные;
5. выставить error flag и продолжить;
6. сбросить FIFO/stream recovery.
```

Самый чистый вариант — backpressure:

```
s_axis_tready = !fifo_full
```

Но не каждый источник умеет ждать. Например, ADC или внешний PHY может продолжать выдавать данные независимо от готовности.

Тогда FIFO должен быть достаточно глубоким или должна быть явная политика потери данных.

---

# 37. Async FIFO и непрерывный источник данных

Важное ограничение:

> если источник генерирует данные быстрее, чем приемник способен читать в среднем, FIFO переполнится.

Даже бесконечно большой FIFO только отложил бы проблему.

Например:

```
source average = 200 MB/s
destination average = 150 MB/s
```

Разница:

```
50 MB/s
```

FIFO будет заполняться со скоростью 50 MB/s.

Значит, нужно:

```
снизить input rate;  
увеличить output rate;  
добавить backpressure;  
дропать данные;  
сжимать/фильтровать данные;  
изменить архитектуру.
```

---

# 38. Когда async FIFO не нужен

Async FIFO не стоит использовать везде подряд.

Он может быть избыточен для:

```
одного status flag;
редкого start pulse;
одного configuration bit;
простого request/acknowledge;
медленно меняющегося mode signal;
reset crossing.
```

В этих случаях лучше использовать:

```
bit synchronizer
pulse synchronizer
handshake
reset synchronizer
```

Async FIFO нужен именно для **очереди данных**.

---

# 39. Когда async FIFO нужен почти точно

Async FIFO почти всегда уместен для:

```
потоковых данных;
burst traffic;
AXI-Stream CDC;
передачи sample data;
перехода между user_clk и processing_clk;
передачи пакетов;
сглаживания неравномерной готовности consumer;
clock domain boundary между IP-блоками.
```

Признак:

```
есть последовательность слов, и важно сохранить порядок
```

Тогда FIFO — естественное решение.

---

# 40. Порядок данных в FIFO

FIFO означает:

```
First In, First Out
```

То есть порядок сохраняется:

```
write: A, B, C, D
read:  A, B, C, D
```

Async FIFO не сортирует данные, не объединяет и не интерпретирует их. Он только сохраняет порядок записи при чтении.

Если видите нарушение порядка, обычно причина не в “асинхронности”, а в:

```
ошибке wr_en;
ошибке rd_en;
overflow/underflow;
неправильном reset;
неверной упаковке sideband;
ошибке в packet layer;
неверном использовании standard/FWFT mode.
```

---

# 41. Reset async FIFO

У async FIFO есть reset, и это отдельная важная тема.

Кратко:

```
reset должен корректно сбросить обе стороны FIFO;
после reset нельзя сразу считать flags валидными без учета busy/reset completion;
у XPM FIFO есть специальные reset behavior и иногда reset busy signals;
reset crossing не стоит делать “на глаз”.
```

Здесь подробно не разворачиваю, потому что у тебя отдельно запланирована заметка **reset-domain interaction**. Но в практическом дизайне reset async FIFO — одно из мест, где часто появляются ошибки.

---

# 42. Constraints для async FIFO

Кратко:

```
wr_clk и rd_clk должны быть правильно описаны в XDC;
если clocks асинхронны, между ними должны быть корректные CDC constraints;
нельзя пытаться timing close обычные paths между unrelated clock domains;
XPM/IP обычно содержит или подразумевает правильные CDC-структуры, но clocks все равно должны быть описаны корректно.
```

Подробно это лучше раскрывать в отдельной теме **CDC constraints**, чтобы не смешивать RTL-архитектуру и XDC.

---

# 43. Проверка в simulation

Simulation async FIFO проверяет не metastability, а protocol.

Что стоит проверять:

```
1. данные не теряются;
2. порядок данных сохраняется;
3. нет записи при full;
4. нет чтения при empty;
5. корректно работает backpressure;
6. correct behavior на burst;
7. correct behavior при разных частотах wr_clk/rd_clk;
8. correct behavior при случайных паузах чтения и записи;
9. correct behavior после reset;
10. sideband-сигналы идут вместе с data.
```

Полезный testbench-подход:

```
source генерирует счетчик или sequence number;
FIFO передает данные;
destination проверяет, что sequence number идет строго по порядку.
```

Например:

```
write: 0, 1, 2, 3, 4, ...read:  0, 1, 2, 3, 4, ...
```

Если destination увидел:

```
0, 1, 2, 4
```

значит потеряно слово `3`.

Если увидел:

```
0, 1, 2, 2, 3
```

значит есть повтор.

Если увидел:

```
0, 1, 3, 2
```

значит нарушен порядок.

---

# 44. Randomized testbench

Для async FIFO полезна randomized simulation:

```
wr_clk и rd_clk с разными периодами;
случайный src_valid;
случайный dst_ready;
случайные burst;
случайные паузы;
проверка scoreboard.
```

Scoreboard:

```
при успешной записи добавляем слово в reference queue;
при успешном чтении сравниваем dout с первым словом reference queue.
```

Успешная запись:

```verilog
wr_fire = wr_en && !full;
```

Успешное чтение:

```verilog
rd_fire = rd_en && !empty;
```

Для AXI-style:

```verilog
s_fire = s_valid && s_ready;
m_fire = m_valid && m_ready;
```

---

# 45. Hardware debug

В реальном железе для async FIFO полезно выводить в ILA:

Write side:

```
wr_clk
wr_en
full
almost_full
overflow flag
wr_data_count
```

Read side:

```
rd_clk
rd_en
empty
almost_empty
underflow flag
rd_data_count
```

Но нужно помнить:

```
сигналы разных clock domains нельзя бездумно смотреть в одной ILA clock domain
```

Практически лучше:

```
одна ILA на wr_clk domain;
другая ILA на rd_clk domain;
или аккуратно синхронизированные debug flags.
```

---

# 46. Типовой wrapper для AXI-Stream async FIFO

Упрощенный пример для `tdata` + `tlast`.

```verilog

module axis_async_fifo_simple #(
    parameter integer DATA_WIDTH = 32,
    parameter integer FIFO_DEPTH = 1024
)(
    input  wire                  s_axis_aclk,
    input  wire                  s_axis_rst,
    input  wire                  s_axis_tvalid,
    output wire                  s_axis_tready,
    input  wire [DATA_WIDTH-1:0] s_axis_tdata,
    input  wire                  s_axis_tlast,

    input  wire                  m_axis_aclk,
    input  wire                  m_axis_rst,
    output wire                  m_axis_tvalid,
    input  wire                  m_axis_tready,
    output wire [DATA_WIDTH-1:0] m_axis_tdata,
    output wire                  m_axis_tlast
);

    localparam integer FIFO_WIDTH = DATA_WIDTH + 1;

    wire                  fifo_full;
    wire                  fifo_empty;
    wire                  fifo_wr_en;
    wire                  fifo_rd_en;
    wire [FIFO_WIDTH-1:0] fifo_din;
    wire [FIFO_WIDTH-1:0] fifo_dout;

    assign s_axis_tready = !fifo_full;
    assign fifo_wr_en    = s_axis_tvalid && s_axis_tready;
    assign fifo_din      = {s_axis_tlast, s_axis_tdata};

    assign m_axis_tvalid = !fifo_empty;
    assign fifo_rd_en    = m_axis_tvalid && m_axis_tready;

    assign {m_axis_tlast, m_axis_tdata} = fifo_dout;

    cdc_async_fifo #(
        .DATA_WIDTH (FIFO_WIDTH),
        .FIFO_DEPTH (FIFO_DEPTH)
    ) u_fifo (
        .wr_clk       (s_axis_aclk),
        .wr_rst       (s_axis_rst),
        .wr_en        (fifo_wr_en),
        .din          (fifo_din),
        .full         (fifo_full),
        .almost_full  (),

        .rd_clk       (m_axis_aclk),
        .rd_rst       (m_axis_rst),
        .rd_en        (fifo_rd_en),
        .dout         (fifo_dout),
        .empty        (fifo_empty),
        .almost_empty ()
    );

endmodule
```

Это концептуальный пример. В реальном проекте нужно проверить режим чтения FIFO. Такой простой mapping `m_axis_tvalid = !empty` особенно естественен для FWFT FIFO.

---

# 47. Типовая ошибка с standard FIFO mode

Для standard FIFO mode ошибка часто выглядит так:

```verilog
assign m_axis_tvalid = !fifo_empty;
assign m_axis_tdata  = fifo_dout;
assign fifo_rd_en    = m_axis_tvalid && m_axis_tready;
```

Это корректно только если `fifo_dout` уже содержит валидное текущее слово, то есть FIFO работает в FWFT-like режиме.

В standard mode после `rd_en` данные могут появиться позже.

Тогда нужна дополнительная логика регистрации `valid`.

Поэтому для stream wrapper проще использовать FWFT, если IP и архитектура позволяют.

---

# 48. Skid buffer после FIFO

Иногда после FIFO ставят небольшой register slice или skid buffer.

Зачем:

```
улучшить timing;
развязать ready path;
сделать AXI-Stream поведение чище;
избежать длинной combinational path от m_ready до rd_en;
удерживать dout при backpressure.
```

Для AXI-Stream это часто полезно, особенно если downstream logic сложная.

Структура:

```
async FIFO -> small output buffer -> downstream AXIS
```

---

# 49. Backpressure path

В AXI-Stream FIFO есть важный путь:

```
downstream ready влияет на чтение FIFO
upstream ready зависит от full
```

Но между write и read sides нет прямого combinational пути. Это хорошо.

Async FIFO разрывает timing между доменами:

```
s_axis_tready формируется в source clock domain
m_axis_tvalid формируется в destination clock domain
```

Это одно из преимуществ FIFO как CDC boundary.

---

# 50. Async FIFO и latency-sensitive control

FIFO не всегда хорош для control-команд, где важно:

```
команда отправлена;
команда принята;
получен ack;
source знает, что destination действительно обработал команду.
```

FIFO гарантирует доставку в очередь, но не обязательно обработку downstream-логикой.

Если нужен именно “команда выполнена”, то поверх FIFO нужен response path или handshake.

Пример:

```
command FIFO: clk_a -> clk_b
response FIFO/handshake: clk_b -> clk_a
```

---

# 51. Async FIFO и clock stopping

Если один из clocks может останавливаться, нужно быть особенно аккуратным.

Например:

```
wr_clk работает
rd_clk остановлен
```

Тогда FIFO будет заполняться, пока не станет full.

Если source уважает full — данные не потеряются, но поток остановится.

Если source не может остановиться — будет overflow.

Обратная ситуация:

```
rd_clk работает  
wr_clk остановлен
```

FIFO опустеет, затем `empty` станет активным.

Async FIFO может работать с независимыми clocks, но архитектура должна учитывать остановку одного домена.

---

# 52. Async FIFO и reset после clock stop

Если clock остановлен, reset/flags могут вести себя не так, как ожидается.

Например, если read side reset отпущен, но `rd_clk` не тикает, часть read-side логики не выйдет из reset-состояния.

Это снова относится к теме reset-domain interaction, но практически важно помнить:

```
для корректного reset clock domain должен получить clock edges
```

---

# 53. Async FIFO и `valid` output у XPM

У XPM FIFO есть порты вроде:

```
valid
data_valid
wr_ack
overflow
underflow
```

Их смысл зависит от режима FIFO.

Не стоит механически подключать `valid`, не проверив документацию/template.

Для простого FWFT-style wrapper часто достаточно:

```
m_valid = !empty
rd_en = m_valid && m_ready
```

Но если выбран standard mode, может быть полезен именно `valid` от FIFO.

Практическое правило:

```
выбрать один режим чтения;
понять timing diagram;
после этого писать wrapper.
```

---

# 54. Async FIFO и report_cdc

Готовые XPM/IP FIFO Vivado обычно распознает как CDC-структуры.

Но CDC report может показывать paths, связанные с FIFO, если:

```
FIFO написан вручную;
атрибуты не расставлены;
pointers синхронизируются нестандартно;
clocks не описаны;
constraints конфликтуют;
используется самодельная RAM/pointer logic.
```

Для production-проекта это еще один аргумент в пользу XPM или проверенного FIFO-модуля.

---

# 55. Самодельный async FIFO

Написать async FIFO самому можно, но это не лучший первый выбор.

Нужно правильно реализовать:

```
dual-port memory;binary pointers;Gray-code pointers;pointer synchronization;full detection;empty detection;reset behavior;read latency;overflow/underflow behavior;formal/simulation checks;attributes for synchronizers;tool-specific RAM inference.
```

Ошибки в async FIFO часто проявляются редко и трудно воспроизводятся.

Практический совет:

```
для реального Vivado-проекта лучше использовать XPM/IP;самодельный FIFO писать только если есть веская причина.
```

---

# 56. Минимальная концептуальная структура async FIFO

Не как готовый код, а как архитектурная схема:

```
write domain:    wr_ptr_bin    wr_ptr_gray    synchronized rd_ptr_gray    full logicread domain:    rd_ptr_bin    rd_ptr_gray    synchronized wr_ptr_gray    empty logicmemory:    dual-port RAM    write port on wr_clk    read port on rd_clk
```

Схема:

```
                    +-------------------+wr_clk domain       |                   |       rd_clk domain                    |   dual-port RAM   |din ---- write ---->|                   |---- read ----> dout                    |                   |                    +-------------------+wr_ptr_bin  -------------------------- controls write addressrd_ptr_bin  -------------------------- controls read addresswr_ptr_gray ---- sync ----> rd domain ---- empty logicrd_ptr_gray ---- sync ----> wr domain ---- full logic
```

---

# 57. Почему async FIFO считается “правильным CDC”

Потому что data bus не пересекает clock boundary как набор нестабильных битов.

Данные записываются в память в write domain:

```
memory write через wr_clk
```

Данные читаются из памяти в read domain:

```
memory read через rd_clk
```

Через CDC пересекаются только pointers в Gray code, которые специально предназначены для такого пересечения.

То есть async FIFO превращает опасную задачу:

```
передать data bus между clocks
```

в контролируемую задачу:

```
передать pointer state через Gray-code synchronizers
```

---

# 58. Типовые ошибки async FIFO

## Ошибка 1. Использовать `full` не в том домене

```
full используется в rd_clk domain
```

Так нельзя.

---

## Ошибка 2. Использовать `empty` не в том домене

```
empty используется в wr_clk domain
```

Так нельзя.

---

## Ошибка 3. Игнорировать full

```
wr_en активен даже при full
```

Итог:

```
overflowпотеря данныхперезапись старых данныхpacket corruption
```

---

## Ошибка 4. Игнорировать empty

```
rd_en активен даже при empty
```

Итог:

```
underflowповтор старых данныхмусор на выходеошибка sequence
```

---

## Ошибка 5. Неверно понимать read mode

`standard` и `FWFT` имеют разную timing-semantics.

---

## Ошибка 6. Не передать sideband

Передали `tdata`, но забыли `tlast`.

Итог:

```
данные есть, но packet boundary потерян
```

---

## Ошибка 7. Считать, что FIFO исправляет постоянную разницу скоростей

Если средняя скорость записи выше средней скорости чтения, FIFO переполнится.

---

## Ошибка 8. Слишком маленькая глубина

FIFO работает в коротком тесте, но переполняется на реальных burst.

---

## Ошибка 9. Неправильный reset

После reset FIFO иногда стартует с неправильными flags или ложными данными.

---

## Ошибка 10. Самодельный FIFO без тщательной проверки

Особенно опасны ошибки в full/empty на wrap-around.

---

# 59. Практический checklist для async FIFO

Перед использованием async FIFO стоит ответить:

```
1. Какие clock domains соединяются?2. Какой direction передачи?3. Какая ширина data word?4. Есть ли sideband-сигналы?5. Нужно ли сохранять packet boundary?6. Нужен ли AXI-Stream wrapper?7. Какая средняя скорость записи?8. Какая средняя скорость чтения?9. Какие максимальные burst?10. Может ли consumer останавливаться?11. Может ли producer быть остановлен?12. Что делать при full?13. Что делать при empty?14. Какая нужна глубина FIFO?15. Нужен ли almost_full?16. Нужен ли FWFT?17. Как будет проверяться overflow/underflow?18. Как FIFO сбрасывается?19. Какие signals относятся к wr_clk domain?20. Какие signals относятся к rd_clk domain?
```

---

# 60. Практическое правило выбора

Коротко:

```
одиночный level signal    -> bit synchronizerредкий pulse/event    -> pulse/toggle synchronizerодно data word иногда    -> handshakeпоток data words    -> async FIFOAXI-Stream между clocks    -> AXIS async FIFO / AXIS Clock Converter / wrapper around xpm_fifo_asyncразные clocks + burst traffic    -> async FIFO с достаточной depthразные clocks + packet stream    -> async FIFO, data + sideband вместе
```

---

# 61. Главное резюме

**Async FIFO** — основной практический CDC-механизм для потоковых данных.

Он нужен, когда:

```
данные идут последовательностью;нужно сохранить порядок;write и read работают в разных clock domains;частоты могут отличаться;одна сторона может временно останавливаться.
```

Главные правила:

```
1. wr_en, din, full относятся к wr_clk domain.2. rd_en, dout, empty относятся к rd_clk domain.3. full нельзя использовать в read domain.4. empty нельзя использовать в write domain.5. Нельзя писать при full.6. Нельзя читать при empty.7. Data и sideband нужно передавать вместе.8. FIFO depth должна покрывать burst и stall.9. FIFO не исправляет постоянный throughput mismatch.10. Для Vivado лучше использовать XPM FIFO или готовый IP, а не писать async FIFO вручную без необходимости.
```

Короткая формула:

```
Async FIFO = dual-port memory + Gray-code pointer CDC + full/empty logic
```