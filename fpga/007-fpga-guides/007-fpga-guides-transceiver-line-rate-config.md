## Line rate config

**Line rate config** в контексте FPGA/Vivado и блока **Transceiver Configuration** — это настройка целевой **line rate** для GT-подсистемы. Это не просто поле в Wizard, куда вводится число в Gb/s, а решение о том, какую серийную скорость должен поддерживать link, какой **PLL** будет ее генерировать, какой **reference clock** для этого подойдет, какая **data width** потребуется внутри transceiver и с какой частотой потом будет работать параллельный интерфейс на стороне логики. В Vivado/AMD IP line rate задается в gigabits per second, а допустимый диапазон зависит от типа transceiver и может дополнительно ограничиваться выбранным device.

Главная мысль здесь такая: **line rate — это первичный параметр transceiver design.** Когда engineer выбирает line rate, он фактически задает основу для всей дальнейшей схемы clocking. Даже в IBERT AMD определяет “protocol” как комбинацию **line rate / data width / reference clock rate**, то есть line rate рассматривается не отдельно, а как центральная часть конфигурации.

### Почему это отдельная полноценная подтема

Внутри общего раздела **Transceiver Configuration** есть темы более обзорные — refclk, PLL, reset, buffers, debug. А **line rate config** — это уже узкая тема, потому что именно здесь сходятся почти все ключевые зависимости: protocol, кодирование, **PLL choice**, **reference clock**, **data width**, **TX/RX user clocks** и предельные возможности device. Поэтому ошибочно думать, что line rate — это просто “скорость линка”. На практике это параметр, который влияет на весь design вокруг GT: от Quad clocking и XDC до bring-up и достижимой частоты в обычной fabric logic.

### Что такое line rate в практическом смысле

В практическом смысле **line rate** — это **raw serial bit rate** на одной lane. AMD прямо формулирует это для Aurora 8B/10B: line rate — это **unencoded bit rate**, с которым данные передаются по serial link. Это важное уточнение, потому что пользователь часто мысленно смешивает **line rate** и полезную пропускную способность, а это не одно и то же. При наличии line coding часть полосы уходит в protocol overhead.

Например, для **64B/66B** overhead около 3%, а для **8B/10B** — 25%. AMD прямо указывает это в Aurora documentation. Значит, одинаковый nominal line rate дает разный payload throughput в зависимости от encoding scheme. Поэтому при line rate config надо думать не только “сколько Gb/s я хочу на проводе”, но и “какой полезный bandwidth я реально получу после этого кодирования”.

### С чего нужно начинать line rate config

Правильный порядок обычно такой: **полезная скорость / protocol target → тип coding → target line rate → PLL → compatible refclk → interface width / user clocks**. Такой порядок хорошо совпадает с тем, как AMD tools и IP сами смотрят на задачу. В Wizard line rate вводится как один из основных параметров, а дальше tool уже подбирает compatible clocking options, включая reference clock selections и QPLL-related settings. Для secondary QPLL PG182 прямо говорит, что после ввода **line rate of second core** инструмент показывает все compatible **actual reference clock frequency** options.

Из этого следует важная вещь: **не надо начинать с refclk, если line rate еще не определен.** Line rate — более фундаментальная величина для datapath transceiver. Уже от нее дальше выбираются диапазоны **PLL**, делители и вся clocking topology. Поэтому line rate config — это одна из самых ранних архитектурных decisions в GT design.

### Связь line rate с PLL choice

Одно и то же desired line rate не всегда одинаково удобно реализуется через разные **PLLs**. Для UltraScale/UltraScale+ AMD documents показывают, что **CPLL** и **QPLL** имеют разные line-rate ranges. Например, в DS930 для GTH эти ranges зависят от **output divider** и от того, используется ли CPLL, QPLL0 или QPLL1; кроме того, верхний предел зависит от speed grade. В PG380 также отмечено, что для одного из GTH use cases CPLL и QPLL имеют разные maximum line rates, и это влияет на допустимые SDI режимы.

Отсюда возникает важное правило: **line rate нельзя выбирать отдельно от PLL topology.** Если line rate уходит выше области, где CPLL еще разумен, design естественно двигается к QPLL-based clocking. Это же видно и в IBERT for UltraScale GTH: AMD прямо пишет, что use of a **QPLL is recommended for line rates above 8 Gb/s**.

### Связь line rate с reference clock

После выбора target line rate почти сразу возникает вопрос: какой **reference clock frequency** сможет legal и удобно поддержать этот режим. В PG182 AMD прямо указывает, что после задания line rate Wizard предлагает совместимые **actual reference clock frequency** варианты для соответствующего QPLL use case. В JESD204 PHY docs также сказано, что **reference clock and core clock frequency constraints vary depending on the selected line rate and reference clock**, а generated XDC уже отражает эту связь.

Это важное следствие: **line rate config и refclk setup — не две независимые задачи.** Обычно сначала фиксируется target line rate, а уже затем подбирается подходящий refclk, который соответствует legal решению по **PLL/divider** и реальному плану платы. Если начать наоборот — от “удобной частоты oscillator” — легко получить design, который формально собирается, но имеет менее clean clocking topology или неудобный путь к multi-lane расширению.

### Связь line rate с data width

Это одна из самых недооцененных частей темы. Когда engineer меняет line rate, меняется не только serial domain, но и требования к **parallel interface** внутри transceiver. Документация по CPRI прямо показывает, что при **8B/10B encoded line rates** transceiver конфигурируется с **40-bit internal data width** для использования встроенной 8B/10B encode/decode logic. Для этих режимов на transmit side может потребоваться другой режим TX interface и соответствующий **TXUSRCLK**, а на receive side — свой вариант RX interface clocking.

В других protocols набор соотношений будет отличаться, но общий принцип остается: **line rate всегда связан с interface width.** Даже в IBERT AMD определяет protocol как **line rate/data width/reference clock rate combination**, а не только как line rate + refclk. Это хороший reminder: ширина параллельного datapath — полноценная часть line rate config, а не вторичная detail.

### Связь line rate с TXUSRCLK / RXUSRCLK

На user side line rate почти всегда проявляется через **TXUSRCLK / RXUSRCLK / USER_CLK**. Aurora docs прямо говорят, что **USER_CLK/SYNC_CLK frequency** рассчитывается на основе **line rate** и **transceiver interface width**. В hardware debug section Aurora также отмечает, что transceiver генерирует `txoutclk` based on the line rate parameter, а уже от него строится `user_clk`.

Практически это означает следующее: чем выше line rate при фиксированной interface width, тем выше частота parallel clock на стороне логики. И наоборот, увеличение interface width может снизить требуемую частоту. Именно поэтому line rate config нельзя делать, не думая о том, выдержит ли соседняя fabric logic нужный clock и насколько удобно этот clock встроится в остальную систему.

### Line rate и device limits

Очень важная часть темы — **line rate is always device-limited**. В user-facing Aurora docs AMD пишет, что available line-rate range depends on transceiver type and can be limited by the selected device. В DS930 для GTH отдельно указаны minimum/maximum line rates и конкретные ranges для CPLL/QPLL0/QPLL1 в зависимости от divider и speed grade. Это уже не soft recommendation, а hard capability envelope of the silicon.

Из этого следует простой, но важный вывод: **line rate config нельзя переносить между devices “по аналогии”.** Даже если protocol одинаковый, legal line-rate window, recommended **PLL choice** и комфортный запас могут отличаться между GTH, GTY, разными packages и разными speed grades. Поэтому каждый новый target device требует отдельной проверки этих assumptions against datasheet and transceiver user guide.

### Line rate и throughput scaling по lanes

Еще один системный момент: line rate почти никогда не существует “в одиночку”. В многолинейных protocols aggregate throughput зависит от **line rate per lane** и **number of lanes**. Aurora documentation прямо пишет, что throughput зависит от числа transceivers, transceiver type и target line rate selected for the transceivers; single instance can use up to 16 lanes, а итоговый throughput масштабируется от сотен Mb/s до сотен Gb/s.

Это важный design choice. Иногда user пытается поднять aggregate bandwidth за счет максимального line rate на одной lane, хотя system было бы проще сделать через большее число lanes на более умеренном per-lane rate. А иногда наоборот: extra lanes усложняют placement, channel bonding и PCB, и выгоднее поднять per-lane line rate. Значит, **line rate config** всегда нужно рассматривать вместе с **lane strategy**, а не отдельно.

### Как правильно смотреть на line rate во Vivado

Во Vivado line rate полезно воспринимать не как isolated GUI field, а как входную точку в целый набор зависимостей. Wizard — это supported method of configuring UltraScale transceivers, он автоматически применяет primitive parameters под application и упрощает helper clocking / port enablement. Поэтому на prototype stage наиболее здоровый flow обычно такой: сначала в Wizard задать realistic line rate target, затем посмотреть, какие **PLL**, **refclk** и **clocking** choices tool считает compatible, а уже потом интегрировать это в larger user design.

То есть хороший подход — не пытаться вручную “угадать” line-rate-related attributes GT primitive в начале проекта. Гораздо надежнее сначала получить working configuration from Wizard, example design-level XDC и базовый bring-up path, а потом уже при необходимости уточнять детали. Это особенно полезно потому, что line rate config затрагивает сразу несколько зависимых параметров, и manual setup здесь быстро становится error-prone.

### Как об этом полезно думать на практике

Полезно воспринимать эту тему не как выбор одной цифры, а как выбор рабочей точки всей системы. Когда скорость в линии растет, вместе с ней растут требования к источнику тактирования, к запасу по устройству, к частоте параллельной части и к аккуратности всей архитектуры вокруг transceiver. Поэтому хорошая настройка почти всегда выглядит как компромисс: не максимальная цифра любой ценой, а такая скорость, при которой канал легко поднимается, соседняя логика не начинает страдать по частоте, а схема остается понятной для отладки и дальнейшего расширения.

Именно по этой причине в реальном проекте полезно задавать себе несколько простых вопросов. Какая полезная скорость действительно нужна приложению? Какая часть полосы уйдет в кодирование? Можно ли получить нужный результат меньшей скоростью на линии, но большим числом lanes? Или наоборот, лучше уменьшить число lanes и поднять скорость на каждой из них? Такие вопросы звучат базово, но именно они обычно отделяют устойчивую конфигурацию от конфигурации, которая формально допустима, но неудобна в реальном железе.

Есть и еще один важный момент. Чем раньше line rate зафиксирован как осознанное архитектурное решение, тем проще становятся все соседние темы: выбор опорной частоты, проверка ограничений по устройству, оценка частоты пользовательского интерфейса, подготовка XDC и логика bring-up. Если же эта скорость долго остается “примерной”, то остальные решения тоже начинают приниматься примерно, и потом приходится пересобирать значительную часть clocking scheme уже после первых проблем на плате.

### Как line rate влияет на bring-up

На этапе bring-up неверный line rate часто маскируется под другие проблемы. В hardware debug guide Aurora есть прямой совет: если `user_clk` frequency не находится в expected range, нужно проверить frequency of the transceiver reference clock and PLL attributes. Это косвенно показывает, что ошибка в line-rate-related configuration может проявиться как “неправильный user clock”, а дальше уже потянуть за собой reset sequence, training failures и misleading debug symptoms.

Поэтому practical strategy такая: если link не поднимается, сначала проверять не только reset FSM и serial polarity, но и базовую триаду **line rate / refclk / PLL**. Во многих случаях проблема оказывается именно в том, что nominal line rate был выбран без учета legal divider solution, последствий для user clock или device-specific limit.

### Типичные ошибки

У подтемы **line rate config** есть несколько очень типичных ошибок. Первая — путать **line rate** и **payload throughput**. Вторая — выбирать line rate, не учитывая **encoding overhead**. Третья — задавать line rate без одновременной проверки **PLL range** и **reference clock compatibility**. Четвертая — смотреть только на serial speed и забывать про **interface width** и **TX/RX user clocks**. Пятая — переносить “удачную конфигурацию” с другого device, не проверяя current speed-grade limits.

Есть и более тонкая ошибка: считать, что maximum supported line rate always является лучшей рабочей точкой для prototype. Даже current Aurora docs отмечают, что для better timing performance с default synthesis strategy line rate нередко стоит снижать относительно максимума, а при других strategies потолок может быть достигнут. Это не универсальное правило для всех designs, но хороший reminder: **maximum legal line rate** и **most practical project line rate** — не всегда одно и то же.

### Практический итог

**Line rate config** — это полноценная инженерная подтема внутри **Transceiver Configuration**. Она отвечает не на вопрос “какую цифру поставить в Wizard”, а на более глубокий вопрос: какой serial bit rate нужен системе, как он соотносится с encoding overhead, какой **PLL** и **refclk** его поддержат, какую **interface width** он потребует и во что это превратится на уровне user clocks и всей архитектуры линка.

Хороший **line rate config** обычно означает следующее: target line rate выбран от **payload requirement** и **protocol coding**; проверены **device/speed-grade limits**; понятно, используется **CPLL** или **QPLL**; подобран compatible **reference clock**; осознаны последствия для **data width** и **TXUSRCLK/RXUSRCLK**; bring-up проверяется через clean Wizard-based configuration, а не только внутри большого final design.

Если сказать совсем коротко: **line rate config** — это настройка центральной скорости serial world, от которой потом раскладывается вся остальная transceiver architecture. И чем раньше эта скорость выбрана правильно, тем проще дальше становятся refclk setup, PLL selection, user clocking и реальный hardware bring-up.