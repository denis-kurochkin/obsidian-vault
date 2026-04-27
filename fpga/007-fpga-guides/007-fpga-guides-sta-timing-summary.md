## Timing summary

**Timing summary** в контексте FPGA/Vivado и блока **Synthesis report analysis** — это основной high-level report, который показывает, как текущий design соотносится с заданными timing constraints: где timing met, где есть violations, по каким типам checks они возникают и какие numbers нужно смотреть в первую очередь. В UG906 AMD прямо пишет, что **Report Timing Summary** дает comprehensive overview всех timing checks и содержит достаточно деталей, чтобы начать анализ и debug timing issues; timing analysis доступен на любом этапе flow после synthesis.

Главная мысль здесь такая:  
**timing summary — это не “еще один отчет после synthesis”, а главный early indicator того, насколько RTL и constraints уже складываются в жизнеспособную timing model.** На этапе synthesis он особенно полезен, потому что позволяет увидеть первые признаки будущих проблем до полноценного implementation. При этом AMD отдельно уточняет, что в synthesized design timing engine оценивает net delays по connectivity и fanout, а точность ниже, чем после placement/routing; особенно это важно для путей, где есть pre-placed cells, I/O или GT. Значит, post-synthesis timing summary — это ранний прогноз, а не final truth.

### Почему это отдельная полноценная подтема

Внутри блока **Synthesis report analysis** тема **timing summary** выделяется отдельно потому, что это первая сводка, где одновременно сходятся три вещи:  
**constraints**, **реальная структура после synthesis** и **числовая оценка риска**. UG949 прямо говорит, что Timing Summary report provides high-level information on the timing characteristics of the design compared to the constraints provided, а UG906 рекомендует использовать именно этот report для timing signoff и как starting point для анализа violations. Даже в Design Runs window основные timing scores run’а показываются как **WNS, TNS, WHS, THS, WBSS, TPWS**, и если timing not met, AMD советует начинать именно с Timing Summary Report.

То есть **timing summary** — это не просто краткий вывод “прошло/не прошло”. Это точка, где engineer должен понять: проблема в missing constraints, в самой architecture, в synthesis result, в physical effects или в конкретном классе timing checks. Именно поэтому эту тему полезно изучать отдельно, а не растворять в общем разговоре про timing closure.

### Что обычно входит в timing summary

В practical Vivado flow timing summary собирает high-level информацию по основным timing checks и сводным scores. На уровне runs и log AMD явно выделяет как минимум:

- **WNS** — Worst Negative Slack
- **TNS** — Total Negative Slack
- **WHS** — Worst Hold Slack
- **THS** — Total Hold Slack
- **WBSS** — Worst Bus Skew Slack
- **TPWS** — Total Pulse Width Slack

Это видно и в Design Runs window, и в implementation log, где router phases показывают estimated post-routing timing summary именно через WNS/TNS/WHS/THS, а UG906 отдельно напоминает, что Timing Summary Report нужно смотреть еще и для **Pulse Width timing summary** и информации о violations или missing constraints.

Практически это означает, что timing summary — это **сводка по нескольким классам проблем сразу**, а не только по setup. Очень частая ошибка — смотреть только на WNS и игнорировать hold, pulse width или bus skew. В маленьком design это иногда проходит безболезненно, но в серьезном prototype такой подход быстро начинает скрывать реальные issues.

### С чего полезно начинать чтение timing summary

Хороший порядок чтения обычно такой:

1. Сначала смотри, есть ли вообще violations.
2. Потом смотри, **какого они типа** — setup, hold, pulse width, bus skew.
3. Затем смотри на сводные numbers — WNS/TNS/WHS/THS.
4. После этого переходи к detailed path analysis через **report_timing**.

UG906 прямо говорит, что **Report Timing** нужен для investigation timing issues flagged in Report Timing Summary и для проверки coverage/validity specific timing constraints. А UG906 Category 1: Timing подчеркивает, что timing characteristics path включают **Path Type, Requirement, Slack, Timing Exception**, и при debugging missing or incorrect constraints первым делом надо смотреть на path requirement.

Иными словами, timing summary — это **front page**, а не весь анализ целиком. Он отвечает на вопрос “где горит”, но не всегда сразу отвечает “почему”. Именно поэтому зрелый flow почти всегда делает так: сначала timing summary, затем targeted `report_timing` по failing paths.

### Как понимать WNS

**WNS** — это самый плохой, то есть наиболее отрицательный slack среди проверяемых путей данного типа. В практическом смысле это ответ на вопрос: **насколько худший path не укладывается в requirement**. Если WNS отрицательный, timing по крайней мере на одном path уже не met. Если WNS положительный, это еще не значит, что весь design идеален, но это хороший первый сигнал по соответствующему классу checks. AMD использует WNS как один из основных timing scores и в Design Runs window, и в QoR assessment details.

На synthesis stage WNS особенно полезен как **быстрый индикатор architectural danger**. Но важно помнить оговорку UG906: post-synthesis delay estimates основаны на connectivity и fanout, а не на реальном routed path, поэтому synthesis WNS — это не signoff number, а ранняя оценка. Если WNS уже сильно плохой после synthesis, это серьезный warning. Если он слегка плохой или около нуля, интерпретировать это нужно осторожнее, в контексте architecture и будущего implementation.

### Как понимать TNS

**TNS** — это суммарный negative slack по всем violating paths данного класса. В practical engineering sense это отвечает на вопрос: **насколько проблема локальна или системна**. Можно иметь один очень плохой path и скромный TNS, а можно иметь умеренно плохой WNS, но большой TNS, если failing paths много. AMD использует TNS вместе с WNS как одну из базовых timing metrics и в QoR assessment, и в run summary.

Это различие очень полезно.  
Если **WNS сильно плохой, а TNS маленький**, часто сначала ищут один-две локальные architectural hotspots.  
Если **WNS умеренно плохой, но TNS большой**, это уже часто говорит о более широкой structural problem: возможно, плохой clocking, тяжелая hierarchy, нехватка pipelining, плохой floorplan direction или некорректный constraints model. Это не буквальная формула из одного абзаца manual, а инженерная интерпретация того, зачем Vivado вообще показывает WNS и TNS вместе.

### Как понимать WHS и THS

**WHS** и **THS** — это аналогичные summary numbers для **hold checks**. Очень важно, что hold timing — это отдельный класс проблем, и positive setup picture не гарантирует отсутствие hold issues. В Design Runs window AMD показывает WHS и THS наравне с WNS/TNS; implementation log тоже использует эти метрики в estimated post-routing timing summary.

Практически это значит:  
если engineer смотрит только на WNS/TNS, он видит только половину картины.  
Hold violations often требуют другой тип reasoning, чем setup violations. Setup чаще связан с длинными datapaths, heavy logic, routing delay и Fmax pressure. Hold может всплывать из-за clock relationships, skew effects, overly short paths, clocking topology и некоторых optimization side effects. Timing summary полезен именно тем, что он показывает оба мира на одной стартовой панели.

### Pulse width и bus skew: почему их нельзя игнорировать

UG906 прямо напоминает, что Timing Summary Report нужен, чтобы увидеть **Pulse Width timing summary**, а Design Runs window добавляет **WBSS**, то есть Worst Bus Skew Slack. Это важно потому, что часть engineers привыкает читать timing summary только как setup/hold report. На практике это уже шире. **Pulse width** checks связаны с корректностью clock pulse characteristics, а **bus skew** — с согласованностью arrival times для групп сигналов, где это критично.

Для prototyping это особенно важно, когда проект содержит source-synchronous interfaces, wide buses, сложные IO constraints или несколько тесно связанных control/data groups. Если эти классы checks присутствуют в design, timing summary нужно читать полностью, а не только по первым двум строкам.

### Timing summary после synthesis: что он реально умеет сказать

На этапе synthesis timing summary особенно полезен как **early diagnosis tool**. UG901 отдельно говорит, что после synthesis можно view reports, open and analyze synthesized design, а UG906 указывает, что timing analysis available at any point after synthesis. При этом synthesized-design timing based on estimated net delays means, что report очень хорош для **раннего структурного анализа**, но хуже подходит как окончательный verdict.

Хорошая practical интерпретация post-synthesis timing summary такая:

- если violations already severe, architecture likely needs attention;
- если summary выглядит clean, это хороший sign, но не proof окончательного успеха;
- если report странный или противоречивый, надо проверять constraints coverage и validity.

UG903 отдельно напоминает, что после synthesis стоит загружать synthesized design с теми же synthesis XDC in memory и запускать timing summary, потому что некоторые pre-synthesis constraints после transformations synthesis могут больше не применяться так, как ожидалось.

### Timing summary и constraints quality

Очень важный момент: timing summary отражает не только качество RTL, но и качество **constraints model**. UG906 Category 1: Timing прямо советует при debugging missing or incorrect timing constraints сначала смотреть на **path requirement**. А UG903 предупреждает, что после synthesis некоторые hierarchical objects и names могут измениться, из-за чего pre-synthesis constraints могут apply некорректно.

Это означает, что плохой timing summary не всегда равен “плохой логике”, а хороший timing summary не всегда равен “все constraints идеальны”. Иногда проблема в том, что часть paths timed неверно, часть clocks не созданы как expected, или constraints coverage неполная. Именно поэтому summary report нужно читать не в отрыве от constraints, а как **отчет о взаимодействии design и constraints**.

### Когда после timing summary надо идти в report_timing

UG906 ясно разделяет роли: **Report Timing Summary** — для overview и signoff/start analysis, а **Report Timing** — для investigation specific timing paths. Если summary показал violation, следующий шаг — не просто перечитывать numbers, а открыть конкретные worst paths и посмотреть:

- path type;
- requirement;
- slack;
- logic vs net delay;
- timing exception, если есть;
- startpoint/endpoint pattern.

UG906 отдельно пишет, что Report Timing helps investigate issues flagged in Report Timing Summary or verify coverage and validity of specific timing constraints.

Практически timing summary отвечает на вопрос **“сколько и где болит”**, а report_timing — **“почему именно болит”**. Эти два отчета нужно воспринимать как связку, а не как конкурирующие инструменты.

### Что timing summary говорит о готовности prototype

В контексте **Prototyping** timing summary особенно полезен, потому что prototype часто еще далек от финальной optimization, но уже должен быть **engineering-realistic**. Если timing summary после synthesis показывает грубые violations, это сильный сигнал, что project может быстро упереться в deeper structural issues. Если summary умеренно чистый, prototype likely имеет шанс развиваться дальше без немедленной architectural переделки. UG906 даже выделяет **QoR Assessment**, где Vivado reviews worst-case paths per clock group and checks WNS/TNS/WHS/THS to identify potential failures earlier in the flow.

То есть timing summary в prototype flow — это один из самых ранних **go/no-go indicators**. Не в том смысле, что он окончательно решает судьбу проекта, а в том смысле, что он очень быстро показывает, насколько reasonable текущая комбинация RTL + constraints + target device.

### Типичные ошибки

Самая частая ошибка — смотреть только на **WNS** и игнорировать остальную summary picture. Но Vivado явно показывает и **TNS**, и **WHS/THS**, и дополнительные timing scores вроде **WBSS** и **TPWS**, если они релевантны.

Вторая ошибка — трактовать post-synthesis timing summary как final signoff truth. UG906 отдельно оговаривает, что synthesized-design delay estimates основаны на connectivity/fanout и менее точны, чем implemented results.

Третья ошибка — не проверять constraints quality. UG903 прямо предупреждает, что synthesis transformations могут изменить applicability некоторых constraints, и timing summary после synthesis нужно смотреть с тем же XDC context.

Четвертая ошибка — пытаться лечить проблему прямо по summary numbers, не открывая specific failing paths через `report_timing`. UG906 clearly separates these reports for overview versus investigation.

### Практический итог

**Timing summary** — это полноценная подтема внутри блока **Synthesis report analysis**. Она отвечает не просто на вопрос “есть ли timing violations”, а на более важный вопрос: **как текущий design в целом выглядит относительно timing constraints и какие классы проблем уже видны после synthesis или implementation**. Official Vivado guidance позиционирует Timing Summary report как основной overview/signoff report, доступный после synthesis; он показывает WNS/TNS/WHS/THS и другие timing scores, а затем направляет engineer к более детальному `report_timing` analysis.

Если сказать совсем коротко: **timing summary** — это первая страница timing-истории проекта. По ней еще нельзя понять все детали, но уже можно понять, healthy ли timing model, где начинаются реальные риски и куда копать дальше — в constraints, в architecture, в specific paths или в later implementation effects.