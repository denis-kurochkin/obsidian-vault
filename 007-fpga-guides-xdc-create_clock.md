**`create_clock`** в контексте FPGA/Vivado и блока **Constraints writing (XDC/SDC)** — это базовая timing-команда, которой задают **primary clock** или **virtual clock**. Для Vivado это одна из самых фундаментальных constraints-команд, потому что именно clock characteristics используются timing engine для расчета path requirements и последующего вычисления **slack**. В UG903 прямо сказано, что **primary clock** может быть определен только командой `create_clock`, а в разделе про supported SDC commands для `create_clock` перечислены ключевые опции `-period`, `-name`, `-waveform` и `-add`.

Главная мысль здесь такая:  
**`create_clock` — это точка, с которой для tool начинается понятие времени в проекте.** Пока clock не описан, timing engine не знает, какой period считать допустимым, где находятся clock edges и какие register-to-register paths вообще нужно анализировать. Именно поэтому UG903 относит `create_clock` к числу timing constraints, которые реально влияют на synthesis results, а UG949 советует проверять, что в design не осталось endpoints без clock, через `check_timing` и категорию `no_clock`.

### Почему это отдельная полноценная подтема

Внутри блока **Constraints writing (XDC/SDC)** команда `create_clock` выделяется отдельно потому, что почти все остальные timing constraints опираются на уже существующую clock model. Если ты не создал clock правильно, дальше начинают искажаться:

- timing requirements;
- `set_input_delay` / `set_output_delay`;
- clock interaction analysis;
- setup/hold reports;
- derived/generated clocks;
- часть wizard recommendations.

UG903 прямо показывает recommended sequence: сначала определяются clocks, а Timing Constraints Wizard анализирует netlist, clock connectivity и existing constraints, чтобы искать missing timing constraints.

Проще говоря: **ошибка в `create_clock` обычно не выглядит как одна маленькая неточность в XDC**. Она часто превращается в неправильную timing picture всего проекта. Поэтому тему полезно рассматривать не как “одну команду Tcl”, а как основу всей clock definition strategy.

### Что именно делает create_clock

На практическом уровне `create_clock` создает clock object с конкретными характеристиками: именем, period и waveform. Эти параметры становятся частью timing model Vivado. UG912 отдельно подчеркивает, что source point clock definition задает **time zero** и point of propagation, которые timing engine использует при вычислении delays. Это очень важный detail: clock определяется не только как “частота 100 MHz”, а как конкретная временная ссылка в design.

Именно поэтому два seemingly похожих constraints могут быть неэквивалентны, если clock определен в другом source point или с другой waveform. Для basic synchronous designs это часто не замечают, но в более сложных systems уже phase, duty cycle и source location начинают влиять на analysis более заметно. UG903 и UG912 как раз подчеркивают, что timing engine использует характеристики clock waveform и point of propagation, а не только число периода.

### Primary clock: где create_clock нужен в первую очередь

UG903 определяет **primary clock** как board clock, который входит в design через input port или через gigabit transceiver output pin, например recovered clock. Для такого clock именно `create_clock` является базовой командой определения. При этом AMD рекомендует задавать board clock **на input port**, а не на output clock buffer. В example для `sysclk` прямой рекомендуемый XDC такой:

create_clock -period 10 [get_ports sysclk]

В этом же примере UG903 поясняет, что board clock входит через port `sysclk`, затем проходит через input buffer и clock buffer, а определять его лучше именно на порту.

Это правило очень важно practically. Если у тебя есть внешний oscillator, differential board clock или системный входной clock, нормальная starting point для Vivado — именно top-level input port. Так tool строит естественную clock propagation model от реальной точки входа в chip, а не от какого-то внутреннего buffer node.

### Virtual clock: create_clock без source object

Есть второй важный режим использования `create_clock` — **virtual clock**. UG903 прямо пишет, что virtual clock создается командой `create_clock` **без указания source object**, и такой clock не привязан к netlist element внутри design. Обычно он нужен для `set_input_delay` и `set_output_delay`, когда внешний интерфейс должен быть описан через отдельную reference clock model, а не через реальный внутренний propagated clock.

Это означает, что `create_clock` — не только про реальные board clocks. Это еще и способ создать **timing reference outside the netlist**, когда нужно описать external timing environment. Для prototyping это особенно полезно на интерфейсах, где внешний device задает свои timing relations, а твоя FPGA должна только корректно встроиться в эту модель.

### Минимальный смысл period и waveform

В supported SDC syntax UG903 перечисляет для `create_clock` ключевые параметры `-period` и `-waveform`. Period задает длительность clock cycle, а waveform определяет положение edges внутри периода. Это важно не только для красивого описания clock, а потому что timing engine реально использует эти характеристики для setup/hold analysis. В общем разделе **About Clocks** UG903 говорит, что timing engine uses clock characteristics to compute path requirements and report timing margin by slack computation.

На практике это значит следующее. Если у тебя обычный 50% duty cycle без phase shift, часто достаточно только `-period`. Но как только duty cycle нетривиальный или у тебя есть specific edge relationship, waveform становится частью real timing intent. То есть `create_clock` нельзя воспринимать как “частота и все”. Даже в простом design tool строит timing from edges, not just from frequency.

### Почему define clock на правильном объекте так важно

UG912 отдельно подчеркивает, что source of primary clock defines time zero and point of propagation. Это делает выбор объекта у `create_clock` очень важным. Если external board clock defined on top-level port, timing engine видит естественную точку старта clock tree. Если designer случайно определяет его слишком поздно или на неподходящем object, модель propagation меняется. Именно поэтому UG903 explicitly recommends defining the board clock on the input port, not on the output of the clock buffer.

Для prototyping это особенно полезная дисциплина:  
**сначала правильно описать primary clocks на реальных точках входа**, и только потом заниматься generated clocks, IO delays и clock interaction rules. Такой порядок уменьшает вероятность, что потом придется переделывать всю clock model из-за одной неудачной стартовой точки.

### create_clock и generated clocks: где граница

Очень важно не путать `create_clock` и `create_generated_clock`. UG903 прямо разделяет эти две команды: primary clocks определяются `create_clock`, а generated clocks — `create_generated_clock`. В примере деления clock на 2 сначала создается primary clock на входе `clkin` через `create_clock`, а затем clock `clkdiv2` описывается уже как generated clock от pin `REGA/Q`.

Практически это значит, что `create_clock` не нужно использовать для каждого внутреннего derived clock, если он уже является функцией другого clock. Для таких случаев правильнее строить model через `create_generated_clock`, чтобы Vivado видел родительскую clock relationship. А `create_clock` остается командой для **первичных** или **виртуальных** временных reference points.

### Порядок применения create_clock в XDC

UG903 отдельно напоминает, что XDC commands interpreted sequentially, а для equivalent constraints **last constraint takes precedence**. В example с двумя `create_clock` на одном и том же input port второй clock override’ит первый, если не используется `-add`. Это одна из самых важных practical особенностей `create_clock`: не только сам текст команды важен, но и место, где она читается в sequence.

То есть если в проекте есть несколько XDC layers, IP-level constraints или generated wizard files, `create_clock` надо рассматривать как часть **constraint ordering problem**. Неправильный read order может привести не к syntax error, а к тихому override clock definition. Для prototype это опаснее, чем явная ошибка, потому что timing model становится другой, а project все еще “продолжает работать”.

### Опция -add: когда она нужна

Из того же UG903 example следует важная вещь: если несколько clock definitions прикрепляются к одному объекту, без `-add` более поздний `create_clock` override’ит предыдущий. Значит, `-add` нужен в тех случаях, когда designer осознанно хочет иметь несколько clocks на одном source object, а не заменить старое определение новым. Supported SDC syntax for `create_clock` прямо включает опцию `-add`.

Практически это не самый частый case для beginner flow, но очень важный для зрелого XDC writing:  
**не каждый повторный create_clock — это harmless duplicate.**  
Часто это либо accidental override, либо intended multi-clock attachment, и tool будет вести себя по-разному в зависимости от использования `-add`.

### create_clock и synthesis

UG903 прямо указывает, что из timing constraints именно `create_clock`, `create_generated_clock`, `set_input_delay`, `set_output_delay`, `set_clock_groups`, `set_false_path`, `set_max_delay` и `set_multicycle_path` имеют real impact on synthesis results. Это делает `create_clock` не просто post-synthesis analysis aid, а активной частью early flow.

Для prototyping это очень полезно понимать. Иногда engineer думает, что clock constraints нужны “только для implementation”. Для Vivado это не так. Если primary clocks заданы корректно, synthesis уже получает более правильную timing context. Значит, `create_clock` — это одна из тех XDC команд, которые стоит писать не в конце, а практически в самом начале нормального project setup.

### Как проверять, что create_clock сработал правильно

Есть несколько базовых способов проверки. Во-первых, UG906 говорит, что **Clock Summary** section в Timing Summary Report показывает все clocks в design — созданные через `create_clock`, `create_generated_clock` и даже auto-derived clocks. Во-вторых, эта секция эквивалентна по смыслу тому, что показывает `report_clocks`. В-третьих, UG949 советует проверять, что в design нет unconstrained endpoints, через `check_timing` и категорию `no_clock`.

Практически правильный workflow такой:

- после добавления `create_clock` посмотреть `report_clocks` или Clock Summary;
- убедиться, что имя, period и waveform именно такие, как ожидалось;
- проверить `check_timing`, чтобы не осталось paths без clock coverage;
- затем уже переходить к input/output delays и дальнейшим constraints.

Это особенно важно, потому что clock definition может быть syntactically valid, но attached not where you thought, overridden later in XDC, или неполно покрывать design. Проверка после записи constraint здесь обязательна.

### Timing Constraints Wizard и create_clock

UG903 описывает **Timing Constraints Wizard** как инструмент, который identifies missing timing constraints on synthesized or implemented design. Он анализирует netlist, clock nets connectivity и existing timing constraints. Это делает wizard особенно полезным для темы `create_clock`, потому что missing or incorrect primary clocks — одна из самых базовых причин плохой timing model.

Но важно понимать границу: wizard помогает **находить missing pieces**, а не заменяет понимание clock architecture. Хороший `create_clock` почти всегда начинается не с wizard, а с осознанного knowledge о том, какие board clocks реально входят в design, где они входят и какова их intended timing role. Wizard полезен как validation tool, а не как единственный источник истины.

### create_clock и scope

UG903 в разделе про XDC scoping mechanism отдельно отмечает, что timing clocks, defined by `create_clock` or `create_generated_clock`, are visible throughout the design regardless of current instance setting. Это очень важный practical point: clocks — это глобальная timing сущность, а не локальный constraint одного маленького instance.

Это означает, что ошибки в `create_clock` тоже обычно глобальны по последствиям. Неверное clock definition редко остается локальной проблемой одного submodule. Оно меняет timing view всего проекта там, где этот clock propagates. Поэтому clock constraints требуют более аккуратного scoping thinking, чем многие локальные physical constraints.

### create_clock и GT/recovered clocks

UG903 определяет primary clock как board clock, entering through an input port or a gigabit transceiver output pin, for example a recovered clock. При этом есть device-specific note: для UltraScale и UltraScale+ timer automatically derives clocks on GT output ports, а явное определение primary clocks на GT output only needed for 7 series. Это важный нюанс именно для Vivado users, потому что GT-related clocking behavior depends on family.

Практически lesson такой:  
**не все recovered/GT clocks требуют одинакового create_clock treatment на разных device families.**  
Поэтому при работе с transceivers полезно всегда сверяться с конкретной family behavior, а не переносить blindly старый XDC шаблон между поколениями FPGA.

### Типичные ошибки при create_clock

Самая частая ошибка — задать `create_clock` не на том объекте. Для board clock AMD прямо рекомендует input port, а не output clock buffer. Если сделать наоборот, clock model start point уже смещается.

Вторая ошибка — путать primary и generated clocks. Если внутренний derived clock описан вторым `create_clock` вместо `create_generated_clock`, tool теряет часть parent-child timing relationship. UG903 прямо разделяет эти два класса clock definitions.

Третья ошибка — accidental override в XDC order. UG903 отдельно показывает, что equivalent clock definitions without `-add` resolve in favor of the last one. Это особенно опасно в multi-file projects.

Четвертая ошибка — написать `create_clock` и не проверить `report_clocks` / Clock Summary / `check_timing no_clock`. Тогда design может уже жить с неполной или неверной clock coverage, а engineer узнает об этом позже и в менее очевидной форме.

Пятая ошибка — считать, что `create_clock` нужен только для later implementation. UG903 прямо включает его в список constraints, которые реально влияют на synthesis results.

### Практический итог

**`create_clock`** — это полноценная подтема внутри блока **Constraints writing (XDC/SDC)**, потому что она отвечает не просто на вопрос “как задать частоту”, а на более важный вопрос: **как создать для Vivado правильную временную reference model проекта**. По официальным материалам AMD, `create_clock` задает primary или virtual clocks, определяет source point timing propagation, влияет на synthesis results, участвует в foundation для всех остальных timing constraints и должен проверяться через `report_clocks`, Clock Summary и `check_timing`.

Если сказать совсем коротко: **`create_clock` — это первая и самая базовая timing truth в XDC.** Если она задана правильно, дальше clocks, IO delays и timing analysis начинают складываться в coherent model. Если она задана плохо, почти весь дальнейший constraints flow оказывается построен на неверной временной основе.