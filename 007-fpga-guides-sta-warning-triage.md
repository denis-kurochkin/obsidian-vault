## Warning triage

**Warning triage** в контексте FPGA/Vivado и блока **Synthesis report analysis** — это не просто чтение списка сообщений, а системная сортировка warnings и critical warnings по реальному инженерному риску: что надо исправлять немедленно, что указывает на structural problem в RTL или constraints, а что можно временно отложить без потери контроля над проектом. В Vivado есть несколько уровней сообщений и отчетов, а severity сама по себе уже несет смысл: **Warning** означает, что результат может быть sub-optimal, потому что constraints или specifications могут применяться не так, как задумано; **Critical Warning** означает, что часть user input или constraints не будет применена либо нарушает best practices, и AMD настоятельно рекомендует это проверить.

Главная мысль здесь такая:  
**не все warnings одинаково опасны, но triage нельзя сводить к механическому “исправить все подряд” или “игнорировать пока собирается”.** Хороший triage отвечает на три вопроса:  
какое сообщение говорит о потере корректности модели;  
какое сообщение бьет по future timing/QoR;  
какое сообщение пока только сигнализирует о локальном несовершенстве, но не ломает flow. Именно поэтому Vivado дает не только Messages window, но и отдельные отчеты вроде `report_methodology`, `report_drc`, `report_cdc`, а UG906 прямо советует уделять особое внимание **Critical Warnings**, потому что они влияют и на timing closure, и на sign-off quality.

### Почему это отдельная полноценная подтема

Внутри блока **Synthesis report analysis** тема **warning triage** важна потому, что synthesis редко проваливается из-за одной “фатальной” ошибки. Гораздо чаще проект проходит шаг synthesis или implementation, но оставляет после себя длинный хвост warnings, и именно качество triage определяет, превратится ли этот хвост в controlled engineering backlog или в хаотичный future debug. AMD прямо пишет, что DRC messages, Critical Warnings и Warnings нужно review early in the flow, иначе проблемы вылезают позже; а для methodology checks сказано отдельно, что violations нужно address, особенно Critical Warnings.

### Сначала нужно понимать источник warning, а не только его текст

Одна из самых полезных привычек — сразу различать, **откуда** пришло сообщение. В Vivado есть по меньшей мере четыре важных класса источников:

- обычные tool messages из synthesis/implementation log и Messages window;
- **DRC** violations;
- **methodology** violations;
- **CDC** report violations.

Это важно, потому что они ведут себя по-разному. Например, severity у обычных сообщений можно менять через `set_msg_config`, но для `report_cdc` AMD прямо отмечает, что `set_msg_config` на его severities не действует, потому что этот отчет не использует message manager. А для DRC и Methodology есть отдельные reports, waivers и свой жизненный цикл.

### Первый уровень triage: severity действительно имеет смысл

Хороший triage почти всегда начинается с severity. Vivado различает как минимум **Advisory**, **Warning**, **Critical Warning**, **Error** и **Fatal**. Для DRC AMD формулирует смысл severity так: **Warning** — design results might be sub-optimal, потому что constraints/specifications могут не примениться как intended; **Critical Warning** — часть user input или constraints не будет применена либо нарушает best practices, и это нужно разбирать. Отсюда следует практическое правило: **Critical Warning почти всегда должен попадать в верх triage-очереди**, даже если flow пока не остановился.

Особенно это важно потому, что в Vivado **Critical Warnings в ранних стадиях могут превратиться в Errors позже**. UG906 прямо пишет, что Critical Warning DRCs в early stages later become Errors during implementation flow and prevent bitstream creation. UG949 добавляет, что Critical Warning DRCs during implementation become Errors during bitstream generation. Это означает, что triage по Critical Warning — не “полезная уборка”, а часто прямой способ не встретить жесткую блокировку на этапе `write_bitstream`.

### Второй уровень triage: сообщения, которые ломают саму модель проекта

После severity полезно выделять warnings, которые означают, что tool уже **не видит проект так, как ты думаешь**. Сюда обычно относятся:

- unconstrained или неверно constrained I/O/clock cases;
- violations, где constraints/user input not applied as intended;
- methodology warnings про broken design practices;
- CDC findings уровня Critical, где crossing классифицируется как unsafe или unknown.

Почему они так опасны: они не просто ухудшают качество результата, а искажают базовую модель design. UG895/UG899 прямо определяют Critical Warning как случай, где user input or constraints will not be applied or do not adhere to best practices. Для CDC UG906 показывает, что critical rules соответствуют случаям вроде unsynchronized single-bit/multi-bit CDC, async reset without synchronization, combinatorial logic before synchronizer, fanout before synchronizer и multi-clock fanin.

Практический вывод отсюда простой:  
**если warning говорит “tool больше не верит в твои assumptions”, такой warning надо triage’ить раньше, чем большинство optimization warnings.**

### Methodology warnings: это не косметика

Vivado отдельно дает `report_methodology`, и UG906 подчеркивает, что methodology DRCs зависят от стадии flow: на elaborated RTL это lint-style checks, на synthesized design — netlist and constraint checks, а на implemented design — implementation and timing checks. В том же разделе AMD прямо советует address all methodology violations, especially Critical Warnings. Это означает, что methodology warnings хорошо подходят для **раннего triage**, потому что они часто показывают проблему еще до того, как она превратится в concrete timing failure.

Очень полезно воспринимать methodology warning не как “идеологию tool”, а как **early prediction**: если violated methodology already flagged on synthesized design, у этого часто есть downstream cost по QoR, timing closure или maintainability. Поэтому в зрелом triage methodology warnings обычно идут сразу после hard Errors/Critical Warnings, связанных с constraints correctness.

### DRC warnings: у них короткий путь до блокировки bitstream

С DRC triage правило еще жестче. UG906 прямо рекомендует review DRC messages, Critical Warnings and Warnings early, а examples показывают, что unconstrained I/O can be reported as Critical Warning after synthesis and later surface again post-route, where at `write_bitstream` it is elevated to Error. Поэтому DRC-warning triage — это не “посмотрю после timing”, а почти всегда early-stage activity.

Хороший practical принцип здесь такой:  
**все DRC Critical Warnings считать blocking until disproven.**  
Даже если run еще идет дальше, это не значит, что сообщение можно safely отложить. AMD явно предупреждает, что такие вещи позже становятся hard blockers для bitstream generation.

### CDC warnings и criticals нужно triage’ить особым способом

CDC — это отдельный мир. Для `report_cdc` Vivado не использует обычный message manager, и severity rules здесь свои. UG906 показывает конкретные CDC rules: unsynchronized single-bit and multi-bit CDC, async reset without synchronization, combinatorial logic before synchronizer, fanout before synchronizer и multi-clock fanin marked as **Critical**; synchronized-but-missing-ASYNC_REG и некоторые mux/CE-based topologies идут как **Warning**; clean synchronized cases — как **Info**.

Но тут есть важный нюанс для triage: **report_cdc по умолчанию показывает только одно нарушение на endpoint**, по правилу precedence. UG906 прямо говорит, что if multiple violations exist for the same endpoint, only the highest-precedence one is reported by default; если его waive’нуть, появится следующее. Есть и режим `-all_checks_per_endpoint`, чтобы видеть полный набор checks. Это делает CDC triage особенным: sometimes one visible Critical is only the top layer of the problem.

Практически это означает:  
**CDC triage нельзя завершать по одному report screenshot.**  
Нужно понимать precedence, safe/unsafe categories и при необходимости раскрывать все checks per endpoint. Иначе есть риск “починить” верхнюю проблему и пропустить вторую, которая скрывалась ниже.

### Triage должен быть stage-aware

Очень полезно сортировать warnings еще и по тому, **на какой стадии** они возникли. UG906 прямо делит methodology checks по стадиям: RTL lint-style checks на elaborated design, netlist/constraint checks на synthesized design, implementation and timing checks на implemented design. Это дает хороший triage framework:

- на **elaborated RTL** важнее semantic and lint-like issues;
- на **synthesized design** — structural warnings, constraint applicability, CDC, methodology;
- на **implemented design** — DRC, routing, timing, congestion-related warnings.

Такой подход помогает не смешивать сообщения разной природы. Например, inferred latch warning на elaborated/synthesized stage — это обычно RTL semantics issue. А router congestion critical warning — это уже implementation-stage physical issue. Они обе важны, но triage actions у них разные.

### Не все warnings нужно “чинить”, но почти все нужно классифицировать

Хороший triage не равен слепому zero-warning policy. Некоторые warnings бывают:

- временно ожидаемыми на промежуточной стадии;
- уже известными и допустимыми under current prototype assumptions;
- связанными с external IP state or versioning and not immediately blocking;
- низкоприоритетными compared with architecture-breaking issues.

Но даже такие warnings должны быть **классифицированы**, а не просто оставлены в общем шуме. UG906 поддерживает formal waiver flow для DRC, CDC и Methodology: можно report only waived violations или all violations with `-waived` / `-no_waiver`, и перед final bitstream AMD советует проверить, что waived only expected issues remain.

Это очень полезная инженерная дисциплина:  
warning может остаться unresolved, но не должен оставаться **unowned**.

### Waiver — это часть triage, а не способ забыть проблему

UG906 прямо говорит: after you define waivers and before final bitstream, verify that only expected violations are waived. То есть waiver — это не suppression for comfort, а controlled statement: “мы понимаем эту проблему и сознательно принимаем ее на данном этапе”. Более того, `report_cdc`, `report_drc` и `report_methodology` поддерживают специальные режимы показа waived violations, чтобы их можно было отдельно ревизовать.

Практический вывод:  
**если warning triage зрелый, waived set должен быть маленьким, объяснимым и reviewable.**  
Если waived list растет бесконтрольно, triage уже перестает быть triage и превращается в hiding mechanism.

### Suppression и severity changes нужно использовать очень осторожно

Vivado позволяет менять message severity и suppress messages через `set_msg_config`, а также через IDE. Но AMD прямо предупреждает: use caution when demoting critical warnings, because these messages flag problems that might result in errors later in the design flow. Можно также limit the number of repeated messages, а default message limit is 100.

Это делает правильную политику очень простой:  
**triage сначала, suppression потом.**  
Сначала нужно понять, что сообщение означает и к какому классу риска относится. И только после этого можно решать, нужно ли понизить noise, ограничить repetition или suppress known-benign case. Иначе tool noise reduction легко превращается в bug concealment. Особенно это важно потому, что для некоторых report-based flows, например `report_cdc`, severity вообще не управляется обычным message mechanism.

### Хорошая practical шкала triage

Для реальной работы удобно держать простую шкалу из четырех уровней.

**1. Blocking now**  
Errors, Fatal, DRC Critical Warnings, warnings about unapplied constraints, severe CDC Criticals, messages that can block bitstream or invalidate timing model. DRC Critical Warnings later become Errors at bitstream generation, so these belong here by default.

**2. Fix before trusting QoR**  
Methodology Critical Warnings, synthesized-design structural warnings, control/constraint issues, CDC Warnings that indicate incomplete synchronization practice, and warnings suggesting sub-optimal results due to misapplied specifications. These may not block the run immediately, but they can poison timing closure and sign-off quality.

**3. Track and schedule**  
Non-blocking warnings whose impact is understood, version/upgrade-related messages, known prototype compromises, or local optimization issues that do not currently invalidate function or timing model. These can remain open, but only with owner and rationale.

**4. Suppress or limit only after proof**  
Repeated benign messages, already reviewed known IDs, or tool-noise cases where the engineering meaning is exhausted. Use `set_msg_config` only after that proof, not instead of it.

### Что обычно оказывается хорошим workflow

Хороший **warning triage workflow** обычно выглядит так. Сначала смотри **Critical Warnings and Errors** из current stage log/messages. Затем запускай или открывай **report_methodology**, **report_drc**, а при multi-clock design — **report_cdc**. После этого группируй находки по risk class: model-breaking, QoR-threatening, local-and-known. Потом решай: fix now, fix later with owner, или explicit waiver. И только в самом конце думай о suppression/limit policies for repeated noise. Такой workflow хорошо согласуется с тем, как AMD позиционирует review of DRC/Methodology/CDC reports и caution around severity changes.

### Типичные ошибки

Самая частая ошибка — triage по количеству сообщений, а не по их инженерному смыслу. Один Critical Warning about unapplied constraints может быть опаснее, чем десятки ordinary warnings. Это прямо следует из официальных определений severity.

Вторая ошибка — demote critical warnings too early. AMD отдельно предупреждает, что demoting critical warnings is dangerous because they may become errors later in the flow.

Третья ошибка — считать CDC warnings обычными messages. Для `report_cdc` действуют свои precedence rules, и lower-level issues can be hidden by default.

Четвертая ошибка — использовать waivers как свалку. UG906 рекомендует перед final bitstream explicitly review waived violations и убедиться, что waived only expected items.

Пятая ошибка — откладывать DRC Critical Warnings “до implementation”. UG906 и UG949 прямо советуют review them early because later they turn into Errors and can stop bitstream creation.

### Практический итог

**Warning triage** — это полноценная подтема внутри блока **Synthesis report analysis**, потому что она отвечает не просто на вопрос “есть ли warnings”, а на более важный вопрос: **какие сообщения реально меняют trust level к design, constraints и sign-off path**. Official Vivado guidance показывает, что severity meaningful, DRC and Methodology violations need early review, CDC has its own precedence model, waivers must be audited, and suppression/severity changes should be used with caution.

Если сказать совсем коротко: **warning triage** — это искусство быстро отделять tool noise от messages, которые уже сейчас подсказывают, что проект перестает соответствовать твоим assumptions. И чем раньше в проекте появляется такая дисциплина, тем меньше шанс, что настоящий blocker будет спрятан среди “еще ста предупреждений”.