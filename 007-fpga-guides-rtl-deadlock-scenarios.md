## Deadlock scenarios

**Deadlock scenarios** в контексте FPGA/Vivado и **Digital Design/RTL** — это такие архитектурные ситуации, в которых поток данных или транзакций перестает делать **forward progress**, потому что несколько blocks или channels начинают ждать друг друга по кругу. Для streaming RTL это обычно выглядит как вечный `ready=0` или как зависший `valid`, который никогда не получает handshake. Для AXI memory-mapped fabric deadlock часто проявляется как внешне “живой” interconnect, в котором handshake на одном interface вроде произошел, но ожидаемая транзакция дальше уже никогда не появляется. AMD прямо пишет, что одним из самых частых симптомов AXI protocol violation является apparent lock-up connected core, когда transfer completes on one side of the Interconnect, but the resultant transfer is never issued on the expected output interface.

Главная мысль здесь такая:  
**deadlock почти никогда не возникает из одной плохой строки RTL.**  
Обычно это результат неправильной **dependency structure** между handshake signals, buffers, arbitration, reset behavior или ordering rules. Arm в AXI specification прямо подчеркивает, что dependency rules между handshake signals нужны именно для предотвращения deadlock, и формулирует базовое правило: **VALID must not depend on READY**, тогда как receiver может ждать VALID перед тем, как assert READY.

### Почему это отдельная полноценная подтема

Внутри блока про **Backpressure propagation** тема **deadlock scenarios** выделяется отдельно потому, что не всякая остановка — это deadlock. Обычный stall может быть нормальным и временным: downstream busy, FIFO full, arbitration delay, memory latency. Deadlock начинается тогда, когда system уже не способна сама восстановить движение, потому что условия выпуска данных, приема данных и освобождения ресурсов образуют **closed wait cycle**. Arm AXI spec специально задает handshake dependency rules “to prevent a deadlock situation”, а AMD PG059 отдельно разбирает **cyclic dependency avoidance** и приводит конкретный пример, как deadlock возникает в AXI Crossbar при определенной комбинации outstanding transactions, IDs и out-of-order responses.

### С чего полезно начинать thinking

Самый полезный способ смотреть на тему — не как на “редкую багу”, а как на вопрос архитектурных зависимостей.  
Для любого потока полезно спросить:

- кто ждет кого;
- какое условие нужно для assert `valid`;
- какое условие нужно для assert `ready`;
- нужен ли release buffer space, чтобы движение возобновилось;
- нет ли ситуации, где оба конца ждут событие, которое может произойти только после уже состоявшегося handshake.

Arm specification дает здесь фундаментальное правило: source **не имеет права** ждать `READY`, прежде чем assert `VALID`; после assert `VALID` этот сигнал должен оставаться asserted до handshake; transfer occurs only when both `VALID` and `READY` are High. Именно это правило ломает многие потенциальные circular waits еще на уровне local interface contract.

### Базовый handshake deadlock

Самый классический **deadlock scenario** в ready/valid-based RTL — это когда source ждет `ready`, прежде чем поднимать `valid`, а destination ждет `valid`, прежде чем поднимать `ready`. На бумаге оба блока “ведут себя аккуратно”. На практике никто не делает первый шаг, и transfer never starts. Arm прямо запрещает такую source-side зависимость: **the VALID signal of the AXI interface sending information must not be dependent on the READY signal of the receiving interface**. Receiver, наоборот, может ждать VALID. Это asymmetric rule, и оно не случайно: именно так убирается один из самых опасных типов deadlock на single channel.

В чистом streaming RTL вне formal AXI semantics это правило полезно сохранять почти без изменений. Если producer говорит себе “я выставлю `valid`, только когда consumer уже подтвердил `ready`”, ты уже близко к deadlock-prone architecture. Хорошая stage должна уметь **advertise availability independently** от готовности downstream side, а stall handling уже делать отдельно через hold, skid buffer или FIFO. Это уже инженерный вывод из Arm handshake model, но именно так она обычно переносится в FPGA pipeline design.

### Cross-channel deadlock в write path

Есть deadlock scenarios сложнее, чем просто один channel. Arm AXI spec отдельно предупреждает про write transaction dependencies. Там есть очень конкретный caution: **master must not wait for AWREADY before driving WVALID**, потому что deadlock может случиться, если slave одновременно ждет `WVALID` перед тем, как assert `AWREADY`. Для AXI4/AXI5 spec также говорит, что master must not wait for slave to assert `AWREADY` or `WREADY` before asserting `AWVALID` or `WVALID`.

Это очень важная practical мысль для RTL. Deadlock часто живет не внутри одного handshake pair, а **между каналами**. То есть address channel и data channel по отдельности могут быть “правильными”, но combined dependency policy уже образует circular wait. Если master решил не выпускать write data, пока адрес не accepted, а slave решил не принимать адрес, пока не увидел готовность data side, link зависает не из-за invalid signal levels, а из-за wrong dependency graph. Это как раз тот случай, когда protocol-compliant local logic по ощущениям “почти правильная”, но system-level sequencing уже неверная.

### Cyclic dependency через interconnect

Следующий класс — **multi-transaction cyclic dependency**, уже не на уровне одного source и one destination, а на уровне fabric/interconnect. AMD в PG059 отдельно описывает риск deadlock, когда есть несколько transaction IDs, несколько outstanding transactions, больше одного slave device, и slave devices могут давать **out-of-order responses**. В этом случае появляется potential cyclic dependency risk. Затем AMD приводит конкретный example: два masters, два slaves, одинаковые ID threads внутри каждого master, перекрестные reads, и дальше responses приходят в таком порядке, что Crossbar не может пропустить ответ ни одному master, потому что каждый ждет completion другого transaction first.

Это очень хороший system-level example того, что deadlock может возникать **без единой ошибки в одном handshake wire**. Все signals могут toggl’иться legal way, все transactions по отдельности допустимы, но глобальный ordering state уже сделал system non-progressing. Именно поэтому AMD AXI Crossbar применяет метод **Single Slave per ID**: для каждого ID thread outstanding transactions в одном направлении могут идти только к одному MI slot at a time. AMD прямо пишет, что это rule stalls later transaction until earlier response completes, и тем самым removes interdependencies that could cause deadlock.

### Buffer-induced deadlock

Есть и другой тип: **buffer-induced deadlock**. Здесь проблема не в handshake rule itself, а в том, что cycle in the graph замыкается через ограниченные очереди. AMD в Vitis tutorial прямо пишет, что **poorly sized dataflow channels can cause loss of performance and/or deadlock**. Хотя этот пример относится к DATAFLOW flows, сама архитектурная идея полностью применима и к RTL: если несколько processes или blocks связаны FIFO-like channels в цикле, недостаточная глубина буферов может сделать систему неспособной продвинуть хотя бы один token, который разорвал бы circular wait.

В обычном RTL это выглядит так: block A не читает, пока не запишет в свой output buffer; block B не читает, пока не запишет в свой; оба output buffers full; ни один block не освобождает вход, потому что ждет возможность выгрузить выход. Формально это уже не simple backpressure, а настоящий deadlock by bounded storage. Practical lesson отсюда такой: **все cycles in dataflow graph требуют отдельной buffer analysis**, а не только проверки локального `ready/valid`.

### Reset-related deadlock and hang

Очень неприятный класс проблем — **reset-related deadlock** или lock-up after reset. AMD для AXI Interconnect прямо пишет, что infrastructure cores **do not support partial resetting**: whenever one interface is reset, all interfaces must be reset, resets must overlap, and none of the connected cores may reset one end of an AXI interface connection without the other end overlapping that reset cycle. Во время reset these cores deassert all `valid` and `ready` outputs.

Практически это означает, что “локальный reset одной стороны канала” может создать system state, в котором одна часть fabric уже снова ждет handshake, а другая еще не вернулась в согласованное состояние. На waveform это может выглядеть как бесконечный wait, хотя корень проблемы вообще не в functional datapath, а в неправильной **reset overlap policy**. Для FPGA RTL это очень важный вывод: deadlock scenarios надо проверять не только в steady-state, но и на reset entry/exit.

### Clock-related pseudo-deadlock

Иногда внешне deadlock похож на handshake bug, а реальный источник — неверная **clock relationship**. AMD в PG059 пишет, что synchronous clock conversion mode требует, чтобы SI and MI clocks remained edge-aligned at all times; если их active edges drift apart, можно увидеть malfunction, including dropped transfers or lock-up. AMD even recommends reconfiguring the clock conversion core to asynchronous conversion to isolate this issue.

Это важная practical граница: не каждый “hang” является логическим deadlock в strict sense. Иногда это **clocking-induced lock-up**, который снаружи выглядит как permanent stall. Но для RTL engineer effect тот же: no forward progress. Поэтому в реальном debug полезно держать в голове отдельный вопрос: deadlock вызван circular wait logic, или interface state machine просто зашел в broken state из-за CDC/clock-converter assumptions.

### Protocol-violation deadlock

Есть сценарии, где design зависает не потому, что circular wait был задуман архитектурой, а потому что кто-то нарушил protocol contract. AMD пишет, что custom IP violating AXI protocol rules часто вызывает apparent lock-up connected core; AXI Interconnect and Crossbar are especially vulnerable to such violations. Отдельно в AXI Firewall IP AMD описывает, что firewall нужен для защиты одной части AXI network от issues caused by the opposite network, such as **protocol violations or timeout hangs**; when failure is detected, firewall blocks problematic traffic and can terminate outstanding transfers toward the protected side.

Это очень полезно для design thinking. Иногда engineer анализирует deadlock как чисто internal property своего блока, а на самом деле причина — внешняя non-compliant endpoint logic. Поэтому в больших Vivado systems **protocol checker / firewall / timeout monitoring** — это не роскошь, а практическая защита от “необъяснимых” hangs, которые локально выглядят как deadlock in your RTL. AMD прямо рекомендует использовать protocol checker before deploying custom IP or modified IP.

### Deadlock через ordering assumptions

Еще один subtle scenario связан с ordering. AMD AXI Crossbar в разделе error signaling указывает, что certain bad response-ID situations permanently block the entire response by the Crossbar, and this can cause the problematic slave and any master expecting the response to hang. То есть если ordering/ID mapping assumptions нарушены, deadlock-like hang может быть вызван уже не backpressure как таковым, а тем, что fabric больше не может legally route completion to the waiting requester.

Для обычного RTL lesson здесь такой: **если system relies on in-order completion, ID ownership or packet return mapping, эти правила должны быть explicit и легко проверяемы**. Deadlock часто появляется не в data motion, а в completion path: request ушел, а legal way вернуть response уже потерян. В streaming systems это бывает похоже на case, когда packet descriptor ушел вперед, payload stalled, а downstream side ждет metadata completion, которая сама зависит от освобождения stalled path. В memory-mapped AXI такие вещи formalized сильнее, но architectural pattern общий.

### Как распознавать deadlock scenario в архитектуре заранее

Есть несколько очень сильных warning signs.

Первый — **mutual wait symmetry**: и upstream, и downstream side имеют условие “я начну только после тебя”. Arm AXI rules специально ломают эту симметрию в пользу source-driven `VALID`.

Второй — **cross-channel dependency without clear master rule**: address path ждет data path, data path ждет address path, response path ждет completion elsewhere. Arm write-channel dependency rules как раз существуют против такого класса ошибок.

Третий — **cycles with bounded buffers**: если topology содержит loop, а его разрыв зависит от FIFO space, buffer depth должен анализироваться как часть correctness, не только performance. AMD DATAFLOW tutorial прямо указывает, что poorly sized channels can cause deadlock.

Четвертый — **partial reset or mismatched reset release** на интерфейсах. AMD запрещает partial resetting AXI Interconnect connections.

Пятый — **misaligned clock assumptions** in synchronous clock-conversion paths. AMD указывает, что clock-edge drift can cause lock-up.

### Как practically защищаться

Самая базовая защита — соблюдать handshake asymmetry: source-driven `VALID`, stable data/control until handshake, no circular dependence of source on destination `READY`. Это фундамент из Arm AXI spec.

Следующий уровень — intentionally break cycles. Там, где есть feedback through storage, полезно явно проектировать **FIFO depth**, ownership rules и one-way progress conditions. На AXI interconnect level AMD делает это через **Single Slave per ID**, то есть вводит architectural restriction, которая сознательно убирает dangerous concurrency pattern. Это очень хороший пример общего принципа: иногда deadlock предотвращается не “умной FSM”, а правильно выбранным ограничением на допустимое поведение системы.

Еще один practical слой — recovery and isolation. Там, где protocol violations or timeout hangs возможны из-за внешних IP, полезны **protocol checkers**, firewall-like isolation и timeout-based containment. AMD AXI Firewall как раз существует, чтобы problematic traffic не рушил весь protected side and to restore normal operation after separate reset of the monitored portion.

### Что проверять в simulation и debug

Хорошая проверка **deadlock scenarios** почти никогда не ограничивается “прошел обычный testbench”. Полезно специально проверять:

- source, который долго держит `valid` under backpressure;
- destination, который поднимает `ready` only after seeing `valid`;
- cross-channel skew between address and data;
- bounded-buffer cycles under worst-case occupancy;
- reset overlap and staggered release;
- clock-conversion assumptions;
- protocol checker assertions on handshake/order rules.

Для system-level AXI designs AMD дополнительно советует использовать protocol checker IP before deploying custom or modified IP, а при unexplained lock-up смотреть именно на symptoms, где handshake seems complete on one side but not issued on the other. Это хороший debug heuristic: искать не только “какой signal залип”, а **какая dependency never becomes satisfiable**.

### Типичные ошибки

Самая частая ошибка — считать, что deadlock возможен только в “сложных interconnect systems”. На практике даже один ready/valid stage pair может deadlock’нуть, если source mistakenly waits for `ready`. Arm spec прямо запрещает это.

Вторая ошибка — лечить deadlock только добавлением buffer, не понимая dependency graph. Buffer может помочь, а может просто отложить hang на несколько cycles или сделать его workload-dependent. AMD tutorial про FIFO sizing полезен именно тем, что показывает: **depth matters**, but only in the context of actual process interaction.

Третья ошибка — забывать про reset and clock assumptions. В реальном Vivado system apparent deadlock нередко вызван violated reset-overlap rule или wrong synchronous clock-converter assumption, а не “логикой потока” в чистом виде.

Четвертая ошибка — доверять внешнему IP без protocol checking. AMD прямо предупреждает, что custom IP protocol violations often lead to lock-up symptoms in AXI Interconnect.

### Практический итог

**Deadlock scenarios** — это полноценная подтема внутри блока **Backpressure propagation**, потому что она отвечает не на вопрос “есть ли stall”, а на более глубокий вопрос: **может ли система вообще восстановить движение, если несколько условий ожидания замыкаются в круг.** Arm AXI spec задает базовые dependency rules, которые убирают локальные handshake deadlocks; AMD PG059 показывает, как deadlock возникает на уровне interconnect and ordering, и как architectural restriction вроде **Single Slave per ID** его предотвращает; AMD debug/firewall materials добавляют practical layer про protocol violations, timeout hangs, reset overlap и containment.

Если сказать совсем коротко: **deadlock** в RTL — это почти всегда не “плохой stall”, а **замкнутая зависимость без механизма first progress**. И чем раньше engineer начнет рисовать не только datapath, но и граф ожиданий между `valid`, `ready`, buffers, IDs, resets и clocks, тем меньше шанс, что design будет выглядеть корректно в локальных waveforms и зависать на настоящем железе.