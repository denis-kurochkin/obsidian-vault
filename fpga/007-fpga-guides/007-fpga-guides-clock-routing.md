# RTL / FPGA Clock Routing

## Что это такое

**Clock routing** в FPGA — это способ, которым тактовые сигналы распространяются внутри кристалла от источника clock к регистрам, блокам памяти, DSP, SERDES, I/O и другим синхронным элементам.

Проще говоря, это ответ на вопрос:

> по каким ресурсам внутри FPGA идет clock и почему для него нельзя относиться к маршрутизации так же, как к обычному data signal.

Clock routing — это не просто "провести провод от точки A к точке B".

Для clocks в FPGA обычно существуют специальные ресурсы:

- dedicated clock networks;
    
- глобальные и региональные clock lines;
    
- специальные clock buffers;
    
- выделенные маршруты для низкого skew и предсказуемой задержки.
    

Именно поэтому тема clock routing — это часть общей clocking architecture, а не просто detail implementation.

---

## Интуитивная идея

Обычный логический сигнал обычно идет туда, куда нужно функционально, и инструмент сам ищет доступный routing path.

Clock — совсем другой класс сигнала.

Почему:

- он должен приходить к очень большому числу регистров;
    
- желательно почти одновременно;
    
- с минимальным skew;
    
- с контролируемой задержкой;
    
- с низкой чувствительностью к случайной реализации маршрута.
    

Если data signal может терпеть обычную fabric routing, то clock обычно требует **специальной инфраструктуры распределения**.

То есть clock routing — это про массовое, точное и предсказуемое разведение сигнала времени по кристаллу.

---

## Главная идея

Clock routing нужен для того, чтобы сделать clock:

- глобально или регионально доступным;
    
- достаточно одновременным в разных точках дизайна;
    
- пригодным для timing analysis;
    
- устойчивым к вариациям маршрута.
    

Главный practical смысл такой:

> clock должен идти по специализированным clock resources, а не по случайной обычной логической маршрутизации.

Именно это позволяет FPGA оставаться предсказуемой синхронной системой.

---

## Чем clock routing отличается от обычной routing

Это один из самых важных вопросов темы.

### Обычная routing

Используется для:

- combinational nets;
    
- control signals;
    
- datapath;
    
- локальных связей между блоками.
    

Здесь допустимо, что:

- задержки различаются;
    
- маршрут зависит от placement;
    
- нагрузка сильно различается;
    
- сигнал идет только в ограниченное число точек.
    

### Clock routing

Используется для:

- clocks;
    
- иногда clock-like signals, признанных инструментом как специальные;
    
- массовой доставки clock к большому числу sequential elements.
    

Здесь нужно:

- минимальный skew;
    
- предсказуемая задержка;
    
- хорошая повторяемость;
    
- специальная поддержка timing analysis;
    
- использование dedicated clock resources.
    

Именно поэтому clock routing — это отдельная архитектурная категория.

---

## Почему обычная routing плоха для clocks

Если clock идет как обычный fabric signal, появляются риски:

- большой skew;
    
- непредсказуемая задержка;
    
- сильная зависимость от placement/routing;
    
- проблемы timing closure;
    
- ухудшение качества clock distribution;
    
- большее потребление ресурсов и возможные tool warnings.
    

То есть даже если периодический сигнал "функционально доходит", это еще не означает, что он годится как настоящий system clock.

Это одна из центральных идей всей темы.

---

## Основные элементы clock routing в FPGA

Конкретные названия зависят от семейства FPGA, но логика обычно общая.

## 1. Global clock networks

Это специальные сети, предназначенные для распространения clocks по большой части кристалла или по всему кристаллу.

Они нужны, когда clock используется широко:

- много регистров;
    
- несколько крупных подсистем;
    
- весь top-level domain.
    

Их цель:

- минимизировать skew;
    
- дать clock доступ ко многим регионам FPGA;
    
- обеспечить хороший timing baseline.
    

---

## 2. Regional clock networks

Иногда clock нужен не во всей FPGA, а только в определенной области.

Тогда могут использоваться региональные clock resources.

Это полезно, когда:

- домен локален;
    
- не нужно грузить глобальную сеть;
    
- важна локальная эффективность и ограничение области распространения clock.
    

---

## 3. Clock buffers

Clock buffers — это специальные буферы, которые подключают источник такта к clock routing infrastructure.

Их роль:

- ввести clock в dedicated clock network;
    
- обеспечить правильное распределение;
    
- иногда — деление, gating, selection или regional driving в пределах архитектуры FPGA.
    

Конкретные типы зависят от vendor/family, но общая мысль одна:

**clock buffer — это не просто обычный буфер, а часть clock distribution fabric.**

---

## 4. Clock-capable input paths

Внешний clock обычно подается не на произвольный пин, а на специальные clock-capable inputs.

Это важно, потому что путь от входа к clock network тоже является частью архитектуры clock routing.

Если clock завести не туда, можно получить ограничения или ухудшение clock quality/routing flexibility.

---

## Clock routing и clock buffers

Clock buffer обычно является границей между:

- источником clock;
    
- и dedicated distribution network.
    

Например:

- внешний oscillator приходит на clock-capable pin;
    
- дальше проходит через input buffer / clock buffer;
    
- потом разводится по global/regional network.
    

То же относится к generated clocks из PLL/MMCM: они обычно тоже дальше идут через clocking/routing resources, а не просто через fabric logic.

То есть хороший clock routing почти всегда включает правильное использование clock buffers.

---

## Clock routing и skew

Одна из главных причин существования dedicated clock routing — это борьба со skew.

**Clock skew** — это различие момента прихода одного и того же clock в разные точки дизайна.

Если skew большой:

- ухудшается setup/hold picture;
    
- timing становится менее предсказуемым;
    
- могут появляться трудные для анализа проблемы между регистрами.
    

Dedicated clock routing как раз и нужен, чтобы skew был малым и контролируемым.

То есть clock routing — это фактически архитектурный инструмент управления skew.

---

## Clock routing и insertion delay

Помимо skew важна и общая задержка clock distribution.

Clock приходит не мгновенно. От источника до регистра есть определенный путь.

Эта задержка должна быть:

- понятной timing tools;
    
- достаточно предсказуемой;
    
- совместимой с выбранной clock architecture.
    

Clock routing не обязательно делает задержку маленькой в абсолютном смысле, но делает ее **предсказуемой и хорошо анализируемой**.

А для timing это часто важнее, чем "минимально возможная" задержка сама по себе.

---

## Clock routing и generated clocks

Generated clocks тоже нуждаются в правильной маршрутизации.

Если clock создан через:

- PLL;
    
- MMCM;
    
- dedicated divider/buffer;
    

это еще не конец задачи. Дальше его нужно корректно подать в clock network.

То есть clock generation и clock routing — разные, но тесно связанные темы:

- generation отвечает за частоту/фазу/происхождение;
    
- routing отвечает за физическую доставку clock к нагрузкам.
    

---

## Clock routing и logic-generated clocks

Это очень важный practical case.

Если clock создается обычной RTL-логикой, например делителем на счетчике, то часто возникает вопрос:

- можно ли потом использовать этот сигнал как clock?
    

Проблема в том, что такой сигнал обычно не является "естественным" участником нормальной clock routing infrastructure.

В результате:

- он может идти по fabric;
    
- skew может быть плохим;
    
- timing analysis усложняется;
    
- tools могут ругаться или требовать специальных constraints;
    
- архитектура становится хуже.
    

Именно поэтому логическое деление clock и последующее использование как полноценного такта считается плохой практикой во многих FPGA-проектах.

---

## Clock routing и clock enable

Эта тема тесно связана с вопросом:

- нужен ли новый clock,
    
- или достаточно clock enable.
    

Если задача — просто обновлять часть логики реже, то часто лучше:

- оставить один нормальный clock по dedicated clock network;
    
- использовать CE-пульс внутри этого домена.
    

Так мы:

- не создаем новый fabric-routed pseudo-clock;
    
- не усложняем clock routing;
    
- не плодим clock domains;
    
- сохраняем более чистую timing architecture.
    

То есть хороший clock routing часто начинается с решения **не создавать лишний clock вообще**.

---

## Clock routing и clock domain architecture

Каждый новый real clock должен иметь внятный routing story:

- откуда он приходит;
    
- через какой buffer/resource входит в сеть;
    
- где он распространяется;
    
- какие блоки им питаются.
    

Если этого нет, clock domain architecture обычно слабая.

То есть clock routing — это не только implementation detail, но и часть архитектурного вопроса:

> насколько осмысленно и чисто вообще построены clock domains в проекте?

---

## Clock routing и floorplanning

Хотя пользователь не всегда делает явный floorplan, физическая близость блоков к clock resources и используемым регионам FPGA может иметь значение.

Особенно это важно для:

- крупных multi-clock designs;
    
- high-speed interfaces;
    
- regional clock usage;
    
- designs с жесткими timing margin.
    

Clock routing зависит от физической структуры кристалла, поэтому placement и routing не совсем независимы от архитектуры clock domains.

То есть хороший clocking design обычно учитывает физическую топологию хотя бы на концептуальном уровне.

---

## Clock routing и power/resource efficiency

Clock networks — это дорогие и важные ресурсы FPGA.

Если бездумно плодить clocks, можно:

- перегрузить clock resources;
    
- усложнить implementation;
    
- ухудшить power picture;
    
- усложнить timing closure.
    

Поэтому хорошие проекты обычно:

- минимизируют количество clocks;
    
- используют regional/global networks осознанно;
    
- не тратят clock routing на то, что можно решить CE или локальной логикой.
    

---

## Clock routing и gating

Clock gating в FPGA требует особой осторожности.

Наивный подход:

- взять clock;
    
- пропустить через combinational gate типа `clk & en`;
    
- использовать результат как новый gated clock.
    

Почему это плохо:

- появляются glitches;
    
- нарушается нормальная clock routing discipline;
    
- ухудшается skew/timing;
    
- сигнал может идти как обычная логика, а не как proper clock.
    

Если gating нужен, его обычно нужно делать через предназначенные для этого архитектурные механизмы vendor FPGA, а не обычной логикой.

Но во многих случаях снова оказывается, что лучше использовать CE, а не gating clock.

---

## Типичные ошибки

### Ошибка 1. Использовать fabric-routed signal как полноценный clock

Сигнал периодический, но route у него не clock-quality.

### Ошибка 2. Делить clock RTL-логикой и считать, что дальше это обычный новый domain

На практике это часто приводит к плохому clock routing.

### Ошибка 3. Игнорировать специальные clock-capable pins и buffers

Входной clocking path тоже важен.

### Ошибка 4. Создавать слишком много clocks без понимания, как они будут разводиться

Clock routing ресурсы конечны и архитектурно значимы.

### Ошибка 5. Делать наивный clock gating через combinational logic

Это почти всегда плохой путь.

### Ошибка 6. Считать, что если сигнал дошел функционально, значит routing достаточно хорош

Для clock этого критерия недостаточно.

### Ошибка 7. Не учитывать skew/insertion delay как часть архитектуры

Clock routing — это не только "есть соединение", но и качество временной доставки.

---

## Практические правила

### 1. Clock должен идти по dedicated clock resources

Это базовое правило.

### 2. Не использовать обычную fabric logic для построения "настоящих" clocks без очень веской причины и архитектурной поддержки

Иначе routing quality обычно хуже.

### 3. Минимизировать число real clocks

Каждый новый clock требует routing resources, constraints и domain discipline.

### 4. Если нужен более редкий функциональный шаг, сначала подумать про CE

Это часто лучше, чем новый clock.

### 5. Внешние clocks заводить через clock-capable paths

Начало clock route тоже важно.

### 6. Generated clock должен иметь не только generation story, но и routing story

Нужно понимать, как он дойдет до нагрузки.

### 7. Clock gating делать только архитектурно корректным способом

И во многих случаях избегать вовсе.

---

## Хорошая архитектурная картина

Полезный healthy pattern выглядит так:

### Шаг 1. Есть один или несколько valid clock sources

Например, внешние reference clocks или outputs PLL/MMCM.

### Шаг 2. Эти clocks входят в dedicated clock infrastructure через соответствующие buffers/resources

То есть clock сразу попадает на предназначенный для него путь.

### Шаг 3. Clock domains ограничены и осмысленны

Нет лишних pseudo-clocks.

### Шаг 4. Для медленной логики там, где возможно, используется clock enable

Чтобы не плодить дополнительные routed clocks.

### Шаг 5. Clock distribution понятна с точки зрения skew, reset, timing и constraints

Это и есть зрелая clock routing architecture.

---

## Пример хорошего мышления

Вместо вопроса:

- "можно ли этот toggling signal использовать как clock?"
    

лучше спрашивать:

- пойдет ли он по dedicated clock network;
    
- recognized ли он как proper clock architecture element;
    
- каков будет skew;
    
- есть ли лучший вариант через CE или PLL/MMCM output;
    
- как это повлияет на timing и CDC.
    

Такой подход гораздо ближе к реальной FPGA-инженерии.

---

## Clock routing и constraints

Хотя routing — это физическая тема, timing tools должны понимать clock architecture логически.

Для этого нужны корректные clock constraints:

- primary clocks;
    
- generated clocks;
    
- relationships between clocks;
    
- asynchronous groups там, где нужно.
    

Но важно помнить: constraint не превращает плохой fabric-routed pseudo-clock в хороший архитектурный clock.

Constraint помогает анализу, но не заменяет правильную физическую routing architecture.

Это очень важный практический момент.

---

## Главное, что нужно запомнить

**Clock routing** в FPGA — это специализированная доставка тактовых сигналов через dedicated clock resources с целью обеспечить низкий skew, предсказуемую задержку и корректную timing architecture.

Главный practical смысл темы:

- clock нельзя воспринимать как обычный signal;
    
- clocks должны идти по предназначенной для них инфраструктуре;
    
- fabric-routed pseudo-clocks почти всегда хуже настоящих clock paths;
    
- хороший clock routing тесно связан с clock generation, CE-vs-clock decisions, reset и CDC.
    

Золотое правило:

**если сигнал должен быть настоящим clock, у него должен быть настоящий clock routing path.**

---

## Краткая памятка

### Что такое clock routing

Специальная маршрутизация clocks внутри FPGA.

### Чем отличается от обычной routing

- меньше skew
    
- более предсказуемая задержка
    
- специальные сети и буферы
    
- ориентация на массовую доставку к sequential logic
    

### Плохие признаки

- fabric-routed pseudo-clock
    
- логическое деление clock
    
- combinational clock gating
    
- слишком много лишних clocks
    

### Хорошие признаки

- dedicated clock resources
    
- clock-capable inputs
    
- clock buffers
    
- минимум real clock domains
    
- CE там, где новый clock не нужен
    

### Золотое правило

**clock должен распространяться как clock, а не как случайный data net.**

---

## Мини-шаблон архитектурного мышления

Для каждого clock в проекте полезно явно ответить:

1. Откуда он приходит?
    
2. Через какой buffer/resource попадает в clock network?
    
3. Global или regional routing ему нужен?
    
4. Почему это именно real clock, а не задача для CE?
    
5. Каков ожидаемый skew/insertion story?
    
6. Какие блоки питаются этим clock?
    
7. Как этот clock constrained и как связан с другими domains?
    

Если на эти вопросы есть ясные ответы, clock routing architecture обычно находится на хорошем инженерном уровне.