**Domain partitioning** в контексте FPGA/Vivado и System-Level/SoC Awareness — это архитектурное разделение всей системы на отдельные **domains**, внутри которых действуют свои общие правила работы. Чаще всего в этой теме под domain понимают прежде всего **clock domain**, но на практике полезно думать шире: система делится не только по clock, но и по **reset**, по типу **traffic**, по скорости **data path**, по роли блока в системе и по способу **integration**.

Главная идея такая:  
нельзя строить большую FPGA system как один сплошной кусок RTL, где все со всем связано напрямую. Намного лучше заранее разбить design на понятные области, у каждой из которых есть свои boundary, свои interface rules и свои способы связи с соседями.

Именно это и есть **domain partitioning**.

---

## Зачем это вообще нужно

Когда проект маленький, кажется, что можно просто “подключать блоки по мере необходимости”. Но когда система растет, появляются типичные проблемы:

- слишком много случайных связей между блоками,
- непонятно, где проходит **CDC boundary**,
- сложно понять, какой блок к какому **clock** относится,
- reset topology становится запутанной,
- трудно делать **timing closure**,
- трудно верифицировать и debug system behavior,
- любое изменение в одном месте начинает ломать соседние части.

Хорошее **partitioning** решает именно это. Оно делает систему более modular и более predictable. Ты не просто видишь набор модулей, а видишь структуру: где **control plane**, где **data plane**, где **high-speed I/O**, где **memory subsystem**, где **clock crossings**, где локальная логика, а где system backbone.

---

## Что такое domain в практическом смысле

В FPGA-проекте domain — это не абстрактная “область на схеме”, а реальная часть системы, внутри которой соблюдаются единые architectural rules.

Например, domain может быть таким:

- все блоки работают от одного `clk_axi`,
- все блоки используют один local reset policy,
- все интерфейсы внутри domain синхронные и без CDC,
- все block-to-block connections сделаны через один тип interface,
- внутри domain одинаковые требования по latency и throughput.

То есть **domain** — это область с внутренней однородностью.  
А **partitioning** — это осознанное отделение одной такой области от другой.

---

## Самый важный вариант: clock domain partitioning

В контексте темы **Clock domain strategy** чаще всего речь идет именно о разделении системы на **clock domains**.

Например, в одном SoC/FPGA design могут быть:

- **system/control domain** на 100 MHz,
- **AXI interconnect domain** на 150 MHz,
- **DDR/UI domain** на частоте memory controller,
- **video or sensor domain** от внешнего interface clock,
- **PCIe domain** от своего reference clock,
- **slow peripheral domain** для UART, GPIO, housekeeping logic.

Сам факт наличия нескольких clocks — это нормально. Проблема начинается тогда, когда эти domains возникают хаотично.

Хорошая архитектура говорит не просто “у нас есть 5 clocks”, а говорит:

- почему именно эти 5 clocks нужны,
- какие blocks живут в каждом domain,
- где проходят crossings,
- какой **CDC method** используется на каждой границе,
- какие interfaces разрешены между domains.

---

## Цель domain partitioning

Цель не в том, чтобы сделать “как можно больше domains”.  
Наоборот, хорошее partitioning ищет баланс:

- domains должно быть достаточно мало, чтобы system была manageable;
- domains должно быть достаточно много, чтобы каждая часть работала в своем подходящем режиме.

Это всегда **trade-off**.

Если domains слишком мало, система становится перегруженной:  
в одном clock приходится тянуть и slow control logic, и high-speed datapath, и внешние interfaces с разной природой.

Если domains слишком много, появляется перегрузка по CDC, constraints, debug и integration.

Значит, задача архитектора — найти естественные границы.

---

## По каким признакам систему делят на domains

Обычно domain boundaries появляются не случайно, а по нескольким понятным причинам.

### 1. Разная clock nature

Самый очевидный случай — блоки реально живут в разных clock worlds.  
Например:

- внешний ADC дает свой sample clock,
- DDR controller работает в своем UI clock,
- PCIe subsystem диктует свой clocking scheme,
- core logic работает от system PLL.

Если clocks независимые или слабо связанные, это почти наверняка разные domains.

### 2. Разная performance requirement

Некоторым блокам нужен высокий **throughput** и tight timing, а другим нужен только простой control.  
Тогда выгодно отделить быстрый **datapath domain** от медленного **control domain**.

Это важно, потому что control logic часто создает лишнюю combinational complexity, а datapath требует чистой pipeline structure. Когда все смешано, timing и readability обычно ухудшаются.

### 3. Разный тип interface

Очень полезно делить систему по **interface-oriented boundaries**.  
Например:

- AXI4 memory-mapped area,
- AXI-Stream processing chain,
- external serial interfaces,
- configuration/status fabric.

Если domain имеет единый тип interface, его проще reuse, verify и document.

### 4. Разная reset behavior

Иногда blocks логично разделять, потому что у них разная reset semantics.  
Например:

- один domain должен подниматься сразу после power-up,
- другой стартует только после lock от MMCM/PLL,
- третий resetится отдельно по software command.

Если не учитывать это заранее, reset network превращается в источник системных ошибок.

### 5. Разный ownership внутри architecture

На большом project часто есть и organizational reason:  
одна команда делает **packet processing**, другая — **DMA/AXI**, третья — **board interfaces**.  
Грамотное partitioning помогает согласовать не только logic, но и development flow.

---

## Как выглядит хорошее partitioning

Хорошее **domain partitioning** обладает несколькими признаками.

Во-первых, для каждого domain можно в одном-двух предложениях объяснить его purpose.  
Например:  
“Этот domain обслуживает streaming data path от sensor input до formatter”.  
“Этот domain содержит register control, status collection и software-visible logic”.

Во-вторых, у domain есть четкая граница.  
То есть понятно:

- какие сигналы входят,
- какие выходят,
- какой protocol используется,
- где нужен CDC,
- кто владеет backpressure,
- где находится buffering.

В-третьих, domain internally coherent.  
Внутри него не должно быть смеси из случайных clocks, нестабильных resets и ad-hoc connections.

В-четвертых, crossing между domains стандартизирован.  
Не “каждый раз по-разному”, а по повторяемому pattern.

---

## Что считается плохим partitioning

Плохое **domain partitioning** обычно узнается быстро, даже без formal review.

Например, плохой признак — когда clock domain boundary проходит “между двумя случайными регистрами”, потому что так было удобно подключить.  
Еще плохой признак — когда wide data bus идет напрямую между domains, а рядом отдельно синхронизируется только `valid`, и при этом никто не гарантирует data coherency.

Еще один плохой case — когда blocks формально относятся к разным domains, но у них нет четкого interface contract. Тогда каждый новый engineer начинает по-своему решать, как именно передавать pulse, status, packet, reset и enable.

Также плохой sign — когда partitioning сделано по file structure, а не по system behavior.  
То, что модули лежат в разных папках, не означает, что архитектура действительно разделена правильно.

---

## Базовые типы domains в FPGA system

На практике удобно мыслить систему через несколько типовых domains.

### Control domain

Это область, где живут:

- configuration registers,
- state machines верхнего уровня,
- status aggregation,
- interrupt/control logic,
- software-facing AXI-Lite or CSR logic.

Обычно здесь важны clarity, determinism и простота debug.  
Сверхвысокий throughput тут не нужен.

### Data domain

Это область, где идет основная обработка потока данных:

- filtering,
- framing,
- packetization,
- DSP chain,
- format conversion,
- stream processing.

Тут важны pipeline, throughput, latency budget, backpressure behavior.

### Memory domain

Это область вокруг BRAM/URAM/DDR interface и memory controller.  
Часто у нее свои clocks, свои handshake rules и свой scheduling behavior.

### I/O domain

Это все, что живет рядом с внешним миром:

- PHY-facing logic,
- source-synchronous interfaces,
- SERDES-adjacent blocks,
- sensor or converter interfaces.

Часто именно здесь появляются нестандартные clocks и самые чувствительные CDC boundaries.

### Management / bring-up domain

Иногда полезно отдельно выделить domain для startup, monitoring, error handling, debug hooks, version/status registers, test modes.  
Это особенно полезно в сложных SoC-like design.

---

## Границы между domains: что должно быть определено заранее

Когда ты разделил system на domains, следующая задача — правильно описать **boundary**.  
У границы должны быть явно заданы ответы на несколько вопросов.

**Какой тип информации пересекает границу?**  
Single-bit flag, pulse, multi-bit control word, packet stream, burst data?

**Граница synchronous или asynchronous?**  
Domains синхронно связаны, derived from same source, или fully asynchronous?

**Кто управляет flow?**  
Есть ли `valid/ready`, request/acknowledge, FIFO full/empty, token-based transfer?

**Нужен ли buffering?**  
Если скорости отличаются, почти всегда нужен FIFO или другой decoupling mechanism.

**Что происходит при reset?**  
Какая сторона выходит из reset раньше? Можно ли принимать трафик сразу? Нужна ли initialization handshake?

Если этого нет, domain partitioning формально есть, а practically — нет.

---

## CDC как следствие partitioning

Очень важная мысль:  
**CDC не должен быть случайной проблемой, он должен быть ожидаемым следствием partitioning.**

То есть правильный flow мышления такой:

1. Сначала ты определяешь domains.
2. Потом отмечаешь их boundaries.
3. Потом для каждой boundary выбираешь CDC method.

А не наоборот:  
сначала рисуешь произвольные связи, а потом героически вставляешь synchronizer туда, где Vivado или simulation показали проблему.

Типовые методы тут известны:

- **2-FF synchronizer** для single-bit level signal,
- **pulse synchronizer** для event transfer,
- **handshake** для control transaction,
- **async FIFO** для continuous stream,
- **dual-clock buffer** для packet/data transfer,
- **Gray-coded counters** для некоторых pointer-based schemes.

Важно понимать: выбор CDC method зависит не от “предпочтений”, а от природы interface.

---

## Domain partitioning и Vivado

В Vivado хорошее partitioning помогает почти во всех engineering tasks.

### Timing closure

Когда domains отделены правильно, легче:

- задавать clocks,
- описывать generated clocks,
- понимать asynchronous groups,
- анализировать critical paths,
- отделять real timing issue от CDC issue.

Timing report становится более readable, потому что path structure соответствует architecture.

### Constraints

Если partitioning сделан хорошо, constraints пишутся более естественно.  
Ты понимаешь, где нужен normal timing analysis, где CDC-aware treatment, где false path, где clock group separation, где input/output timing.

Плохая архитектура почти всегда порождает messy XDC.

### IP integration

В больших system Vivado IP blocks часто уже имеют свою clock/reset model.  
Если твоя system partitioned cleanly, их легче встроить.  
Если нет — начинается постоянная локальная адаптация, wrapper-on-wrapper и рост complexity.

### Debug

В ILA/debug flow тоже проще, когда известно:

- в каком domain смотреть signal,
- где ожидать задержку,
- где boundary,
- где packet еще “сырое”, а где уже “переформатированное”.

---

## Практический подход к partitioning на старте проекта

Хороший способ — не начинать сразу с RTL, а сначала сделать простую **domain map** системы.

На одной схеме или в таблице полезно зафиксировать:

- имя domain,
- его main clock,
- reset source,
- назначение,
- входные и выходные interfaces,
- связи с соседними domains,
- CDC mechanism на каждой связи.

Например, неформально это может выглядеть так:

- `ctrl_clk` domain — AXI-Lite registers, status, configuration;
- `stream_clk` domain — main processing pipeline;
- `ddr_ui_clk` domain — DMA and memory access;
- crossing `ctrl_clk -> stream_clk` — config sync through handshake;
- crossing `stream_clk -> ddr_ui_clk` — async FIFO;
- crossing `ddr_ui_clk -> ctrl_clk` — status flags through synchronizers.

Такой map очень быстро выявляет архитектурные слабые места еще до написания большого объема code.

---

## Полезный architectural принцип

Один из самых полезных принципов здесь такой:

**Внутри domain должно быть просто.  
Сложность должна концентрироваться на boundary — но в контролируемом виде.**

Это очень хорошее правило.  
Если внутри каждого domain куча локальных clocks, ad-hoc resets и нестандартные mini-crossings, значит partitioning не сработало.

Domain должен быть “тихой зоной”, где логика строится обычными synchronous rules.  
Граница — это особое место, где ты осознанно добавляешь synchronizer, FIFO, bridge, rate adaptation или protocol conversion.

---

## Domain partitioning и повторное использование блоков

Когда domain boundaries хорошо сформулированы, блоки легче reuse.  
Причина простая: reusable block должен иметь ясный operational context.

Например, блок намного проще переносить между project, если про него сразу понятно:

- в каком clock domain он живет,
- допускает ли он backpressure,
- как resetится,
- synchronous ли у него interfaces,
- кто отвечает за CDC — сам block или внешний wrapper.

Если этого нет, reuse становится болезненным, потому что каждый раз приходится заново разбираться с implicit assumptions.

---

## Частая ошибка: partitioning по интуиции, а не по dataflow

Очень часто engineer делит design так, как “красиво выглядит по module hierarchy”.  
Но правильнее делить по реальному **dataflow** и **controlflow**.

Нужно смотреть:

- где реально течет data,
- где меняется rate,
- где появляются очереди,
- где есть arbitration,
- где software вмешивается в processing,
- где появляются независимые clocks,
- где нужен deterministic latency,
- где приемлем elastic behavior.

Именно эти вещи должны определять domain boundaries, а не только эстетика hierarchy.

---

## Как понять, что partitioning уже пора пересмотреть

Есть несколько явных симптомов.

Если при каждом изменении interface приходится править много несвязанных модулей, значит boundaries неудачные.

Если CDC logic размазана по всей системе, а не сосредоточена в понятных местах, значит domains отделены плохо.

Если один block получает сразу 3–4 unrelated clocks, это часто признак architectural confusion.

Если timing analysis постоянно показывает paths, которые трудно даже логически объяснить, возможно, partitioning не соответствует реальному поведению system.

Если reset sequencing регулярно вызывает баги при startup, почти всегда стоит пересмотреть domain boundaries и reset ownership.

---

## Практический итог

**Domain partitioning** — это основа системного мышления в FPGA.  
Это не просто “разделить design на куски”, а задать правильную структуру всей системы:

- где проходят естественные architectural boundaries,
- какие blocks образуют единый domain,
- как domains взаимодействуют,
- где и каким способом выполняется CDC,
- как clock, reset, throughput и interface policy согласованы между собой.

Хорошее partitioning дает сразу несколько выигрышей:

- проще **timing closure**,
- проще **CDC analysis**,
- проще **integration**,
- проще **debug**,
- проще **scaling** проекта,
- проще **reuse** модулей,
- меньше скрытых system-level bugs.

Если сказать совсем коротко:  
**domain partitioning** — это способ превратить большой FPGA design из “набора связанных модулей” в понятную и управляемую систему.