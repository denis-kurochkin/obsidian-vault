## Input/output delay

**Input/output delay** в контексте FPGA/Vivado и блока **Constraints writing (XDC/SDC)** — это описание **внешнего временного окружения** FPGA на границе кристалла. Команды `set_input_delay` и `set_output_delay` нужны потому, что Vivado анализирует timing только **внутри** FPGA; все задержки, которые живут **вне** кристалла — во внешнем устройстве, на плате, в трассах между clock и data — нужно явно сообщить tool’у через constraints. AMD прямо формулирует это так: чтобы точно смоделировать external timing context, для input и output ports нужно задавать timing information beyond FPGA boundaries именно через `set_input_delay` и `set_output_delay`.

Главная мысль здесь такая:  
**input/output delay — это не “уточнение после create_clock”, а обязательная часть timing-модели интерфейса.** Clock говорит Vivado, когда происходят reference edges. А `set_input_delay` и `set_output_delay` говорят, **где относительно этих edges реально находятся данные на ножке FPGA**. Без этих constraints timing внутри ядра может считаться формально, но картина на границе с внешним миром будет неполной или неверной. AMD в UG949 отдельно пишет, что defining good constraints идет по шагам, и первые два шага как раз задают default timing requirements через **clock waveforms** и **I/O delay constraints**; там же сказано, что input/output delay constraints must be specified to describe external paths to and from the device interface.

### Почему это отдельная полноценная подтема

Внутри блока **Constraints writing (XDC/SDC)** тема **input/output delay** выделяется отдельно потому, что она находится на границе между FPGA и внешней системой. `create_clock` описывает сами reference clocks. `set_input_delay` и `set_output_delay` описывают уже **отношение data к этим clocks на интерфейсе**. Если `create_clock` — это внутренняя временная ось проекта, то input/output delay — это способ связать эту ось с реальной платой и внешними chips. AMD прямо пишет, что такие delays определяются **relative to a clock** that is usually also generated on the board and enters the device, а в некоторых случаях delays надо задавать **relative to a virtual clock**, если waveform связанного внешнего clock отличается от board clock.

То есть здесь речь уже не про “внутренний timing FPGA сам по себе”, а про **system-level timing contract**. Именно поэтому эта тема так важна для prototyping: prototype очень легко может “хорошо выглядеть” по внутреннему timing и при этом иметь неверно описанный внешний интерфейс. Тогда design будет misleadingly clean inside Vivado и проблемным в реальном железе. Это не отдельная экзотика, а нормальное следствие того, что Vivado timing engine не видит внешнюю часть тракта сам по себе.

### Что такое set_input_delay

Команда **`set_input_delay`** задает input path delay на input port **relative to a clock edge at the interface of the design**. AMD уточняет, что с точки зрения application board этот delay — это фазовая разница между:

1. данными, которые идут от внешнего chip через плату к input package pin FPGA, и
2. соответствующим reference board clock.  
    При этом input delay может быть как **положительным**, так и **отрицательным**, в зависимости от относительной фазы clock и data на границе устройства.

Это очень важная practical идея. Многие engineers поначалу мыслят input delay как “сколько данных идет до FPGA”. Но для Vivado это не просто абсолютная задержка data trace. Это **временное положение входных данных относительно reference clock edge**, которое уже включает system-level relationship между external launch side, board routing и точкой входа в FPGA. Именно поэтому одно и то же physical подключение нельзя корректно описать одной “просто цифрой длины трассы”, если неясно, относительно какого clock эта цифра считается.

### Что такое set_output_delay

Команда **`set_output_delay`** задает output path delay output-порта **relative to a clock edge at the interface of the design**. AMD поясняет, что на application board этот delay представляет фазовую разницу между:

1. данными, которые идут от output package pin FPGA через плату к другому устройству, и
2. соответствующим reference board clock.  
    Как и input delay, output delay тоже может быть **положительным** или **отрицательным** — это зависит от относительной фазы data и clock уже **вне** FPGA.

Практически это означает, что `set_output_delay` — это не “внутренний clock-to-out FPGA”, а **внешнее timing requirement на выходе интерфейса**. То есть ты говоришь Vivado не только “когда у меня щелкает регистр внутри”, а “когда данные должны оказаться валидными для соседнего устройства с учетом board path и его setup/hold expectations”. Именно поэтому output delay в UG949 выражается через `Tsetup`, `Thold`, data delay и clock arrival terms outside/inside FPGA.

### Почему input/output delay всегда завязан на clock

И `set_input_delay`, и `set_output_delay` определяются **относительно clock**. Для Vivado это принципиально. В UG903 отдельно сказано, что хотя в стандартном SDC опция `-clock` может быть optional, **для Vivado она обязательна**. Relative clock может быть либо **design clock**, либо **virtual clock**. Для output delays AMD отдельно рекомендует: если используется virtual clock, брать **тот же waveform**, что и у design clock, связанного с этими outputs, чтобы timing path requirement оставался realistic.

Это один из важнейших моментов темы.  
**Input/output delay без правильно выбранного reference clock почти всегда теряет смысл.**  
Само значение delay — это не автономная величина. Это всегда “насколько data сдвинута относительно конкретного reference edge”. Поэтому правильное написание этих constraints начинается не с числа, а с ответа на вопрос: **какой clock реально является reference для данного интерфейса**. UG949 прямо советует для этого смотреть board schematics, design schematics и, при необходимости, использовать `report_timing -from/-to [get_ports ...]`, чтобы понять, к каким clocks port относится внутри design.

### Primary clock, virtual clock или generated clock

UG949 прямо выделяет три варианта reference clock для I/O constraints:  
**primary clock**, **virtual clock** и **generated clock**. Это очень полезное разделение. В простом system-synchronous интерфейсе часто достаточно привязать `set_input_delay` или `set_output_delay` к уже определенному board clock. Но если внешний интерфейс живет относительно clock с другой waveform или его удобнее моделировать вне netlist, лучше использовать **virtual clock**. AMD отдельно пишет, что virtual clock нужен в тех случаях, когда I/O path связан с clock, waveform которого отличается от board clock, и что virtual clock convenient for modeling jitter or source latency scenarios without modifying the design clock.

Практически хороший подход такой:  
если reference clock реально входит в design и именно им логично описывать интерфейс — используй design clock;  
если нужно описать внешний мир аккуратнее и не смешивать его напрямую с внутренним propagated clock — используй virtual clock;  
если интерфейс привязан к derived clock inside design — смотри в сторону generated clock. UG949 прямо указывает на все три варианта как на нормальные способы привязки I/O delay constraints.

### Min и max: почему одной цифры часто недостаточно

UG903 отдельно подчеркивает, что для `set_input_delay` и `set_output_delay` есть опции **`-min`** и **`-max`**. Они задают разные значения для:

- **min delay analysis** — hold/removal;
- **max delay analysis** — setup/recovery.  
    Если ни `-min`, ни `-max` не указаны, одно и то же значение применяется и к min, и к max analysis.

Это очень важная practical тема. В реальном интерфейсе почти никогда не бывает так, что setup-side и hold-side внешнего пути симметричны. External device может иметь разные worst-case и best-case characteristics, board routing тоже дает range, а не одну магическую константу. Поэтому **полноценное описание интерфейса обычно требует и `-max`, и `-min`**, а не одной общей цифры. UG903 прямо приводит examples, где input delay и output delay задаются разными значениями для `-max` и `-min`.

### Что означают input delay(max) и input delay(min)

UG949 дает очень полезную формулу для input delays. Для input path, относительно interface clock:

- `Input Delay(max) = Tco(max) + Ddata(max) + Dclock_to_ExtDev(max) - Dclock_to_FPGA(min)`
- `Input Delay(min) = Tco(min) + Ddata(min) + Dclock_to_ExtDev(min) - Dclock_to_FPGA(max)`

Там же отдельно сказано, что отрицательный input delay означает: data приходит к интерфейсу устройства **раньше**, чем launch clock edge.

Практически это дает очень полезную рамку мышления.  
`-max` здесь соответствует **setup-side worst case**, а `-min` — **hold-side best/earliest case**.  
То есть при написании `set_input_delay` ты обычно моделируешь не “типичную” задержку, а **внешний временной коридор**, внутри которого данные могут оказаться на pin FPGA. Именно в этом и состоит ценность корректного I/O constraint: он заставляет Vivado проверять внутреннюю логику против реального worst/best-case окна интерфейса, а не против усредненной интуиции.

### Что означают output delay(max) и output delay(min)

UG949 аналогично дает формулы для output delays:

- `Output Delay(max) = Tsetup + Ddata(max) + Dclock_to_FPGA(max) - Dclock_to_ExtDev(min)`
- `Output Delay(min) = Ddata(min) - Thold + Dclock_to_FPGA(min) - Dclock_to_ExtDev(max)`

И отдельно поясняет, что output delays refer to the minimum and maximum time outside the device required for the path to be functional under all conditions.

Практический смысл такой:  
`set_output_delay -max` обычно моделирует **setup requirement внешнего приемника**, а `-min` — **hold-side requirement** на том же интерфейсе. То есть здесь ты constrain’ишь не “как быстро FPGA умеет выдавать data”, а “успеет ли внешний chip корректно ее принять”. Это очень важное отличие: output delay — это уже не внутренняя характеристика FPGA path в чистом виде, а часть **совместной модели FPGA + board + external receiver**.

### Почему board schematics здесь критичны

UG949 в разделе про constraining input and output ports прямо подводит к тому, что reference clocks и delay values нужно определять с **system-level perspective**: identify clocks related to each port, browse the board schematics, browse the design schematics, use report timing from or to the port. Это очень практичная рекомендация: значения `set_input_delay` и `set_output_delay` нельзя качественно придумать “из головы” только по RTL.

Именно здесь тема I/O delays резко отличается от многих внутренних constraints. Для внутреннего path engineer часто может опереться на netlist и reports. Для input/output delay нужно еще понимать:

- какой внешний device участвует в интерфейсе;
- какой у него clocking relation;
- какие у него `Tco`, `Tsetup`, `Thold`;
- как разведены board traces clock/data;
- есть ли phase shift или source-synchronous behavior.  
    Vivado помогает проверить модель, но исходные числа приходят **не из synthesis**, а из system-level knowledge о плате и соседнем устройстве.

### DDR и несколько активных edges

Для DDR-подобных интерфейсов UG903 отдельно показывает использование `-clock_fall` и `-add_delay`. `-clock_fall` означает, что constraint applies to timing paths launched by the **falling edge** of the relative clock; без этой опции Vivado assumes only the rising edge. Также AMD отдельно предупреждает: не путать `-clock_fall` с `-rise` и `-fall`, потому что последние относятся к **data edge**, а не к **clock edge**.

Опция **`-add_delay`** обязательна, если:

- уже существует max или min delay constraint на том же порту,
- и ты хочешь задать второй max/min constraint на тот же порт.  
    AMD прямо говорит, что это обычно используется для DDR interfaces или вообще когда port нужно constrain’ить относительно нескольких clock edges или нескольких clocks.

Практически это значит, что для DDR нельзя ограничиться одной строчкой delay. Нужно явно описать rising-edge и falling-edge cases, а при добавлении второго значения на тот же порт использовать `-add_delay`. В UG903 есть прямые DDR examples и для `set_input_delay`, и для `set_output_delay`.

### Feed-through path: когда нет register-to-register внутри

UG903 отдельно приводит важный пример: чтобы constrain’ить **pure combinational path** между I/O ports, нужно определить и input delay, и output delay **relative to a previously defined virtual clock**. В example дается path `DIN -> DOUT`, где при virtual clock period 10 ns, input delay 4 ns и output delay 1 ns внутренний combinational requirement становится 5 ns.

Это очень полезный case, потому что он показывает, что I/O delays — это не только про стандартный “external register -> FPGA register” или “FPGA register -> external register” случай. Они нужны и тогда, когда FPGA выполняет просто combinational forwarding between ports. В таком случае именно сумма input/output constraints формирует допустимое внутреннее окно на обработку сигнала внутри FPGA.

### Ограничения по объектам: к чему можно применять эти команды

UG903 отдельно пишет, что `set_input_delay` можно применять только к **input** или **bi-directional ports**; clock input ports automatically ignored, internal pins generally not valid targets, за исключением специальных случаев вроде STARTUPE3. Аналогично `set_output_delay` применим только к **output** или **bi-directional ports**; internal pin не является обычной целью этой команды, опять же за исключением специальных documented cases.

Практически это помогает избежать одной из типичных ошибок: пытаться constrain’ить этими командами произвольные внутренние nets. В большинстве случаев input/output delay — это именно **constraint на границе интерфейса**, а не general-purpose path exception. Если задача уже живет внутри netlist, обычно нужно смотреть на другие timing constraints, а не на I/O delay commands.

### Почему input/output delay идут после clock constraints

UG949 в разделе **Defining Timing Constraints in Four Steps** прямо говорит, что первые два шага timing methodology — это default timing requirements from **clock waveforms** and **I/O delay constraints**. В той же структуре сначала идет **Defining Clock Constraints**, а затем **Constraining Input and Output Ports**. Это полностью логично: пока reference clocks не описаны, корректно задать I/O delays невозможно.

Практический workflow отсюда очень прямой:

1. сначала создаешь primary/generated/virtual clocks;
2. затем определяешь input/output delays относительно этих clocks;
3. потом уже переходишь к clock groups, CDC constraints и timing exceptions.  
    Если сделать наоборот, можно получить syntactically valid XDC, но плохую или двусмысленную timing model.

### Как practically находить правильный reference clock для порта

UG949 дает очень полезный practical совет: после того как clocks определены, можно использовать `report_timing -from [get_ports ...] -sort_by group` или `report_timing -to [get_ports ...]`, чтобы понять, к каким clocks внутри design относится конкретный port. Дальше можно создать input/output delay relative to the reported clock, rerun report, и если порт still related to more than one clock, добавить соответствующие constraints и повторить проверку.

Это очень сильная техника именно для prototyping, когда интерфейсы уже есть, а exact clock relationships inside design не всегда очевидны с первого взгляда. Вместо гадания “какой clock поставить в `-clock`” можно использовать timing engine как помощника и проверить, какие path groups Vivado реально видит от данного порта.

### Timing Constraints Wizard и I/O delays

UG903 отдельно пишет, что Timing Constraints Wizard analyzes all paths from input ports to identify their destination clocks and active edges, а для outputs — source clocks inside the design and their active edges. На этой основе wizard рекомендует basic system synchronous input/output delay constraints.

Это полезно понимать правильно. Wizard хорош как **validation and discovery tool** — он помогает найти missing or obvious I/O constraints. Но сами numbers delay он не “угадывает” из воздуха: значения должны прийти из board-level timing knowledge и specs external device. То есть wizard помогает связать port с clock, но инженер все равно обязан понимать system timing context.

### Типичные ошибки

Самая частая ошибка — **вообще не задавать input/output delays**, ограничившись только `create_clock`. AMD прямо говорит, что для точного моделирования external timing context timing information for input and output ports must be given through `set_input_delay` and `set_output_delay`.

Вторая ошибка — использовать одну “красивую” цифру без раздельных `-min` и `-max`, когда реальный интерфейс явно имеет разные setup-side и hold-side условия. UG903 прямо поддерживает separate min/max analysis, а UG949 дает разные формулы для max и min both for inputs and outputs.

Третья ошибка — выбрать неправильный reference clock или не использовать virtual clock там, где он нужен. UG903 и UG949 прямо выделяют design clock, virtual clock и generated clock как разные valid references для I/O constraints.

Четвертая ошибка — забыть про `-add_delay` в DDR-like cases. AMD отдельно пишет, что без `-add_delay` второй max/min delay на тот же port не задается как дополнительный independent case.

Пятая ошибка — пытаться трактовать I/O delay как purely internal constraint. Это не constraint “про логику FPGA как таковую”, а constraint “про отношение FPGA к внешнему миру”. Поэтому числа должны приходить из schematics, datasheets и interface timing budget, а не только из внутреннего netlist reasoning.

### Практический итог

**Input/output delay** — это полноценная подтема внутри блока **Constraints writing (XDC/SDC)**, потому что она отвечает не просто на вопрос “как написать `set_input_delay` или `set_output_delay`”, а на более важный вопрос: **как встроить FPGA в реальную внешнюю timing-систему платы**. Official Vivado guidance прямо говорит, что эти constraints нужны для modeling external timing context beyond FPGA boundaries; они задаются относительно design or virtual clock, поддерживают separate min/max analysis, используются для normal system-synchronous and DDR-style cases и должны проверяться по timing reports и clock relationships port’ов.

Если сказать совсем коротко: **`set_input_delay` и `set_output_delay` превращают внутренне корректный timing FPGA в корректный timing интерфейса.** Без них проект может быть хорошо constrained внутри кристалла, но плохо описан на границе с внешним миром — а именно эта граница в prototyping чаще всего и определяет, заработает ли железо как ожидалось.