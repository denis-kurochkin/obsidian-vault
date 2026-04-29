## Setup/hold analysis

**Setup/hold analysis** в контексте FPGA/Vivado и блока **Static Timing Analysis (STA)** — это базовая проверка того, успевают ли данные прийти к принимающему sequential element достаточно **рано** для корректного захвата и при этом не меняются **слишком рано после** захвата. В Vivado после того как на clock net определен timing clock, timing analysis выполняет и **setup**, и **hold** checks на data pin destination flip-flop при pessimistic, but reasonable operating conditions; безопасная передача получается тогда, когда и setup slack, и hold slack положительны.

Главная мысль здесь такая:  
**setup отвечает на вопрос “успеют ли данные к нужному фронту”, а hold — “останутся ли они стабильными сразу после этого фронта”.** Эти две проверки связаны, но не взаимозаменяемы. UG906 прямо отмечает, что hold relationship directly tied to setup relationship: setup analysis проверяет безопасный захват в pessimistic scenario, а hold analysis гарантирует, что данные не изменятся слишком рано после capture edge.

### Почему это отдельная полноценная подтема

Внутри блока **STA** тема **setup/hold analysis** выделяется отдельно потому, что именно через нее инженер впервые начинает по-настоящему читать timing path. В UG906 timing analysis описан через timing paths между sequential elements, controlled either by the same clock or by different clocks, а в UG949 отдельно подчеркивается, что reviewing timing slack удобно вести через упрощенные equations для setup и hold. То есть setup/hold — это не “один из видов report”, а фундамент всего timing reasoning в Vivado.

Для prototyping это особенно важно. Prototype может быть еще не идеально оптимизирован, но если engineer не различает setup и hold problems, он почти неизбежно будет неправильно интерпретировать violations и выбирать неверные способы исправления. UG906 прямо пишет, что timing summary используется для signoff и начала debugging timing issues, а implementation log и Design Runs window показывают отдельно **WNS/TNS** и **WHS/THS**, то есть setup и hold уже на уровне верхней сводки считаются разными классами проблем.

### Что такое timing path в setup/hold analysis

Чтобы понять setup/hold, полезно сначала видеть сам **timing path**. UG906 определяет timing path как connectivity between instances in the design; в цифровом design path формируется между двумя sequential elements, controlled by same or different clocks. Внутри timing report такой path обычно распадается на source clock path, data path и destination clock path. Это важно, потому что setup/hold никогда не вычисляются “по одной цифре delay”, а всегда на основе отношений между data movement и clock edges на обоих концах path.

Практически это означает, что даже в простом register-to-register case Vivado смотрит не только на combinational logic между регистрами, но и на то, когда launch edge пришел к source register, когда capture edge пришел к destination register, какой skew между ними образовался, какая uncertainty учтена и какой internal setup/hold requirement имеет destination element. Именно поэтому один и тот же datapath delay может вести себя по-разному в разных clock relationships.

### Setup analysis: что именно проверяется

**Setup analysis** проверяет, что данные успевают дойти до capture point до нужного активного edge с учетом max datapath delay, clock skew, uncertainty и setup time destination element. UG949 дает упрощенную форму уравнения:

`Slack (setup/recovery) = setup path requirement - datapath delay (max) + clock skew - clock uncertainty - setup/recovery time`

Практический смысл тут очень прямой. Если max datapath delay слишком большой, setup slack падает. Если clock uncertainty велика, setup slack тоже падает. Если skew благоприятный, он может помочь setup; если неблагоприятный — ухудшить его. Поэтому setup violation чаще всего означает не просто “слишком много LUT”, а одну из комбинаций: длинный combinational path, тяжелый routing, плохое clock relation, большая uncertainty или неудачное расположение логики. UG906 в категории timing прямо советует при analysis смотреть на path requirement, slack и type of path first.

### Hold analysis: что именно проверяется

**Hold analysis** проверяет противоположную сторону: данные не должны стать новыми слишком быстро после launch, чтобы destination element не увидел раннюю смену вместо старых данных. UG949 дает упрощенную форму hold equation:

`Slack (hold/removal) = hold path requirement + datapath delay (min) - clock skew - clock uncertainty - hold/removal time`

Здесь критично то, что в hold analysis участвует **min datapath delay**, а не max. Поэтому hold problem по своей природе часто противоположна setup problem. Для setup path “слишком длинный” — плохо. Для hold path “слишком короткий” — тоже плохо. UG906 отдельно подчеркивает, что hold relationship tied to setup relationship, но проверяет другой временной риск: не early arrival before the receiving element has safely finished with the previous data.

### Почему setup и hold нельзя путать

Одна из самых частых инженерных ошибок — воспринимать hold как “тот же setup, только в другую сторону”. На практике это ведет к неправильным решениям. Если setup violation часто можно ослабить снижением clock frequency или архитектурным pipelining, то hold violation так не лечится. UG906 прямо пишет, что design might still function in the lab if setup is failing by a small amount because you can lower the clock frequency, but if hold timing fails, the design most likely does not function. Там же отмечено, что router prioritizes fixing hold before setup.

Из этого следует очень важный practical вывод:  
**hold violations обычно опаснее в смысле немедленной работоспособности железа**, а setup violations чаще связаны с достижением target frequency. Это не делает setup второстепенным, но помогает правильно расставить инженерные приоритеты при triage timing issues.

### Роль clock skew в setup/hold analysis

UG949 отдельно подчеркивает, что setup и hold slacks зависят от **clock skew**, причем skew входит в equations по-разному, а timing analysis всегда вычисляет skew по определенной internal модели. Это важно, потому что многие начинающие инженеры мыслят только data path, а clock distribution воспринимают как фон. На деле skew — полноценный участник setup/hold balance.

Практически это означает, что один и тот же logic path может иметь разную судьбу в зависимости от physical clock tree situation. Иногда setup path ухудшается в основном из-за data delay. Иногда не меньше вредит skew/uncertainty. А hold path особенно чувствителен к сочетанию very short datapath и неблагоприятного skew. Именно поэтому в Vivado timing debugging нельзя ограничиваться чтением only logic levels. Нужно смотреть и на clock side path.

### Роль clock uncertainty

UG949 включает **clock uncertainty** в обе упрощенные slack equations. Это очень полезное напоминание: setup/hold analysis в Vivado строится не на “идеальных фронтах”, а на realistic timing model, где есть uncertainty margin. Чем выше uncertainty, тем меньше запас по setup и hold.

Для prototyping это особенно важно, потому что prototype часто содержит несколько clock domains, PLL/MMCM structures, I/O timing assumptions и changing constraints maturity. Если uncertainty model плохая или неполная, engineer может получить misleading STA picture. То есть setup/hold analysis всегда нужно читать в связке с quality of clock constraints.

### Положительный slack и отрицательный slack

В setup/hold analysis главным итоговым числом является **slack**. UG949 прямо пишет, что safe data transfer occurs when both setup and hold slacks are positive. Это простое правило, но за ним стоит важная инженерная дисциплина: смотреть не только на факт violation, но и на его magnitude и distribution. Один path с небольшим отрицательным setup slack и множество paths с плохим hold slack — это очень разные ситуации.

Практически положительный setup slack означает, что max data arrival still fits before required capture edge. Положительный hold slack означает, что min arrival still does not violate the stability window after capture. Отрицательный slack, соответственно, показывает, на сколько design выходит за пределы допустимого timing budget. Именно вокруг этого числа потом строятся **WNS/TNS** для setup и **WHS/THS** для hold.

### Setup/hold analysis после synthesis и после implementation

UG906 отдельно оговаривает, что timing analysis available at any point after synthesis, но accuracy differs by stage. В synthesized design net delays are estimated from connectivity and fanout, so the picture is useful early, but less accurate than post-place/post-route timing. Это означает, что setup/hold analysis после synthesis и после implementation решают схожую задачу, но интерпретируются по-разному.

После synthesis setup/hold violations — это чаще **ранний structural indicator**. Они помогают увидеть architectural risk, missing constraints или obviously hard paths. После implementation те же violations уже ближе к signoff reality. Для prototyping полезно уметь жить в обоих режимах: post-synth использовать как early forecast, post-route — как real decision point о готовности design к железу.

### Input setup/hold как отдельный слой анализа

UG906 отдельно имеет report level для **Input Ports Setup/Hold**. Там сказано, что для каждого input port report lists worst-case setup and hold requirements relative to the reference clock and also shows the internal clock used to capture the input data. Это очень важное расширение базовой темы: setup/hold analysis касается не только register-to-register paths внутри fabric, но и I/O boundary.

Это особенно полезно в prototyping, где внешние interfaces часто оказываются не менее критичны, чем внутренние clocks. Если `set_input_delay` задан правильно, Vivado может уже на уровне input setup/hold report показать worst-case window для порта или bus. Для input buses UG906 отдельно говорит, что worst-case data window for the entire bus equals the sum of largest setup and hold values, а при наличии IDELAY even reports an optimal tap point.

### Почему setup/hold analysis всегда связана с constraints quality

UG949 и UG906 постоянно связывают timing analysis с quality of constraints. В UG906 category timing советует при debugging missing or incorrect timing constraints first check the **path requirement**. Это очень сильная practical мысль: setup/hold analysis не существует в вакууме. Если clocks заданы неправильно, если I/O delays missing, если clock groups or exceptions overly broad, то и setup/hold picture становится неверной.

То есть плохой setup slack не всегда означает “плохая логика”, а хороший setup slack не всегда означает “все хорошо”. Иногда проблема в том, что design timed against the wrong requirement. Именно поэтому зрелый STA workflow всегда проверяет и **timing path**, и **constraint model**.

### Как practically читать setup/hold problem

Хороший practical reading обычно идет в такой последовательности. Сначала смотри, **какой это check** — setup или hold. Потом смотри **requirement**. Затем смотри, что доминирует: datapath delay, skew, uncertainty или endpoint requirement. UG949 прямо рекомендует reviewing timing slack through simplified setup/hold equations because they let you identify which factor impacts slack. UG906 дальше предлагает использовать detailed reports like Report Timing and Report Design Analysis to inspect worst setup paths or hold-sensitive characteristics.

Практически setup path чаще требует questions вроде:  
слишком ли глубокая combinational logic, слишком ли длинный routing, не нужен ли pipelining, правильный ли clock relationship.  
Hold path чаще требует questions вроде:  
не слишком ли короткий path, нет ли problem in clock relation, не возник ли issue from constraints or implementation side effects.  
Именно различение этих двух investigative styles и делает setup/hold analysis зрелым инженерным инструментом.

### Почему hold fixing иногда портит setup

UG906 отдельно рассматривает scenario **Determining if Hold-Fixing is Negatively Impacting the Design**. Там сказано, что router prioritizes fixing hold timing before setup timing; usually it can do so without hurting setup, but in some cases — often caused by design or constraint errors — setup timing can degrade significantly. Это очень важный practical lesson. Setup и hold связаны, и aggressive hold-fixing can consume routing freedom or worsen other paths.

Из этого следует полезный вывод:  
если в design есть крупные hold violations, их нельзя рассматривать как isolated nuisance. Они способны изменить общую timing landscape. Поэтому large hold problems полезно понимать и по возможности снижать рано, а не только ждать, что router magically resolves everything at the end. UG949 прямо советует, что для hold violations larger than about 0.4 ns it can be advantageous to reduce them prior to routing.

### Common engineering intuition: setup про frequency, hold про immediacy

Если очень упростить инженерную картину, то:

- **setup** больше связан с achievable clock frequency and long paths;
- **hold** больше связан с immediate correctness of very short paths and clock edge relationships.

Такое упрощение не заменяет equations, но хорошо помогает в triage. Оно хорошо согласуется с AMD guidance о том, что small setup failure sometimes can be tolerated by lowering frequency, while hold failure usually means design most likely does not function.

Но важно не упростить слишком сильно. Setup/hold analysis — это не просто “long vs short path”. Это всегда совместная модель:

- launch edge;
- capture edge;
- datapath delay;
- clock path;
- skew;
- uncertainty;
- endpoint timing requirements.

### Типичные ошибки

Самая частая ошибка — смотреть только на **setup** и игнорировать **hold**. Vivado верхнеуровнево показывает отдельно WNS/TNS и WHS/THS именно потому, что это два независимых risk classes.

Вторая ошибка — трактовать post-synthesis setup/hold numbers как final signoff truth. UG906 прямо оговаривает, что post-synth net delay estimates are less accurate than implemented timing.

Третья ошибка — лечить setup and hold одной и той же инженерной логикой. AMD guidance показывает, что they differ structurally and even router priorities treat them differently.

Четвертая ошибка — не проверять constraints quality, когда setup/hold picture выглядит странно. UG906 рекомендует в таких случаях first check path requirement.

### Практический итог

**Setup/hold analysis** — это полноценная подтема внутри блока **Static Timing Analysis (STA)**, потому что она отвечает не просто на вопрос “есть ли timing violation”, а на более важный вопрос: **сможет ли каждая передача данных между launch и capture points произойти безопасно как до, так и сразу после capture edge**. Official Vivado guidance показывает, что timing analysis always performs setup and hold checks once clocks are defined, что slack equations для них различаются, что hold relationship tied to setup but checks a different failure mode, и что both analyses must be interpreted together with clocks, constraints and design stage.

Если сказать совсем коротко: **setup говорит, успели ли данные, hold говорит, не пришли ли они слишком рано.** И именно на этой паре проверок держится вся practical ценность STA в Vivado — от ранней оценки prototype после synthesis до финального timing signoff после implementation.