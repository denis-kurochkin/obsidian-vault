## False/multicycle paths

**`set_false_path`** и **`set_multicycle_path`** в контексте FPGA/Vivado и блока **Constraints writing (XDC/SDC)** — это две разные timing-exception команды, которые часто путают, хотя смысл у них принципиально разный. `set_false_path` говорит Vivado, что определенный path **не должен анализироваться вообще**. `set_multicycle_path` говорит другое: path **должен анализироваться**, но для него требуется **больше одного clock cycle**. В UG903 это сформулировано напрямую: `set_false_path` indicates that a logic path should not be analyzed, а `set_multicycle_path` indicates the number of clock cycles required to propagate data from start to end of a path.

Главная мысль здесь такая:  
**false path — это про “не таймить”, multicycle path — это про “таймить иначе”.**  
Если path реально должен оставаться функционально проверяемым и оптимизируемым, но работает не каждый cycle, AMD рекомендует использовать именно **multicycle**, а не false path. UG903 прямо дает tip: use a Multicycle constraint in place of a False Path constraint when your intent is only to relax timing on a synchronous path, but the path still must be timed, verified, and optimized. UG949 при этом отдельно предупреждает, что false path обычно трудно доказать как действительно безопасный, и AMD generally does not recommend using it unless the associated risk is understood and accepted.

### Почему это отдельная полноценная подтема

Внутри блока **Constraints writing (XDC/SDC)** тема **false/multicycle paths** важна потому, что именно здесь engineer получает возможность не просто описать clocks, а **изменить саму timing model** для части design. Это уже не базовый constraint вроде `create_clock`, а осознанное исключение из default timing assumptions. UG903 относит `set_false_path` и `set_multicycle_path` к timing exceptions, а UG949 выделяет для них отдельные sections в timing-closure methodology. При этом UG903 также говорит, что обе команды входят в список timing constraints, которые реально влияют и на synthesis results, а не только на поздний implementation analysis.

Именно поэтому ошибка в этих constraints обычно намного опаснее, чем ошибка в “обычном” report-oriented Tcl. Неправильный false path может скрыть реальную functional timing problem. Неправильный multicycle может дать красивый slack на path, который на самом деле still must operate every cycle. UG949 отдельно приводит common mistakes for multicycle, а UG903 показывает, что false path and multicycle participate in a strict precedence model with other exceptions.

### Что такое false path в практическом смысле

**False path** — это path, который tool не должен включать в slack computation и ordinary timing analysis. UG949 формулирует это очень прямо: false path exceptions can be added to timing paths to **ignore slack computation** on these paths. При этом AMD сразу добавляет важную оговорку: usually it is difficult to prove that a path does not need timing to be functional, even with simulation tools.

Практически false path применяют тогда, когда path **не является functional synchronous timing requirement** для данного режима работы. Типичные примеры по смыслу methodology — asynchronous CDC paths after proper synchronization, mutually exclusive operational structures, configuration/status paths that are not part of active cycle-based timing behavior, или другие non-functional timing relationships. Но очень важно: false path — это не способ “убрать неудобный violation”. Это способ сказать: **этот path не должен участвовать в обычной timing модели вообще**. Именно поэтому AMD подчеркивает, что risk of using false paths needs proper assessment.

### Что такое multicycle path в практическом смысле

**Multicycle path** — это path, который остается timed, но получает requirement в **несколько cycles** вместо одного. UG949 формулирует базовое правило так: multicycle path constraints must reflect design functionality and must be applied on paths that do not have an active clock edge at every cycle, on either the source clock, the destination clock, or both clocks. UG835 уточняет, что `set_multicycle_path` specifies path multipliers for setup and hold analysis, with respect to the source clock or destination clock.

Это очень важная practical разница. При multicycle path tool по-прежнему:

- считает slack,
- проверяет path against requirement,
- видит path как часть active timing model,
- может оптимизировать его в соответствии с новым requirement.

То есть multicycle не выключает visibility path, а **перенастраивает expected timing relationship**. Именно поэтому UG903 рекомендует multicycle вместо false path, когда path still must be timed and verified.

### Когда выбирать false path, а когда multicycle

Это, пожалуй, главный practical вопрос всей темы. Самая полезная короткая формула такая:

- если path **не должен таймиться вообще** как обычный synchronous requirement — думай про **false path**;
- если path **должен таймиться**, но **не каждый cycle активен** — думай про **multicycle path**.

AMD буквально дает этот distinction в UG903: use multicycle when your intent is only to relax timing requirements on a synchronous path, but the path still must be timed, verified and optimized. UG949, со своей стороны, явно не любит ложные false paths без сильного обоснования.

Практически это означает следующее. Если у тебя есть properly synchronized CDC crossing, и обычный inter-clock timing model не имеет смысла, false path может быть уместен. Но если у тебя synchronous enable-driven path, который реально потребляет данные раз в два или раз в четыре cycles, то false path почти всегда слишком груб. Здесь правильнее multicycle, потому что path все еще functional and synchronous, просто slower in terms of active edge relationship.

### False path и CDC

UG903 прямо пишет, что timing analysis of CDC paths can be fully ignored by using `set_false_path` or `set_clock_groups`, or partially analyzed by using `set_max_delay -datapath_only`. Это очень важная граница. Для asynchronous CDC false path действительно может быть частью valid methodology, но AMD одновременно показывает, что это **не единственный** и не всегда лучший вариант. Если path delay between asynchronous domains still matters, например для multibit CDC coherence, AMD рекомендует заменять false-path-style ignoring на `set_max_delay -datapath_only`, а для spread между bits использовать `set_bus_skew`.

Отсюда следует один из самых важных practical выводов:  
**false path в CDC не равен “проблема решена”.**  
Он всего лишь выключает обычный timing analysis. Если для correctness still важны bounded datapath delay или bounded inter-bit skew, нужны другие constraints поверх или вместо false path. UG903 именно это и подчеркивает в разделе про constraining asynchronous signals.

### Почему false path считается опасной командой

UG949 прямо пишет, что false path difficult to prove and generally not recommended unless risk is acceptable. Причина простая: как только ты объявил path false, Vivado перестает считать для него обычный slack, и этот path исчезает из привычной timing visibility. Если path был помечен false incorrectly, ты можешь получить design, который “timing clean” в reports, но functional incorrect in hardware.

Именно поэтому хороший false-path workflow почти всегда требует ответить на очень строгий вопрос:  
**почему этот path действительно не нуждается в synchronous timing analysis для functional correctness?**  
Если на этот вопрос нельзя ответить коротко и уверенно, false path обычно подозрителен. UG949 фактически подталкивает к такой осторожности, а UG903 дополнительно напоминает, что false path — это полноценная timing exception, влияющая на downstream precedence model.

### Почему multicycle path тоже опасен, хотя он “мягче”

На первый взгляд multicycle кажется safer: path же все еще timed. Но у этой команды есть свой класс ошибок. UG949 прямо говорит, что multicycle path must reflect actual functionality and be applied only where path not active every cycle. Если это условие нарушено, constraint становится не relaxation of real design behavior, а просто искусственным ослаблением requirement.

Кроме того, multicycle имеет тонкую особенность: нужно правильно понимать, **что происходит с hold analysis**. UG949 в common mistakes explicitly warns against relaxing setup without adjusting hold back to same launch and capture edges when path not functionally active every cycle. Иначе hold requirement can become very high and impossible to meet. Это один из самых известных pitfalls темы.

### Setup и hold в multicycle path

UG835 описывает `set_multicycle_path` как command for setup and hold analysis path multipliers. Это значит, что multicycle affects **не только setup**, но и **hold picture**, и именно здесь начинаются многие ошибки. В Vivado multicycle обычно требует осознанной пары действий: relax setup by N cycles and then adjust hold accordingly, чтобы path still referenced correct launch/capture edge relationship. UG949 прямо выделяет раздел “Relaxing the Setup Requirement While Keeping Hold Unchanged” внутри multicycle methodology, а в common mistakes предупреждает, что без корректной hold adjustment можно получить impossible hold requirement.

Практический смысл здесь такой:  
**multicycle almost never should be typed from memory “одной строкой и готово”, если engineer не понимает resulting hold model.**  
В реальном flow после multicycle нужно обязательно смотреть timing reports и убедиться, что path requirement changed exactly as intended for both setup and hold.

### -start и -end: от какого clock считать cycles

UG949 и UG835 прямо говорят, что path multiplier in multicycle is expressed in clock cycles, either based on the **source clock** when `-start` is used, or the **destination clock** when `-end` is used. Это важный practical detail, особенно если source и destination clocks differ. В простом same-clock case engineer это иногда почти не замечает. Но как только clocks different or edge relationships nontrivial, `-start` versus `-end` становится частью real meaning of the exception.

То есть multicycle path — это не просто “число 2 или 3”. Это еще и вопрос: **относительно какого clock domain measured these additional cycles make sense**. И именно поэтому multicycle constraints особенно требуют path-level understanding, а не шаблонного копирования между проектами.

### Что tool делает с path после false path и после multicycle

Это полезно разделить очень явно.

После **false path**:

- обычный slack computation for that path ignored;
- path effectively removed from standard timing analysis of this kind;
- downstream reports may no longer show that path as ordinary failing candidate.

После **multicycle path**:

- path remains in timing model;
- requirement changes to multiple cycles;
- slack is still computed against new requirement;
- path can still be verified and optimized under that updated model.

Именно это distinction и делает тему столь важной. False path меняет **visibility** path. Multicycle меняет **expected timing budget** for a still-visible path.

### False/multicycle paths и synthesis

UG903 прямо включает и `set_false_path`, и `set_multicycle_path` в список timing constraints, которые have real impact on synthesis results. Это очень важно для prototyping, потому что многие думают об exception constraints как о purely implementation-stage tuning. Для Vivado это не так: если exception model задана рано, она already influences synthesis context.

Практический вывод:  
если false/multicycle path действительно часть архитектурного intent, лучше задать его корректно рано, а не “прикрутить потом после первых bad timing numbers”. Но это одновременно повышает цену ошибки: неверный exception может начать искажать not only reports, but also upstream optimization decisions.

### Precedence и почему exceptions могут конфликтовать

UG903 отдельно описывает **Exceptions Priority**. Там сказано, что priority between False Path, Max/Min Delay and Multicycle Path can be altered with `-reset_path`, а **clock group constraint cannot be overridden**. Также max/min delay or multicycle can override previously defined false path or max/min only when both constraints use exactly the same `-from/-to/-through` arguments and latest constraint uses `-reset_path`.

Это очень важная practical тема. Она означает, что timing exceptions — это не просто независимые строки XDC. Они образуют **precedence system**. Поэтому если в проекте уже есть `set_clock_groups`, потом path-level multicycle or false path may not work as expected. А если один и тот же path covered несколькими exception types, without explicit control you can easily end up with a different active constraint than you intended.

### Почему clock groups иногда лучше, а иногда хуже false path

UG903 и UG949 показывают четкое различие. Если нужно выключить timing **между entire clock groups в обе стороны**, `set_clock_groups` often cleaner. Если нужно выключить only specific direction or subset of paths, path-level `set_false_path` usually more precise. Но при этом UG903 подчеркивает, что в asynchronous CDC scenarios clock groups are often recommended over false path — until the moment you need partial analysis or `set_max_delay -datapath_only`, because `set_clock_groups` has higher precedence and can hide more than you want.

Отсюда очень полезный engineering principle:  
**use the weakest exception model that still accurately matches the design.**  
Если нужен path-level surgical exception, не обязательно выключать whole clock interaction. Если нужен complete domain-level ignore, path-level false paths may be unnecessarily verbose.

### Common mistakes для multicycle

UG949 в разделе **Common Mistakes** для multicycle прямо выделяет две особенно опасные ошибки.

Первая: **relaxing setup without adjusting hold back to same launch and capture edges**. Тогда hold requirement becomes very high and often impossible to meet. Вторая: **setting a multicycle path between incorrect points in the design**, because engineer assumes only one path exists from a startpoint cell to an endpoint cell, while endpoint may have multiple data input pins, including clock enable and reset pins, active on consecutive edges.

Эти две ошибки хорошо показывают общий характер multicycle path: здесь недостаточно “примерно знать, что data идет реже”. Нужно понимать exact startpoints, endpoints and functional gating semantics. Иначе constraint looks elegant but does not correspond to actual hardware behavior.

### Common mistakes для false path

Для false path UG949 подчеркивает другую проблему: **difficult to prove non-functionality**. То есть основная ошибка здесь — объявить path false, хотя он все еще функционально требует bounded timing. В CDC это особенно заметно: если ты скрываешь path false path’ом, а для correctness still нужен bounded datapath delay or skew, ordinary false path alone is insufficient. UG903 поэтому и указывает на `set_max_delay -datapath_only` and `set_bus_skew` as additional or alternative tools.

Практически это означает, что false path особенно опасен на:

- multibit CDC;
- interfaces with coherence requirements;
- paths that are not synchronous in the classic sense, but still have bounded delay requirements;
- cases where engineer uses false path just to silence reports.

### Когда path лучше constrain’ить не false/multicycle, а max/min delay

UG903 прямо показывает, что besides false path and multicycle, there are also `set_max_delay` and `set_min_delay`, which override default setup/hold requirements with user-specified values. UG949 additionally says that if you need to add extra timing margin between clocks, the safest overconstraint method is usually `set_clock_uncertainty`, not modifying clock edges. То есть false/multicycle — не единственные “advanced timing” tools, и иногда они просто не лучший fit.

Особенно это касается CDC datapath-only cases. UG903 прямо рекомендует `set_max_delay -datapath_only` for some asynchronous scenarios where you still want bounded delay rather than complete ignore. Это отличный пример того, как engineer sometimes reaches for false path too early, while the real requirement is “not synchronous setup/hold, but still bounded propagation”.

### Как проверять, что false/multicycle path сработал правильно

После добавления exception нельзя ограничиваться тем, что warning исчез. Нужно проверять **timing reports**. UG949 напоминает, что timing path report shows how slack is computed on any logical path for any timing check, а UG906/UG949 consistently position detailed timing reports as the place where you verify requirement, slack, path type and active exceptions. Если path объявлен false, нужно осознанно понимать, что он исчезнет из обычной slack-driven visibility. Если path multicycle, нужно открыть report and verify that requirement changed exactly as intended for setup and hold.

Практический workflow здесь обычно такой:

1. identify exact path and its functional intent;
2. add false path or multicycle constraint;
3. rerun timing reports on that same path class;
4. verify active exception and resulting requirement/slack;
5. check that no unintended neighboring paths were also covered.

### Что особенно важно в prototyping

В prototype flow false/multicycle paths особенно коварны. Prototype часто допускает rough QoR, but it very плохо переносит **wrong timing assumptions hidden by exceptions**. Если engineer слишком рано раздает false paths, design может быстро стать “timing-clean on paper” and unstable on hardware. Если multicycle поставлен шаблонно без proper hold handling, design может получить unrealistic hold model и странные failures позже. AMD timing-closure methodology в целом подталкивает к тому, чтобы timing exceptions reflect **actual functionality**, а не служили cosmetic cleanup.

Поэтому хороший prototype discipline обычно такая:

- false path only when you can explain why path should not be analyzed functionally;
- multicycle only when path really inactive on some cycles;
- CDC cases assessed separately;
- path-level report verification mandatory after each exception change.

### Типичные ошибки

Самая частая ошибка — использовать **false path вместо multicycle**, просто чтобы быстро убрать violation. UG903 explicitly says multicycle should be used when the intent is only to relax synchronous timing while keeping path timed.

Вторая ошибка — объявлять **false path на CDC**, когда path still needs bounded datapath delay or bit-to-bit skew control. UG903 прямо указывает на `set_max_delay -datapath_only` and `set_bus_skew` for such cases.

Третья ошибка — задавать **multicycle setup without proper hold adjustment**. UG949 names this as a common mistake and warns that hold requirement can become impossible.

Четвертая ошибка — ставить multicycle between incorrect points or too broad object sets, especially where endpoint has multiple meaningful pins like CE/reset/data. UG949 explicitly warns about this.

Пятая ошибка — забывать про **precedence**. Clock groups cannot be overridden, and false/multicycle interplay is subject to explicit priority rules and sometimes `-reset_path`.

### Практический итог

**False/multicycle paths** — это полноценная подтема внутри блока **Constraints writing (XDC/SDC)**, потому что она отвечает не на вопрос “как убрать violation”, а на более важный вопрос: **какую timing semantics этот path реально имеет в архитектуре**. Official AMD guidance проводит очень четкую границу: `set_false_path` means do not analyze the path; `set_multicycle_path` means analyze the path with a multi-cycle requirement. AMD generally advises caution with false paths, recommends multicycle when the path still must be timed, and separately warns about hold handling and endpoint selection mistakes for multicycle constraints.

Если сказать совсем коротко: **false path скрывает path из обычной timing картины, multicycle path меняет его timing budget, но оставляет path видимым.** И главная инженерная задача здесь — не выбрать “самую удобную” команду, а выбрать ту, которая точно соответствует реальной functional природе пути.