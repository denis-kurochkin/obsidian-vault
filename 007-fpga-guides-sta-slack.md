## Slack interpretation

**Slack** в контексте FPGA/Vivado и блока **Static Timing Analysis (STA)** — это итоговая разница между тем, **сколько времени path имеет по requirement**, и тем, **сколько времени path реально тратит** с учетом data delay, clock skew, uncertainty и требований самого sequential element. В UG949 AMD прямо показывает упрощенные equations для setup/recovery и hold/removal slack, а UG906 описывает timing path как сочетание source clock path, data path и destination clock path.

Главная practical мысль такая: **slack — это не просто “прошло или не прошло”, а сжатое объяснение баланса между requirement и реальным путем**. Положительный slack означает, что path укладывается в заданную timing model. Отрицательный slack означает violation. Но для нормальной инженерной интерпретации этого мало: нужно понимать, **почему именно** slack получился таким — из-за слишком жесткого requirement, из-за длинного datapath, из-за плохого skew, из-за большой uncertainty или из-за ошибки в constraints. UG949 прямо рекомендует читать slack через упрощенные equations, чтобы увидеть, какой вклад вносит каждый фактор.

### Что именно означает положительный и отрицательный slack

Для **setup/recovery** Vivado использует логику вида:  
`slack = requirement - max datapath delay + clock skew - clock uncertainty - setup/recovery time`.  
Для **hold/removal** логика другая:  
`slack = requirement + min datapath delay - clock skew - clock uncertainty - hold/removal time`.  
Из этого сразу следует две важные вещи. Во-первых, positive slack означает timing margin. Во-вторых, setup и hold slacks нельзя интерпретировать одинаково, потому что в одном случае опасен слишком длинный path, а в другом — слишком короткий.

Очень полезно мысленно перевести slack в простой вопрос:  
**насколько path далек от границы допустимого поведения**.  
Если slack положительный, у path есть запас. Если он отрицательный на 0.050 ns, path опаздывает или нарушает hold-window на 50 ps. Если отрицательный на 1 ns, это уже не “мелкий шум implementation”, а сильный симптом архитектурной или constraint-level проблемы. UG906 прямо использует slack как базовый индикатор violations в Timing Summary и рекомендует начинать timing debug именно с этого отчета.

### Почему slack нельзя читать без requirement

Одна из самых частых ошибок — смотреть только на число slack, не глядя на **path requirement**. UG906 отдельно подчеркивает, что path requirement — это разница по времени между capture edge и launch edge, и что при debugging timing issues первым делом полезно оценить, не слишком ли tight сам requirement для данной topology. В разделе **Timing Violations** AMD прямо рекомендует сначала проверить именно requirement, прежде чем переходить к более глубокому анализу path delay.

Практически это означает: одинаковый slack может иметь разный смысл в разных условиях.  
Например, `-0.200 ns` при требовании 2 ns и `-0.200 ns` при требовании 10 ns — это не одинаковая инженерная ситуация. В первом случае, возможно, path просто слишком агрессивен для выбранной частоты. Во втором это может быть уже признак странной implementation, неправильной clock model или плохо написанных constraints. Поэтому правильная интерпретация slack почти всегда начинается не с самого slack, а с пары **requirement + slack**.

### Как понимать WNS и TNS через slack

На уровне сводных отчетов Vivado агрегирует slack в метрики вроде **WNS** и **TNS**. **WNS** — это worst negative slack, то есть самый плохой violating path. **TNS** — total negative slack, то есть суммарный negative slack по violating paths. Это полезно, потому что один сильно плохой path и много умеренно плохих paths — это разные классы проблем. UG906 использует Timing Summary как главный обзорный отчет и отдельно описывает timing scores в run-level reporting.

Практически можно читать это так.  
Если **WNS сильно плохой, а TNS небольшой**, скорее всего, есть несколько локальных hotspots.  
Если **WNS умеренно плохой, но TNS большой**, проблема часто уже системная: много похожих violating paths, возможно, одна общая архитектурная причина, clock relation problem или widespread routing issue. Это не формальная “математика из учебника”, а полезная инженерная интерпретация того, зачем Vivado показывает aggregated slack metrics, а затем предлагает перейти к detailed `report_timing`.

### Что slack говорит о datapath

UG906 в разделе **Category 1: Timing** рекомендует разбирать datapath через **Path Delay, Logic Delay и Net Delay**. Это один из самых полезных способов интерпретации slack. Если **Logic Delay** составляет больше примерно 50% total path delay, AMD советует смотреть на logic depth и cell types — возможно, нужен RTL change или другая synthesis strategy. Если доминирует **Net Delay**, надо смотреть на physical characteristics, fanout, placement spread и hold-fix detours.

Отсюда следует очень важный practical принцип:  
**отрицательный slack сам по себе не говорит, что именно плохо**.  
Он только говорит, что path не уложился. А вот соотношение **logic delay** и **net delay** уже подсказывает направление решения. Большой logic delay чаще ведет к вопросам про pipelining, simplification, retiming, arithmetic structure. Большой net delay чаще ведет к вопросам про floorplanning, congestion, fanout и physical spread. Именно поэтому хороший slack interpretation всегда включает breakdown path delay, а не только итоговую цифру violation.

### Что slack говорит о clock topology

UG949 отдельно советует смотреть на **clock skew** как на самостоятельный contributor к slack. Если величина skew близка по модулю к величине negative slack и сам skew превышает несколько сотен пикосекунд, AMD рекомендует review clock topology, потому что skew likely major contributor. Это очень сильная practical подсказка: path может быть “плохим” не столько из-за логики, сколько из-за clock-side structure.

Это особенно полезно в prototype flow, где designer часто интуитивно винит datapath first. Если slack плохой, первое желание — вставить register, упростить RTL или снизить frequency. Но если проблема в skew, такие действия могут помочь меньше, чем ожидается. Поэтому зрелая интерпретация slack всегда задает второй вопрос:  
**это data-path problem или clock-path problem?**  
UG949 как раз и предлагает смотреть на skew contribution в уравнении slack, а не только на logic depth.

### Почему post-synthesis slack и post-route slack — это не одно и то же

UG906 отдельно оговаривает, что в **synthesized design** Vivado оценивает net delays по connectivity и fanout, а в **implemented design** использует уже actual routing information. Поэтому post-synthesis slack — это полезный early forecast, но не final signoff truth. Для окончательного signoff AMD прямо рекомендует использовать Timing Summary report после полного routing.

Практически это означает:  
если post-synthesis slack уже сильно плохой, это серьезный warning.  
Если он слегка плохой или около нуля, interpretation должна быть осторожной: часть проблем может уйти или, наоборот, проявиться сильнее после placement and routing.  
Если post-route slack плохой, это уже ближе к реальному verdict по железу.  
То есть один и тот же slack надо читать по-разному в зависимости от стадии flow. Это одна из самых важных дисциплин в prototyping: не перепугаться слишком рано, но и не игнорировать сильные ранние симптомы.

### Почему slack нельзя интерпретировать без проверки constraints

UG906 в **Check Timing** section прямо пишет, что Timing Summary показывает issues related to missing or incomplete timing constraints, и для полноценного signoff все path endpoints должны быть constrained. Если constraints неполные или неверные, slack теряет инженерный смысл, потому что path timed against the wrong model or not timed at all. UG906 в timing-debug guidance также советует при странной картине first check the path requirement.

Это означает очень важную вещь: **good slack on a badly constrained design is not good news**.  
И наоборот, sometimes ugly slack is partly a symptom of missing or incorrect constraints rather than of bad hardware structure. Поэтому хороший slack interpretation всегда живет в связке с:

- `report_timing_summary`,
- `check_timing`,
- `report_timing`,
- и общей проверкой clock/IO/exception model.  
    Без этого slack превращается в цифру без контекста.

### Как practically читать slack в report_timing

UG949 пишет, что timing path report gives detailed information on how slack is computed on any logical path for any timing check. А UG906 говорит, что `report_timing` помогает investigate issues flagged in Timing Summary or verify the coverage and validity of specific timing constraints. Это означает, что правильная работа со slack почти всегда двухшаговая:  
сначала смотри summary, потом раскрывай конкретный path.

Хороший practical workflow обычно такой:

1. Найти worst failing path в Timing Summary.
2. Открыть `report_timing` на этот path class.
3. Посмотреть requirement, slack, skew, uncertainty, path delay.
4. Разделить datapath на logic delay и net delay.
5. Понять, локальна ли проблема или это симптом более широкой pattern.  
    UG906 отдельно рекомендует в analysis смотреть именно на path characteristics reports и category-based breakdown.

### Что slack говорит о Fmax, а что не говорит

UG949 дает практическую формулу для оценки maximum frequency через WNS:  
`FMAX (MHz) = max(1000 / (Ti - WNSi))`,  
где `Ti` — target clock period, а `WNSi` — worst negative slack. Это полезно, потому что показывает связь slack с достижимой частотой. Но тут важно не упростить чрезмерно: WNS помогает оценить pressure against current frequency target, но не заменяет полноценный path analysis.

Практически отрицательный setup slack действительно можно использовать как грубую оценку того, насколько текущая target frequency ambitious. Но slack не говорит сам по себе, **почему** Fmax не достигнут. Для этого все равно нужно разбирать datapath, skew, constraints quality и physical characteristics. То есть связь `slack ↔ Fmax` полезна, но только как верхнеуровневый engineering heuristic, а не как полная диагностика.

### Типичные ошибки при интерпретации slack

Самая частая ошибка — смотреть только на знак slack и не смотреть на requirement, skew, uncertainty и path composition. UG949 как раз поэтому и дает simplified equations: чтобы engineer видел contributors, а не только итог.

Вторая ошибка — считать post-synthesis slack final truth. UG906 отдельно предупреждает, что synthesized-design delays are estimated and less accurate than implemented timing.

Третья ошибка — лечить любой negative slack одинаково. UG906 clearly separates logic-dominated and net-dominated cases and recommends different investigation paths.

Четвертая ошибка — доверять хорошему slack при incomplete constraints. UG906 прямо указывает на роль Check Timing section и необходимость constrain all endpoints for complete signoff.

### Практический итог

**Slack interpretation** — это полноценная подтема внутри блока **STA**, потому что она отвечает не просто на вопрос “есть ли violation”, а на более важный вопрос: **что именно итоговый slack говорит о requirement, datapath, clock topology и качестве timing model**. Official Vivado guidance показывает, что slack вычисляется из requirement, datapath delay, skew, uncertainty и cell timing requirements; что summary numbers вроде WNS/TNS полезны для быстрой оценки масштаба проблемы; и что meaningful interpretation всегда требует перехода от summary к detailed path analysis.

Если сказать совсем коротко: **slack — это числовой итог всей timing-истории path, а не просто красная или зеленая лампочка.** И чем лучше engineer умеет разложить этот итог на requirement, logic delay, net delay, skew и constraints quality, тем быстрее он переходит от “у меня violation” к “я понимаю, почему он возник и как именно его исправлять”.