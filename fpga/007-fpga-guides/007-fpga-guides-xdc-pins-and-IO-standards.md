## Pins and I/O standards

**Pins and I/O standards** в контексте FPGA/Vivado и блока **Constraints writing (XDC/SDC)** — это тема про то, как связать логические top-level ports проекта с реальными physical pins корпуса и одновременно задать electrical behavior этих выводов. В Vivado это обычно делается через свойства **`PACKAGE_PIN`** и **`IOSTANDARD`**. `PACKAGE_PIN` задает, на какой package pin будет посажен top-level port, а `IOSTANDARD` задает, по какому programmable I/O standard этот port будет работать как input, output или bidirectional signal. Для top-level ports AMD рекомендует использовать именно `PACKAGE_PIN`, а не `LOC`; `LOC` следует использовать для размещения логических ячеек внутри FPGA fabric.

Главная мысль здесь такая:  
**назначение pins и выбор I/O standard — это не “оформление проекта перед bitstream”, а часть базовой hardware model.** Когда ты пишешь эти constraints, ты не просто выбираешь удобные ножки, а описываешь для tool физическую и электрическую реальность проекта: в каком bank живет сигнал, с каким voltage family он совместим, можно ли этот сигнал поставить рядом с другими, не конфликтует ли он с configuration settings, differential-pair rules или bank-level restrictions. Именно поэтому AMD выделяет целый отдельный I/O and Clock Planning flow и советует валидировать I/O planning через DRC, а финальную проверку делать уже на implemented design и перед bitstream generation.

### Почему это отдельная полноценная подтема

Внутри блока **Constraints writing (XDC/SDC)** тема **Pins and I/O standards** выделяется отдельно потому, что она находится на границе между RTL, PCB и device-specific physics. Clock constraints и timing exceptions в основном описывают временную модель. А pin constraints и I/O standards описывают уже реальное подключение FPGA к внешнему миру. Ошибка в этой области часто не выглядит как “слегка плохой QoR”. Она приводит к тому, что port не может быть legally placed, bank voltage assumptions расходятся с выбранным standard, либо bitstream вообще не удается выпустить из-за DRC violations. UG899 прямо говорит, что способы конфигурирования device, device constraints и configuration voltage влияют на I/O and clock planning.

Для prototyping это особенно важно. На ранней стадии prototype иногда еще можно терпеть недоидеальный timing или временно тяжелую hierarchy, но pin/I/O mistakes терпятся намного хуже. Если неверно выбраны pins или standards, дальше вся цепочка верификации становится менее надежной: simulation может быть хорошей, synthesis — тоже, а вот реальное железо уже не соответствует assumptions проекта. Именно поэтому Vivado поддерживает отдельный **I/O planning project**, а flow можно начинать с пустого I/O planning project, с RTL design project или с post-synthesis netlist project.

### Что именно делает PACKAGE_PIN

Свойство **`PACKAGE_PIN`** определяет конкретное соответствие между логическим top-level port и физическим package pin на корпусе выбранного device. UG912 формулирует это буквально как placement top-level port to a physical package pin on the device и отдельно рекомендует использовать `PACKAGE_PIN` для портов. Это значит, что именно через это свойство ты фиксируешь физический выход или вход сигнала наружу.

Практически это очень важно, потому что pin assignment — это не просто “куда удобно поставить сигнал”. На разных package pins у device могут быть разные роли и ограничения: clock-capable pins, differential pair membership, multifunction pins, принадлежность к конкретному I/O bank, связь с dedicated resources. UG899 отдельно показывает, что в I/O planning flow есть представление device resources, Package Pins window, I/O Ports window и DRC checks, а auto-placement obeys I/O standard and differential pair rules and places global clock pins appropriately. То есть сам tool уже исходит из того, что pins — это структурированная physical resource model, а не flat list of names.

### Что именно делает IOSTANDARD

Свойство **`IOSTANDARD`** задает programmable I/O standard для input, output или bidirectional port. UG912 прямо пишет, что `IOSTANDARD` configures input, output, or bidirectional ports on the target device. Это не “украшение” порта, а electrical contract между FPGA и внешней схемой. От него зависят допустимые voltage levels, совместимость с bank settings, часть DRC rules и сам legal status выбранного pin assignment.

Есть и очень жесткое practical правило: AMD отдельно пишет, что **you must explicitly define an IOSTANDARD on all ports in an I/O Bank before Vivado will create a bitstream**. При этом IOSTANDARD не применяется к **GTs** и **XADCs**. Это один из самых важных фактов всей темы. Он означает, что “оставлю DEFAULT, потом разберусь” — плохая стратегия для нормального project flow. Для обычных I/O banks явное определение standard — это часть обязательной модели проекта, а не желательная доработка на потом.

### Почему pins и I/O standards нельзя рассматривать отдельно

Самая частая conceptual ошибка — думать, что сначала можно выбрать pins, а потом уже назначить standards. На практике это одна связанная задача. `PACKAGE_PIN` определяет physical placement порта, а `IOSTANDARD` определяет, какой electrical mode этот pin должен поддерживать. Совместимость этих двух вещей зависит от bank, VCCO-related assumptions, device configuration и соседних ports. UG912 показывает, что `IO_BANK` связан и с package_pin, и с port, и с I/O standard being implemented by the I/O block. Это очень наглядно показывает, что pin, bank и standard образуют одну систему.

Именно поэтому хороший инженерный порядок обычно такой: сначала ты понимаешь interface requirements и bank-level electrical picture, потом распределяешь ports по pins, а уже затем валидируешь это DRC’ами и I/O planning tools. Если же делать наоборот — “раскидаем по свободным ножкам, потом подгоним standards” — легко попасть в неочевидные conflicts, которые проявятся уже далеко после написания RTL. UG899 как раз строит I/O planning как отдельный design flow с validation steps, а не как последнее косметическое действие перед implementation.

### Bank thinking: почему I/O bank важнее, чем кажется

Очень полезно мыслить не отдельными pins, а **I/O banks**. Практически большинство важных ограничений живет именно на уровне bank: совместимость IOSTANDARD, configuration voltage assumptions, наличие специальных pins, grouping of interfaces. UG912 прямо связывает `IO_BANK` с package pins, ports и implemented I/O standard. UG899 дополнительно показывает, что configuration-related properties и DRC checks зависят от bank settings.

Это значит, что хороший pin planning почти всегда bank-centric. Когда engineer смотрит только на “свободные pins”, он пропускает более важный вопрос: **не разрушает ли это bank-level electrical model**. Особенно это заметно в проектах с mixed-voltage interfaces, с несколькими внешними buses, с clock inputs и с memory-like groupings, где локально красивое решение по pins может оказаться плохим на уровне whole bank. UG899 прямо подчеркивает, что I/O planning нужен для координации между FPGA designer и PCB designer и для проверки design requirements до финальной реализации.

### Конфигурационные настройки тоже влияют на I/O planning

В этой теме есть еще одна важная, но часто недооцененная часть — **device configuration settings**. UG899 прямо говорит, что configuration method, device constraints и configuration voltage влияют на I/O and clock planning. Для некоторых devices нужно правильно задать **CFGBVS** и **CONFIG_VOLTAGE**; DRCs then are issued based on IOSTANDARD and CONFIG_VOLTAGE settings for the bank. Для example с configuration bank AMD даже показывает, что неверная комбинация может вызывать critical warnings уровня **CFGBVS-4**.

Практический смысл здесь очень большой: I/O constraints — это не только то, что ты пишешь на top-level ports. Это еще и общая configuration context выбранного device. Если не учитывать эту часть, можно получить ситуацию, где порты individually выглядят разумно, но global device setup делает комбинацию illegal. Именно поэтому в серьезном prototyping flow device configuration лучше фиксировать до того, как pin planning ушел слишком далеко.

### Differential pairs, clock pins и специальные категории pins

Не все pins одинаковы по назначению. В UG899 отдельно отмечено, что в I/O planning GUI clock-capable pins и VREF pins имеют специальные обозначения, а dedicated I/O pins относятся к устройству, а не к bank. Кроме того, auto-place I/O obeys differential pair rules and places global clock pins appropriately. Это означает, что часть ports нельзя safely раскидывать без учета special pin classes.

Для prototyping это особенно важно на clock inputs и differential interfaces. Там pin choice влияет не только на удобство трассировки на плате, но и на саму доступность нужных hardware resources. Поэтому если в проекте есть external clocks, differential data, reference inputs или multifunction pins, тема **Pins and I/O standards** быстро перестает быть “простым set_property PACKAGE_PIN” и превращается в полноценную архитектурную задачу.

### Как это обычно выглядит в XDC

На базовом уровне XDC для этой темы выглядит очень просто:

```XDC
set_property PACKAGE_PIN E3 [get_ports clk_in]  
set_property IOSTANDARD LVCMOS33 [get_ports clk_in]  
  
set_property PACKAGE_PIN H17 [get_ports led0]  
set_property IOSTANDARD LVCMOS18 [get_ports led0]
```

Для `IOSTANDARD` UG912 прямо дает XDC syntax вида `set_property IOSTANDARD value [get_ports port_name]`. Для `PACKAGE_PIN` UG912 отдельно определяет это свойство как assignment of a top-level port to a physical package pin.

Но deceptively simple syntax здесь не должна обманывать. Простота Tcl-строки не означает простоту самой инженерной задачи. Самое важное в этой теме — не написать две команды, а убедиться, что выбранный pin legal, выбранный standard legal, bank picture coherent, configuration settings consistent и что после этого DRC действительно чистый.

### Почему DRC здесь обязателен, а не факультативен

После того как pins и I/O standards назначены, проект нужно валидировать. UG899 прямо говорит: after performing I/O and clock planning, validate your design, run DRCs, and for final validation implement the design and generate a bitstream. Это очень важная последовательность. В этой теме DRC — не последняя бюрократия, а основной способ проверить, что electrical and physical rules реально соблюдены.

Есть и очень практичные примеры таких DRCs. В tutorial flow AMD показывает aggregated violations вроде **UCIO-1** и **NSTD-1**; при этом `UCIO-1` aggregates unconstrained pin-related violations, а `NSTD-1` covers ports without proper I/O standard assignment. В примере AMD отдельно показывает, что одна aggregated violation может включать десятки и сотни отдельных объектов. Это хороший reminder: один DRC ID в report может означать не “одну мелочь”, а системную проблему с большим числом ports.

### Что означает хороший результат в prototyping

Для prototype хороший результат по этой теме — это не просто “bitstream generated”. Обычно он выглядит так:  
все top-level ports имеют осмысленные pin assignments;  
все обычные I/O ports в своих banks имеют явно заданные IOSTANDARDs;  
bank-level choices совместимы с configuration and voltage model;  
clock and differential ports сидят на подходящих pins;  
DRC clean или, как минимум, оставшиеся items осознанно объяснены. Всё это хорошо соответствует I/O planning flow, который AMD описывает как путь от раннего planning до final validation with an implemented design.

Для prototyping это особенно важно потому, что pin plan часто потом начинает жить дольше, чем сам первый RTL. Если сразу сделать pins and standards небрежно, проект очень быстро обрастает зависимостями: PCB, test scripts, lab setups, connectors, daughter cards, pin labels. Поэтому на практике грамотные pin constraints часто экономят больше времени, чем еще одна optimization в логике. Это инженерный вывод из самой структуры AMD I/O planning flow, где pin planning вынесен в отдельную дисциплину.

### Типичные ошибки

Самая частая ошибка — задать `PACKAGE_PIN`, но не задать `IOSTANDARD` на обычных I/O ports. Для Vivado это критично: AMD прямо пишет, что before bitstream every port in an I/O bank must have explicit IOSTANDARD, за исключением GTs и XADCs.

Вторая ошибка — использовать для top-level ports `LOC` вместо `PACKAGE_PIN`. UG912 прямо рекомендует для ports именно `PACKAGE_PIN`, а `LOC` — для logic cells.

Третья ошибка — думать о pins по одному сигналу и не смотреть на bank-level совместимость. Это особенно опасно при mixed-voltage interfaces и configuration-sensitive banks. UG912 и UG899 вместе показывают, что pin, bank и IOSTANDARD связаны, а DRCs зависят от IOSTANDARD plus configuration settings.

Четвертая ошибка — не запускать DRC сразу после назначения pins/standards. AMD рекомендует делать I/O validation через DRC и финальную проверку на implemented design.

Пятая ошибка — воспринимать auto-placement как замену инженерного решения. UG899 действительно пишет, что auto-place obeys I/O standard and differential pair rules, но это не отменяет необходимости понимать interface grouping, bank usage и board intent. Auto-placement — это инструмент, а не архитектурная ответственность.

### Практический итог

**Pins and I/O standards** — это полноценная подтема внутри блока **Constraints writing (XDC/SDC)**, потому что она отвечает не только на вопрос “куда посадить сигнал”, а на более важный вопрос: **как описать physical и electrical подключение FPGA к внешнему миру так, чтобы Vivado, плата и выбранный device жили в одной модели.** По официальным материалам AMD, `PACKAGE_PIN` задает assignment top-level port to a package pin, `IOSTANDARD` задает electrical standard порта, explicit IOSTANDARD required for ordinary I/O banks before bitstream, а I/O planning нужно валидировать через DRC и final implemented design checks.

Если сказать совсем коротко: **эта тема про то, чтобы порт в RTL стал реальным, legal и электрически корректным выводом на FPGA.** И чем раньше pins, banks, standards и configuration assumptions собраны в coherent model, тем спокойнее дальше идут synthesis, implementation и lab bring-up.