## Constraint injection

**Constraint injection** в контексте FPGA/Vivado — это не официальный термин, а удобное инженерное название для всей темы, где constraints не просто “лежат в XDC”, а **вводятся в design flow** в конкретный момент, в конкретном порядке и в конкретном scope. Идея здесь такая: важно не только **что** ты написал в constraint, но и **куда**, **когда** и **от чьего имени** этот constraint попадает в проект. В Vivado constraints могут входить через XDC files, `read_xdc`, unmanaged Tcl scripts, IP-level XDC, OOC constraints и interactive edits в открытом netlist. UG903 отдельно отмечает, что XDC можно читать через `read_xdc` или через constraints set, а unmanaged Tcl scripts допускают полноценные Tcl конструкции и могут смешиваться с XDC в одном наборе constraints.

### Почему это отдельная тема

Когда engineer впервые работает с Vivado, constraints часто воспринимаются как статическое приложение к RTL: есть `top.xdc`, в нем clocks, pins и несколько timing exceptions. Но в реальном project flow constraints почти всегда приходят из нескольких источников сразу: user XDC, IP output products, block design, OOC modules, Tcl automation и wizard-generated additions. В этот момент становится важным уже не просто “есть ли constraint”, а **как он injected into the design** и как он взаимодействует с другими constraints. Именно поэтому constraint injection полезно выделять в отдельную подтему: она про архитектуру constraint flow, а не только про синтаксис команд.

### Смысл темы в одной фразе

Если сказать коротко, то **constraint injection** — это дисциплина, по которой constraints попадают в design **осознанно, последовательно и локально там, где это оправдано**. То есть engineer управляет не только содержимым XDC, но и источником constraint, processing order, stage of the flow, scope и границей между top-level и submodule/IP constraints. Это уже уровень system thinking, а не просто Tcl editing.

### Откуда constraints вообще попадают в Vivado

Vivado допускает несколько способов ввода constraints. Базовый вариант — обычные **XDC files**, которые читаются через `read_xdc` или добавляются в project constraints set. Кроме них можно использовать **unmanaged Tcl scripts**: в отличие от XDC, они могут содержать полноценный Tcl control flow, conditions и loops. При этом UG903 подчеркивает, что XDC files принимают только `set`, `list` и `expr` из built-in Tcl, а unmanaged Tcl scripts не управляются Vivado так же, как обычные XDC edits; кроме того, constraints, сгенерированные таким script-based способом, не сохраняются обратно как managed XDC content.

Это уже первый важный урок по теме injection: **форма ввода constraints влияет на их жизненный цикл**. XDC удобно для controlled project constraints. Unmanaged Tcl удобно для generated or conditional constraints, automation и специальных flows. Но как только в design смешиваются оба способа, engineer уже обязан понимать, как именно tool читает, хранит и повторно применяет эти ограничения.

### Почему порядок применения важен

Одна из самых важных идей здесь — **XDC is sequential**. UG949 прямо говорит, что XDC следует Tcl-style interpretation rules: переменные должны быть определены до использования, clocks должны быть созданы до constraints, которые на них ссылаются, а для эквивалентных constraints одинакового precedence действует правило **the last one applies**. Если один path покрывается несколькими timing exceptions, вступает в силу уже не просто порядок чтения, а **precedence model**.

Это означает, что constraint injection — это всегда еще и **ordering problem**. Недостаточно просто иметь все нужные строки somewhere in the project. Нужно, чтобы они были введены в правильной последовательности. Например, `create_clock` должен появиться раньше `set_input_delay`, `set_output_delay`, `set_clock_groups` или timing exceptions, которые используют этот clock object. А overlapping exceptions нужно писать так, чтобы не возникала зависимость от неочевидных priority rules.

### Project mode и Non-Project mode

В **Project Mode** порядок XDC/Tcl files в constraints set определяет порядок чтения: файл сверху читается раньше, файл снизу — позже. Порядок можно менять прямо в IDE или Tcl-командой `reorder_files`. В **Non-Project Mode** аналогичную роль играет последовательность вызовов `read_xdc`. Это не деталь интерфейса, а часть timing model, потому что именно этот порядок определяет, какие constraints уже существуют к моменту чтения следующего файла.

Отсюда следует очень practical вывод: многие проблемы вида “почему constraint не сработал” на самом деле оказываются не syntax issue, а issue of **read order**. Файл с clock definition прочитался позже файла с delays. Более общий exception пришел после более локального. Или новый wizard-generated constraint был добавлен в конец sequence и случайно изменил behavior уже существующей timing model.

### Constraint injection — это не только timing, но timing здесь главный

Vivado constraints включают не только timing, но и **physical / configuration constraints**. Однако именно timing constraints сильнее всего зависят от правильного injection order, потому что они строят модель времени step by step. UG949 рекомендует практический flow из четырех шагов: сначала **clock constraints**, потом **I/O delays**, потом **clock groups / CDC constraints**, и только после этого — **timing exceptions**. Там же сказано, что первый серьезный набор timing constraints для нового design лучше собирать уже на **post-synthesis netlist**, чтобы можно было нормально использовать timing reports и validation.

Это хороший инженерный принцип: **constraint injection должен идти по слоям модели времени**. Сначала ты описываешь clocks. Затем внешние интерфейсы. Затем relations между clock domains. И только потом делаешь exceptions. Когда этот порядок нарушается, timing model становится менее читаемой и менее надежной.

### Что значит “правильный уровень injection”

Одна из самых тонких частей темы — **scope**. Constraint можно ввести на top-level, на уровне IP, на уровне submodule или как OOC-specific constraint. И это не эквивалентные решения. Vivado использует **XDC scoping mechanism** для IP-level constraints: например, `get_ports` внутри IP XDC может быть автоматически преобразован к top-level port или к hierarchical pin, в зависимости от того, как IP соединен с верхним уровнем и где находятся I/O buffers. UG903 также отдельно отмечает, что некоторые constraints, например `set_input_delay`, `set_output_delay` и `IOSTANDARD`, могут применяться только к top-level ports, а не к hierarchical pins.

На практике это значит, что constraint injection должен быть **локальным по смыслу**. Constraints, описывающие interface или behavior конкретного IP, желательно держать ближе к IP. Но при этом нужно следить, чтобы такие constraints не “утекали” за границу модуля и не меняли top-level intent там, где этого не планировалось. Особенно это опасно для timing exceptions.

### Самая опасная зона — timing exceptions из IP/submodule

UG903 отдельно предупреждает, что timing exception, заданный в IP XDC, может иметь более высокий precedence и override top-level constraints, если написать его слишком глобально. Типичный пример — false path между двумя global clocks: внутри IP такая запись может выглядеть логично, но на уровне всего design она способна “отключить” timing analysis там, где эти же clocks должны анализироваться нормально. Поэтому AMD рекомендует для IP-level XDC использовать **point-to-point exceptions**, то есть constraints на локальные startpoints и endpoints внутри самого IP, а не broad global exceptions between clocks.

Это один из центральных принципов **constraint injection**:  
**inject exception as locally as possible**.  
Чем глобальнее exception, тем выше риск, что он изменит timing model там, где engineer этого не ожидал. Особенно опасны `set_clock_groups` и broad `set_false_path`, потому что они стоят высоко в precedence hierarchy и способны “погасить” целые области timing analysis.

### Precedence: где injection становится реально опасным

В Vivado timing exceptions имеют иерархию приоритета. Для overlapping cases порядок от более высокого к более низкому такой: `set_clock_groups`, затем `set_false_path`, затем `set_max_delay` / `set_min_delay`, затем `set_multicycle_path`. UG903 также уточняет, что внутри одного типа exception более специфичный constraint имеет более высокий precedence, чем более общий, а для некоторых конфликтов возможен `-reset_path`, но AMD рекомендует вообще избегать ситуаций, где один и тот же path покрывается несколькими разными exception types. Проверять реальный результат советуют через `report_exceptions`.

Отсюда вытекает важная мысль. **Constraint injection** — это не искусство “дописать еще одно исключение”, а искусство не создавать конфликтующие модели одного и того же path. Чем меньше overlapping exceptions, тем прозрачнее design intent и тем меньше вероятность, что через полгода никто уже не сможет объяснить, почему конкретный path перестал анализироваться.

### CDC как пример правильного и неправильного injection

Тема CDC очень хорошо показывает, что constraint injection нельзя сводить к одной favorite команде. Для asynchronous CDC paths Vivado допускает полное исключение timing analysis через `set_false_path` или `set_clock_groups`, а также частичный контроль через `set_max_delay -datapath_only`. Для Gray-coded buses и случаев, где нужно ограничить latency между asynchronous clocks, UG949 рекомендует именно `set_max_delay -datapath_only`; для multibit spread используется `set_bus_skew`.

Практически это означает следующее. Если engineer “по привычке” injected global `set_clock_groups -asynchronous` между двумя clocks, он может случайно убить полезный analysis там, где нужен был более локальный CDC constraint. То есть хороший injection в CDC — это не максимально широкий exception, а **минимально достаточный constraint**, соответствующий actual crossing structure.

### OOC и split constraints

Особый класс проблем появляется в **Out-Of-Context** flow. Когда модуль synthesized as OOC, top-level synthesis видит внутри него black box. Поэтому top-level synthesis constraints не могут ссылаться на internal pins, nets или cells этого OOC module. UG903 прямо рекомендует в таких случаях разделять constraints на две части: synthesis-only и implementation-only XDC, используя свойства `USED_IN_SYNTHESIS` и `USED_IN_IMPLEMENTATION`. После link на implementation stage это ограничение снимается, потому что DCP уже объединяется с top netlist и internal objects становятся разрешимыми.

Это важный пример того, что **constraint injection зависит от stage of the flow**. Один и тот же constraint может быть valid and useful в implementation, но invalid at synthesis. Поэтому зрелый Vivado flow почти всегда требует не “одного большого XDC на все случаи”, а осмысленного разделения по stage и visibility.

### Synthesis-only и implementation-only injection

Vivado по умолчанию применяет XDC files и Tcl scripts и к synthesis, и к implementation. Но это поведение можно изменить через `USED_IN_SYNTHESIS` и `USED_IN_IMPLEMENTATION`. Этот механизм нужен не только для OOC, но и вообще для clean constraints architecture: то, что полезно для synthesis object model, не всегда имеет смысл для implementation, и наоборот. UG903 отдельно добавляет еще один нюанс: `DONT_TOUCH`, заданный в synthesis XDC, не подчиняется этим свойствам и может пройти дальше в implementation независимо от `USED_IN_IMPLEMENTATION`.

Хорошая практика здесь такая: clocks, basic I/O timing и базовые architectural relations обычно должны быть consistent across the flow; constraints, которые адресуют objects, появляющиеся только post-synth или post-link, лучше отделять; physical fine-tuning и implementation-specific directives не стоит без необходимости смешивать с ранними synthesis constraints. Такой подход уменьшает число invalid or silently ineffective constraints и делает injection заметно более предсказуемым.

### Wizard, interactive edits и generated constraints

Отдельная важная зона — constraints, которые появляются не вручную, а через **Timing Constraints Wizard** или интерактивную работу в открытом design. UG949 рекомендует использовать Wizard для поиска missing constraints в первых трех шагах timing methodology. UG903 уточняет, что в Project Mode Wizard сохраняет новые рекомендации в target XDC file с processing order `NORMAL`, причем место этого файла в sequence критично; в Non-Project/DCP modes новые constraints добавляются в конец общей sequence, и если их нужно поднять выше из-за dependencies или precedence, sequence потом надо править вручную.

Это очень важный момент. **Generated constraints are not magically correct just because a wizard created them.** Они все равно входят в конкретную sequence и подчиняются тем же rules of ordering and precedence. Поэтому после любого wizard-based injection engineer должен проверить не только содержание, но и **место в sequence**.

### `source` против `read_xdc`

На первый взгляд это detail, но для темы injection он важен. UG903 отмечает различие между `source` и `read_xdc`: constraints, импортированные через `source`, не сохраняются в checkpoint в том же порядке, что при применении; constraints, импортированные через `read_xdc`, сохраняются первыми, а затем — импортированные через `source`. Если нужно сохранить sequence exactly as applied, AMD рекомендует `read_xdc -unmanaged` вместо `source`.

Этот нюанс хорошо показывает общий смысл темы. Constraint injection — это не только “прочитал файл”, а еще и вопрос **как этот файл живет дальше в checkpoint, rerun и debug flow**. Для reproducible projects такие seemingly small differences иногда оказываются принципиальными.

### Что считается хорошей архитектурой constraint injection

Хороший flow обычно выглядит так: сначала есть layering — clocks, затем I/O and interface timing, потом clock relations / CDC model, затем timing exceptions и только после этого physical refinements; затем есть ownership — top-level constraints описывают top-level intent, IP constraints остаются local, OOC constraints отделены, generated constraints либо коммитятся в controlled XDC, либо воспроизводимо пересоздаются scripts; и наконец есть validation — timing reports, review of invalid constraints, `report_exceptions` и проверка после synthesis, а не только после route. Такой подход напрямую соответствует рекомендациям UG949 по timing methodology и UG903 по ordering, scoping и precedence.

Очень полезно мысленно перестать думать “у меня есть пять XDC files” и начать думать: какие constraints inject primary timing model, какие inject interface assumptions, какие inject CDC policy, какие inject exceptions и какие inject physical intent. Такой взгляд обычно лучше обычного file-centric thinking. Один и тот же `constraints.xdc` может быть архитектурно плохим, если в нем вперемешку лежат clocks, asynchronous groups, late physical patches и temporary debug exceptions. И наоборот, несколько файлов могут быть очень здоровой структурой, если каждый отвечает за свой layer и читается в правильном порядке.

Еще один полезный взгляд на тему такой: constraints не стоит воспринимать как “последнюю заплатку перед implementation”. Гораздо надежнее думать о них как о части исходной архитектуры проекта. То есть вместе с RTL, clocking, CDC и reset strategy engineer заранее определяет, какая модель времени должна получиться у инструмента. Тогда constraints перестают быть реакцией на проблемы и становятся частью замысла системы. Именно в этом смысле хороший порядок ввода ограничений часто важнее, чем еще один локальный exception.

И тогда review таких файлов становится намного проще: видно, что действительно задает намерение проекта, а что появилось случайно и требует пересмотра.

### Типичные ошибки

У этой темы есть повторяющийся набор ошибок. Первая — считать, что constraint “существует”, значит он уже работает, хотя query может ничего не находить или object еще не создан. Вторая — писать слишком глобальные exceptions, особенно `set_clock_groups` и global `set_false_path`. Третья — смешивать top-level и IP-local constraints без явной scoping strategy. Четвертая — игнорировать difference between synthesis and implementation visibility, особенно в OOC flow. Пятая — blindly trust Wizard output, не проверяя порядок injection и real effect on reports. Шестая — латать timing violations новыми exceptions вместо проверки correctness самой timing model. Все эти ошибки прямо связаны с тем, как Vivado применяет constraints по order, scope, precedence и design stage.

### Практический итог

**Constraint injection** — это полноценная инженерная подтема Vivado flow. Она описывает, как constraints входят в design, в каком порядке, в каком scope и на каком этапе. Смысл темы в том, что **constraints — это не просто текстовые файлы, а активные правила, которые строят timing and implementation model design step by step**. Когда injection организован хорошо, tool видит именно ту timing model, которую engineer реально имеет в виду. Когда организован плохо, project получает hidden overrides, invalid queries, broad exceptions и очень трудную отладку timing behavior.