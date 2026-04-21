## Reset sequence

**Reset sequence** в контексте FPGA/Vivado и блока **Transceiver Configuration** — это порядок, по которому transceiver проходит через включение, инициализацию **PLL**, запуск **TX/RX user clocks**, подтверждение `TXUSERRDY/RXUSERRDY`, при необходимости процедуру **buffer bypass**, и только после этого выходит в нормальную работу. В **GT Wizard** этот порядок обычно собирается из **reset controller helper block**, блоков **user clocking** и, если нужно, блоков **buffer bypass**. AMD прямо указывает, что reset state machines в Wizard реализуют последовательности из user guides для **GTH/GTY**, а связи между reset logic, user clock logic и bypass logic уже предусмотрены в типовом потоке.

Главная мысль здесь такая:  
**reset sequence у GT — это часть clocking architecture, а не просто отдельный reset signal.**  
Если для обычной fabric logic грубый reset иногда еще проходит, то для transceiver неправильный порядок почти сразу дает странные симптомы: есть `pll_lock`, но нет `*_done`; есть `TXUSRCLK/RXUSRCLK`, но link не выходит в устойчивое состояние; после смены режима datapath уже обновлен, а прием или передача не возвращаются в рабочий режим. Поэтому AMD выделяет reset sequencing как отдельную тему в **PG182**, а **Open IP Example Design** рекомендует как reference для базовой **power-on-reset sequence** и первого подъема.

### Почему это отдельная полноценная подтема

Внутри общего раздела **Transceiver Configuration** есть темы более обзорные: **reference clock setup**, **line rate config**, **CPLL vs QPLL**, **buffer modes**. А **reset sequence** — это уже отдельная узкая тема, потому что именно здесь все эти решения начинают взаимодействовать во времени. Reset зависит от того, какой **PLL** выбран, есть ли shared **QPLL**, стабильны ли **TXUSRCLK/RXUSRCLK**, используется ли **buffer bypass**, нужен ли **datapath-only reset** после dynamic change и есть ли device-specific нюансы вроде **CPLL calibration**. Поэтому reset sequence — это уже не набор портов, а согласование всей GT subsystem во времени.

### С чего начинается правильный reset sequence

Правильная последовательность начинается не с пользовательского pulse на reset input, а с наличия **free-running clock** для reset logic. В **PG182** AMD прямо пишет, что **reset controller helper block** требует вход `gtwiz_reset_clk_freerun_in`, который должен переключаться на заданной частоте еще до device configuration; этот же clock используется для подъема core и для разных helper blocks. Для части конфигураций с **CPLL** этот же freerun clock должен также подаваться на `drpclk_in`. Это важная деталь: reset sequence у GT живет не в том же мире, что и основной datapath, а опирается на отдельный стабильный clock domain.

Следующий шаг — дождаться, пока сам transceiver скажет, что питание и внутренние условия готовы. AMD явно указывает: после configuration нельзя сразу начинать дергать reset inputs; сначала transceiver power должно быть reported as good. **Reset helper block** держит **PLL** и datapath resources в reset, пока `GTPOWERGOOD` не станет High на всех channels, и только потом проходит один раз через **reset all state machine**. Практический смысл простой: сначала **GTPOWERGOOD**, потом первичная инициализация, и только затем любые последующие user resets.

### Какие reset inputs вообще есть

В **GT Wizard** reset control специально разделен по глубине воздействия. В **PG182** перечислены такие входы:

- `gtwiz_reset_all_in`
- `gtwiz_reset_tx_pll_and_datapath_in`
- `gtwiz_reset_tx_datapath_in`
- `gtwiz_reset_rx_pll_and_datapath_in`
- `gtwiz_reset_rx_datapath_in`

Для них AMD задает общую базовую семантику: это **active-High asynchronous pulse** длительностью не меньше одного периода `gtwiz_reset_clk_freerun_in`, который запускает соответствующий reset process. По смыслу эти входы делят reset на три уровня: полный **reset_all**, reset уровня **PLL + datapath**, и reset уровня только **datapath**. Именно это и делает reset sequence управляемым: можно выбирать не “сильнее или слабее”, а именно тот объем, который реально нужен.

Это очень важное место для системного мышления. У GT reset — не одна универсальная кнопка. Есть reset для начального старта, есть reset для случаев, когда затронут **PLL/refclk** мир, и есть локальный reset только для datapath. Из этого следует простое правило: **не использовать `reset_all` как ответ на любое изменение в системе**. Чем точнее выбран объем, тем меньше побочных эффектов на shared resources, соседние lanes и весь link. Такой подход прямо поддерживается и общей port model из **PG182**, и заметкой AMD по приложению, где разные reset inputs применяются для разных типов dynamic changes.

### Что делает `reset_all`

`gtwiz_reset_all_in` — это самый тяжелый reset в обычной Wizard-based конфигурации. AMD описывает его как user signal для reset **PLLs** и active data directions transceiver primitives; в SDI application note отдельно сказано, что этот reset обычно используется во время старта. На практике именно он задает базовый подъем после power-up или после ситуации, когда нужно заново привести и TX, и RX части канала в known state.

Но здесь важно понимать границу. После первого старта reset helper block уже сам проходит через **reset-all state machine**, а AMD рекомендует ждать либо `gtpowergood_out`, либо одновременно `gtwiz_reset_tx_done_out` и `gtwiz_reset_rx_done_out`, прежде чем запускать дальнейшие resets. То есть **первичный старт** и **рабочий reset** — это два разных режима. В первом режиме система только формирует исходное состояние. Во втором режиме верхняя logic уже должна осознанно выбирать нужный уровень reset.

### Когда нужен `*_pll_and_datapath_in`

Reset уровня **PLL + datapath** нужен тогда, когда изменение затрагивает не только local path, но и источник serial/user clocking. В **PG182** эти входы определены как reset соответствующего data direction вместе с associated **PLLs**. В SDI application note AMD дает очень наглядный пример: `gtwiz_reset_tx_pll_and_datapath_in` и `gtwiz_reset_rx_pll_and_datapath_in` особенно полезны, когда меняется **reference clock** для TX PLL или RX PLL. Это хороший шаблон и для более общего GT design: если меняется refclk source, тип **PLL**, или другие условия, влияющие на clock generation, одного datapath reset обычно уже недостаточно.

Отсюда удобно вывести простое правило. Если затронут **PLL world**, надо думать в терминах `*_pll_and_datapath_reset`. Если базовая clocking topology не меняется, а нужно только заново ввести transmit или receive path в корректное состояние, тогда часто хватает **datapath-only reset**. Это не универсальная таблица на все случаи, но именно так AMD documents разводят смысл этих reset inputs.

### Когда хватает `*_datapath_in`

Reset уровня только **datapath** используется там, где **PLL** и общая clocking structure уже остаются правильными, а повторная инициализация нужна только для RX или TX path. В **PG182** эти порты так и определены: reset только transmit или receive data direction transceiver primitives, без reset associated PLLs. В SDI note AMD показывает типичный пример для RX: после dynamic changes на стороне приема — например, изменения `rxcdrhold`, equalization mode, `RXCDR_CFG` или `RXOUT_DIV` — достаточно выполнить `gtwiz_reset_rx_datapath_in`; если в одной последовательности меняется сразу несколько таких параметров, одного datapath reset после всех изменений достаточно. Для TX datapath в application note используется похожая логика при смене части transmit-side настроек.

Это делает **datapath-only reset** очень важным инструментом для runtime reconfiguration. Вместо полного restart of the link можно локально переинициализировать только ту часть GT, которая действительно требует повторного входа в valid state. Выигрыш понятен: меньше disturbance для соседней logic, меньше лишних действий вокруг shared resources и более чистый control flow. Но такой подход работает только тогда, когда engineer уверен, что сторона **PLL/refclk** не затронута.

### Почему user clocks — часть reset sequence

Одна из самых важных и часто недооцененных идей — **reset sequence не может корректно завершиться, пока не подтверждена стабильность TXUSRCLK/RXUSRCLK**. В **PG182** прямо сказано, что wiring между **reset helper block** и **user clocking helper blocks** существует для правильной реализации `TXUSERRDY` и `RXUSERRDY`. Более того, когда helper blocks вынесены в example design, порты `gtwiz_reset_userclk_tx_active_in` и `gtwiz_reset_userclk_rx_active_in` должны быть asserted High, когда `TXUSRCLK/TXUSRCLK2` и `RXUSRCLK/RXUSRCLK2` уже active and stable; только тогда transmitter и receiver reset sequence могут завершиться.

Со стороны самих user clocking helper blocks правило тоже описано явно. Для `gtwiz_userclk_tx_reset_in` и `gtwiz_userclk_rx_reset_in` AMD пишет, что active-High assertion нужно держать до тех пор, пока `gtwiz_userclk_*_srcclk_in/out` не станет stable. После этого helper block формирует `gtwiz_userclk_*_active_out`, который и подтверждает, что local user clock resources вышли из reset. Это означает очень важную вещь: в хорошем GT design **clock stabilization** и **reset completion** — это одна сцепленная последовательность, а не две независимые ветки.

### Роль `TXUSERRDY` / `RXUSERRDY`

На уровне primitive ports `TXUSERRDY` и `RXUSERRDY` иногда воспринимаются как обычные control signals. Но в контексте Wizard-based reset sequence они являются частью coordination между reset state machine и user clocking network. **PG182** прямо говорит, что wiring between reset controller helper block and user clocking helper blocks существует именно для **proper TXUSERRDY and RXUSERRDY signaling**. То есть в нормальном потоке эти сигналы не должны мыслиться как случайные manual toggles без связи с clocks и done indicators.

Практически это удобно помнить так:  
**TX/RX user-ready нельзя отделять от готовности user clocks.**  
Если `TXUSRCLK2` или `RXUSRCLK2` еще не stable, а reset logic уже пытается завершить sequence, проект может попасть в странное промежуточное состояние. Формально часть reset ports уже отработала, но datapath еще не опирается на действительно готовый local clock domain. Именно поэтому типовой Wizard flow старается связать эти вещи автоматически.

### Что означают `gtwiz_reset_tx_done_out` и `gtwiz_reset_rx_done_out`

Эти два статуса — главные признаки того, что reset sequence действительно завершился. **PG182** определяет `gtwiz_reset_tx_done_out` как **active-High indication** завершения transmitter reset sequence, а `gtwiz_reset_rx_done_out` — как такой же признак для receiver. При старте AMD рекомендует ждать либо `gtpowergood_out`, либо обе done-indications, прежде чем инициировать дополнительные resets. Это значит, что `*_done` — не декоративные флаги, а реальные operational checkpoints всего процесса.

Есть и более важный нюанс. Если после завершения reset sequence **PLL**, который clock’ит TX или RX datapath, теряет lock, соответствующий `gtwiz_reset_*_done_out` deasserts, но **sequence does not automatically restart**. AMD прямо пишет, что в такой ситуации требуется **user intervention**. Это очень важное system-level правило: reset sequence автоматизирует корректный порядок подъема, но не является полностью self-healing mechanism. Политика recovery после later loss-of-lock должна быть реализована в верхнем control layer.

### Buffer bypass и reset sequence

Если в проекте bypass’ятся TX или RX elastic buffers, тема reset sequence становится еще теснее связанной с другими helper blocks. **PG182** прямо говорит, что wiring между reset helper block и buffer bypass controller helper blocks используется для запуска **buffer bypass sequences upon completion of the reset sequences**. Для TX buffer bypass AMD дополнительно указывает, что `gtwiz_buffbypass_tx_reset_in` нужно release as soon as `TXUSRCLK2` is stable и до того, как transmitter datapath reset sequence полностью завершится; по умолчанию `gtwiz_reset_tx_done_out` соединяется с `gtwiz_buffbypass_tx_resetdone_in`. Для RX bypass helper block аналогично рассчитан на synchronous reset pulse сразу после стабилизации `RXUSRCLK2`.

Практический вывод очень важен: **buffer bypass нельзя рассматривать отдельно от reset sequence**. Если buffers bypassed, то получить `pll_lock` и даже `tx/rx resetdone` еще недостаточно; нужно, чтобы phase-alignment или bypass-related procedure также прошла в правильный момент относительно user clocks и datapath reset completion. Поэтому конфигурации с bypass почти всегда требуют более аккуратной общей coordination, чем варианты с включенными elastic buffers.

### Почему старт и reset во время работы — это разные режимы

Очень полезно разделять два сценария.

Первый — **initial power-on / configuration bring-up**. Здесь система только выходит в базовое состояние: free-running reset clock уже работает, transceiver power приходит в `GTPOWERGOOD`, reset-all sequence проходит один раз, user clocks стабилизируются, done outputs поднимаются, optional bypass procedures завершаются. Именно этот сценарий AMD советует сначала смотреть через **Open IP Example Design**, который показывает basic power-on-reset sequence и link functionality через **PRBS generators/checkers**.

Второй сценарий — **рабочий reset во время работы**. Здесь system уже работает, и reset нужен из-за refclk change, mode change, datapath reconfiguration, PLL relock problem или software-controlled reinit. Здесь уже нельзя просто повторять power-on logic. Нужно понимать, какой уровень reset реально нужен, какие clocks должны оставаться running, не мешают ли друг другу несколько dynamic procedures и кто в управляющей части координирует recovery flow. Именно такую coordinated model AMD показывает в SDI application note, где control module carefully coordinates reset and dynamic change activities.

### Почему example design так полезен именно для reset sequence

Для темы **reset sequence** example design особенно полезен, потому что он показывает не только то, что “link works”, но и **какой именно базовый порядок подъема предполагает Wizard**. **UG1192** прямо говорит, что **Open IP Example Design** создает отдельный проект, который демонстрирует basic power-on-reset sequence transceiver и показывает link functionality with **PRBS generators/checkers**; его можно использовать и как simulation reference, и как упрощенный отдельный проект для проверки transceiver interface до интеграции в большой проект.

Это очень практичный совет. Если твой final design не выходит в устойчивое состояние, полезно сначала проверить, проходит ли тот же transceiver configuration cleanly в example design. Если проходит, значит проблема, скорее всего, в собственной coordination вокруг reset, user clocks, buffer bypass или runtime control. Если не проходит даже example design, тогда искать нужно уже на уровне base clocking, refclk, **PLL** или board signal integrity. Для reset sequence это один из самых быстрых способов отделить architecture problem от integration problem.

### Device-specific нюансы: CPLL calibration

У reset sequence есть и device-specific детали. Для **UltraScale+** в **GTHE4/GTYE4** AMD добавляет **CPLL Calibration block**, который можно включить параметром `INCLUDE_CPLL_CAL`; в описании сказано, что этот блок оценивает frequency, at which **CPLL lock** is achieved, а default values, corresponding to the line rate, можно получить через open-example-design step или по указанной формуле. Сам факт существования такого блока хорошо показывает, что reset sequence зависит не только от abstract FSM, но и от реального analog behavior выбранного **PLL** в конкретном поколении device.

Практический смысл здесь не в том, что “всегда надо включать calibration”, а в том, что **reset sequence нельзя проектировать полностью device-agnostic**. Если проект использует **CPLL** на UltraScale+ и проявляет нестабильность вокруг reset или lock behavior, нужно проверять не только общий RTL, но и vendor-specific recommendations именно для этой GT family.

### Как на это правильно смотреть в системе

Полезно мыслить о **reset sequence** не как о “пачке reset ports”, а как о coordination между несколькими подсистемами:

- **free-running reset clock domain**
- transceiver **power-good / PLL lock**
- **TX/RX user clock generation**
- primitive-level `TXUSERRDY/RXUSERRDY`
- optional **buffer bypass**
- status outputs `gtwiz_reset_tx_done_out` / `gtwiz_reset_rx_done_out`
- higher-level control logic, которая решает, когда и какой reset запускать.

Такой взгляд особенно полезен потому, что большинство реальных bugs возникают не внутри одной из этих частей, а **на границе между ними**. Например, freerun clock уже есть, но user clock helper block released too early. Или **PLL** снова lock’нулся, но верхний control layer не повторил нужный reset scope. Или RX datapath уже перенастроен, а вместо local reset случайно был дан полный `reset_all`, который unnecessarily disturbed TX side и соседние channels. Это не экзотика, а типичные следствия того, что reset sequence не был осмыслен как system-level process. Этот вывод хорошо согласуется с общим guidance AMD, где helper blocks и coordinated control logic предлагаются как основной recommended path.

### Типичные ошибки

У этой подтемы есть несколько повторяющихся ошибок.

Первая — считать, что достаточно одного общего `reset_all`, и Wizard “сам все сделает” в любой runtime situation. На деле `reset_all` уместен прежде всего для старта или полного reinit, а для refclk change, mode change или datapath change часто нужен более узкий объем reset.

Вторая — забывать, что **user clocks должны быть active and stable** до завершения TX/RX reset sequence. Если этот handshake нарушен, `reset_done` может не прийти в ожидаемом виде или link окажется в промежуточном состоянии.

Третья — игнорировать тот факт, что после потери **PLL lock** `gtwiz_reset_tx_done_out` / `gtwiz_reset_rx_done_out` deassert, но sequence не перезапускается автоматически. Без явной recovery policy в верхнем control logic такая система может просто “застыть” после кратковременной clocking problem.

Четвертая — пытаться рассматривать **buffer bypass** отдельно от reset. Если bypass-enabled design не запускает alignment/bypass procedure в нужный момент, формально reset часть уже может быть завершена, а practical link readiness — еще нет.

Пятая — first-time debug сразу в большом mission design, без проверки через example design. Для reset sequence это особенно вредно, потому что одновременно смешиваются GT bring-up, собственная control logic, application protocol и board-level issues.

### Практический итог

**Reset sequence** — это полноценная инженерная подтема внутри **Transceiver Configuration**. Она отвечает не на вопрос “какой reset port использовать”, а на более глубокий вопрос: **как во времени согласовать power-good, PLLs, user clocks, TXUSERRDY/RXUSERRDY, optional buffer bypass и рабочее восстановление так, чтобы GT надежно выходил в working state и корректно переживал повторную инициализацию.**

Хороший **reset sequence** обычно означает следующее:  
есть стабильный **free-running reset clock**;  
старт начинается только после `GTPOWERGOOD`;  
для первичного подъема используется корректный **reset-all flow**;  
для изменений во время работы выбирается правильный уровень между **PLL + datapath** и **datapath-only** reset;  
completion контролируется через `gtwiz_reset_tx_done_out` / `gtwiz_reset_rx_done_out`;  
user clocks и optional **buffer bypass** встроены в ту же последовательность;  
а recovery after loss-of-lock обрабатывается верхним control logic явно, а не “по надежде”.

Если сказать совсем коротко: **reset sequence** — это choreography всего transceiver bring-up. И чем раньше engineer начинает видеть в reset не один сигнал, а согласованный процесс во времени, тем быстрее проект перестает быть “капризным GT” и становится управляемой serial subsystem.