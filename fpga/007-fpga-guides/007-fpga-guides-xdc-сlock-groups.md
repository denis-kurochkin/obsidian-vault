## Clock groups

**`set_clock_groups`** в контексте FPGA/Vivado и блока **Constraints writing (XDC/SDC)** — это команда, которая **отключает timing analysis между группами clocks**, которые ты явно укажешь, но **не отключает timing внутри одной и той же группы**. По умолчанию Vivado пытается таймить взаимодействие **между всеми clocks** в проекте, пока ты явно не скажешь обратное через `set_clock_groups` или через другие timing exceptions. Важная practical разница с `set_false_path` такая: `set_clock_groups` действует **в обе стороны** между группами clocks, а `set_false_path` обычно задают направленно через `-from` и `-to`.

Главная мысль здесь такая:  
**clock groups нужны не для “красивого XDC”, а для правильной модели отношений между clock domains.** Если два clocks реально asynchronous, mutually exclusive или вообще не должны анализироваться друг против друга, Vivado без `set_clock_groups` будет продолжать считать их timed interaction, а это часто приводит к ложным inter-clock violations, странным small requirements и misleading timing picture. AMD прямо рекомендует анализировать отношения между каждым master clock и связанными generated clocks и помнить, что в Vivado каждое clock interaction timed, пока оно не объявлено asynchronous или false path.

### Почему это отдельная полноценная подтема

Внутри блока **Constraints writing (XDC/SDC)** тема **clock groups** выделяется отдельно потому, что она задает не просто один path exception, а **глобальную политику взаимодействия clock domains**. Ошибка в `set_clock_groups` редко остается локальной: она либо оставляет лишние timed paths между несвязанными clocks, либо, наоборот, слишком широко выключает timing analysis там, где часть путей все еще нужно анализировать. UG903 прямо пишет, что `set_clock_groups -asynchronous` имеет **более высокий приоритет**, чем обычные timing exceptions; если тебе нужно продолжать constrain’ить и report’ить часть путей между asynchronous clocks, использовать надо **timing exceptions only**, а **не** `set_clock_groups`.

Проще говоря, `set_clock_groups` — это **сильная** команда. Она удобна, когда clock relationship действительно можно описать целиком на уровне групп. Но она опасна, если ее применяют “на всякий случай”, не разобрав topology crossings и не проверив, остались ли пути, которые все еще должны быть видимы timing engine. Именно поэтому UG903 выделяет для clock groups отдельный раздел с разбиением на **asynchronous**, **logically exclusive** и **physically exclusive** cases.

### Что именно делает set_clock_groups

В practical sense `set_clock_groups` говорит Vivado:  
**“между этими clock groups timing analysis выполнять не нужно”**.  
При этом внутри каждой группы clocks продолжают анализироваться normally. Это особенно важно понимать в многодоменных системах: команда не “убивает все timing”, а только cross-group interaction. UG903 формулирует это буквально: command disables timing analysis between groups of clocks you identify, and not between the clocks within the same group.

Из этого следует полезное инженерное правило:  
если у тебя есть несколько clocks, и часть из них synchronous/related, а часть — нет, группировать их нужно так, чтобы **внутригрупповые отношения оставались meaningful**, а **межгрупповые** отключались только там, где это реально justified. Это не просто синтаксис `-group`, а способ формально выразить clock architecture проекта.

### Три основных типа clock groups

В Vivado у `set_clock_groups` обычно используют три главных режима:

- **`-asynchronous`**
- **`-logically_exclusive`**
- **`-physically_exclusive`**

Эти варианты перечислены и в UG903, и в списке supported SDC commands. Они описывают **разные причины**, почему timing между clock groups надо выключить.

Это очень важно, потому что одна из частых ошибок — использовать `-asynchronous` для любого неудобного inter-clock case. На самом деле exclusive clocks и asynchronous clocks — это не одно и то же. Если выбрать неправильную категорию, tool получит неверную architectural story о проекте.

### Asynchronous clock groups

**Asynchronous clock groups** используют тогда, когда clocks **не имеют безопасного временного отношения** и paths между ними нужно игнорировать в timing analysis. UG903 прямо пишет, что **asynchronous clocks and unexpandable clocks cannot be safely timed**, и timing paths between them can be ignored using `set_clock_groups`.

Это самый частый practical case: два реально независимых clock domains, между которыми есть CDC logic — например, **two-flop synchronizer**, async FIFO или другой осознанный crossing mechanism. UG903 отдельно говорит, что для asynchronous signals paths between two asynchronous clock domains can be disabled with `set_clock_groups` (recommended) or `set_false_path` (not recommended), **при условии**, что inter-clock domains already properly designed with synchronizer or FIFO. И там же AMD добавляет важную оговорку: даже если timing analysis между доменами отключен, path delay между ними **не должен быть unnecessarily high**.

Из этого следует важный practical вывод:  
**`set_clock_groups -asynchronous` не заменяет CDC design.**  
Сначала ты должен правильно построить crossing, а уже потом отключать несмысловой inter-clock timing. Если CDC logic unsafe, то `set_clock_groups` только скроет проблему из обычного timing analysis, но не исправит архитектуру.

### Когда wizard рекомендует asynchronous clock groups

UG903 отдельно описывает, когда **Timing Constraints Wizard** рекомендует `set_clock_groups -asynchronous`. Это происходит, когда:

- **all paths have synchronizers in both directions**;
- **no path is covered by `set_max_delay -datapath_only` in either direction**, потому что `set_clock_groups` имеет более высокий precedence и override’ит existing `set_max_delay`.

Это очень полезный критерий. Он показывает, что Vivado сам считает `set_clock_groups -asynchronous` уместным только там, где CDC topology уже выглядит “полностью закрытой” синхронизаторами и где не требуется сохранять отдельные path-level constraints. То есть wizard не рекомендует эту команду безусловно для любого clock pair, который просто кажется asynchronous.

### Почему set_clock_groups сильнее обычных timing exceptions

UG903 и UG949 отдельно подчеркивают, что `set_clock_groups` имеет **higher precedence** than timing exceptions. UG949 формулирует это очень прямо: command is not considered a timing exception even though it is equivalent to two `set_false_path` commands between two clocks; it has **higher precedence** than timing exceptions. UG903 отдельно предупреждает, что `set_clock_groups -asynchronous` override’ит regular exceptions.

Практически это означает очень важную вещь:  
если ты поставил `set_clock_groups -asynchronous` между двумя clocks, а потом пытаешься **локально** constrained/report’ить часть путей между ними, это, скорее всего, уже не сработает так, как ты ожидаешь. Именно поэтому для mixed cases, где часть CDC paths должна быть видна и контролируема, AMD советует использовать **timing exceptions only**, а не clock groups.

### Логически exclusive clock groups

**Logically exclusive clocks** — это clocks, которые определены **на разных source points**, но share part of their clock tree because of a **multiplexer** or other combinational logic. UG903 прямо так и определяет logically exclusive clock groups.

Это очень полезный case для FPGA design, потому что внутри clock mux structure Vivado может видеть несколько clocks на одном shared tree, хотя в реальном hardware одновременно активен только один mode. И вот здесь `-logically_exclusive` нужен не потому, что clocks asynchronous, а потому, что они **mode-exclusive** по смыслу логики.

UG903 различает два варианта logically exclusive clocks:

- **with no interaction**
- **with interaction**

### Logically exclusive clocks with no interaction

Если logically exclusive clocks **не имеют timing paths between each other** нигде, кроме shared clock tree logic, Timing Constraints Wizard рекомендует ставить clock groups **directly on them**. Это описано в UG903 для case “Logically Exclusive Clock Groups with No Interaction”.

Это удобный и чистый вариант. По сути, ты говоришь Vivado: “эти clocks делят часть clock tree, но между их functional domains paths анализировать не нужно”. Такой case обычно проще всего описывается на уровне самих clocks, без дополнительных clock copies.

### Logically exclusive clocks with interaction

Гораздо более интересный и опасный case — **logically exclusive clocks with interaction**. UG903 пишет, что Timing Constraints Wizard identifies logically exclusive clocks that have timing paths between each other elsewhere than just on the logic connected to the shared clock tree. В этом случае прямое применение clock groups к исходным clocks может быть слишком грубым.

Именно поэтому UG903 рекомендует более тонкий прием: создать **generated clocks that are copies** of `clkA` and `clkB`, but that exist only on the shared clock tree, и применить `set_clock_groups` уже **к этим copies**, чтобы paths **outside** shared clock tree still remained normally timed. Это один из самых сильных practical lessons всей темы: иногда clock groups надо ставить **не на исходные clocks**, а на специально выделенные local clock objects.

Это критически важно, потому что показывает:  
**clock groups — это не только команда “отключить все между A и B”, а инструмент аккуратного моделирования конкретной части clock topology.**  
Если применить ее слишком широко, можно потерять полезный timing analysis там, где clocks все еще interact elsewhere in the design.

### Physically exclusive clock groups

**Physically exclusive clocks** — это clocks, которые defined on the **same source point** and propagate on the **same clock tree**. UG903 приводит именно такое определение. Типичный пример — несколько `create_clock` на одном и том же input port для mutually exclusive external modes.

Здесь логика очень простая: hardware physically не может одновременно иметь оба clocks active on that same source/tree, хотя Vivado способен propagates multiple timing clocks on the same tree for multi-mode reporting convenience. Поэтому `-physically_exclusive` используется, чтобы объяснить tool’у, что эти clock definitions mutually exclusive **по физике**, а не просто по логике mux behavior.

### Clock groups и CDC constraints

UG903 прямо связывает тему **clock groups** с **CDC constraints**. В разделе About CDC Constraints AMD пишет, что CDC paths are timing paths with different launch and capture clocks, и что paths between synchronous clocks, but covered by false path constraints, are consequently treated as asynchronous CDCs. Wizard analyzes CDC topology and recommends clock groups or false path constraints whenever it is safe to do so.

Практически это значит:  
если между двумя clocks реально есть CDC logic, вопрос не только в том, timed ли path, но и в том, **как Vivado классифицирует crossing**. Слишком грубое clock grouping может скрыть часть paths из обычного timing analysis, но CDC reasoning при этом все равно остается архитектурно важным. Поэтому хорошие clock groups обычно идут рука об руку с нормальным CDC design: synchronizers, `ASYNC_REG`, async FIFOs и понятная crossing topology.

### Почему set_clock_groups и set_false_path — не одно и то же

UG903 отдельно пишет, что `set_false_path` disables timing only in the **specified direction** using `-from`/`-to`, а `set_clock_groups` disables timing analysis **between groups** in both directions. Это ключевое различие.

На практике это означает:

- если тебе нужно выключить **все** междоменные пути между двумя группами clocks — `set_clock_groups` often cleaner;
- если тебе нужно выключить только **конкретные directions** или только **часть paths** — чаще подходит `set_false_path` или более локальные exceptions.

Именно поэтому AMD в UG903 прямо пишет: если нужно constrain and report **some paths** between asynchronous clocks, надо использовать **timing exceptions only**, not `set_clock_groups`. То есть `set_clock_groups` хорош там, где domain relationship действительно можно описать целиком на уровне clocks, а не на уровне отдельных path subsets.

### Как проверять clock groups после записи

После написания `set_clock_groups` его нужно **обязательно проверять**. Для этого Vivado дает **Report Clock Interaction**. UG906 прямо говорит, что его можно открыть из Reports > Timing > Report Clock Interaction или вызвать Tcl-командой `report_clock_interaction -name clocks_1`. UG945 также подчеркивает, что after or during constraints creation you must verify that constraints are complete and safe, и напоминает, что clocks timed together by default unless otherwise specified by clock groups or other exceptions.

Это очень важный practical шаг. Правильный workflow обычно такой:

1. define all primary/generated clocks;
2. посмотреть **clock interaction** picture;
3. добавить `set_clock_groups` там, где отношения действительно asynchronous or exclusive;
4. снова открыть **Report Clock Interaction** и проверить, что закрыты именно те pairs, которые должны быть закрыты.

UG906 также описывает **matrix color coding** для clock interaction report и прямо говорит, что user-defined false path or clock group constraints cover all paths crossing from source clock to destination clock. Это очень удобно для визуальной проверки того, что именно выключено твоими constraints.

### Clock interaction report как основной инструмент проверки

Clock Interaction Report полезен еще и потому, что он показывает не только “timed or not”, но и помогает увидеть **подозрительные clock pair relationships**. UG906 отдельно описывает **Clock Pair Classification**, где есть такие категории, как missing common primary clock, missing common node, missing common phase, missing common period и presence of a virtual clock. Это очень помогает понять, действительно ли clocks unrelated, или проблема в том, что clock relationships просто описаны неполно.

Практический вывод здесь очень простой:  
**перед тем как ставить `set_clock_groups`, полезно сначала убедиться, что clock pair действительно должна быть asynchronous/exclusive, а не просто плохо описана через clocks/generated clocks.**  
Именно для этого сочетание `report_clocks` + `report_clock_interaction` обычно гораздо полезнее, чем сразу писать исключение “по ощущениям”.

### Когда set_clock_groups лучше не использовать

UG903 прямо предупреждает о ситуации, когда `set_clock_groups -asynchronous` лучше **не** применять: если нужно constrain and report some paths between asynchronous clocks. Также wizard рекомендует asynchronous clock groups only if **all** paths in both directions already go through synchronizers and no path needs `set_max_delay -datapath_only`.

Это означает, что `set_clock_groups` плохо подходит для mixed cases, где:

- только часть crossings safe to ignore;
- часть CDC paths still needs datapath limit;
- существуют meaningful inter-domain timing checks outside synchronizers;
- хочется сохранить visibility конкретных paths в reports.

В таких ситуациях лучше работать через **path-level constraints**, а не выключать analysis целиком между clock groups. Иначе легко получить слишком “чистую” timing картину, в которой важные crossings просто исчезли из анализа.

### Типичные ошибки

Самая частая ошибка — ставить `set_clock_groups -asynchronous` **слишком рано**, просто потому что inter-clock requirement выглядит маленьким или path неудобно закрывать. UG906 прямо отмечает, что paths with very small setup requirements often indicate missing timing exceptions or incorrect clock relationships. То есть сначала надо понять relationship clocks, а уже потом решать, asynchronous ли они на самом деле.

Вторая ошибка — путать **asynchronous** и **exclusive** cases. Если clocks mutually exclusive по mux/mode logic или physically share same source point, это уже не просто asynchronous domains, и правильнее использовать `-logically_exclusive` или `-physically_exclusive`.

Третья ошибка — применять clock groups к исходным clocks в case logically exclusive clocks **with interaction**, где нужно было создать local copies on shared clock tree и группировать уже их. UG903 специально приводит такой recommended pattern.

Четвертая ошибка — забывать, что `set_clock_groups` override’ит более локальные exceptions. Из-за этого можно случайно потерять `set_max_delay -datapath_only` или другие useful constraints на CDC paths.

Пятая ошибка — не проверять результат через **Report Clock Interaction**. В таком случае constraint может быть syntactically valid, но покрывать больше clock pairs, чем ты реально хотел.

### Практический итог

**Clock groups** — это полноценная подтема внутри блока **Constraints writing (XDC/SDC)**, потому что она отвечает не на вопрос “как выключить неудобный timing”, а на более важный вопрос: **какие отношения между clock domains в проекте реально существуют и как их формально сообщить Vivado**. Official AMD guidance показывает, что `set_clock_groups` disables timing analysis between clock groups, поддерживает asynchronous/logically exclusive/physically exclusive cases, имеет higher precedence than timing exceptions и должен проверяться через Clock Interaction Report.

Если сказать совсем коротко: **`set_clock_groups` — это инструмент для описания архитектуры clock domains, а не просто способ убрать violations из отчета.** Когда он применен правильно, timing model становится чище и честнее. Когда слишком широко или слишком рано — проект легко теряет полезный timing visibility именно там, где она еще была нужна.