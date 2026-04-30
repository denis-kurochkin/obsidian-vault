# MTBF concept внутри CDC

**MTBF** — это **Mean Time Between Failures**, то есть среднее время между отказами.

В контексте **Clock Domain Crossing** MTBF обычно означает:

> как часто metastability в synchronizer может привести к реальной ошибке в пользовательской логике.

Это не функциональный параметр RTL, а вероятностная характеристика физического поведения триггеров.

---

# 1. Зачем вообще нужен MTBF

Когда сигнал пересекает clock domain, принимающий триггер может захватить его в момент изменения.

Например:

```
clk_a domain                      clk_b domain

signal_a  --------------------->  FF in clk_b
```

Если `signal_a` меняется слишком близко к фронту `clk_b`, нарушаются условия:

```
setup time
hold time
```

Триггер может попасть в **metastable state**.

MTBF нужен, чтобы оценить:

```
насколько редко metastability приведет к ошибке
```

---

# 2. Metastability нельзя полностью исключить

Это ключевая мысль.

Synchronizer не делает metastability невозможной.

Он только делает вероятность ее распространения дальше очень маленькой.

Даже хороший two-flop synchronizer:

```
async_signal -> FF1 -> FF2 -> dst_logic
```

не дает математической гарантии “ошибки никогда не будет”.

Он дает очень высокую вероятность, что `FF1` успеет стабилизироваться до того, как его значение захватит `FF2`.

---

# 3. Что означает “failure” в MTBF

Failure в CDC-контексте — это не обязательно поломка FPGA.

Это может быть:

```
неправильно принятый flag
потерянный event
лишний event
неправильный переход state machine
ложный старт операции
ошибка full/empty внутри FIFO
редкий зависон control logic
```

То есть failure — это момент, когда metastability стала видимой на уровне логики.

---

# 4. Почему обычная RTL simulation не показывает MTBF

RTL simulation работает с дискретными значениями:

```
0
1
X
Z
```

Но metastability — это аналоговое состояние триггера.

В реальном кристалле выход триггера может некоторое время находиться между логическим `0` и `1`.

Обычная simulation этого не моделирует.

Поэтому simulation может показать, что CDC-код “работает”, но это не доказывает хороший MTBF.

---

# 5. Классический two-flop synchronizer

Базовая схема:

```
async_signal -> FF1 -> FF2 -> synced_signal
                    clk_dst
```

`FF1` принимает на себя риск metastability.

`FF2` защищает остальную логику.

Смысл:

```
если FF1 стал metastable,
у него есть почти один период clk_dst,
чтобы стабилизироваться до захвата FF2
```

---

# 6. Почему одного FF недостаточно

Плохой вариант:

```
async_signal -> FF1 -> dst_logic
```

Если `FF1` стал metastable, пользовательская логика сразу видит его выход.

Это может привести к тому, что разные элементы логики интерпретируют сигнал по-разному.

Например:

```
одна часть схемы увидела 0
другая часть схемы увидела 1
```

или сигнал стабилизировался слишком поздно и нарушил timing уже дальше по цепочке.

---

# 7. Что делает второй FF

Второй FF не “исправляет” первый.

Он просто ждет еще один такт.

```
FF1 получает async input
FF2 получает уже задержанную версию FF1
```

Если metastability в `FF1` успела разрешиться, `FF2` получает нормальный `0` или `1`.

Если не успела — возможен отказ.

MTBF как раз оценивает вероятность этого редкого случая.

---

# 8. Упрощенная формула MTBF

Часто MTBF описывают формулой такого вида:

```
MTBF ≈ exp(Tresolve / τ) / (Fclk * Fdata * C)
```

Где:

|Параметр|Смысл|
|---|---|
|`Tresolve`|время на стабилизацию metastability|
|`τ`|технологический параметр триггера|
|`Fclk`|частота принимающего clock|
|`Fdata`|частота переключения входного сигнала|
|`C`|технологическая/эмпирическая константа|

На практике важнее не сама формула, а зависимости.

---

# 9. Главная зависимость

Самое важное:

```
MTBF экспоненциально зависит от Tresolve
```

То есть небольшое увеличение времени на стабилизацию может очень сильно увеличить MTBF.

Именно поэтому добавление третьего FF иногда дает огромный выигрыш по надежности.

---

# 10. Что такое `Tresolve`

`Tresolve` — это время, которое есть у metastable-состояния, чтобы стабилизироваться.

Для two-flop synchronizer грубо:

```
Tresolve ≈ Tclk_dst - routing_delay - setup_time_FF2 - clock_uncertainty
```

То есть это не весь период clock.

Из периода вычитаются:

```
задержка маршрута FF1 -> FF2
setup time второго FF
clock skew / uncertainty
внутренние задержки
```

Поэтому физическая реализация synchronizer важна.

---

# 11. Почему placement влияет на MTBF

Если `FF1` и `FF2` стоят рядом, routing delay между ними маленький.

```
FF1 -> короткий routing -> FF2
```

Тогда `Tresolve` больше.

Если Vivado разместил их далеко:

```
FF1 -> длинный routing -> FF2
```

routing delay больше, `Tresolve` меньше, MTBF хуже.

Поэтому synchronizer — это не только RTL-шаблон, но и физическая структура, которую нужно правильно реализовать.

---

# 12. Зачем нужен `ASYNC_REG`

В Vivado synchronizer-регистры обычно помечают так:

```verilog
(* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)
reg [1:0] sync_ff;
```

`ASYNC_REG` сообщает инструменту:

```
это не обычная pipeline-цепочкаэто synchronizer chain
```

Практический эффект:

```
Vivado лучше распознает CDC structure
Vivado старается размещать FF stages близко
уменьшается routing delay
увеличивается Tresolve
улучшается MTBF
```

---

# 13. Зачем нужен `SHREG_EXTRACT = "NO"`

Короткая цепочка регистров может быть оптимизирована в shift register LUT / SRL.

Для обычного shift register это нормально.

Для synchronizer — плохо.

CDC synchronizer должен быть реализован именно на flip-flops:

```
FF -> FF
```

а не как LUT-based shift structure.

Поэтому часто добавляют:

```verilog
(* SHREG_EXTRACT = "NO" *)
```

---

# 14. Почему `Fclk` влияет на MTBF

`Fclk` — это частота принимающего clock domain.

Чем выше частота, тем меньше период clock.

Например:

```
100 MHz -> 10 ns
250 MHz -> 4 ns
500 MHz -> 2 ns
```

При 500 MHz у metastability намного меньше времени на стабилизацию между `FF1` и `FF2`.

Поэтому быстрые destination domains требуют более аккуратного CDC.

---

# 15. Почему `Fdata` влияет на MTBF

`Fdata` — это не обязательно частота source clock.

Это частота реальных изменений async-сигнала.

Например:

```
clk_src = 100 MHz
signal_src меняется 1 раз в секунду
```

Тогда `Fdata` примерно 1 Hz, а не 100 MHz.

Но если сигнал toggles почти каждый такт:

```
clk_src = 100 MHz
signal_src меняется ~50 MHz
```

риск metastability events намного выше.

Чем чаще сигнал меняется, тем больше попыток попасть около фронта `clk_dst`.

---

# 16. Status flag и fast toggle — это разные риски

Сравним два crossing:

```
pll_locked    меняется редко
data_toggle   меняется часто
```

Оба могут идти через two-flop synchronizer.

Но MTBF у них будет разный, потому что частота переключений разная.

Редкий status flag обычно безопаснее.

Частый toggle требует большего внимания.

---

# 17. Почему нельзя механически считать все CDC одинаковыми

Два crossing могут выглядеть одинаково в RTL:

```
signal -> 2 FF synchronizer
```

Но иметь разный риск.

Риск зависит от:

```
частоты dst clock
частоты изменения input signal
качества placement
количества synchronizer stages
технологии FPGA
важности сигнала
числа таких crossings в системе
```

Поэтому CDC-review должен учитывать не только структуру, но и смысл сигнала.

---

# 18. Два FF или три FF

Обычные варианты:

```
2 FF — базовый synchronizer
3 FF — повышенная надежность
```

Two-flop:

```
async -> FF1 -> FF2 -> logic
```

Three-flop:

```
async -> FF1 -> FF2 -> FF3 -> logic
```

Третий FF дает metastability еще один такт на стабилизацию.

Цена:

```
+ выше MTBF- на 1 такт больше latency
```

---

# 19. Когда обычно хватает двух FF

Два FF часто используют для:

```
медленных status flags
configuration bits
link_up
locked
done
ready
error_seen
manual control signals
```

Условия:

```
сигнал меняется редко
latency 2 такта допустима
failure не приводит к catastrophic behavior
clock frequency не экстремально высокая
```

---

# 20. Когда стоит рассмотреть три FF

Три FF разумно использовать, если:

```
destination clock быстрый
сигнал часто переключается
устройство должно работать годами
отказ дорогой
сигнал влияет на critical state machine
crossing используется во многих экземплярах
методология проекта требует 3-stage synchronizers
```

Пример:

```
control signal, который может сбросить/запустить крупный data path
```

лучше сделать надежнее, если extra latency не мешает.

---

# 21. Почему нельзя просто всегда ставить много FF

Можно подумать:

```
чем больше FF, тем лучше
```

С точки зрения MTBF — да.

Но есть цена:

```
увеличивается latency
усложняется control timing
можно нарушить protocol
может увеличиться resource usage
события могут приходить быстрее, чем chain успевает их отразить
```

Для level flag это обычно не проблема.

Для event/pulse/control protocol задержка может быть важной.

---

# 22. MTBF и latency

Synchronizer добавляет задержку в destination domain.

Для 2 FF:

```
примерно 2 такта clk_dst
```

Для 3 FF:

```
примерно 3 такта clk_dst
```

Но с точки зрения source domain задержка не является строго фиксированной, потому что clocks независимы.

Практически:

```
destination увидит изменение через несколько своих тактов
```

Поэтому CDC-logic должна быть tolerant к этой задержке.

---

# 23. MTBF не решает потерю pulse

Важно не путать две проблемы.

MTBF связан с metastability.

Но если короткий pulse просто не попал на фронт destination clock, это не MTBF-отказ.

Это ошибка CDC-протокола.

Например:

```
pulse_src длится 1 такт clk_src
clk_dst медленнее
```

Destination может вообще не увидеть pulse.

Даже идеальный synchronizer не исправит это.

Для pulse нужны:

```
toggle synchronizer
pulse stretching
handshake
FIFO
```

---

# 24. MTBF не делает data bus atomic

Еще одна частая ошибка:

```
поставим 2 FF на каждый бит data bus
```

Это может улучшить metastability risk каждого отдельного бита, но не гарантирует целостность слова.

Например:

```
data_src: 8'h0F -> 8'h10
```

Destination может временно увидеть смесь старых и новых битов.

MTBF тут не решает основную проблему.

Для data bus нужны:

```
handshake
Gray code
async FIFO
```

---

# 25. MTBF и reconvergence

Допустим, есть два связанных сигнала:

```
a_src
b_src
```

Они синхронизируются отдельно:

```
a_src -> sync -> a_dst
b_src -> sync -> b_dst
```

Даже если у каждого synchronizer хороший MTBF, destination может увидеть изменения в разные такты.

Это называется reconvergence problem.

MTBF говорит о вероятности metastability failure, но не гарантирует сохранение отношений между сигналами.

Если сигналы связаны логически, нужен protocol, а не независимые synchronizers.

---

# 26. MTBF одного synchronizer и MTBF всей системы

Если в проекте один synchronizer с очень большим MTBF — это хорошо.

Но в реальном FPGA могут быть сотни или тысячи CDC-points.

Упрощенно:

```
чем больше независимых CDC crossings,тем выше суммарная вероятность редкого отказа
```

То есть system-level MTBF хуже, чем MTBF одного crossing.

Практический вывод:

```
не плодить случайные CDCделать crossings централизованноиспользовать FIFO/handshake вместо набора отдельных flagsдокументировать CDC boundary
```

---

# 27. Пример system-level мышления

Плохо:

```
20 отдельных status/control сигналов идут между clk_a и clk_bкаждый синхронизируется где-то локально
```

Лучше:

```
сгруппировать control transferиспользовать handshakeиспользовать command/status register schemeиспользовать FIFO для событий
```

Так проще проверить архитектуру и меньше риск reconvergence/semantic ошибок.

---

# 28. MTBF и async FIFO

В async FIFO data bus не пересекает clock domain напрямую.

Данные записываются в memory на `wr_clk`, а читаются на `rd_clk`.

Через CDC проходят в основном:

```
write pointerread pointer
```

Обычно pointers передаются в Gray code через synchronizers.

MTBF async FIFO связан с надежностью этих pointer synchronizers.

Поэтому качественный async FIFO использует:

```
Gray code pointers2+ FF synchronizersправильную full/empty logicправильный reset behavior
```

---

# 29. Почему Gray code помогает, но не отменяет MTBF

Gray code делает так, что при переходе pointer меняется только один бит.

Это снижает риск получить произвольное мусорное значение pointer.

Но этот один бит все равно пересекает clock domain.

Значит, metastability всё еще возможна.

Поэтому Gray-code pointer всё равно проходит через synchronizer chain.

---

# 30. MTBF и toggle synchronizer

Toggle synchronizer передает событие как изменение состояния бита:

```
event -> toggle changes
```

Destination синхронизирует toggle-бит и делает edge detect.

MTBF здесь относится к synchronizer chain, через который проходит toggle.

Но есть отдельное ограничение:

```
если события идут слишком часто,destination может пропустить четное количество toggles
```

Это уже не MTBF-проблема, а protocol/rate limitation.

---

# 31. MTBF и source register

Желательно, чтобы сигнал перед CDC был зарегистрирован в source domain.

Хорошо:

```
always @(posedge clk_src) begin    flag_src <= condition_src;end
```

Потом:

```
cdc_sync_bit u_sync (    .clk_dst  (clk_dst),    .async_in (flag_src),    .sync_out (flag_dst));
```

Плохо:

```
assign flag_comb = a & b | c;
```

и сразу в synchronizer.

Почему хуже:

```
combinational glitches увеличивают effective Fdataбольше переключений — хуже MTBFсложнее CDC analysisсложнее понять semantics crossing
```

---

# 32. Effective toggle rate

Иногда кажется, что сигнал меняется редко, но из-за combinational logic он glitch-ит.

Например:

```
assign event_async = cond_a & cond_b;
```

Если `cond_a` и `cond_b` меняются не одновременно, на `event_async` могут быть короткие всплески.

С точки зрения synchronizer это дополнительные переключения.

То есть `Fdata` фактически выше.

Практическое правило:

```
перед CDC лучше регистрировать source signal
```

---

# 33. MTBF и fanout первого stage

Нельзя использовать выход первого FF synchronizer в логике.

Плохо:

```
assign logic_a = sync_ff1 & enable;
```

Правильно:

```
assign logic_a = sync_ff2 & enable;
```

`sync_ff1` — это именно та точка, где metastability наиболее вероятна.

Его fanout должен быть минимальным:

```
sync_ff1 -> только sync_ff2
```

---

# 34. MTBF и логика между stages

Плохо:

```
always @(posedge clk_dst) begin    sync_ff1 <= async_in;    sync_ff2 <= sync_ff1 & mask;end
```

Между stages synchronizer не должно быть logic.

Правильно:

```
always @(posedge clk_dst) begin    sync_ff1 <= async_in;    sync_ff2 <= sync_ff1;endassign synced_masked = sync_ff2 & mask;
```

Почему:

```
logic между stages добавляет delayуменьшает Tresolveможет ухудшить MTBFломает распознавание synchronizer chain
```

---

# 35. MTBF и duplicated synchronizers

Плохая структура:

```
async_signal -> sync A -> logic Aasync_signal -> sync B -> logic B
```

Даже если оба synchronizer имеют хороший MTBF, они могут выдать изменение в разные такты.

Лучше:

```
async_signal -> one synchronizer -> fanout synced_signal
```

То есть:

```
сначала синхронизировать один раз,потом раздавать уже synchronized signal
```

---

# 36. Как practically выбирать synchronizer depth

Хорошее инженерное правило:

|Ситуация|Типичный выбор|
|---|---|
|Медленный 1-bit status|2 FF|
|Редкий control flag|2 FF|
|Быстрый dst clock|3 FF|
|Часто переключающийся signal|3 FF или другой protocol|
|Safety/reliability-critical|3 FF + review|
|Data bus|Не FF-chain|
|Event stream|FIFO/handshake|
|Async FIFO pointers|XPM/IP default или 2–3 stages|

Это не абсолютный закон, но хороший старт.

---

# 37. Почему vendor primitives полезны

В Xilinx/Vivado есть XPM CDC primitives:

```
xpm_cdc_singlexpm_cdc_array_singlexpm_cdc_pulsexpm_cdc_handshakexpm_cdc_grayxpm_fifo_async
```

Они полезны потому что:

```
имеют правильные attributesлучше распознаются Vivadoуменьшают риск RTL-ошибкидают параметр DEST_SYNC_FF / CDC_SYNC_STAGESулучшают читаемость CDC intent
```

Но они не отменяют архитектурного выбора.

Например:

```
xpm_cdc_array_single не делает data bus atomic
```

---

# 38. Что можно проверять в simulation

MTBF напрямую simulation не проверяет.

Но simulation полезна для проверки CDC-протокола.

Проверять стоит:

```
события не теряютсянет лишних событийdata сохраняет порядокvalid/ready работает корректнонет overflow/underflowsource держит data stable во время handshakedestination корректно реагирует на latency
```

То есть simulation проверяет digital protocol, а не analog metastability.

---

# 39. Что можно проверять в Vivado

В Vivado полезно смотреть:

```
report_cdcreport_timingатрибуты ASYNC_REGplacement synchronizer stagesотсутствие logic между stagesclock domain definitions
```

Но важно помнить:

```
CDC constraints не делают crossing безопасным
```

Они только описывают timing-analysis intent.

Без правильного RTL-паттерна хороший XDC не спасет.

Подробно это лучше вынести в отдельную заметку **CDC constraints**.

---

# 40. Когда нужен настоящий расчет MTBF

Во многих FPGA-проектах MTBF вручную не считают для каждого crossing.

Обычно используют проверенную methodology.

Но расчет может понадобиться, если:

```
устройство работает годами без обслуживаниямного одинаковых CDC crossingsочень быстрые clocksочень высокая частота переключенийесть safety/reliability requirementнужно обосновать 2 FF против 3 FFзаказчик требует reliability analysis
```

Для точного расчета нужны данные:

```
параметры FPGA-технологииτ и Cчастоты clockчастота переключения signalrouting delayssetup timesчисло synchronizersтребуемый system-level MTBF
```

---

# 41. Ошибки понимания MTBF

## Ошибка 1

```
MTBF = 100 лет, значит 100 лет точно не будет ошибки
```

Нет.

Это статистическое среднее, а не гарантия.

---

## Ошибка 2

```
2 FF полностью убирают metastability
```

Нет.

Они резко уменьшают вероятность распространения metastability.

---

## Ошибка 3

```
если simulation прошла, MTBF хороший
```

Нет.

RTL simulation не моделирует физическое metastable decay.

---

## Ошибка 4

```
можно синхронизировать data bus побитно, если поставить 3 FF
```

Нет.

Это улучшает MTBF отдельных битов, но не гарантирует atomic data transfer.

---

## Ошибка 5

```
CDC-проблемы решаются только constraints
```

Нет.

CDC решается RTL-архитектурой. Constraints только помогают timing/tools правильно интерпретировать clocks.

---

# 42. Практический checklist по MTBF

Для каждого single-bit CDC crossing можно спросить:

```
1. Это действительно single-bit level signal?2. Как часто он переключается?3. Какой destination clock frequency?4. Достаточно ли 2 FF?5. Нужны ли 3 FF?6. Есть ли ASYNC_REG?7. Запрещен ли SRL extraction?8. Нет ли logic между stages?9. Используется ли только final stage?10. Source signal зарегистрирован?11. Нет ли duplicated synchronizers?12. Не является ли это частью multi-bit protocol?
```

Если хотя бы один ответ вызывает сомнение, crossing нужно пересмотреть.

---

# 43. Короткий практический пример

```
module cdc_sync_bit #(    parameter integer STAGES = 2,    parameter INIT_VALUE = 1'b0)(    input  wire clk_dst,    input  wire async_in,    output wire sync_out);    (* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)    reg [STAGES-1:0] sync_ff = {STAGES{INIT_VALUE}};    always @(posedge clk_dst) begin        sync_ff <= {sync_ff[STAGES-2:0], async_in};    end    assign sync_out = sync_ff[STAGES-1];endmodule
```

Для обычного status flag:

```
cdc_sync_bit #(    .STAGES(2)) u_link_up_sync (    .clk_dst  (sys_clk),    .async_in (link_up_async),    .sync_out (link_up_sys_clk));
```

Для более критичного crossing:

```
cdc_sync_bit #(    .STAGES(3)) u_critical_flag_sync (    .clk_dst  (fast_clk),    .async_in (critical_flag_async),    .sync_out (critical_flag_fast_clk));
```

---

# 44. Главное резюме

**MTBF** — это способ думать о надежности synchronizer с точки зрения вероятности metastability failure.

Главные идеи:

```
1. Metastability нельзя полностью исключить.2. Synchronizer снижает вероятность ее распространения.3. MTBF экспоненциально зависит от времени стабилизации.4. Больше Tresolve — лучше MTBF.5. Быстрый destination clock уменьшает Tresolve.6. Частые переключения async-сигнала увеличивают риск.7. 2 FF — базовый вариант.8. 3 FF — для быстрых, частых или критичных crossings.9. ASYNC_REG и placement реально влияют на MTBF.10. MTBF не решает pulse loss, data bus atomicity и protocol mistakes.
```

Коротко:

```
MTBF — это не “CDC работает или не работает”.MTBF — это “насколько редко правильно сделанный synchronizer может дать физически обусловленный отказ”.
```

Практическая формула выбора:

```
медленный level flag  -> 2 FFкритичный/быстрый flag -> 3 FFpulse/event           -> toggle/handshakedata word             -> handshakestream                -> async FIFO
```