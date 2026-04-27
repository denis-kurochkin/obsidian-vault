## Inferred latches

**Inferred latches** в контексте FPGA/Vivado и блока **Synthesis report analysis** — это latches, которые Vivado выводит из RTL не потому, что ты явно описал latch-based storage, а потому что код логически требует “удерживать предыдущее значение”, когда часть условий не покрыта явным присваиванием. В UG901 AMD прямо пишет, что **inferred latches** часто появляются из-за HDL coding mistakes, например из-за **incomplete `if`** или **incomplete `case`** statements; при этом Vivado выдает warning и пишет в log type и size распознанных latches.

Главная мысль здесь такая:  
**inferred latch в synthesis report — это почти всегда structural symptom, а не просто “еще один sequential element”.**  
Да, latch — это легальная hardware structure, и Vivado умеет их распознавать. Но для FPGA design AMD отдельно отмечает, что latches **in general are not recommended**, потому что их трудно test и validate в hardware; кроме того, latch-related timing и latch loops требуют отдельного внимания в analysis flow.

### Почему это отдельная полноценная подтема

Внутри блока **Synthesis report analysis** тема **inferred latches** выделяется отдельно потому, что это очень характерный случай, когда synthesis report показывает не просто utilization change, а **semantic mismatch** между intended RTL style и реально выведенной hardware model. Если ты ожидал pure combinational block, а получил latch, то это уже не маленький cosmetic effect. Это означает, что у блока появилась memory behavior, а вместе с ним — новые timing paths, возможность latch loops и более сложная debug picture. UG901 прямо говорит, что неполное покрытие присваиваний в process/always block приводит к hardware with internal state or memory, то есть к **Flip-Flops or Latches**.

### Что такое latch в practical RTL sense

На языке RTL latch появляется тогда, когда output должен сохранять старое значение, если новое явно не назначено. Для tool это означает: комбинационная логика уже недостаточна, нужен state-holding element. UG901 формулирует это очень прямо для VHDL: process становится sequential, когда некоторые assigned signals **not explicitly assigned in all paths within the process**, и generated hardware then has internal state or memory — either **Flip-Flops or Latches**. Аналогично для SystemVerilog в UG901 прямо различаются `always_ff` для flip-flops и `always_latch` для latches.

Это важный conceptual момент. **Latch не “ошибка syntax”, а логическое следствие incomplete assignment model.** Поэтому проблема почти всегда не в одном missing `else` как таковом, а в том, что designer mentally думал про combinational function, а описал conditionally retained value. Именно поэтому inferred latch так полезен как synthesis-analysis signal: он показывает разрыв между твоей architectural idea и тем, что tool понял из RTL.

### Почему inferred latches обычно нежелательны в FPGA

Для FPGA/prototyping inferred latches чаще рассматривают как red flag. AMD в UG949 пишет, что latches **not recommended in general** и что их трудно test и validate в hardware. Это важный vendor-level signal: latch technically supported, но это не preferred mainstream style для обычного synchronous FPGA design. Дополнительно Vivado выделяет отдельную категорию **latch_loops** в Check Timing section, потому что loops, проходящие через latches, не входят в обычные combinational-loop checks и могут влиять на calculations of latch time borrowing.

Практически это означает три вещи.  
Во-первых, latch делает behavior более чувствительным к level-sensitive timing assumptions, а не только к clean edge-based clocking.  
Во-вторых, latch усложняет analysis и signoff, потому что появляются time-borrow related considerations.  
В-третьих, latch easier to infer accidentally, чем intentionally maintain clean and justified latch-based microarchitecture. Именно поэтому в обычном FPGA RTL inferred latch обычно трактуют как bug until proven otherwise.

### Как inferred latches обычно появляются

Самый типичный сценарий — неполный **`if`** или **`case`**. UG901 прямо говорит, что inferred latches are often the result of HDL coding mistakes such as incomplete `if` or `case` statements. На практике это означает примерно такие классы ситуаций:

- не у всех веток есть присваивание output;
- внутри combinational `always`/process часть signals assigned, а часть “забыты” на некоторых paths;
- default assignment отсутствует, а отдельные branches задают значение только иногда;
- `case` не покрывает все legal states и не имеет safe default behavior.

Ключевая идея тут не в том, что “нужен `default` любой ценой”, а в том, что **каждый signal в combinational block должен иметь fully defined value on every execution path**, если ты не хочешь storage inference. Иначе tool честно делает вывод: чтобы выход сохранился между evaluation windows, нужен latch.

### Где Vivado показывает inferred latches

Есть несколько точек, где Vivado помогает их увидеть.

Первая — **synthesis log**. UG901 прямо говорит, что log file reports the type and size of recognized latches и что Vivado issues a warning for inferred latch instances. То есть уже на уровне synthesis transcript latch обычно не остается незамеченным.

Вторая — **RTL lint / elaborated RTL stage**. В Tcl reference для `synth_design -rtl` AMD перечисляет lint-style messages, и среди них есть **`INFER-1 (inferred latch)`** и **`INFER-2 (incomplete case statement)`**. Это очень полезно, потому что часть latch bugs можно ловить еще до обычного synthesis run, на этапе early RTL checking.

Третья — уже открытый design и Tcl-level object queries. В UG835 команда **`all_latches`** описана как способ вернуть список всех latches, объявленных в current design. Это удобно, когда хочется не просто читать warnings, а целенаправленно собрать и просмотреть latch instances.

Четвертая — timing/check reports. В UG906 Check Timing section отдельно упоминает **`latch_loops`**, то есть loops through latches, которые влияют на timing analysis. Это уже не про сам факт inference, а про то, что latch presence начинает менять downstream analysis flow.

### Почему inferred latches важны именно на этапе synthesis report analysis

На этапе **synthesis report analysis** inferred latch особенно ценен потому, что это ранний signal. После synthesis ты уже видишь, что design structural intent сместился: combinational region превратилась в sequential one. И это можно исправить еще до того, как project начнет обрастать placement, routing и timing-noise. UG901 прямо рекомендует анализировать synthesis results как отдельную часть flow, а presence of inferred latches — как раз типичный пример того, что synthesis already told you something important about RTL quality.

Для prototyping это особенно полезно. Prototype часто терпит грубые QoR compromises, но очень плохо терпит **silent semantic surprises**. Если block неожиданно получил latch behavior, lab debug может стать намного неприятнее: waveform “почти похожа”, но выход удерживает stale value не тогда, когда ты mentally ожидал. Поэтому inferred latch — это скорее semantic readiness issue, чем просто optimization issue.

### Как читать inferred latch как symptom, а не как isolated warning

Очень полезно не останавливаться на уровне “warning exists”. Нужно понять, **почему latch вообще стал логически нужен**. Обычно тут есть несколько patterns.

Первый pattern — combinational decoder, который на некоторых conditions должен был выдавать explicit default, но не делает этого.  
Второй — output signal written in some branches of a complex `if/case`, but not in all.  
Третий — mixing of control and data decisions, где designer хотел “не менять output, если условие не сработало”, но не осознал, что это уже storage behavior.  
Четвертый — state-machine side logic, где outputs partially assigned only for selected states.

То есть inferred latch нужно читать как вопрос:  
**какое именно значение должно было быть на этом signal в paths, где assignment отсутствует?**  
Если ответ “любое фиксированное combinational default”, latch unwanted.  
Если ответ “должно сохраняться предыдущее значение”, тогда, возможно, latch действительно intended — но это уже нужно явно признать и оформить соответствующим style.

### Inferred latch и timing analysis

Хотя основная проблема latch обычно semantic, timing side тоже важна. UG906 напоминает, что timing analysis доступен уже после synthesis, а Check Timing section умеет показывать **latch_loops**. UG949 дополнительно говорит, что `set_max_time_borrow` — expert-level latch-related command, и вообще latches not recommended in general. Из этого следует practical вывод: как только в design появились latch elements, timing picture уже может выйти за рамки привычного edge-triggered reasoning.

Здесь важно не переусложнять: наличие одного inferred latch не означает автоматически catastrophic timing failure. Но это означает, что path analysis potentially перестает быть “обычной synchronous story”, а значит latch presence стоит закрывать на synthesis stage, если только latch architecture не сделана намеренно и осознанно.

### Inferred latch и latch loops

Отдельно стоит выделить **latch loops**. UG949 пишет, что good design should not have combinational loops, потому что timing engine breaks such loops and they are not reported as normal timing paths, which can lead to incorrect hardware behavior even if overall timing looks met. UG906 в Check Timing отдельно перечисляет **latch_loops** как special category. Это очень важное сочетание: latch plus feedback structure может создать analysis blind spots или по крайней мере сильно усложнить reasoning about correctness.

Практически это значит, что если inferred latch появился внутри feedback-heavy control logic, риск уже не только в “лишнем storage”, а в том, что часть behavior начнет зависеть от loop structures, которые timing engine анализирует special way. Для prototype это особенно нежелательно, потому что debug becomes both functional and timing-sensitive at once.

### Когда latch может быть intentional

Нужно отдельно сказать: **latch не всегда ошибка**. UG901 прямо поддерживает `always_latch` в SystemVerilog, то есть tool recognizes intentional latch coding style. Это полезно как граница между two situations:

- **inferred latch by accident**;
- **explicit latch by design intent**.

Если latch действительно нужен, лучше, чтобы это было видно из code style и design documentation. Тогда synthesis warning уже не surprise, а expected artifact architecture. Но даже в этом случае AMD советует осторожность: latches generally not recommended and time-borrow related handling is expert territory. Поэтому intentional latch в FPGA design — это скорее special-case decision, а не default style.

### Как practically искать и исправлять inferred latches

Хороший practical workflow обычно такой.

Сначала смотри synthesis warnings и log: есть ли latch recognition messages. Потом, если нужно, запускай early RTL lint/elaboration, где `INFER-1` может показать проблему еще до полного synthesis. Затем открывай synthesized design и прицельно смотри latch instances через **`all_latches`**. После этого возвращайся в RTL и проверяй каждый suspicious block на complete assignment coverage. Если latch presence unexpected, обычное исправление — сделать combinational intent explicit: задать defaults или покрыть все legal paths assignments. Если latch expected, тогда лучше оформить это как deliberate latch-style code, а не как accidental incomplete logic.

Этот workflow хорош тем, что разделяет две задачи:  
сначала **обнаружить latch structurally**,  
потом **решить, соответствует ли он architectural intent**.  
Именно второе обычно важнее первого. Warning сам по себе — это только повод начать analysis.

### Типичные ошибки

Самая частая ошибка — игнорировать inferred latch warning как harmless synthesis noise. UG901 прямо подчеркивает, что warning нужен, чтобы designer verified whether inferred latch functionality was intended. То есть сам tool explicitly предлагает не проходить мимо такого сообщения.

Вторая ошибка — пытаться “починить” latch уже на уровне implementation or timing constraints. Обычно это RTL semantics issue, а не placement problem. Синтез уже честно сказал, что code implies storage. Исправлять нужно cause, а не downstream symptoms. Это вывод из самой природы latch inference, описанной в UG901.

Третья ошибка — считать, что partial assignments acceptable, если simulation “примерно выглядит правильно”. На synthesis side неполные assignments превращаются в real hardware memory element. UG901 прямо связывает not explicitly assigned in all paths with generated memory.

Четвертая ошибка — использовать latch unintentionally inside larger feedback/control structures. Тогда issue уже не только в одном storage element, а в possible latch loops and harder timing/debug behavior.

### Практический итог

**Inferred latches** — это полноценная подтема внутри блока **Synthesis report analysis**, потому что она отвечает не на вопрос “есть ли warning”, а на более важный вопрос: **не превратил ли synthesis твой supposedly combinational or edge-based RTL в level-sensitive storage behavior.** AMD UG901 прямо говорит, что inferred latches часто возникают из-за incomplete `if/case`, Vivado warns about them and reports recognized latches in the log; UG835 дает `all_latches` для direct inspection; UG906 и UG949 показывают, что latch presence затрагивает и timing/check flow through latch loops and time-borrow considerations.

Если сказать совсем коротко: **inferred latch** — это один из самых полезных early warnings synthesis stage. Он часто означает не “дизайн чуть менее красивый”, а “твоя RTL semantics уже отличается от того, что ты, скорее всего, имел в виду”. И чем раньше такие места отлавливаются и осмысляются, тем спокойнее потом проходят prototyping, timing analysis и lab debug.