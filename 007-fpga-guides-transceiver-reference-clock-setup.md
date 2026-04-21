## Reference clock setup

**Reference clock setup** в контексте FPGA/Vivado и блока **Transceiver Configuration** — это проектирование и настройка опорного **reference clock** для transceiver subsystem. Это не просто выбор частоты на вход GT, а решение о том, **какой source используется, какой PLL от него работает, в какой Quad приходит refclk, какие line rates он должен поддержать и как этот выбор влияет на bring-up всей serial link architecture**. В UltraScale GTH transceivers организованы по **Quads**: в каждом Quad есть четыре channels и один common block с **QPLL0/QPLL1**, а у каждого channel есть собственный **CPLL**. При этом RX и TX могут быть настроены на использование **QPLL0, QPLL1 или CPLL** как clock source.

Главная идея здесь такая:  
**reference clock — это основа работы transceiver PLL, а не просто очередной fabric clock.**  
Именно поэтому refclk нужно рассматривать как часть общей **transceiver clocking architecture**, а не как локальную настройку одного IP. Если refclk подобран неудачно, проблемы обычно проявляются не в одном месте, а по всей цепочке: PLL lock, startup behavior, link bring-up, BER margin, устойчивость к reset/restart. Это особенно важно потому, что AMD отдельно задает для transceiver именно **reference clock phase-noise masks**, а для некоторых protocols, например PCIe, protocol-specific mask может заменять generic device-level mask.

---

### Зачем вообще выделять эту тему отдельно

На уровне большого раздела **Transceiver Configuration** можно говорить про PLL, Wizard, reset, buffers, debug и line rate в целом. Но **reference clock setup** — это отдельная подтема, потому что именно здесь сходятся сразу несколько слоев design:

- **protocol / target line rate**,
- **CPLL vs QPLL topology**,
- **Quad placement**,
- **oscillator quality**,
- **board-level routing**,
- **Vivado Wizard choices**.

То есть эта тема не про один параметр, а про точку, где встречаются **system architecture**, **PCB reality** и **Vivado configuration flow**. Если engineer неправильно мыслит про refclk, он потом часто пытается лечить проблему через reset logic, GT attributes или protocol debug, хотя root cause находится гораздо раньше — в самой опорной clock scheme. Это уже инженерный вывод из того, что AMD docs отдельно и подробно описывают refclk requirements, oscillator selection и PLL dependencies как самостоятельные вещи.

---

### Что на самом деле включает в себя reference clock setup

Когда говорят “setup reference clock”, часто слишком сильно упрощают тему до вопроса “156.25 MHz или 125 MHz?”. На практике сюда входят несколько связанных решений.

Во-первых, нужно выбрать **reference clock source**. Это означает не только frequency, но и тип источника, electrical characteristics и то, соответствует ли он требованиям transceiver input. В PCB guide AMD прямо перечисляет критерии выбора oscillator: **frequency range, output voltage swing, jitter, rise/fall times, noise specification, duty cycle и frequency stability**. То есть refclk source оценивается не по одному числу, а как полноценный analog-quality input.

Во-вторых, нужно определить, **какой PLL использует этот refclk**. В UltraScale GTH **QPLL0/QPLL1** живут на уровне Quad и могут распределять clock на все четыре channels, а **CPLL** принадлежит одному channel. Значит, выбор refclk всегда связан с вопросом: это **shared Quad-level clocking** или **per-channel clocking**. Без ответа на этот вопрос нельзя считать reference clock setup завершенным.

В-третьих, нужно учитывать **legal frequency combinations** для конкретной line-rate configuration. Wizard в advanced clocking section прямо показывает, что он подбирает **compatible frequencies** для reference clock под выбранную конфигурацию и line rate. Это означает, что правильный путь мышления идет не “от удобной частоты”, а “от требуемой линии и допустимых PLL/refclk combinations”.

---

### С чего надо начинать thinking

Одна из самых типичных ошибок — начинать с board oscillator:  
“У меня на плате есть такая frequency, значит от нее и пойдем”.

Для **reference clock setup** более зрелый порядок обратный:

**protocol / encoding / target line rate → PLL choice → compatible refclk options → Quad placement → actual oscillator source**

Такой порядок лучше соответствует тому, как и сами AMD tools смотрят на задачу: сначала определяется нужная transceiver configuration, а уже потом выбираются **actual compatible reference clock frequencies**. В PG182 это видно напрямую: Wizard рассчитывает и предлагает допустимые значения refclk под нужную configuration.

Это важный architectural момент. Один и тот же oscillator может быть “допустимым”, но не обязательно **удачным**. Он может вести к неудобной PLL topology, ограничивать shared clocking внутри Quad или делать system менее масштабируемой. Поэтому хороший **reference clock setup** начинается не с выбора “удобной частоты”, а с ответа на вопрос:  
**какая refclk architecture лучше поддерживает нужный line rate и нужную структуру link.**

---

### Связь с CPLL и QPLL

Внутри этой темы нельзя обойти вопрос **CPLL vs QPLL**, потому что refclk всегда выбирается не “в пустоту”, а для конкретного PLL consumer.

Для GTH в UltraScale QPLL0/QPLL1 находятся в common block и распределяются на весь Quad, тогда как CPLL локален для одного channel. Это сразу задает два разных класса решений:

- **shared refclk / shared PLL model** через QPLL,
- **local refclk / local PLL model** через CPLL.

Отсюда появляется practical правило. Если architecture multi-lane, внутри одного Quad и с общей high-speed clocking strategy, обычно естественно думать через **QPLL-based setup**. Если задача более локальная, single-lane или channel-isolated, может быть уместнее **CPLL-based setup**. Важно не то, что один вариант “лучше”, а то, что **reference clock setup нельзя проектировать отдельно от PLL topology**. Это одна инженерная задача, просто на двух уровнях описания.

Есть и еще один practical нюанс. Для UltraScale+ AMD отдельно предупреждает, что в некоторых ситуациях **CPLL may not reliably lock** после configuration, при повторной подаче reference clock или после assert/deassert `CPLLPD`; для таких случаев Wizard предусматривает **CPLL Calibration block**. Это хороший пример того, что выбор refclk и выбор PLL влияют не только на steady-state, но и на реальный bring-up behavior.

---

### Почему line rate и refclk нельзя рассматривать независимо

В transceiver thinking есть сильный соблазн отделить “data path settings” от “clock settings”. Для **reference clock setup** это неверно. Refclk существует именно для того, чтобы PLL мог сгенерировать правильную serial clocking environment под нужный **line rate**. Поэтому refclk всегда связан с target bit rate через PLL/divider structure.

Это хорошо видно и из Wizard flow, где compatible reference clock values выбираются под конкретную configured core/line-rate combination, и из transceiver docs, где возможности CPLL и QPLL описываются через поддерживаемые диапазоны line rate. Для GTH в одном из official documents даже прямо показано, что **CPLL и QPLL имеют разные maximum line-rate capabilities**, а значит выбор refclk нельзя делать без оглядки на требуемый serial rate.

Практически это означает:  
если engineer говорит “я уже выбрал reference clock, осталось понять line rate”, значит thinking почти наверняка перевернут. Правильнее говорить наоборот:  
“я знаю line rate и topology, теперь подбираю legal и convenient refclk”.

---

### Quad placement как часть reference clock setup

Очень важная системная мысль: **reference clock setup тесно связан с placement по Quads**.

Поскольку **QPLL0/QPLL1** являются Quad-level resources, а сами channels сгруппированы в Quad, вопрос “где будут стоять lanes” влияет не только на data routing, но и на то, насколько clean получится refclk distribution и сможет ли design удобно использовать shared PLL resources. Это не вторичная деталь, а часть архитектуры transceiver subsystem.

Кроме того, для transceiver reference clocks существуют и ограничения на то, сколько Quads может быть запитано от одного external clock pin pair. В документации Aurora для FPGA designs прямо указано, что в **UltraScale** один внешний refclk pair не должен обслуживать более **пяти Quads** или **двадцати GTHE3/GTHE4 channels**. Это особенно важно в больших prototyping systems: refclk topology должна масштабироваться не только логически, но и физически.

Из этого следует полезное правило:  
**не фиксировать lane placement до того, как хотя бы грубо продуман refclk plan**.  
Иначе легко получить ситуацию, когда data lanes уже “удобно” разложены, а clock architecture потом вынужденно усложняется. Это не отдельный факт из одной строки документации, а инженерный вывод из Quad-based structure и ограничений refclk distribution.

---

### Качество source важнее, чем часто кажется

Для обычных fabric clocks engineer нередко думает в первую очередь о частоте. Для transceiver **reference clock** этого недостаточно. AMD прямо публикует **phase-noise masks** для типовых refclk frequencies 156.25, 312.5 и 625 MHz, причем отдельно для QPLL и CPLL, и указывает, что для других frequencies нужно использовать nearest phase-noise mask. Также там сказано, что protocol-specific mask, например для PCIe, имеет приоритет над generic one.

На board side UG583 отдельно подчеркивает, что oscillator selection должен учитывать **jitter, rise/fall times, duty cycle, noise и frequency stability**, а MGTREFCLK differential input internally terminated at **100 Ω differential impedance**. Это значит, что refclk source нельзя считать абстрактным идеальным сигналом; его electrical quality напрямую входит в задачу setup.

Именно поэтому **reference clock setup** — это пересечение FPGA и PCB design.  
Здесь нельзя качественно закончить работу только на стороне Vivado. Хороший result получается тогда, когда board source, refclk pins, PLL choice и target protocol рассматриваются как одна система.

---

### Что нельзя путать внутри этой темы

Одна из частых причин путаницы — смешивание разных clock roles.

Есть **reference clock**, который приходит на transceiver и служит опорой для PLL.  
Есть **recovered clock**, который получается уже из RX path.  
Есть **user clocks**, связанные с TX/RX user-side logic и helper clocking around the core.  
И есть отдельные system/helper clocks, которые могут участвовать в reset control, DRP logic и example design. Это видно и в transceiver docs, и в Wizard material, где refclk и user-side clocks фигурируют как разные сущности.

Для **reference clock setup** это очень важно. Пока engineer mentally не разделил эти clocks, он будет говорить “clock у GT” как будто это одна вещь. А зрелое проектирование начинается с того, что роли названы отдельно:  
что является **PLL reference**, что является **recovered timing**, а что является **user-side distribution**. Тогда и сама architecture становится намного яснее.

---

### Как правильно использовать Vivado на этом этапе

Во **Vivado GT Wizard** тему refclk полезно воспринимать не как поле для ввода числа, а как часть общей **clocking model**. Wizard подбирает совместимые **reference clock frequencies**, строит варианты advanced clocking и заставляет мыслить legal combinations, а не произвольными экспериментами с primitive attributes. Это особенно полезно на раннем этапе, когда важнее получить working topology, чем сразу делать deep manual tuning.

Для hardware bring-up AMD рекомендует использовать **Open IP Example Design** после настройки transceivers wizard. В отдельном official material сказано, что example design показывает базовую **power-on-reset sequence** и демонстрирует работу link через **PRBS generators and checkers**. В feature summary также отмечены convenience features вроде **differential reference clock buffer instantiation**, PRBS link monitoring и VIO для basic bring-up/debug.

Это очень важный practical момент именно для **reference clock setup**.  
Refclk лучше сначала проверять в максимально чистой среде — через Wizard example design или IBERT-like flow — а не внутри большого пользовательского проекта. Если link не поднимается уже там, значит искать нужно в первую очередь clocking, PLL, reset sequencing и lane mapping, а не в application logic. Это уже инженерный вывод из рекомендованного official bring-up flow.

---

### Типичные ошибки

У этой подтемы есть несколько повторяющихся ошибок.

Первая — выбирать refclk по принципу **board convenience**, а не от **line rate + PLL topology**.  
Вторая — думать о refclk отдельно от **Quad placement**.  
Третья — путать **reference clock**, **recovered clock** и **user clocks**.  
Четвертая — недооценивать требования к **phase noise / jitter / duty cycle**, как будто речь идет о простом digital clock.  
Пятая — проверять первый раз refclk уже в большом design, минуя example design или другой упрощенный bring-up flow. Все эти ошибки логически следуют из official docs по transceiver architecture, oscillator selection и Wizard bring-up.

Есть и более тонкая ошибка: делать слишком много уникальных refclk sources без явной необходимости. Сама по себе документация не формулирует это как универсальный запрет, но из ограничений на distribution, Quad-based PLL model и сложности bring-up прямо следует инженерный вывод:  
**лишние уникальные refclk domains быстро увеличивают system complexity**.

---

### Практический итог

**Reference clock setup** — это полноценная инженерная подтема внутри **Transceiver Configuration**.  
Она отвечает не на вопрос “какую частоту подать”, а на гораздо более системный вопрос:

**как правильно встроить refclk в PLL topology, line-rate target, Quad placement, board source quality и Vivado bring-up flow.**