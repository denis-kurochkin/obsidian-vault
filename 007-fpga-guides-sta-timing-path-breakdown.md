## Timing path breakdown

**Timing path breakdown** в контексте FPGA/Vivado и блока **Static Timing Analysis (STA)** — это разбор timing path на составные части, из которых Vivado потом вычисляет **requirement**, **delay** и итоговый **slack**. В UG906 timing path описывается как связь между startpoint и endpoint, а сам path в отчете обычно делится на три основные секции: **Source Clock Path**, **Data Path** и **Destination Clock Path**. Там же отдельно разбираются **launch edge** и **capture edge**, то есть временные точки, относительно которых строится весь analysis.

Главная practical мысль такая:  
**timing violation почти никогда не объясняется одной цифрой slack.**  
Чтобы понять причину, нужно разложить path на clock-side и data-side contributions и посмотреть, что именно “съедает” запас: слишком tight **requirement**, большой **logic delay**, большой **net delay**, плохой **clock skew** или высокая **uncertainty**. AMD прямо рекомендует для этого смотреть detailed timing report, потому что он показывает, как именно slack computed on a given path.

### Почему это отдельная полноценная подтема

Внутри блока **STA** тема **timing path breakdown** важна потому, что именно она переводит timing analysis из уровня “есть violation / нет violation” в уровень инженерного понимания причин. UG906 в разделе **Category 1: Timing** советует при анализе path смотреть на **Path Type**, **Requirement**, **Slack**, **Timing Exception**, а для datapath — отдельно на **Path Delay**, **Logic Delay** и **Net Delay**. Это уже не обзорный timing summary, а начало root-cause analysis.

Для prototyping это особенно полезно, потому что на ранних стадиях проекта часто надо быстро отличить:  
это architectural problem в RTL,  
или issue в clocking,  
или symptom незакрытого floorplanning/routing,  
или вообще consequence плохих constraints.  
Без path breakdown все эти причины легко смешиваются, и engineer начинает лечить “плохой slack” слишком общими методами. UG949 прямо подводит к тому, что slack нужно читать через contributors, а не как isolated number.

### Из чего состоит timing path

В UG906 timing path sections явно разбиты на:

- **Source Clock Path**
- **Data Path**
- **Destination Clock Path**

Это очень полезная модель.

**Source Clock Path** — это все, что относится к приходу launch clock edge к стартовой sequential cell.  
**Data Path** — это путь данных от launch point до capture point, включая logic и nets.  
**Destination Clock Path** — это все, что относится к приходу capture clock edge к конечной sequential cell.

Практически это означает, что timing problem может сидеть в любой из этих трех частей. Если engineer смотрит только на data logic между регистрами, он видит лишь центральную часть path, но не видит, например, clock skew contribution или плохую clock distribution. UG949 отдельно показывает, что **clock skew** входит прямо в slack equations, а значит timing path always must be read as combined clock-data-clock structure.

### Source clock path: что это и почему он важен

**Source Clock Path** — это часть анализа, которая описывает, как launch edge доходит до startpoint. В timing report именно отсюда Vivado берет временную reference для старта движения данных. В общей структуре path, описанной в UG906, source clock path — отдельная section, а в UG949 slack equations показывают, что launch/capture relationship и clock skew directly affect result.

Практический смысл тут такой: если source-side clock distribution необычная, если у clock есть extra delay, если path идет через сложную clock topology, все это может повлиять на effective launch timing. В простых synchronous designs engineer обычно не думает о source clock path отдельно, но при более сложной clocking architecture именно он начинает влиять на slack не меньше data logic. UG949 прямо советует отдельно оценивать skew contribution, если он comparable to negative slack.

### Data path: центральная часть breakdown

**Data Path** — это то, что инженеры обычно первым делом называют “сам timing path”: launch register, combinational logic, routing nets, endpoint pin. Но в Vivado datapath detail разбивается дальше на:

- **Path Delay**
- **Logic Delay**
- **Net Delay**

UG906 прямо советует смотреть именно на это разбиение. Если **Logic Delay** составляет больше примерно 50% total path delay, нужно проверять **logic depth** и типы cells; если доминирует **Net Delay**, надо смотреть на physical characteristics, high fanout nets и hold-fix detours.

Это, пожалуй, самый useful practical rule всей темы. Он позволяет быстро отделить две разные инженерные ситуации:

- path плохой потому, что в нем слишком много logic work на один cycle;
- path плохой потому, что logic уже не так страшна, но placement/routing makes it physically expensive.

### Logic delay: что он обычно означает

Если в breakdown видно, что path mainly bad because of **Logic Delay**, это обычно указывает на проблемы уровня RTL/microarchitecture: слишком много LUT levels, тяжелый decode, длинная mux chain, сложная arithmetic cone или просто недостаток pipelining. UG906 прямо советует при high logic-delay percentage examine logic depth and cell types, а при необходимости менять RTL or synthesis settings.

Практически такой path часто лечится не floorplanning, а архитектурными изменениями:

- добавить pipeline stage,
- упростить combinational decision,
- разбить крупный block,
- изменить arithmetic structure,
- проверить retiming opportunities.  
    UG949 также советует после synthesis or `opt_design` раньше всего искать longest paths and improve them, because this often most strongly impacts timing and power QoR.

### Net delay: что он обычно означает

Если в breakdown dominant **Net Delay**, это уже другой класс проблемы. UG906 прямо пишет: if Net Delay dominates a path with a reasonable requirement, look at physical and property characteristics, check for **high fanout nets** or **hold fix detours**. Это уже намекает не столько на “плохую логику”, сколько на плохую physical realization path inside the device.

Практически это часто означает:

- registers or blocks physically too far apart,
- path crosses congested region,
- one control/data net has too much fanout,
- router had to take detours,
- hierarchy or floorplan caused poor locality.  
    Для prototype flow это особенно важно: если engineer incorrectly attacks a net-delay problem by rewriting arithmetic, он легко тратит время не туда. Path breakdown как раз помогает этого избежать.

### Destination clock path: вторая clock-side половина

**Destination Clock Path** — это capture-side часть timing path, то есть как capture edge приходит к endpoint. Как и source clock path, эта section участвует в формировании effective relationship between launch and capture edges. В UG906 destination clock path explicitly part of timing-path sections, а UG949 связывает source/destination clock delays через **clock skew**.

Практически здесь важно понимать: path может выглядеть “длинным” не только потому, что data late, но и потому, что destination clock path creates unfavorable skew. Если skew и negative slack близки по величине, UG949 рекомендует review clock topology, потому что skew likely major contributor. Это очень сильная диагностика: иногда timing path breakdown показывает, что root cause sits more in clock distribution than in data logic.

### Launch and capture edges: где рождается requirement

UG906 в timing path structure отдельно выделяет **Launch and Capture Edges**. А UG949 в разделе про tight requirements пишет, что Vivado identifies path requirements by expanding each clock out to 1000 cycles and determining where the closest non-coincident edge alignment occurs. Если 1000 cycles are not sufficient, report shows **Not Expanded**, и clocks надо трактовать как asynchronous for timing purposes.

Это очень важный practical момент. **Requirement** не берется “просто из периода одного clock”. Он получается из реального отношения launch/capture edges. Для same-clock paths это часто выглядит просто, но для inter-clock analysis edge alignment может быть намного тоньше. Именно поэтому timing path breakdown всегда полезно читать вместе с type of path: intra-clock or inter-clock, expanded or not expanded, regular synchronous or exception-covered.

### Requirement как часть path breakdown

UG906 прямо рекомендует смотреть на **Path Type and Requirement** в начале timing analysis. Requirement показывает, сколько времени дано path according to current timing model. Если requirement unexpectedly small, даже умеренный datapath delay может дать bad slack. UG949 в разделе **Identifying Tight Timing Requirements** как раз показывает, что Vivado computes closest valid edge alignment, и именно оно может давать unexpectedly tight path relationships.

Практически это значит, что timing path breakdown начинается не с вопроса “сколько delay”, а с вопроса **“какой requirement вообще у этого path и почему именно такой?”**  
Если requirement absurdly tight, сначала нужно проверить:

- correct ли clock relationship,
- правильно ли заданы clocks/generated clocks,
- не missing ли clock exceptions,
- нет ли inter-clock case that should be treated differently.

### Slack как итог всего breakdown

UG949 прямо пишет, что timing path report shows how **slack** is computed, а simplified equations позволяют выделить contributions of requirement, datapath delay, skew, uncertainty and endpoint timing requirement. То есть **slack** — это не отдельная “четвертая сущность”, а итог всей структуры path breakdown.

Поэтому хороший timing path breakdown всегда отвечает на вопросы:

- какой requirement;
- сколько занимает datapath;
- из datapath что logic, а что net;
- каков skew;
- какова uncertainty;
- насколько итоговый slack driven by each of these pieces.  
    Если этого разбора нет, negative slack остается только красной цифрой. Если разбор есть, violation превращается в диагностируемую engineering problem.

### Path breakdown и setup/hold

Timing path breakdown полезен и для **setup**, и для **hold**, но интерпретация differs. UG949 показывает, что setup slack uses **max datapath delay**, а hold slack uses **min datapath delay**. Это означает, что один и тот же path physically может иметь разные dominant contributors в setup и hold views. Например, длинная logic chain often hurts setup, а слишком short path plus skew often hurts hold.

Практически это важно потому, что engineer не должен думать, будто один breakdown “на все случаи”. Нужно понимать, **какой timing check сейчас смотрится**. UG906 тоже в path analysis рекомендует сначала видеть **Path Type** — setup or hold — before deeper analysis.

### Post-synthesis и post-route breakdown: чем они отличаются

UG906 отдельно оговаривает, что в synthesized design Vivado estimates net delays based on connectivity and fanout, while in implemented design it uses actual routing information. Это значит, что **timing path breakdown after synthesis** полезен как ранний forecast, но **post-route breakdown** уже намного ближе к physical truth. Особенно это касается **Net Delay** и **clock skew**.

Практически это читается так:

- если post-synthesis path already has terrible logic-delay breakdown, проблема likely architectural;
- если post-synthesis path looks acceptable, but post-route net delay explodes, проблема likely physical;
- if skew becomes significant only after routing, clock topology and placement deserve extra attention.  
    Именно поэтому prototype flow выигрывает от умения читать breakdown на нескольких стадиях, а не только в final routed design.

### Report Timing и Report Design Analysis

UG906 и UG949 вместе показывают два очень полезных инструмента для этой темы.  
**`report_timing`** дает детальную структуру конкретного path и how slack is computed.  
**`report_design_analysis`** помогает смотреть **path characteristics** сразу для worst paths, включая setup path characteristics tables. UG949 прямо показывает пример команды `report_design_analysis -max_paths 50 -setup`.

Практически это значит:

- если нужно понять один конкретный worst path — иди в `report_timing`;
- если нужно увидеть pattern across many bad paths — useful `report_design_analysis`.  
    Один path может вводить в заблуждение. Но когда несколько worst paths all logic-dominated or all net-dominated, path breakdown начинает показывать already system-level trend.

### Что обычно оказывается полезной инженерной интерпретацией

Хороший timing path breakdown обычно приводит не к “общему ощущению”, а к одному из нескольких типовых диагнозов.

**1. Tight requirement problem**  
Requirement слишком мал relative to intended design behavior. Сначала проверяй clocks, generated clocks, inter-clock relationship, exceptions.

**2. Logic-dominated path**  
Too much logic on one cycle. Нужны pipeline, simplification, retiming, RTL changes.

**3. Net-dominated path**  
Physical spread, congestion, fanout, detours. Нужны locality/floorplanning/fanout-oriented actions.

**4. Clock-side problem**  
Skew or uncertainty unusually large. Нужна review of clock topology and relationships.

Это и есть главная ценность breakdown: одна и та же negative slack figure начинает раскладываться на понятные classes of root cause.

### Типичные ошибки

Самая частая ошибка — смотреть только на **slack** и не смотреть на **requirement, logic delay, net delay, skew**. UG949 как раз поэтому дает simplified slack equations, а UG906 рекомендует category-based timing analysis.

Вторая ошибка — считать, что “path long” всегда означает плохой RTL. UG906 explicitly separates logic-dominated and net-dominated cases.

Третья ошибка — игнорировать clock-side sections of the path. Source and destination clock paths are explicit parts of timing report, and skew can be a major contributor.

Четвертая ошибка — делать выводы по post-synthesis net breakdown как по final physical truth. UG906 warns that synthesized net delays are estimated, not routed.

### Практический итог

**Timing path breakdown** — это полноценная подтема внутри блока **STA**, потому что она отвечает не просто на вопрос “где violation”, а на более важный вопрос: **из каких частей этот violation складывается**. Official Vivado guidance явно делит timing path на **Source Clock Path**, **Data Path** и **Destination Clock Path**, рекомендует анализировать **Requirement**, **Slack**, **Logic Delay**, **Net Delay**, **Skew** и использовать `report_timing` plus `report_design_analysis` для root-cause analysis.

Если сказать совсем коротко: **timing path breakdown** — это способ превратить “плохой slack” в понятную инженерную картину. Не просто “path не укладывается”, а “не укладывается потому, что requirement tight, logic глубокая, routing длинный или clock topology неблагоприятна”. Именно с этого обычно и начинается зрелый timing debug в Vivado.