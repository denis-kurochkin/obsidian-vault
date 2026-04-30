# CDC constraints в Vivado / XDC

**CDC constraints** — это часть XDC, которая говорит Vivado, как относиться к timing paths между разными clock domains.

Главная мысль:

> CDC constraints не делают переход между clock domains безопасным.  
> Они только говорят timing engine, какие междоменные paths нужно анализировать обычным STA, а какие нужно исключить или ограничить специальным образом.

Безопасность CDC создается RTL-архитектурой:

```
synchronizer
toggle synchronizer
handshake
async FIFO
Gray-code crossing
reset synchronizer
```

А XDC только описывает это для Vivado.

---

# 1. Почему CDC constraints нужны

Vivado по умолчанию пытается анализировать timing paths между clocks в дизайне. Если два clock действительно асинхронны, обычный setup/hold-анализ между ними часто не имеет физического смысла: фаза между clocks неизвестна или постоянно “плывет”. AMD прямо указывает, что для paths между clocks от разных primary clocks или без общего узла их нужно рассматривать как asynchronous clocks и добавлять соответствующие timing exceptions.

Пример:

```
clk_100m от board oscillator
clk_156m от GT reference/user clock
```

Если между ними есть обычный timing path:

```
clk_100m register -> combinational logic -> clk_156m register
```

Vivado может пытаться закрыть setup/hold между unrelated clocks. Это либо даст бессмысленные violations, либо замаскирует настоящую CDC-проблему.

---

# 2. Что CDC constraints не делают

CDC constraints не создают:

```
two-flop synchronizer
async FIFO
handshake
Gray-code protection
atomic transfer для data bus
защиту от потери pulse
```

Плохая логика:

```verilog
always @(posedge clk_b) begin
    data_b <= data_a;
end
```

не становится безопасной после:

```
set_clock_groups -asynchronous \
    -group [get_clocks clk_a] \
    -group [get_clocks clk_b]
```

Constraint только говорит Vivado не анализировать обычный timing между этими clock groups.

---

# 3. Разделение ответственности

Полезно разделять так:

```
RTL:
    как данные реально пересекают clock domain

XDC:
    как Vivado должен анализировать или игнорировать timing paths

report_cdc:
    совпадает ли RTL-структура с ожидаемой CDC-методологией
```

AMD описывает `report_cdc` как structural analysis CDC-переходов; этот отчет нужен для поиска потенциально небезопасных CDC, включая проблемы metastability и data coherency. При этом `report_cdc` фокусируется на структуре crossing и связанных constraints, а не на timing slack.

---

# 4. Главные команды для CDC constraints

В Vivado чаще всего используются:

```
set_clock_groups
set_false_path
set_max_delay -datapath_only
```

Дополнительно для анализа:

```
report_clock_interaction
report_cdc
report_timing_summary
```

---

# 5. `set_clock_groups -asynchronous`

Самая частая команда для полностью асинхронных clock domains:

```
set_clock_groups -asynchronous \
    -group [get_clocks clk_a] \
    -group [get_clocks clk_b]
```

Смысл:

```
пути из clk_a в clk_b не анализируются обычным timing;
пути из clk_b в clk_a тоже не анализируются обычным timing.
```

То есть это двунаправленное исключение между группами.

AMD описывает `set_clock_groups` как команду, похожую на определение false path constraints для data paths между exclusive или asynchronous clock domains. Если в команде задано несколько групп, timing между группами отключается, но внутри одной группы clocks не становятся asynchronous друг к другу.

---

# 6. Пример: два независимых board clocks

Допустим:

```
create_clock -name sys_clk  -period 5.000 [get_ports sys_clk_p]
create_clock -name sfp_clk  -period 6.400 [get_ports sfp_clk_p]
```

Если это реально независимые источники:

```
sys_clk = 200 MHz oscillator
sfp_clk = 156.25 MHz oscillator / GT-related domain
```

то можно описать:

```
set_clock_groups -asynchronous \
    -group [get_clocks sys_clk] \
    -group [get_clocks sfp_clk]
```

После этого обычный STA не будет пытаться закрывать timing между этими двумя domains.

Но RTL между ними все равно должен быть через CDC-структуры.

---

# 7. Несколько clock groups

Если clocks больше двух:

```
set_clock_groups -asynchronous \
    -group [get_clocks {sys_clk clk_100m}] \
    -group [get_clocks {sfp_clk aurora_user_clk}] \
    -group [get_clocks {pcie_user_clk}]
```

Смысл:

```
clocks внутри одной group не разделяются этой командой;между разными groups timing отключается.
```

Это важно.

Если `sys_clk` и `clk_100m` внутри одной group, эта команда не говорит, что они асинхронны между собой.

---

# 8. Опасность слишком широкого `set_clock_groups`

Плохой стиль:

```
set_clock_groups -asynchronous \
    -group [get_clocks *]
```

Или слишком грубые wildcard-ограничения.

Почему опасно:

```
можно случайно отключить timing там, где clocks на самом деле related;
можно скрыть реальные timing violations;
можно сделать report_timing слишком “чистым” искусственно;
можно затруднить CDC/debug.
```

CDC constraints должны быть осознанными и точными.

---

# 9. `set_false_path`

`set_false_path` отключает timing analysis для конкретных paths.

Пример:

```
set_false_path \
    -from [get_clocks clk_a] \
    -to   [get_clocks clk_b]
```

Это направление:

```
clk_a -> clk_b
```

не анализируется.

Но обратное направление:

```
clk_b -> clk_a
```

не отключается, если не добавить вторую команду.

AMD указывает, что `set_false_path` отключает timing только в направлении, заданном через `-from` и `-to`, в отличие от `set_clock_groups`, который отключает анализ между clock groups.

---

# 10. Двунаправленный `set_false_path`

Эквивалентно `set_clock_groups -asynchronous` для двух clocks можно написать:

```
set_false_path -from [get_clocks clk_a] -to [get_clocks clk_b]
set_false_path -from [get_clocks clk_b] -to [get_clocks clk_a]
```

Но обычно для полностью асинхронных доменов удобнее:

```
set_clock_groups -asynchronous \
    -group [get_clocks clk_a] \
    -group [get_clocks clk_b]
```

`set_false_path` полезен, когда нужно отключить не все paths между clocks, а только конкретное направление или конкретные точки.

---

# 11. Когда лучше `set_clock_groups`

Обычно `set_clock_groups -asynchronous` удобно использовать, когда:

```
два clock domains полностью asynchronous;
между ними нет обычных synchronous timing paths;
все crossing сделаны через CDC-структуры;
нужно отключить timing в обе стороны;
нет необходимости задавать частные ограничения на отдельные CDC paths.
```

Пример:

```
set_clock_groups -asynchronous \
    -group [get_clocks sys_clk] \
    -group [get_clocks aurora_user_clk]
```

---

# 12. Когда лучше `set_false_path`

`set_false_path` может быть лучше, когда:

```
нужно отключить только одно направление;
нужно ограничить конкретные registers/pins;
часть paths между clocks должна оставаться timed;
нужно точечно исключить known false path;
clocks в целом related, но конкретный путь функционально false.
```

Пример:

```
set_false_path \
    -from [get_pins u_src_reg/Q] \
    -to   [get_pins u_sync/sync_ff_reg[0]/D]
```

Но такие точечные exceptions требуют аккуратности. Их сложнее сопровождать, особенно если RTL меняется.

---

# 13. `set_max_delay -datapath_only`

Третий важный вариант:

```
set_max_delay -datapath_only <value> \
    -from <source_objects> \
    -to   <destination_objects>
```

В CDC-контексте это часто используют не для обычного setup timing, а чтобы ограничить физическую задержку datapath между source и destination CDC-элементами.

Смысл:

```
не анализировать clock skew как обычный synchronous path;но ограничить максимальную datapath-задержку.
```

AMD отдельно упоминает `set_max_delay -datapath_only` среди вариантов timing exceptions для asynchronous CDC paths. Также `report_cdc` анализирует paths между asynchronous clocks и paths, проигнорированные через false path или max-delay-datapath-only exceptions.

---

# 14. Где `set_max_delay -datapath_only` полезен

Он полезен, когда нельзя просто полностью “забыть” о physical delay.

Например:

```
Gray-code bus crossing
несколько independent bits с требованием bounded skew
CDC structures, где важна ограниченная задержка до synchronizer
```

Для Gray-code pointer crossing в async FIFO идея такая:

```
Gray code гарантирует изменение одного бита за increment,
но physical routing не должен растягивать биты настолько,
чтобы destination увидел некорректную temporal картину.
```

На практике для XPM/IP это часто уже учтено внутри vendor-методологии, но при самодельных CDC-структурах вопрос становится важным.

---

# 15. Почему не всегда надо делать `set_clock_groups`

Важный нюанс: `set_clock_groups` имеет высокий приоритет и может перекрыть более точные ограничения.

AMD указывает, что `set_clock_groups` имеет более высокий precedence, чем timing exceptions, даже если по смыслу похож на две `set_false_path` между clocks.

Практический вывод:

```
если сделать широкое set_clock_groups,
то более точные set_max_delay -datapath_only constraints
могут потерять смысл.
```

Особенно это важно при использовании XPM CDC.

---

# 16. XPM CDC и constraints

Xilinx/AMD XPM CDC primitives часто содержат или предполагают собственные ограничения для CDC-структур. В официальной методологии AMD прямо отмечает, что XPM CDC предоставляет собственные `set_max_delay -datapath_only` constraints, а `set_clock_groups` имеет более высокий приоритет и может их перекрывать.

Практическое правило:

```
если используешь XPM_CDC,
не надо автоматически ставить широкий set_clock_groups,
не проверив, как это влияет на XPM constraints.
```

В простом проекте `set_clock_groups` часто используют, но в более строгой CDC-методологии стоит понимать последствия.

---

# 17. Clock relationship: synchronous или asynchronous

Перед тем как писать CDC constraints, нужно понять отношение clocks.

Варианты:

```
1. synchronous related clocks2. asynchronous clocks3. logically exclusive clocks4. physically exclusive clocks5. clocks with known but complex relationship
```

Нельзя автоматически считать любые два разных clock names асинхронными.

---

# 18. Related clocks

Clocks могут быть related, если они получены из одного источника через MMCM/PLL/BUFGCE_DIV и Vivado знает их relationship.

Пример:

```
clk_100mclk_200m
```

оба сгенерированы из одного MMCM.

В таком случае Vivado может выполнять нормальный timing analysis между ними.

Если между такими domains есть обычный synchronous crossing, его можно закрывать через timing.

Если crossing сделан как CDC через FIFO/synchronizer, тогда constraints должны соответствовать выбранной архитектуре, но нельзя механически объявлять все related clocks asynchronous только потому, что частоты разные.

---

# 19. Asynchronous clocks

Clocks обычно asynchronous, если:

```
разные независимые oscillators;нет общего primary clock;clock может останавливаться независимо;частота/фаза не имеют фиксированной связи;GT/user clock не имеет deterministic relation с system clock;clock приходит извне и не связан с локальным clock.
```

Для таких clocks обычный STA между domains обычно бессмысленен.

Нужно:

```
1. сделать правильный CDC в RTL;2. описать clock relationship в XDC;3. проверить report_cdc.
```

---

# 20. Logically exclusive clocks

`-logically_exclusive` используют, когда clocks логически не активны одновременно в конкретной части схемы.

Например:

```
clock mux выбирает один из нескольких clocks
```

Упрощенный пример:

```
set_clock_groups -logically_exclusive \    -group [get_clocks clk_mode0] \    -group [get_clocks clk_mode1]
```

Но здесь надо быть осторожным: если clocks реально могут одновременно существовать в разных частях FPGA, это не physically exclusive.

---

# 21. Physically exclusive clocks

`-physically_exclusive` используют, когда clocks физически не могут существовать одновременно.

Например:

```
один и тот же input pin может получать либо clock A, либо clock B,но не оба одновременно
```

Пример:

```
set_clock_groups -physically_exclusive \    -group [get_clocks clk_ext_a] \    -group [get_clocks clk_ext_b]
```

Эта тема близка к clock muxing и mode constraints. Для CDC она важна потому, что exclusive clocks — это не то же самое, что asynchronous clocks.

---

# 22. Не путать generated clocks и CDC

Если у тебя есть:

```
sys_clk -> MMCM -> clk_asys_clk -> MMCM -> clk_b
```

то `clk_a` и `clk_b` могут быть разными clock domains, но не обязательно asynchronous.

Vivado может знать их phase/frequency relationship.

В этом случае варианты:

```
если crossing обычный synchronous — закрывать timing;если crossing через CDC — использовать CDC-структуру и соответствующие exceptions;если transfer не каждый такт — возможно multicycle, но только при строгой логике enable.
```

Нельзя просто выбирать constraint по названию clock. Нужно смотреть архитектуру передачи.

---

# 23. Что писать в XDC сначала

Типовой порядок мышления:

```
1. Создать primary clocks.2. Создать generated clocks.3. Проверить, что Vivado видит все clocks.4. Определить relationship между clocks.5. Для asynchronous groups добавить CDC exceptions.6. Для specific CDC structures добавить/проверить max_delay datapath_only, если нужно.7. Запустить reports.
```

Команды:

```
report_clocksreport_clock_interactionreport_cdcreport_timing_summary
```

---

# 24. Пример структуры XDC для CDC

```
# ============================================================# Primary clocks# ============================================================create_clock -name sys_clk \    -period 5.000 \    [get_ports sys_clk_p]create_clock -name sfp_ref_clk \    -period 6.400 \    [get_ports sfp_clk_p]# ============================================================# Clock groups# ============================================================set_clock_groups -asynchronous \    -group [get_clocks sys_clk] \    -group [get_clocks sfp_ref_clk]
```

Но для generated clocks от IP/MMCM names могут отличаться. Поэтому после synthesis полезно проверить:

```
get_clocksreport_clocks
```

---

# 25. Проблема неправильных clock names

Частая ошибка:

```
set_clock_groups -asynchronous \    -group [get_clocks clk_a] \    -group [get_clocks clk_b]
```

Но `get_clocks clk_b` ничего не нашел.

В зависимости от настроек это может быть незаметно или дать warning.

Лучше проверять:

```
get_clocksget_clocks -quiet clk_b
```

И использовать строгие проверки в Tcl scripts:

```
set c0 [get_clocks -quiet clk_a]if {[llength $c0] == 0} {    error "Clock clk_a not found"}
```

---

# 26. `report_clock_interaction`

`report_clock_interaction` полезен, чтобы увидеть отношения между clock pairs:

```
timedignoredasynchronousno pathsunsafe / unconstrainted-looking interactions
```

AMD рекомендует использовать Clock Interaction Report для поиска asynchronous clocks, у которых отсутствуют нужные timing exceptions.

Команда:

```
report_clock_interaction
```

Практически это один из первых отчетов, который стоит смотреть, когда в проекте несколько clocks.

---

# 27. `report_cdc`

`report_cdc` — главный отчет по CDC-структурам.

Команда:

```
report_cdc
```

Или с файлом:

```
report_cdc -file cdc_report.rpt
```

AMD указывает, что `report_cdc` показывает CDC paths в synthesized или implemented design и анализирует paths между asynchronous clocks, clocks without common period, а также synchronous paths, проигнорированные через false path или `set_max_delay -datapath_only`.

---

# 28. Что показывает `report_cdc`

Отчет может показать:

```
safe synchronizermissing synchronizerunknown CDC logiccombinational logic before synchronizermulti-bit CDC problemreconvergence issueCDC bus issuereset-related CDC
```

Идея не в том, чтобы “убрать все строки отчета”, а в том, чтобы каждая CDC-точка была понятна и либо исправлена, либо осознанно признана корректной.

---

# 29. Timing clean ≠ CDC clean

Проект может иметь:

```
report_timing_summary: no violations
```

и при этом иметь опасные CDC.

Почему?

Потому что `set_clock_groups` или `set_false_path` могли убрать междоменные paths из timing analysis.

Поэтому для multi-clock дизайна нужно смотреть минимум два типа отчетов:

```
timing reportsCDC reports
```

---

# 30. CDC clean ≠ functionally correct

Даже если `report_cdc` не показывает Critical, дизайн может быть логически неверен.

Например:

```
toggle synchronizer корректен структурно,но source генерирует события слишком часто
```

Или:

```
FIFO корректен,но packet sideband не передан
```

Vivado может не знать protocol-level intention.

Поэтому CDC проверяется тремя слоями:

```
RTL reviewVivado report_cdcsimulation/formal/protocol checks
```

---

# 31. TIMING-9 / Unknown CDC Logic

Vivado может предупреждать о неизвестной CDC-логике, когда видит asynchronous crossing через `set_false_path`, `set_clock_groups` или `set_max_delay -datapath_only`, но не видит double-register synchronizer на capture side. AMD рекомендует в такой ситуации запускать `report_cdc` для полного анализа и рассмотреть использование XPM_CDC.

Практический смысл:

```
если constraint говорит “это CDC”,а структура не похожа на безопасный CDC,Vivado может выдать warning.
```

Такой warning нельзя просто игнорировать.

---

# 32. Constraints для two-flop synchronizer

Для обычного single-bit synchronizer между полностью asynchronous clocks часто делают:

```
set_clock_groups -asynchronous \    -group [get_clocks clk_src] \    -group [get_clocks clk_dst]
```

А в RTL:

```
(* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)reg [1:0] sync_ff;
```

Vivado видит:

```
path ignored by clock groupsstructure recognized as synchronizer
```

Но если используется XPM_CDC, нужно учитывать его собственные constraints и не перекрывать их слишком широким `set_clock_groups`.

---

# 33. Constraints для async FIFO

Для async FIFO обычно:

```
write clock и read clock описаны как clocks;отношение между ними описано как asynchronous, если они unrelated;FIFO реализован через XPM/IP или проверенный RTL;report_cdc не должен показывать опасные crossing внутри FIFO.
```

Если FIFO — XPM/IP, не надо вручную лезть во внутренние paths без необходимости.

Если FIFO самодельный, нужно быть намного внимательнее:

```
Gray-code pointer pathssynchronizer attributesmax delay / bus skew considerationsfull/empty logicreset logic
```

---

# 34. Constraints для Gray-code crossing

Для Gray-code bus нельзя просто думать:

```
это же Gray code, значит всё безопасно
```

Gray code помогает тем, что за один increment меняется один bit, но physical routing всё равно важен.

Для самодельных Gray-code crossings могут понадобиться специальные constraints, чтобы ограничить datapath delay/skew.

Идея:

```
все bits Gray bus должны доходить до destination synchronizers достаточно согласованно,чтобы protocol assumption оставался верным.
```

В Vivado для таких задач часто применяют `set_max_delay -datapath_only` на соответствующие CDC paths, но точные значения зависят от архитектуры и частот.

---

# 35. Constraints для handshake CDC

Handshake обычно имеет два направления:

```
req: clk_src -> clk_dstack: clk_dst -> clk_src
```

Если clocks asynchronous, оба направления должны быть покрыты constraints.

Например:

```
set_clock_groups -asynchronous \    -group [get_clocks clk_src] \    -group [get_clocks clk_dst]
```

И RTL должен содержать synchronizers:

```
req synchronizer в clk_dstack synchronizer в clk_src
```

Constraint не проверит, что source держит data stable до ack. Это уже protocol-level property.

---

# 36. Constraints для pulse/toggle CDC

Для toggle synchronizer:

```
toggle bit пересекает clock domain как single-bit signal
```

Поэтому timing exception похож на single-bit synchronizer.

Но constraint не проверит rate limitation:

```
source не должен toggl-ить слишком быстро
```

Это проверяется RTL review или simulation/formal.

---

# 37. Multicycle path — не CDC constraint

`set_multicycle_path` часто путают с CDC.

Multicycle подходит, когда clocks related и известно, что data передаются не каждый cycle.

Пример:

```
clk_a и clk_b синхронно связаныenable разрешает transfer раз в N cycles
```

Но для unrelated asynchronous clocks multicycle обычно не решает CDC.

Плохая идея:

```
set_multicycle_path 10 -from [get_clocks clk_a] -to [get_clocks clk_b]
```

если `clk_a` и `clk_b` unrelated.

Для настоящего CDC нужны synchronizer/FIFO/handshake и соответствующие exceptions.

---

# 38. Input/output delay — это не CDC constraints

`set_input_delay` и `set_output_delay` описывают timing относительно внешнего интерфейсного clock.

Они не предназначены для внутреннего CDC между двумя FPGA clock domains.

Если внешний сигнал асинхронен к FPGA clock, его обычно не описывают как обычный synchronous input interface; его вводят через synchronizer или специальный receiver.

Пример:

```
button inputexternal interruptasync GPIO
```

Для них важнее:

```
synchronizerdebounce/filteredge detection after sync
```

---

# 39. Не скрывать violations constraints-ами

Плохая практика:

```
есть failing timing path между clk_a и clk_bдобавили set_false_pathtiming стал clean
```

Если путь действительно CDC и в RTL есть правильный synchronizer — нормально.

Если путь был обычной передачей data/control без CDC — это сокрытие ошибки.

Правильный порядок:

```
1. определить, что это за path;2. понять clock relationship;3. исправить RTL CDC architecture;4. только потом добавить constraints;5. проверить report_cdc.
```

---

# 40. Waivers

Иногда `report_cdc` показывает violation, который разработчик считает допустимым.

В Vivado можно делать waivers, но это нужно использовать аккуратно.

Хороший waiver должен иметь объяснение:

```
почему это безопасно;какой protocol это гарантирует;где это проверяется;почему warning не требует RTL-исправления.
```

Плохой waiver:

```
просто чтобы отчет стал зеленым
```

CDC waivers без инженерного объяснения опасны.

---

# 41. Практический flow для CDC constraints

Хороший порядок работы:

```
1. Выписать все clocks проекта.2. Разделить clocks на related/asynchronous/exclusive.3. Нарисовать CDC crossings.4. Для каждого crossing определить method: sync/FIFO/handshake/etc.5. В RTL явно использовать CDC modules или XPM.6. В XDC описать clock groups/exceptions.7. Проверить get_clocks/report_clocks.8. Запустить report_clock_interaction.9. Запустить report_cdc.10. Исправить Critical/Unknown crossings.11. Проверить, что timing clean не получен за счет сокрытия ошибок.
```

---

# 42. Минимальный пример для проекта с двумя domains

Допустим:

```
sys_clk = 200 MHzsfp_user_clk = 156.25 MHz
```

RTL:

```
status flags -> cdc_sync_bitevents       -> cdc_pulse_togglestreams      -> async FIFO
```

XDC:

```
create_clock -name sys_clk \    -period 5.000 \    [get_ports sys_clk_p]# sfp_user_clk может быть создан IP/generated clock,# поэтому имя нужно проверить через get_clocks/report_clocks.set_clock_groups -asynchronous \    -group [get_clocks sys_clk] \    -group [get_clocks sfp_user_clk]
```

Проверка:

```
report_clocksreport_clock_interactionreport_cdc -file cdc_report.rptreport_timing_summary
```

---

# 43. Checklist по CDC constraints

Для каждой пары clocks:

```
1. Они related или unrelated?2. Есть ли common source?3. Видит ли Vivado generated clocks?4. Есть ли paths между ними?5. Если paths есть, это synchronous transfer или CDC?6. Если CDC, какой RTL method используется?7. Нужен ли set_clock_groups?8. Или лучше set_false_path?9. Или нужен set_max_delay -datapath_only?10. Не перекрывает ли set_clock_groups более точные constraints?11. Что показывает report_clock_interaction?12. Что показывает report_cdc?13. Есть ли Unknown/Critical CDC?14. Нет ли multi-bit crossing без protocol?15. Нет ли combinational logic перед synchronizer?16. Нет ли waivers без объяснения?
```

---

# 44. Частые ошибки

## Ошибка 1

```
Поставить set_clock_groups и считать CDC решенным.
```

Нет. RTL должен быть правильным.

---

## Ошибка 2

```
Объявить related clocks asynchronous без причины.
```

Можно скрыть реальные timing paths.

---

## Ошибка 3

```
Сделать wildcard constraints слишком широко.
```

Можно отключить больше timing paths, чем планировалось.

---

## Ошибка 4

```
Использовать set_false_path, чтобы убрать violation, не проверив RTL.
```

Это может спрятать реальную ошибку.

---

## Ошибка 5

```
Не запускать report_cdc.
```

Timing clean не означает CDC clean.

---

## Ошибка 6

```
Игнорировать предупреждения Unknown CDC Logic.
```

Это часто признак того, что constraint есть, а безопасной CDC-структуры Vivado не видит.

---

## Ошибка 7

```
Не проверять имена clocks после synthesis.
```

Constraints могут не примениться к реальным clock objects.

---

## Ошибка 8

```
Использовать multicycle вместо CDC.
```

Для unrelated clocks это неправильная идея.

---

# 45. Главное резюме

CDC constraints — это способ описать Vivado, как анализировать paths между clock domains.

Главные правила:

```
1. Сначала RTL CDC architecture, потом constraints.2. set_clock_groups отключает timing между clock groups.3. set_false_path точечно отключает paths, часто в одном направлении.4. set_max_delay -datapath_only может ограничивать physical datapath delay без обычного skew analysis.5. Related clocks нельзя автоматически объявлять asynchronous.6. Asynchronous clocks требуют CDC-структур и timing exceptions.7. Timing clean не означает CDC clean.8. report_cdc обязателен для multi-clock проекта.9. Широкие constraints могут скрыть ошибки.10. XPM/IP CDC structures лучше не перекрывать необдуманными global exceptions.
```

Короткая формула:

```
RTL делает CDC безопасным.XDC говорит Vivado, как не анализировать CDC как обычный synchronous timing.report_cdc проверяет, похожа ли структура на безопасный CDC.
```