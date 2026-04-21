## Link debug

**Link debug** в контексте FPGA/Vivado и блока **Transceiver Configuration** — это не одна операция и не один инструмент, а последовательная проверка всей serial chain: **reference clock**, **PLL lock**, **reset sequence**, **TX/RX user clocks**, physical link integrity, pattern correctness и уже потом состояние protocol layer. В AMD flow для этого есть три основные опоры: **Open IP Example Design** c **PRBS generators/checkers**, standalone или **In-System IBERT**, и **Vivado Serial I/O Analyzer**, который умеет делать **BER measurements**, **2D eye scans**, **link scans** и **link sweeps**.

Главная мысль здесь такая:  
**link debug нужно строить слоями, от нижнего уровня к верхнему.** Если начать сразу с packet parser, framing logic или software-visible registers, очень легко потерять время, потому что источник проблемы часто находится ниже: в **refclk**, в **PLL topology**, в reset coordination или в самой margin of the link. Official AMD materials как раз подталкивают к такому поэтапному подходу: сначала simplified example design с PRBS, затем **IBERT/Serial I/O Analyzer**, а уже потом debug внутри полного user design.

### Почему это отдельная полноценная подтема

Внутри общего раздела **Transceiver Configuration** есть темы про **line rate**, **reference clock setup**, **reset sequence**, **buffer modes**. Но **link debug** — это отдельная инженерная подтема, потому что именно здесь все эти решения проверяются в реальном железе. **UG1192** прямо рекомендует использовать **Open IP Example Design** после настройки transceiver configuration: такой проект демонстрирует basic power-on-reset sequence и показывает link functionality через **PRBS pattern generators and checkers**; его можно использовать как simulation reference или как упрощенный отдельный test перед интеграцией остальной системы.

Именно поэтому **link debug** нельзя сводить к фразе “открой ILA и посмотри данные”. У high-speed link есть собственный слой observability и собственный набор методов. **IBERT** и **Serial I/O Analyzer** существуют именно потому, что для transceivers часто важнее сначала увидеть **BER**, margin и behavior analog-oriented tuning, чем смотреть application payload. В AMD guidance IBERT описывается как solution для debugging and optimizing transceivers, с pattern generators/checkers и доступом к **DRP** в реальном времени.

### С чего правильно начинать debug

Самый полезный practical принцип такой: сначала проверить, что link вообще имеет право заработать, и только потом спрашивать, почему он не передает полезные данные. Это означает, что первый слой проверки обычно включает наличие валидного **reference clock**, наличие **PLL lock**, завершение **reset sequence**, подъем **TX/RX user clocks** и хотя бы базовую PRBS-level verification. В protocol-based IP это хорошо видно по exported status. Например, в Aurora-related debug/status ports есть `gt_pll_lock`, который показывает, что `tx_out_clk` stable; наружу также доступны `tx_resetdone_out` и `rx_resetdone_out`, а статусы `channel_up` и `lane_up` показывают завершение channel/lane initialization.

Очень важный вывод отсюда такой: **если нет базовых clock/reset indications, protocol-level debug почти всегда преждевременен.** Нет смысла искать bug в framing, если `lane_up` не asserted или `gt_pll_lock` не stable. Поэтому хороший link debug почти всегда идет в таком порядке: сначала **clock/reset readiness**, потом serial data integrity, потом protocol status, и только потом уже full data-path analysis. Это уже инженерный вывод из того, как AMD разделяет reset/status/debug observability в transceiver- и Aurora-related flows.

### Почему Example Design — это не “учебный мусор”, а рабочий инструмент

Одна из самых полезных рекомендаций AMD — не дебажить первый раз сложный transceiver link сразу внутри большого user project. **Open IP Example Design** из **UG1192** создаёт отдельный Vivado project, который показывает базовый power-on-reset flow и link functionality с помощью **PRBS pattern generators/checkers**. Это не просто demo; это controlled environment, где GT configuration уже минимально окружена нужной clocking/reset logic и не смешана с application-specific RTL.

Практический смысл здесь большой. Если link не работает даже в example design, проблема почти наверняка находится на уровне **board connectivity**, **refclk**, **line rate / PLL setup**, reset choreography или грубой GT misconfiguration. Если же example design работает, а основной проект — нет, тогда круг поиска резко сужается: значит, transceiver base configuration, скорее всего, живая, а ломается уже integration layer, custom control logic или protocol adaptation. Это уже логический вывод из того, что AMD предлагает example design как stand-alone test before the rest of the design is finished.

### PRBS как первый честный тест канала

Для первого этапа bring-up **PRBS** удобен потому, что он отделяет проверку physical path от application payload. AMD прямо подчеркивает, что example designs и IBERT используют **PRBS generators and checkers** для quick hardware bring-up and debug. В таком режиме не нужно сразу разбираться, правильно ли работает packet format, alignment markers или software-side protocol. Достаточно ответить на более базовый вопрос: может ли передатчик и приемник вообще передать pattern без ошибок на нужной скорости.

Это очень важная инженерная идея. **PRBS-level pass** не означает, что весь system-level protocol уже исправен, но **PRBS-level fail** почти всегда означает, что рано идти выше по стеку. Иными словами, PRBS в transceiver debug играет роль честного теста канала: он убирает лишние уровни сложности и показывает, что serial path в принципе способен работать на выбранных **line rate**, **refclk** и **equalization** settings. Это inference, но она прямо опирается на то, как AMD использует PRBS в example design и IBERT as the first link-function check.

### Loopback как способ локализовать проблему

Следующий сильный инструмент — **loopback**. В transceiver-facing interfaces AMD выносит `loopback[2:0]` как отдельный control port, который выбирает normal operation или один из loopback modes. Это делает loopback не экзотическим hack, а штатным способом отладки.

Практически loopback полезен как метод локализации. Если internal или near-end loopback работает, а full external path — нет, это очень сильный признак, что проблема сидит вне core transceiver datapath: в PCB path, connector, cable, optical/electrical module, polarity или соседней части board. Если же даже loopback не проходит, тогда вероятнее fault в clock/reset/PLL/base GT configuration. Это уже инженерный вывод из самого назначения loopback mode как selectable transceiver debug path.

### Какие статусы смотреть в первую очередь

В protocol-based cores полезно начинать не со всех возможных registers, а с короткого набора **first-order status signals**. На примере Aurora docs это хорошо видно: `channel_up` показывает, что channel initialization complete и канал ready to send/receive data; `lane_up` показывает успешную инициализацию каждой lane; `soft_err` сигнализирует о soft error в incoming serial stream на один `user_clk` cycle; `hard_err` остается asserted до reset; а `gt_pll_lock` отражает стабильность clock source для части логики, использующей `tx_out_clk`.

Из этого удобно вывести простой practical rule: если `gt_pll_lock` unstable — смотри вниз, в clocking/PLL/refclk; если `gt_pll_lock` stable, но нет `lane_up` или `channel_up` — смотри в reset sequence, polarity, loopback, lane connectivity, channel bonding или low-level transceiver settings; если channel уже up, но появляются `soft_err` или `hard_err` — смотри в margin, BER, equalization, SI issues или protocol-level data integrity. Это не формальная таблица из одной страницы документации, а удобная diagnostic ladder, построенная по смыслу exported status/debug ports.

### Когда нужен standalone IBERT

**IBERT** полезен тогда, когда нужен максимально чистый взгляд на сам transceiver, почти без application logic. **UG1192** прямо указывает, что example design можно использовать и как stand-alone **IBERT**-style test для transceiver interfaces и signal integrity, а в **Transceiver Debug** AMD описывает IBERT как debug IP для transceiver evaluation с pattern generators/checkers, доступом к **DRP** и работой через **Vivado Serial I/O Analyzer**.

Это особенно полезно на этапе prototyping. Если есть сомнение, что именно ломает link — custom RTL или физика канала — standalone IBERT почти всегда дает более быстрый ответ, чем попытка продолжать debug внутри большого top design. Он отрезает лишние уровни системы и оставляет только то, что связано с transceiver behavior, **BER** и tuning. Это inference из того, как AMD позиционирует IBERT для PMA evaluation, demonstration и stand-alone transceiver interface testing.

### Что дает Vivado Serial I/O Analyzer

**Vivado Serial I/O Analyzer** — это не просто GUI для статусов. По AMD documentation он позволяет создавать **links**, измерять **BER** на нескольких каналах, выполнять **2D eye scans**, запускать **link scans** и **link sweeps**, а также менять transceiver parameters в реальном времени, пока каналы взаимодействуют с системой. Для **In-System IBERT** это особенно важно: tool использует реальные данные дизайна и строит eye scan “на живом” link, а не в искусственной лабораторной конфигурации.

На практике это означает, что Serial I/O Analyzer закрывает тот самый промежуточный слой, которого обычно не хватает между “линк то работает, то нет” и “давай переписывать RTL”. Он показывает не только binary status, но и **margin picture** link. Например, **2D Eyescan** отображается как heat map of the BER value, а scan results можно сравнивать по **Open Area**. А **link sweeps** позволяют прогонять multiple scans with different MGT settings и смотреть, какие настройки дают лучший результат.

### Почему In-System IBERT особенно ценен

Для реального проекта очень важно, что **In-System IBERT** можно интегрировать прямо в working design. **UG908** пишет, что этот IP доступен для UltraScale и UltraScale+ families и позволяет выполнять **2-D eye scans** с помощью Vivado Serial I/O Analyzer, используя data from the design while transceivers interact with the rest of the system. То есть ты получаешь не “идеальную стендовую картинку”, а измерение именно того канала, который работает в своем настоящем system context.

Это дает очень сильный debug advantage. Бывает, что standalone IBERT показывает хороший eye, а в полном design link становится хуже из-за другой clock environment, thermal conditions, соседней логики, traffic pattern или runtime settings. В таких случаях **In-System IBERT** часто оказывается самым честным способом увидеть реальное состояние канала, не вынимая его из user design. Это inference, но оно напрямую следует из того, что AMD подчеркивает real-time eye scan using data from the design itself.

### Где в этой теме место у DRP

**DRP** сам по себе не является “методом debug”, но это очень важный инструмент для controlled experiments. AMD указывает, что **IBERT** дает доступ к transceiver **DRP ports**, а protocol-based IP вроде Aurora тоже умеют выводить **DRP interface** наружу через AXI4-Lite или native mode. Это означает, что tuning параметров и чтение certain transceiver settings можно встроить в systematic debug flow, а не делать через ad-hoc recompilation.

Практический смысл здесь в том, что хороший link debug — это не только observation, но и **hypothesis testing**. Например, можно проверить, как меняется BER или eye opening при другом equalization-related setting, при другой комбинации MGT parameters или при другом runtime mode. Без такого controlled knob debugging поиск причины быстро превращается в guessing. Это inference, но она хорошо опирается на AMD’s description of real-time parameter adjustment plus DRP access.

### Как выглядит хороший порядок действий

Удобно мыслить про **link debug** как про ladder из нескольких ступеней. Сначала проверяется самая нижняя база: **refclk**, **PLL lock**, reset completion, `tx_resetdone/rx_resetdone`, наличие и стабильность user clocks. Затем канал упрощается до PRBS-based check через example design или standalone setup. Потом добавляется **loopback** как средство изоляции внешнего тракта. После этого уже есть смысл смотреть protocol-level status вроде `channel_up`, `lane_up`, `soft_err`, `hard_err`. И только если этот слой понятен, полезно идти в deeper margin analysis через **IBERT**, **2D Eyescan**, **Bathtub**, **link scans** и **link sweeps**.

Этот порядок не является жестким vendor-mandated recipe line by line, но он хорошо следует из того, как AMD разносит функции между example design, IBERT, status/debug ports и Serial I/O Analyzer. Главное достоинство такого подхода в том, что он резко уменьшает хаос: каждый следующий инструмент включается только тогда, когда уже пройден предыдущий уровень верификации.

### Типичные ошибки

У этой подтемы есть несколько очень характерных ошибок. Первая — начинать debug слишком высоко по стеку: смотреть application packets, когда еще нет устойчивого `PLL lock` или `lane_up`. Вторая — игнорировать **Example Design** и сразу тащить весь поиск неисправности в full design. Третья — использовать только binary status, не переходя к **BER / eye / sweep** analysis, хотя проблема явно выглядит как margin issue. Четвертая — не использовать **loopback**, из-за чего внутренняя и внешняя части тракта остаются смешанными. Пятая — менять слишком много параметров сразу и терять причинно-следственную связь, вместо controlled experiments через **IBERT/DRP/Serial I/O Analyzer**. Эти ошибки не перечислены AMD одной таблицей, но логически следуют из рекомендованного ими debug stack.

Есть и более тонкая ошибка: считать, что если `channel_up` asserted и payload иногда проходит, то physical link уже “нормальный”. На практике marginal channel может давать intermittent soft errors, чувствительность к temperature/cable/traffic pattern и плохой запас по BER. Именно для таких случаев AMD и дает инструменты вроде **2D Eyescan**, **Bathtub** и **link sweeps** — они показывают не только факт работоспособности, но и качество этой работоспособности.

### Практический итог

**Link debug** — это полноценная инженерная подтема внутри **Transceiver Configuration**. Она отвечает не на вопрос “какой сигнал посмотреть первым”, а на более глубокий вопрос: **как поэтапно доказать, что serial link исправен — от clock/reset/PLL basis до BER, eye margin и protocol-level readiness.**

Хороший **link debug** обычно означает следующее: сначала проверены **clock/reset prerequisites**; затем link прогнан через **PRBS-based bring-up**; при необходимости использован **loopback** для изоляции физического тракта; status signals вроде `channel_up`, `lane_up`, `soft_err`, `hard_err`, `gt_pll_lock`, `tx_resetdone`, `rx_resetdone` читаются как diagnostic ladder, а не как случайный набор флагов; для deeper analysis подключаются **IBERT**, **In-System IBERT**, **Serial I/O Analyzer**, **2D Eyescan**, **Bathtub**, **link scans** и **sweeps**; а изменение runtime settings делается через controlled debug flow, а не через хаотичное переподключение логики.

Если сказать совсем коротко: **link debug** — это искусство не просто увидеть, что link “не работает”, а быстро определить, **на каком именно уровне он ломается**: в clocking, в reset, в physical path, в margin или уже в самом protocol layer.