# PCIe Link Training Sequence

**PCIe link training** — это процесс, в котором два PCIe-порта находят друг друга, синхронизируются на физическом уровне, договариваются о ширине link, скорости, lane numbering и переходят в рабочее состояние.

В контексте FPGA/Vivado это делает не твой RTL напрямую, а PCIe hard IP / integrated block. Но понимать sequence важно, потому что при bring-up почти все низкоуровневые проблемы видны именно через LTSSM-состояния, negotiated width/speed и link status.

Упрощенная последовательность:

```
Reset / Electrical Idle
        |
Detect
        |
Polling
        |
Configuration
        |
L0
```

Для более высоких скоростей:

```
Detect
  -> Polling
  -> Configuration
  -> L0 at initial speed
  -> Recovery / Equalization
  -> L0 at negotiated higher speed
```

---

# 1. Что делает link training

Link training нужен, чтобы PCIe-порты согласовали базовые параметры физического соединения:

```
есть ли link partner;
какие lanes реально подключены;
какая ширина link возможна;
какая скорость поддерживается;
как пронумеровать lanes;
есть ли lane reversal / polarity inversion;
можно ли перейти к L0;
нужна ли speed change / equalization.
```

AMD описывает LTSSM как часть Physical Layer logical sub-block; этот блок отвечает за link initialization, training и maintenance. Там же Physical Layer делится на logical и electrical части, а обмен идет через GT transceivers.

---

# 2. Где это находится в FPGA PCIe IP

В Vivado PCIe IP link training обычно находится внутри:

```
PCIe hard block
GT transceivers
PHY / MAC logic
LTSSM
```

Пользовательская логика обычно видит только status:

```
ltssm_state
phy_link_up / phy_link_down
phy_link_status
negotiated_width
current_speed
user_reset / user_lnk_up
cfg status signals
```

Твой RTL не генерирует TS1/TS2 ordered sets вручную. Это делает PCIe IP.

---

# 3. Что такое LTSSM в этой теме

**LTSSM** — это **Link Training and Status State Machine**.

В этой заметке LTSSM рассматривается не как полный state machine со всеми ветками, а как последовательность training:

```
Detect -> Polling -> Configuration -> L0
```

Подробный разбор всех состояний LTSSM лучше вынести в отдельную заметку, потому что там уже будут:

```
substates
Recovery
Hot Reset
Disabled
Loopback
L0s / L1 / L2
directed speed changes
equalization scenarios
```

---

# 4. До начала training

Перед тем как PCIe link сможет тренироваться, должны быть выполнены базовые условия:

```
питание стабильно;
REFCLK присутствует и стабилен;
PERST# / reset корректно отпущен;
FPGA PCIe IP сконфигурирован;
GT/PHY reset sequence завершена;
link partner тоже готов.
```

TI application note описывает, что после питания и наличия reference clock PCIe device начинает link training, состоящий из receiver detection, polling и configuration.

В FPGA-практике это значит:

```
если нет REFCLK или reset не отпущен,
LTSSM может вообще не начать нормальный путь к L0.
```

---

# 5. REFCLK

PCIe REFCLK — один из обязательных элементов physical link.

Он используется PCIe-устройством для генерации высокоскоростной передачи данных. В типичных PCIe системах это 100 MHz reference clock.

Для FPGA это связано с:

```
MGTREFCLK pins
GT quad placement
clocking constraints
board clock source
jitter
common clock / separate refclk architecture
```

На уровне link training плохой REFCLK часто выглядит как:

```
LTSSM не проходит Detect/Polling
link нестабилен
link падает в Recovery
не удается поднять higher speed
```

---

# 6. PERST# / fundamental reset

`PERST#` — это fundamental reset PCIe-устройства.

Практический смысл:

```
PERST# low  -> устройство удерживается в reset
PERST# high -> устройство может начинать link initialization
```

TI notes указывает, что PERST# должен удерживаться low, пока power rails и reference clock не стабильны; переход PERST# из low в high обычно означает начало link initialization.

В FPGA Endpoint это критично:

```
если FPGA долго конфигурируется,
если PCIe IP не готов к моменту release,
если PERST# неправильно синхронизирован,
host может не увидеть устройство.
```

Но детали system bring-up лучше оставить для будущей заметки **bring-up phases**.

---

# 7. Electrical Idle

До активной передачи lanes находятся в состоянии, близком к electrical idle.

На этом этапе еще нет нормального packet traffic.

Упрощенно:

```
нет TLP
нет DLLP
нет user traffic
PHY готовится определить, есть ли link partner
```

Для FPGA-разработчика важно:

```
если link stuck до Polling,
это почти всегда lower-level physical/reset/refclk/lane issue,
а не BAR/DMA/TLP issue.
```

---

# 8. Detect state

**Detect** — первый большой этап training.

Задача Detect:

```
понять, есть ли приемник на другой стороне lane
```

То есть устройство пытается определить, подключен ли link partner.

Упрощенно:

```
PCIe transmitter проверяет наличие receiver terminationесли receiver найден -> можно переходить дальшеесли receiver не найден -> оставаться в Detect / retry
```

TI описывает Rx Detect как первый шаг link training: после питания и reference clock устройства запускают receiver-detect circuit на lanes, чтобы определить наличие link partner.

---

# 9. Detect в FPGA debug

Если LTSSM застрял в Detect, обычно думать нужно не про software.

Проверять:

```
есть ли физический link partner;вставлена ли карта / активен ли slot;правильные ли lanes;не перепутаны ли TX/RX;есть ли AC coupling;правильный ли connector/pinout;корректен ли PERST#;есть ли REFCLK;правильный ли GT quad;не отключен ли port в BIOS/host.
```

Если receiver не обнаружен, Transaction Layer вообще еще не участвует.

---

# 10. Detect.Quiet и Detect.Active

В реальных LTSSM есть подстадии Detect, например:

```
Detect.QuietDetect.Active
```

Упрощенная интуиция:

```
Detect.Quiet  — ожидание/пауза/idle часть DetectDetect.Active — активная попытка receiver detection
```

При debug можно видеть циклические переходы между ними.

Это обычно означает:

```
core пытается найти receiver,но не получает условий для перехода дальше.
```

---

# 11. После Detect начинается начальная передача

Когда receiver обнаружен, lanes начинают переходить к обмену training information.

TI app note указывает, что после успешного receiver detection отдельные lanes начинают передавать serial data на 2.5 Gbps, то есть на базовой Gen1 скорости.

Практическая идея:

```
начальная training-последовательность начинается с базовой скорости,а не сразу с максимальной Gen3/Gen4/Gen5.
```

Это важно: даже если устройство поддерживает Gen4, сначала нужно надежно установить базовую связь.

---

# 12. Polling state

**Polling** — этап, где обе стороны начинают обмениваться training sequences.

Задачи Polling:

```
получить bit lock;получить symbol lock;убедиться, что принимаемая последовательность декодируется;начать обмен training information;подготовиться к configuration.
```

TI описывает Polling как стадию, где устройства передают ordered sets, называемые training sequences, на Gen1 speed, чтобы установить bit и symbol lock.

---

# 13. Bit lock

**Bit lock** означает, что receiver смог восстановить clock/data relationship входящего serial stream.

Упрощенно:

```
receiver понял, где находятся биты во входящем потоке
```

Без bit lock невозможно надежно декодировать symbols/blocks.

---

# 14. Symbol lock

**Symbol lock** означает, что receiver может декодировать корректные symbols / ordered sets.

Для Gen1/Gen2 это связано с 8b/10b-символами.

Для Gen3+ encoding другой, но смысл похожий:

```
receiver должен понимать границы и смысл управляющих блоков/последовательностей
```

TI описывает symbol lock как способность receiver декодировать valid 10-bit symbol от transmitter.

---

# 15. Training Sequences: TS1 и TS2

Во время Polling/Configuration PCIe использует специальные ordered sets, которые обычно называют:

```
TS1TS2
```

Они нужны для обмена параметрами link training.

Упрощенно TS1/TS2 помогают передать:

```
link number;lane number;supported rates;training control information;lane polarity / reversal-related information;переход к следующим substates.
```

В FPGA ты обычно не видишь содержимое TS1/TS2 напрямую. Но состояние LTSSM и negotiated speed/width являются результатом этого обмена.

---

# 16. Polling.Active

Типичная интуиция:

```
Polling.Active:    активно посылаются training sequences;    стороны пытаются получить lock;    проверяется, что link partner отвечает.
```

Если LTSSM застрял в Polling.Active, возможные причины:

```
receiver найден, но reliable decode не получается;проблемы signal integrity;ошибка lane mapping;перепутана polarity без корректной инверсии;REFCLK/jitter проблема;неправильная speed capability/config;retimer/redriver issue.
```

---

# 17. Polling.Configuration

После начальной части Polling стороны переходят к более структурированному обмену training information.

Упрощенно:

```
Polling.Configuration:    link partner уже виден;    ordered sets принимаются;    идет подготовка к lane/link configuration.
```

Если stuck здесь, значит physical contact есть, но параметры link еще не согласованы достаточно для Configuration.

---

# 18. Configuration state

**Configuration** — этап, где link договаривается о структуре lanes.

Задачи Configuration:

```
определить link width;назначить lane numbers;согласовать link number;выполнить lane-to-lane deskew;учесть lane reversal;подготовить переход в L0.
```

TI notes описывает Configuration как stage, где выполняется lane-to-lane deskew, определяется link width, и каждая lane получает link number и lane number.

---

# 19. Link width negotiation

Link width — это фактическая ширина соединения:

```
x1x2x4x8x16
```

Vivado IP может быть настроен, например, на `x4`, но фактически link может подняться как `x1`.

Причины:

```
не все lanes подключены;часть lanes не работает;lane mapping ошибка;host/root port ограничен;slot bifurcation;retimer/topology;board issue;IP configured capability отличается от ожидания.
```

AMD configuration status interface показывает negotiated width, причем этот output валиден, когда Data Link initialization завершена.

---

# 20. Lane numbering

Во время Configuration каждая lane получает номер внутри link.

Например для `x4`:

```
lane 0lane 1lane 2lane 3
```

Это важно, потому что multi-lane PCIe передает данные распределенно по lanes.

На Physical Layer нужно понять:

```
какая физическая lane соответствует logical lane 0;есть ли lane reversal;какие lanes участвуют в link.
```

---

# 21. Lane reversal

Lane reversal позволяет некоторым PCIe designs работать, даже если порядок lanes на плате развернут.

Например физически:

```
FPGA lane 0 -> connector lane 3FPGA lane 1 -> connector lane 2FPGA lane 2 -> connector lane 1FPGA lane 3 -> connector lane 0
```

Если IP и link partner поддерживают lane reversal, link может подняться.

AMD Physical Layer documentation отмечает, что physical layer supports lane reversal для multi-lane designs и lane polarity inversion.

---

# 22. Polarity inversion

Если differential pair polarity перепутана:

```
P/N swapped
```

PCIe PHY во многих случаях может это обнаружить и компенсировать.

Это относится к Physical Layer training.

Но лучше не использовать это как замену правильному pinout. При bring-up polarity inversion может быть полезным механизмом спасения, но debug лучше начинать с проверки схемы и constraints.

AMD docs указывают, что Physical Layer поддерживает lane polarity inversion в соответствии с PCIe Base Specification.

---

# 23. Lane deskew

В multi-lane link данные приходят по разным lanes с небольшими задержками.

Configuration выполняет deskew:

```
выравнивает lanes между собой,чтобы данные можно было собрать обратно в правильном порядке.
```

Это особенно важно для:

```
x4x8x16длинных трассretimer topologiesсложных backplane/connector paths
```

TI application note прямо указывает lane-to-lane deskew как часть Configuration stage.

---

# 24. Переход в L0

После успешной Configuration link может перейти в **L0**.

`L0` — нормальное активное состояние link.

Упрощенно:

```
L0 = link trained and operational
```

TI notes описывает L0 как normal operational state, где data и packets can be sent and received.

Но для Vivado debug важно не ограничиваться только словом “L0”.

---

# 25. LinkUp не всегда значит “всё готово”

AMD configuration status interface показывает несколько уровней link status:

```
00 — no receivers detected01 — link training in progress10 — link up, DL initialization in progress11 — link up, DL initialization completed
```

То есть для полноценной работы желательно видеть не только Physical Layer link-up, но и завершенную Data Link initialization.

Практический вывод:

```
LTSSM near L0 / LinkUp — хорошо,но для TLP traffic важен завершенный Data Link initialization.
```

---

# 26. `cfg_phy_link_down`

В AMD PCIe IP есть сигнал вида:

```
cfg_phy_link_down
```

Он отражает status PCIe link based on Physical Layer LTSSM:

```
1 -> Link Down0 -> Link Up
```

AMD docs также отмечают, что по PCIe specification LinkUp может быть `1` в Recovery, L0, L0s, L1 и L2 cfg_ltssm states.

Это важно: Recovery не обязательно означает, что link полностью потерян.

---

# 27. Negotiated speed

После training можно посмотреть фактическую speed:

```
Gen1: 2.5 GT/sGen2: 5.0 GT/sGen3: 8.0 GT/sGen4: 16.0 GT/sGen5: 32.0 GT/s
```

Конкретный IP имеет свои кодировки. Например, AMD UltraScale+ PCIe4 configuration status interface показывает `cfg_current_speed`, где 2.5, 5.0 и 8.0 GT/s имеют отдельные encodings, а для HBM PCIE4C также указывается 16.0 GT/s.

Если ожидали Gen3, а получили Gen1, значит link поднялся, но speed negotiation/equalization не дошли до ожидаемой скорости.

---

# 28. Почему link может сначала подняться на Gen1

Базовая training-последовательность начинается с низкой скорости, чтобы обеспечить совместимость и надежный старт.

TI notes указывает, что после Rx Detect lanes начинают serial transmission на 2.5 Gbps, базовой Gen1 скорости.

Дальше, если обе стороны поддерживают более высокую скорость, link может перейти через speed change / Recovery / equalization.

Упрощенно:

```
сначала установить базовую связь;потом попробовать повысить скорость.
```

---

# 29. Recovery state

**Recovery** — состояние, которое используется для восстановления или изменения параметров link.

Recovery может появляться:

```
после начального L0 для перехода на более высокую speed;при retraining;при ошибках link;после потери lock;при попытке изменить width/speed;во время equalization.
```

В этой заметке Recovery рассматриваем только как часть training path.

Подробно Recovery-сценарии лучше разбирать в будущей теме **LTSSM**.

---

# 30. Equalization для Gen3+

Для Gen3 и выше одной базовой training-последовательности недостаточно: нужна link equalization.

TI notes описывает equalization как процесс оптимизации link для higher data rates; он происходит, если все устройства поддерживают Gen3 или выше, и может повторяться для последующих поколений.

Упрощенно equalization подбирает transmitter/receiver настройки так, чтобы serial link был устойчив на высокой скорости.

---

# 31. Equalization phases

Для Gen3/Gen4 equalization обычно делится на несколько phases.

TI notes описывает phases 0/1 как начальную настройку preset values и переход к Gen3, а phases 2/3 как fine tuning; после завершения phase 3 link может перейти в L0 на Gen3.

Для FPGA debug это означает:

```
link может успешно работать на Gen1,но не проходить Gen3/Gen4 equalization.
```

Тогда проблема часто не в TLP, а в SI/equalization/channel/retimer/refclk.

---

# 32. Начальный L0 и финальный L0

Полезно различать:

```
L0 at lower speedL0 after speed change/equalization
```

Например:

```
ожидали Gen3 x4link сначала поднялся Gen1 x4потом через Recovery попытался Gen3после equalization вернулся L0 Gen3 x4
```

Или неудачный вариант:

```
Gen1 x4 OKRecovery to Gen3failurefallback to Gen1
```

В ILA/Vivado Link Debug это может выглядеть как серия переходов через Recovery.

---

# 33. Что происходит после L0

После L0 начинается нормальная работа lower layers:

```
Data Link initializationflow control initializationDLLP exchangeconfiguration transactionsenumerationTLP traffic
```

Важно:

```
L0 — это не конец всей PCIe bring-up истории.L0 — это конец базовой physical link training истории.
```

Дальше host должен перечислить устройство, назначить BAR, включить command bits и т.д.

Но это уже тема **bring-up phases**.

---

# 34. Training sequence vs enumeration

Не путать:

```
link training
```

и

```
PCIe enumeration
```

Link training происходит на Physical/Data Link level.

Enumeration происходит позже через Configuration TLP.

Упрощенно:

```
LTSSM reaches L0        |Data Link ready        |Root Complex reads config space        |Device appears in OS
```

Если link не L0, enumeration невозможна.

Если link L0, но enumeration не прошла — причина уже может быть выше по стеку или в timing/system bring-up.

---

# 35. Training sequence и Root Port / Endpoint

Обе стороны участвуют в link training:

```
Root Port / Downstream PortEndpoint / Upstream Port
```

Они обмениваются ordered sets и сходятся к согласованной конфигурации link.

Важно:

```
FPGA Endpoint не “поднимает link сама”.Она тренируется вместе с link partner.
```

Если host/root port не готов, отключен или держит reset, FPGA LTSSM будет ждать/повторять training.

---

# 36. Training через retimer/redriver

Если между FPGA и host есть retimer, topology усложняется.

TI notes указывает, что при retimer link может быть разделен на части, и link initialization выполняется отдельно на обеих сторонах retimer.

Практически:

```
Root Port <-> RetimerRetimer <-> Endpoint
```

Проблема training может быть на одном сегменте, а не обязательно прямо между host и FPGA.

---

# 37. Что смотреть в Vivado Hardware Manager

AMD Vivado PCIe Link Debug может показывать LTSSM State Trace и LTSSM State Diagram. В Hardware Manager можно увидеть active PCIe link status, transitions, current link status вроде Gen3x8 и connected GTs; также доступна команда `report_hw_pcie`.

Практически полезно смотреть:

```
последнее LTSSM state;какие states посещались;есть ли циклы Detect <-> Polling;есть ли переходы в Recovery;какая current speed;какая negotiated width;какие GTs используются.
```

---

# 38. Почему trace важнее одного текущего состояния

Если просто посмотреть текущее state, можно увидеть:

```
Detect
```

Но это не говорит, был ли link когда-то в L0.

State trace показывает историю:

```
Detect -> Polling -> Configuration -> L0 -> Recovery -> Detect
```

Это совсем другая ситуация, чем:

```
Detect -> Detect -> Detect
```

Первый случай говорит: link поднимался и потом упал.

Второй: link не прошел начальное обнаружение.

---

# 39. Типовые patterns LTSSM при debug

## Pattern 1

```
Detect only
```

Вероятно:

```
receiver не найден;нет partner;reset/refclk/lane/pinout problem.
```

## Pattern 2

```
Detect -> Polling -> Detect
```

Вероятно:

```
receiver найден, но reliable training не проходит;SI/polarity/lane/refclk issue.
```

## Pattern 3

```
Detect -> Polling -> Configuration -> Detect
```

Вероятно:

```
configuration не завершилась;lane width/deskew/lane mapping issue.
```

## Pattern 4

```
L0 -> Recovery -> L0
```

Может быть нормальным при speed change или transient recovery.

Если часто повторяется:

```
link instability;SI/equalization issue;clocking issue;partner compatibility.
```

---

# 40. Как связать LTSSM и status signals

Полезная карта:

```
Detect:    cfg_phy_link_status = no receivers detected / trainingPolling / Configuration:    link training in progressL0 with DL init:    link up, DL initialization in progressL0 after DL init:    link up, DL initialization completed
```

AMD configuration status interface прямо определяет `cfg_phy_link_status` как 2-bit status: no receivers detected, link training in progress, link up with DL initialization in progress, and link up with DL initialization completed.

---

# 41. Negotiated width valid не всегда сразу

`cfg_negotiated_width` нельзя читать как meaningful в любой момент.

AMD docs указывают, что negotiated width valid when `cfg_phy_link_status == 11b`, то есть когда Data Link initialization complete.

Практическое правило:

```
сначала убедись, что link/DL ready;потом анализируй negotiated width.
```

---

# 42. Current speed и expected speed

После link training полезно сравнить:

```
IP configured maximum speedboard/slot supported speedroot port supported speedactual current speed
```

Если actual speed ниже, это не обязательно ошибка.

Возможные причины:

```
link partner не поддерживает выше;BIOS ограничил speed;signal integrity не позволяет higher speed;equalization failed;retimer limits;Vivado IP configured ниже;PCIe slot электрически ограничен.
```

---

# 43. Link training и user clock

После link training PCIe IP обычно предоставляет user clock / user reset для логики.

Важный порядок:

```
не стартовать user transaction logic,пока PCIe core не сообщил, что link/user interface готов.
```

Иначе твой RTL может начать отправлять запросы раньше, чем link и Data Link Layer готовы.

Детали reset/user clock sequencing лучше разбирать в теме **PCIe integration in FPGA**.

---

# 44. Link training не проверяет твою BAR logic

Если LTSSM дошел до L0, это не значит:

```
BAR decode работает;Completion формируются правильно;DMA descriptors корректны;driver loaded;interrupts работают;CDC внутри FPGA безопасен.
```

Это значит только:

```
нижний physical/link training прошел достаточно,чтобы link стал operational.
```

---

# 45. Link training не проверяет throughput

Даже если link поднялся как Gen3 x4, это не гарантирует ожидаемый throughput.

Дальше влияют:

```
MPS/MRRScreditsoutstanding requestsDMA architecturehost/root complexdriverAXI/stream backpressurebufferingcache/IOMMU
```

Это отдельная будущая тема про **PCIe flow control / credits / throughput limits**.

---

# 46. Минимальный sequence для запоминания

```
1. Reset released, REFCLK stable.2. Detect: найти receiver.3. Polling: получить bit/symbol lock через training sequences.4. Configuration: согласовать lanes, width, numbering, deskew.5. L0: link operational at initial negotiated state.6. Recovery/Equalization: перейти на higher speed, если поддерживается.7. L0 again: normal operation at final speed/width.8. Data Link initialization complete.9. Configuration transactions / enumeration can proceed.
```

---

# 47. Практическая таблица состояний

|Этап|Главная задача|Если застряли здесь|
|---|---|---|
|Detect|Найти receiver|Проверить partner, lanes, REFCLK, PERST#, pinout|
|Polling|Получить lock и обмен training sequences|Проверить SI, polarity, lane mapping, clocking|
|Configuration|Согласовать width/lane numbering/deskew|Проверить multi-lane routing, reversal, width capability|
|L0|Нормальная работа link|Дальше смотреть DL init, enumeration, TLP|
|Recovery|Speed change / retrain / recovery|Проверить equalization, SI, stability, speed capability|

---

# 48. Что важно для FPGA/Vivado

Для PCIe Endpoint в FPGA при link training нужно всегда иметь под рукой:

```
LTSSM stateLTSSM state tracecfg_phy_link_downcfg_phy_link_statuscfg_current_speedcfg_negotiated_widthuser_lnk_up / link_upGT reset done/statusPERST#REFCLK presencePCIe IP configured max speed/widthactual lane connections
```

Vivado PCIe Link Debug особенно полезен, потому что показывает не только текущий state, но и transition history. AMD UG908 указывает, что при включенном link debug core stores LTSSM state transitions, доступные в Vivado Hardware Manager.

---

# 49. Главное резюме

**Link training sequence** — это путь, по которому PCIe PHY/LTSSM переводит link из reset/electrical-idle состояния в рабочее состояние.

Главные этапы:

```
Detect:    найти receiverPolling:    получить bit/symbol lock через training sequencesConfiguration:    согласовать lanes, width, numbering, deskewL0:    normal operational stateRecovery/Equalization:    speed change, retraining, higher-speed stabilization
```

Короткая формула:

```
Detect отвечает: “есть ли кто-то на другом конце?”Polling отвечает: “можем ли мы понимать друг друга?”Configuration отвечает: “какими lanes и шириной работаем?”L0 отвечает: “link готов к нормальному обмену.”Recovery отвечает: “переобучаемся, ускоряемся или восстанавливаемся.”
```

В FPGA/Vivado debug первое правило:

```
если LTSSM не дошел до L0,не начинай debug с BAR, DMA или driver.Сначала проверь physical/link training path.
```