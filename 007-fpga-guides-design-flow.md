# 1. Формулировка задачи и требований

На этом этапе определяется, **что именно должна делать FPGA**.

Что обычно фиксируют:

- функциональность блока или всего устройства;
    
- тактовые домены;
    
- интерфейсы: GPIO, SPI, UART, AXI, LVDS, DDR, transceiver, ADC/DAC и т.д.;
    
- производительность: частоты, пропускная способность, задержки;
    
- ограничения по ресурсам: LUT, FF, BRAM, DSP, transceiver;
    
- требования по reset, startup, power-up;
    
- требования по диагностике и отладке.
    

Что получается на выходе:

- краткая спецификация;
    
- схема верхнего уровня;
    
- карта интерфейсов;
    
- понимание, какие блоки будут свои, а какие IP.
    

Почему это важно:  
если это пропустить, потом появляются “правки по ходу”, от которых ломаются архитектура, тайминги и верификация.

---

# 2. Архитектура и разбиение на блоки

Дальше задача делится на **отдельные модули**.

Например:

- clock/reset subsystem;
    
- интерфейс с ADC;
    
- обработка данных;
    
- буферизация;
    
- интерфейс обмена с внешним миром;
    
- диагностические регистры;
    
- debug infrastructure.
    

На этом этапе решают:

- какие блоки будут работать в каких clock domain;
    
- где нужны FIFO, CDC, synchronizer;
    
- где нужен pipeline;
    
- какая будет структура top-level;
    
- какие блоки независимы и могут разрабатываться отдельно.
    

Что получается:

- block diagram;
    
- описание интерфейсов между модулями;
    
- предварительный план по clocking/reset.
    

Почему это важно:  
хорошая архитектура уменьшает количество проблем с CDC, таймингами и “склеиванием” блоков позже.

---

# 3. RTL design

Это написание кода на:

- Verilog / SystemVerilog / VHDL.
    

Здесь создаются:

- модули;
    
- регистровая логика;
    
- combinational logic;
    
- state machines;
    
- FIFO / counters / datapath;
    
- top-level integration.
    

Хорошая практика на этапе RTL:

- писать **синтезируемый, простой и читаемый код**;
    
- отделять datapath от control path;
    
- явно документировать интерфейсы;
    
- сразу соблюдать reset policy;
    
- не смешивать без необходимости разные clock domain;
    
- не надеяться, что “синтезатор сам разберется”.
    

Что уже стоит учитывать:

- latency;
    
- throughput;
    
- разрядности;
    
- переполнения;
    
- signed/unsigned;
    
- fanout;
    
- возможность pipeline.
    

Результат:

- RTL-исходники проекта.
    

---

# 4. Functional simulation

Это проверка, что логика **делает правильные вещи по функции**, еще без учета реальных задержек кристалла.

Обычно делается:

- testbench;
    
- stimulus;
    
- self-checking checks / assertions;
    
- моделирование нормальных и граничных сценариев.
    

Проверяют:

- правильность FSM;
    
- протоколы интерфейсов;
    
- корректность reset;
    
- формат данных;
    
- поведение на ошибках и краях.
    

Важно:  
на этом этапе **тайминги FPGA еще не проверяются**, проверяется именно функциональность RTL.

Результат:

- testbench;
    
- waveform;
    
- отчет, что модуль ведет себя как ожидается.
    

Это один из самых дешевых этапов исправления ошибок. Чем больше ошибок поймано здесь, тем меньше боли потом.

---

# 5. Lint / static checks / CDC / RDC

Параллельно или сразу после функциональной симуляции имеет смысл делать статический анализ.

Сюда относятся:

- lint: подозрительные конструкции, latch, width mismatch, unreachable branches;
    
- CDC: переходы между тактовыми доменами;
    
- RDC: reset domain crossing;
    
- design rule checks на уровне RTL.
    

Ищут:

- несинхронизированные single-bit crossing;
    
- передачу bus без handshake/FIFO;
    
- неправильную обработку async reset;
    
- неоднозначные конструкции в always block.
    

Почему это важно:  
многие ошибки CDC в обычной функциональной симуляции не видны вообще.

---

# 6. Constraints definition

Это один из ключевых этапов. Здесь описывается для инструментов, **как устроен внешний мир и какие требования к времени**.

Обычно задают:

- `create_clock`;
    
- generated clocks;
    
- input delay / output delay;
    
- false paths;
    
- multicycle paths;
    
- clock groups;
    
- pin assignment;
    
- I/O standards;
    
- transceiver/clock constraints;
    
- иногда физические ограничения размещения.
    

Примеры смысла:

- “этот вход приходит относительно такого-то внешнего такта”;
    
- “эти два домена асинхронны”;
    
- “этот путь не должен анализироваться как обычный timing path”;
    
- “этот порт должен сидеть на конкретном пине и стандарте”.
    

Очень важно:  
без корректных constraints timing report либо бесполезен, либо вводит в заблуждение.

---

# 7. Synthesis

Синтез переводит RTL в технологически-зависимую netlist для конкретной FPGA.

Что делает synthesis:

- разворачивает логику;
    
- оптимизирует выражения;
    
- выбирает LUT/FF/BRAM/DSP;
    
- удаляет неиспользуемую логику;
    
- строит netlist.
    

На этом этапе можно увидеть:

- warnings о trim/unused logic;
    
- inferred RAM/DSP;
    
- latch warnings;
    
- high fanout;
    
- resource utilization;
    
- первые timing estimates.
    

Результат:

- synthesized netlist;
    
- utilization report;
    
- synthesis timing report.
    

Важно понимать:  
синтез уже может показать, что архитектура тяжёлая или код написан неудачно, но окончательный timing еще не здесь.

---

# 8. Post-synthesis analysis

После синтеза обычно смотрят:

- utilization;
    
- critical paths;
    
- clock summary;
    
- inferred structures;
    
- warnings.
    

Здесь часто выявляют:

- лишнюю логику;
    
- неверно распознанные RAM/DSP;
    
- слишком длинные combinational path;
    
- проблемы reset/enable;
    
- случайно неиспользуемые сигналы и блоки.
    

Это хорошая точка для ранней коррекции RTL, до place and route.

---

# 9. Implementation (place & route)

Это уже физическая реализация в кристалле.

Состоит из:

- opt_design;
    
- placement;
    
- physical optimization;
    
- routing.
    

Что делает implementation:

- размещает логические элементы по кристаллу;
    
- прокладывает соединения;
    
- оценивает реальные задержки;
    
- пытается выполнить timing constraints.
    

Именно здесь становятся видны:

- реальные setup/hold slack;
    
- congestion;
    
- длинные маршруты;
    
- плохая floorplan-структура;
    
- последствия неудачной архитектуры.
    

Результат:

- implemented design;
    
- routed netlist;
    
- timing summary;
    
- DRC;
    
- power/utilization reports.
    

---

# 10. Static Timing Analysis (STA)

Это не одна кнопка, а отдельный смысловой этап анализа timing.

Проверяют:

- WNS/TNS;
    
- setup violations;
    
- hold violations;
    
- clock interaction;
    
- unconstrained paths;
    
- path groups;
    
- timing between I/O and internal logic;
    
- CDC-related exceptions correctness.
    

Очень важно смотреть не только “есть ли violations”, но и:

- **какие именно пути критические**;
    
- это datapath, control, reset, CDC, I/O?;
    
- действительно ли путь должен анализироваться?;
    
- не забыты ли constraints?
    

Типовые причины плохого timing:

- длинная combinational logic;
    
- большие mux;
    
- плохое pipeline;
    
- высокий fanout;
    
- неудачный placement;
    
- неправильные constraints;
    
- слишком высокая target frequency.
    

---

# 11. Functional + timing-aware verification after implementation

После реализации часто выполняют:

- post-synthesis simulation;
    
- post-implementation timing simulation.
    

На практике timing simulation используют не всегда, потому что она тяжелая и медленная. Но она полезна:

- для сложных startup/reset сценариев;
    
- для интерфейсов с жесткими временными требованиями;
    
- при подозрениях на расхождение между RTL-моделью и реальной сетью после оптимизаций.
    

Чаще в индустрии основной упор делают на:

- хороший RTL simulation;
    
- CDC checks;
    
- корректные constraints;
    
- STA.
    

---

# 12. Bitstream generation

Когда дизайн реализован и timing/DRC приемлемы, генерируется:

- `.bit`, `.bin`, `.mcs` или другой файл конфигурации.
    

Это уже образ, который можно загружать в FPGA или во внешнюю flash.

---

# 13. Programming device

Загрузка в:

- FPGA напрямую по JTAG;
    
- configuration flash;
    
- boot memory.
    

На этом этапе проверяют:

- стартует ли устройство;
    
- lock-ятся ли PLL/MMCM;
    
- приходит ли reset как надо;
    
- есть ли ожидаемая активность интерфейсов;
    
- не сломан ли pinout.
    

---

# 14. On-board debugging / bring-up

Это этап “оживления” железа.

Используют:

- ILA / VIO;
    
- LED / test pins;
    
- встроенные status registers;
    
- loopback;
    
- pattern generators/checkers;
    
- внешние приборы: scope, LA, BERT, protocol analyzer.
    

Проверяют:

- тактирование;
    
- reset sequence;
    
- обмен по интерфейсам;
    
- alignment training / link up;
    
- реальные данные;
    
- редкие состояния, которые не видны в симуляции.
    

Именно здесь часто всплывают:

- ошибки pinout;
    
- polarity;
    
- reset timing;
    
- неверные assumptions о внешнем устройстве;
    
- board-level issues.
    

---

# 15. Iteration and closure

Реальный FPGA flow почти никогда не линейный. Обычно это цикл:

**RTL → sim → synth → impl → timing/debug → исправления → повтор**

Финальная стадия называется:

- **timing closure** — когда дизайн стабильно выполняет timing;
    
- **functional closure** — когда выполнены требования по функции;
    
- **integration closure** — когда всё вместе работает в системе.
    

---

# Как это выглядит в виде короткой цепочки

Можно свести flow к такой схеме:

1. Требования
    
2. Архитектура
    
3. RTL
    
4. Functional simulation
    
5. CDC/lint/static checks
    
6. Constraints
    
7. Synthesis
    
8. Post-synth analysis
    
9. Implementation
    
10. STA
    
11. Bitstream
    
12. Bring-up/debug on hardware
    
13. Iteration until closure
    

---

# Что важно понимать про “правильный” процесс

## 1. Это не строго последовательный конвейер

Ты не пишешь “сразу весь проект полностью”, а работаешь **итерациями**.

Правильнее так:

- взять блок;
    
- сделать ему RTL;
    
- проверить симуляцией;
    
- встроить в систему;
    
- проверить синтез/constraint/timing на текущем уровне;
    
- идти дальше.
    

## 2. Не все warnings одинаково важны

На промежуточных этапах нормально видеть:

- trim unused logic;
    
- dangling outputs;
    
- incomplete integration artifacts.
    

Ненормально игнорировать:

- latch;
    
- multiple drivers;
    
- CDC issues;
    
- unconstrained clocks/paths;
    
- реальные setup/hold violations на тех частях, которые уже должны работать.
    

## 3. Constraints — часть дизайна, а не “дописать в конце”

Ограничения не нужно откладывать полностью на финал.  
Минимально необходимые constraints должны появляться **сразу, как только появляется соответствующий интерфейс или clock**.

## 4. Timing closure начинается еще на RTL

Если архитектура плохая, place&route уже не спасет.  
Pipeline, CDC, fanout, выбор частот — это architectural decisions, не только implementation issue.

---

# Как принято работать в индустрии

Обычно используют подход **incremental, block-based, verification-driven**:

- сначала определяют архитектуру;
    
- блоки проектируют отдельно;
    
- каждый блок проходит RTL sim;
    
- интеграция идёт постепенно;
    
- constraints вводятся по мере появления интерфейсов;
    
- CDC и lint гоняются регулярно;
    
- implementation делают не на каждом коммите, но регулярно;
    
- timing closure выполняют не в самом конце “с нуля”, а по мере роста проекта.
    

То есть не так:

> сначала напишем всё, потом как-нибудь разберемся

И не так:

> после каждого маленького изменения будем фанатично добиваться идеального timing всего проекта

А так:

> развиваем проект поэтапно, поддерживая его в “контролируемо рабочем” состоянии

---

# Практический flow для реальной работы

Для небольшого/среднего FPGA-проекта удобно так:

## Для каждого нового блока

1. Написать спецификацию блока на полстраницы
    
2. Сделать RTL
    
3. Написать testbench
    
4. Прогнать functional simulation
    
5. Проверить lint/CDC
    
6. Встроить в top
    
7. Добавить необходимые constraints
    
8. Прогнать synthesis
    
9. Посмотреть utilization/warnings
    
10. Периодически прогонять implementation и timing
    

## На уровне всего проекта

- держать top всегда собираемым;
    
- не копить много непроверенных изменений;
    
- регулярно смотреть timing summary;
    
- регулярно чистить warnings;
    
- иметь debug hooks для железа.
    

---

# Где чаще всего ломается flow

Самые частые ошибки:

- писать RTL без нормальной симуляции;
    
- не определять clock/reset architecture заранее;
    
- откладывать constraints на самый конец;
    
- игнорировать CDC;
    
- судить о проекте только по behavioral simulation;
    
- пытаться чинить timing только ткл-опциями, а не RTL/архитектурой;
    
- интегрировать сразу слишком большие куски без локальной проверки.
    

---

# Если привязать к Vivado, то flow примерно такой

1. Создание проекта / добавление RTL
    
2. Добавление IP
    
3. Добавление XDC
    
4. Simulation Sources
    
5. Run Synthesis
    
6. Check synthesized design
    
7. Run Implementation
    
8. Check timing/utilization/DRC
    
9. Generate Bitstream
    
10. Program Device
    
11. Debug with ILA/VIO / reports
    
12. Repeat
    

Но важно помнить: **Vivado flow — это только инструментальная оболочка**, а не весь design flow. Настоящий flow начинается еще до написания RTL.

---

# Самая полезная ментальная модель

Думай о разработке FPGA как о трех параллельных потоках:

## 1. Функция

Делает ли дизайн то, что должен?

Инструменты:

- RTL simulation
    
- testbench
    
- assertions
    

## 2. Временные требования

Успевает ли он это делать на нужной частоте и по нужным интерфейсам?

Инструменты:

- XDC
    
- synthesis/implementation
    
- STA
    

## 3. Интеграция с реальным железом

Правильно ли дизайн взаимодействует с платой и внешними устройствами?

Инструменты:

- pin constraints
    
- ILA/VIO
    
- bring-up
    
- lab debug
    

Хороший flow — это когда все три потока развиваются одновременно, а не по очереди.