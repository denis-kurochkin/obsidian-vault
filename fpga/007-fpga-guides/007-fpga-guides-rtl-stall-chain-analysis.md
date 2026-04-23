## Stall chain analysis

**Stall chain analysis** в контексте FPGA/Vivado и **Digital Design/RTL** — это разбор того, как сигнал остановки потока данных распространяется назад по pipeline или streaming path, какие стадии реально блокируются, где появляется лишняя latency, где растет combinational path и в каком месте backpressure начинает влиять уже не только на function, но и на timing. Это не отдельный protocol, а способ смотреть на уже существующий **flow-control path** как на полноценную часть architecture. В AXI-style interfaces сама передача строится на **VALID/READY handshake**: transfer происходит только когда и `VALID`, и `READY` одновременно High; при этом source обязан удерживать data stable до handshake, а на master/slave interfaces не должно быть combinational paths между input и output signals.

Главная мысль здесь такая:  
**stall chain — это не просто “ready пошел назад”, а временная и структурная зависимость между несколькими stage pipeline.** Если downstream block перестал принимать данные, upstream logic должна либо удерживать текущее слово, либо парковать его в local buffer, либо остановить выпуск новых слов. И чем длиннее эта цепочка, тем выше шанс, что она станет и functional bottleneck, и critical path одновременно. AMD прямо показывает типичную модель backpressure на примере CORDIC: когда output buffer почти full, core прекращает дальнейшие operations, затем перестают разгружаться input buffers, они заполняются, и `TREADY` на входах de-asserts — это и есть нормальное upstream propagation stall condition.

### Почему это отдельная полноценная подтема

Внутри более общего блока про **Backpressure propagation** тема **stall chain analysis** выделяется отдельно потому, что она отвечает уже не на вопрос “есть ли backpressure”, а на вопрос **как именно он проходит через design**. В маленьком тракте достаточно знать, что `ready/valid` работает корректно. Но в длинном datapath этого мало. Нужно понимать:

- сколько stage участвует в stall propagation,
- какие stage останавливаются combinationally, а какие registered,
- где stall приходит за один cycle, а где через buffer,
- где добавляется bubble,
- где pipeline остается throughput-1, а где превращается в bursty path.

Это особенно важно в FPGA, потому что Arm AXI specification одновременно требует корректный **VALID/READY flow control** и запрещает combinational paths между input и output signals на interface. На практике это означает, что длинную stall chain нельзя бесконечно растягивать “чистой combinational ready-логикой” без архитектурных последствий.

### Что такое stall chain в практическом смысле

В practical RTL sense **stall chain** — это последовательность stage, через которые назад распространяется невозможность принять новые данные. В одном design это может быть простой `ready` chain, в другом — `stall`, `busy`, `full/almost_full`, `credit` depletion или внутренняя FSM pause condition. Но суть одна: downstream side сообщает upstream side, что data movement должен быть temporarily stopped.

Самое полезное здесь — различать два уровня:

**Functional stall chain** — кто именно перестает двигать данные.  
**Timing stall chain** — через какую именно logic этот stop condition физически проходит за один cycle.

Именно второй уровень часто ломает Fmax. Даже если behavior верный, ready/stall path может оказаться длиннее data path. Arm specification специально подчеркивает, что VALID/READY — это two-way flow-control mechanism, но combinational path между input и output на interface быть не должно. В FPGA это сразу переводится в практический вопрос: где именно stall propagation должен быть registered или buffered.

### С чего начинается анализ stall chain

Хороший **stall chain analysis** почти всегда начинается не с waveforms всего top-level design, а с явного перечисления stage pipeline и их local contract. Для каждой стадии полезно понимать:

- что считается **accept condition**;
- что считается **hold condition**;
- может ли stage хранить одно слово локально;
- есть ли internal queue, skid buffer или FIFO;
- combinational или registered у нее выходной `ready/stall`;
- что происходит с sideband signals при остановке.

Идея в том, чтобы увидеть stall chain не как “все как-то тормозит”, а как набор конкретных boundaries между stage. На AMD стороне похожая мысль прослеживается в рекомендациях использовать **AXI register slices** и **data FIFOs** в местах, где timing closure или rate adaptation становятся важными: это фактически способ не дать одной длинной stall chain пройти через весь тракт без controlled breakpoints.

### Что именно нужно анализировать

У stall chain обычно есть четыре главных измерения.

#### 1. Reach

Насколько далеко stall уходит назад.  
Есть designs, где backpressure почти мгновенно доходит до самого source. Есть другие, где между блоками стоят FIFOs, поэтому source продолжает еще некоторое время выдавать data, пока буферы не заполнятся. AMD CORDIC guide наглядно показывает именно такой buffered case: сначала стопорится output side, потом internal output buffer, потом input buffers, и только после этого падает input `TREADY`.

#### 2. Latency of stall propagation

Через сколько cycles upstream block узнает, что дальше идти нельзя.  
Если propagation purely combinational, реакция может быть “в тот же cycle”, но timing path становится тяжелым. Если propagation registered, timing обычно лучше, но появляется extra cycle reaction latency. Именно здесь появляются **register slice**, **skid buffer** и small FIFOs как controlled compromise between latency and timing. AMD documents прямо описывают two-deep **register slice (skid buffer)** как способ улучшить timing closure ценой одного cycle latency.

#### 3. Stall granularity

Останавливается вся pipeline целиком или только часть.  
Если в design есть decoupling buffers, stall может локально “застревать” и не доходить мгновенно до upstream source. Если buffers нет, одна slow stage может effectively freeze весь тракт.

#### 4. Timing cost

Сколько logic levels и routing delay сидит в path `ready/stall`.  
Это отдельный вопрос, и он не равен functional correctness. Path может быть полностью правильным по simulation, но именно он станет critical path post-route.

### Почему stall chain часто становится critical path

Это одна из самых важных practical причин изучать тему отдельно. В data pipeline engineer обычно внимательно смотрит на arithmetic path, LUT depth, DSP chain, mux tree. Но в streaming architecture critical path часто живет не там, а в **backward control path** — в цепочке `ready`, `stall`, `full`, `can_accept`. Причина простая: один downstream decision влияет на много upstream stages, а значит logic fanout и dependency depth растут быстро.

Arm AXI rule “no combinatorial paths between input and output” существует не только из protocol purity, но и как сильное architectural ограничение против неконтролируемых handshake loops. А AMD design guidance дополнительно рекомендует **AXI register slices** именно для pipelining AXI4-Stream datapath и облегчения timing closure. Это хороший индикатор того, что long handshake/backpressure paths — реальная timing problem, а не только учебная теория.

### Где stall chain обычно становится плохой

Есть несколько типовых мест, где stall chain резко ухудшается.

**Длинная ready chain через много стадий.**  
Каждая stage добавляет свое условие вида `local_ready = downstream_ready && !local_full && state_ok && sideband_ok`. По отдельности это выглядит harmless, вместе — превращается в длинный combinational cone.

**Смешивание dataflow и controlflow.**  
Когда `ready` зависит не только от buffer state, но и от packet FSM, arbitration, mode bits, error handling и dynamic formatting, backward path становится тяжелее, чем сам data mux.

**Размытые boundaries.**  
Если между stage нет четких register/buffer boundaries, невозможно локализовать stall. В результате один unlucky timing path проходит через половину subsystem.

**Скрытая fanout problem.**  
Один stall condition может управлять большим числом enables, hold controls и valid bits. Даже при небольшой логической depth routing delay уже становится существенным.

### Combinational stall chain и registered stall chain

Очень полезно различать эти два стиля.

**Combinational stall chain** дает быструю реакцию: downstream deasserts `ready`, upstream видит это сразу. Это хорошо для zero-extra-cycle flow control, но плохо для long paths и рискованно в больших системах. На AXI interfaces pure combinational input→output path вообще запрещен spec’ом.

**Registered stall chain** вводит явные state points: stall decision проходит через registers, register slices или small buffers. Timing улучшается, place-and-route становится легче, но stall reaches upstream with one or more cycle delay. Чтобы это не ломало данные, нужны local holding capability, обычно в форме **skid buffer** или elastic stage.

Отсюда вытекает важный принцип:  
**stall chain analysis почти всегда заканчивается не на вопросе “работает ли”, а на вопросе “где должен стоять intentional break in the chain”.**

### Роль skid buffer и register slice

В теме stall chain это почти базовые инструменты. AMD прямо описывает **two-deep register slice (skid buffer)** как средство улучшить system timing closure ценой одного cycle latency. В AXI Interconnect такие slices можно вставлять на channels у SI/MI slots, а в Versal methodology guide отдельно советуют использовать **AXI register slices** для pipelining AXI4-Stream datapath и помощи placer’у.

Почему это важно именно для stall chain:

- skid buffer разрывает длинный backward combinational dependency;
- дает local storage на момент, когда stall приходит не мгновенно;
- сохраняет throughput better, чем naive “просто все зарегистрировать”;
- делает boundary между stage явной и анализируемой.

Практически, если stall chain становится длинной и timing начинает ломаться, **register slice / skid buffer** — это обычно не patch, а архитектурный элемент, который нужно было предусмотреть.

### FIFO и stall absorption

Если skid buffer — это средство локального разрыва stall chain, то **FIFO** — это средство поглощения burst mismatch и rate mismatch. AMD AXI Interconnect docs отдельно отмечают, что **datapath FIFOs** могут улучшить throughput, когда data rate на slots отличается из-за width conversion или clock conversion. Это важный момент: stall chain анализируется не только как сигнал остановки, но и как вопрос, **сколько остановки система может поглотить до того, как начнет тормозить upstream side**.

В хорошем RTL-thinking это различие выглядит так:

- **register slice / skid buffer** — контролирует timing и short-term handshake behavior;
- **FIFO** — контролирует burst absorption, decoupling и longer-term backpressure reach.

Если их роли не разделять, легко получить design, где либо timing плохой, либо latency/area слишком велики, либо и то и другое сразу.

### Stall chain analysis на packet path

Для packetized stream тема становится сложнее, потому что stall касается не только data word, но и **packet boundary semantics**: `last`, `keep`, `user`, sideband flags, sometimes error markers. Functional analysis здесь должен отвечать не только на вопрос “остановились ли данные”, но и на вопрос “сохранилась ли согласованность packet metadata”.

Если stall arrives в момент packet boundary, stage обязана удержать не только payload, но и весь associated sideband set. Иначе появляются самые неприятные bugs: packet duplicated, packet truncated, `last` shifted, boundary lost under stall. Arm handshake rules о stable information until transfer are directly relevant here: data/control information должна оставаться stable, пока VALID asserted и handshake еще не произошел.

### Анализ throughput под stall

Еще один важный взгляд: stall chain нужно оценивать не только бинарно “есть/нет ошибок”, но и по **throughput behavior under pressure**.

Полезные вопросы:

- Сколько consecutive stalls может принять каждая stage без немедленного propagation назад?
- Какой sustained throughput остается при periodic downstream pauses?
- После снятия stall pipeline восстанавливает full throughput сразу или через bubbles?
- Где возникает head-of-line blocking?

AMD CORDIC example снова полезен как pattern: output not draining → output buffer fills → operations stop → input buffers fill → TREADY drops upstream. Это textbook example того, что throughput degradation происходит поэтапно, а не мгновенно. В stall chain analysis важно понимать именно эти этапы.

### Что проверять в simulation

Хороший **stall chain analysis** верификационно почти всегда включает directed stress cases, а не только “обычные данные прошли”.

Нужно проверять как минимум:

- single-cycle stall;
- long continuous stall;
- periodic stall, например 1-off / 1-on или 3-off / 5-on;
- stall на packet boundary;
- stall при заполнении локальных buffers;
- simultaneous stall + mode change/reset boundary, если такое возможно по architecture.

Идея здесь в том, чтобы увидеть не только correctness, но и pattern of propagation: кто первым тормозит, где образуется bubble, кто держит data stable, где начинает падать upstream `ready`.

### Что проверять в timing

После functional simulation следующий шаг — timing-oriented review. Для него полезно смотреть:

- path from downstream `ready/stall` to earliest upstream control point;
- fanout of stall-related nets;
- logic levels in `can_accept` / `ready` equations;
- placement spread of stages tied by the same stall chain;
- появились ли stall nets в worst paths post-synth/post-route.

AMD guidance про **AXI register slices** как средство timing closure — хороший practical marker: если streaming path уже заметен в timing report, значит stall chain deserves explicit architectural attention, а не только local RTL cleanup.

### Как выглядит хороший результат анализа

Хороший итог **stall chain analysis** обычно выглядит так:

- понятно, где stall начинается;
- видно, через какие stage он проходит immediately, а через какие only after buffer fill;
- известны registered breakpoints chain;
- нет illegal combinational handshake structure;
- packet semantics сохраняется под stall;
- ready/stall path не доминирует timing unexpectedly;
- есть осознанное решение, где использовать skid buffer, где FIFO, а где direct propagation еще допустима.

То есть цель анализа — не просто найти bug, а сделать **stall behavior explicit and bounded**.

### Типичные ошибки

У этой темы есть несколько очень повторяющихся ошибок.

**Первая** — считать, что если `valid/ready` формально работает на двух соседних блоках, то вся длинная chain автоматически здорова. На практике проблема часто живет только на длине тракта.

**Вторая** — игнорировать timing cost backward path и смотреть только на forward datapath. Это классическая причина, почему pipeline “логически хорошая”, но Fmax плохой.

**Третья** — пытаться решить long stall chain только глобальным register insertion без local storage. Тогда timing становится лучше, но появляются data loss или bubble artifacts.

**Четвертая** — ставить FIFO там, где нужен был skid buffer, или наоборот. Это приводит либо к лишней latency/area, либо к недостаточному decoupling.

**Пятая** — забывать, что AXI-style interface запрещает combinational input→output path. Попытка “ускорить ready” прямой combinational логикой может привести к non-compliant and fragile interface design.

### Практический итог

**Stall chain analysis** — это полноценная подтема внутри блока **Backpressure propagation**. Она отвечает не на вопрос “есть ли flow control”, а на более глубокий вопрос: **как именно остановка проходит через pipeline, где она буферизуется, где становится timing problem и какой architecture нужен, чтобы stall оставался и корректным, и timing-friendly.** Handshake rules AXI задают базовую discipline для VALID/READY, AMD documentation показывает practical value of **register slices**, **skid buffers** и **FIFOs** для timing closure и controlled backpressure propagation.

Если сказать совсем коротко: **stall chain analysis** — это искусство видеть в сигнале остановки не один `ready=0`, а целую структуру зависимостей по времени, памяти и timing. И чем раньше эта структура становится явной, тем проще сделать streaming RTL одновременно надежной, быстрой и понятной в debug.