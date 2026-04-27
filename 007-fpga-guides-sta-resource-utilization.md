## Resource utilization

**Resource utilization** в контексте FPGA/Vivado и блока **Synthesis report analysis** — это анализ того, во что synthesis реально превратил твой RTL с точки зрения использования ресурсов FPGA: **LUT**, **FF**, **BRAM**, **URAM**, **DSP**, **I/O**, а также связанных структур вроде **control sets** и high-fanout control logic. В Vivado это обычно смотрят через **Report Utilization**, который умеет показывать расход ресурсов по design в целом, по **hierarchy**, по **Pblocks** и по **SLR**. После synthesis Vivado дает возможность открыть synthesized design и анализировать отчеты как стандартную часть flow.

Главная мысль здесь такая:  
**resource utilization — это не просто ответ на вопрос “влезает ли дизайн в кристалл”, а ранний structural portrait проекта.** По utilization видно не только объем логики, но и характер архитектуры: где hierarchy раздута, где неудачно расходуются memory blocks, где слишком много control diversity, где возможны проблемы с packing, congestion и последующим QoR. AMD прямо рекомендует использовать post-synthesis analysis как ранний этап понимания качества design, а utilization report — как один из главных инструментов такого анализа.

### Почему это отдельная полноценная подтема

Внутри блока **Synthesis report analysis** тема **resource utilization** выделяется отдельно потому, что она отвечает не на вопрос “есть ли already bad timing”, а на более базовый вопрос: **какую физическую форму получил RTL после synthesis**. Timing проблемы часто появляются позже, но их корни нередко уже видны в utilization picture: слишком много dispersed logic, неравномерная hierarchy, heavy RAM/DSP concentration, избыточные **control sets**, high-fanout control structure. В UG906 utilization report прямо описывается как средство анализа design utilization по hierarchy, Pblocks и SLR, а complexity-related metrics используются для оценки routing complexity и будущих placement challenges.

### Что на самом деле означает “хороший” utilization

Самая частая ошибка — думать, что хороший utilization означает просто “процент LUT не слишком высокий”. На практике хороший utilization — это более широкое понятие. Важны не только абсолютные числа, но и:

- **distribution** ресурсов по hierarchy;
- соотношение LUT/FF/BRAM/DSP;
- концентрация ресурсов в отдельных частях design;
- число и характер **control sets**;
- признаки будущей routing complexity;
- наличие unusually large blocks, которые уже после synthesis выглядят подозрительно.

Vivado utilization report специально позволяет смотреть не только top-level summary, но и breakdown per hierarchy, что как раз нужно для такого анализа. UG906 также подчеркивает пользу рассмотрения design по Pblocks и SLR, когда важна физическая локализация и competition for resources.

### С чего полезно начинать анализ

Хороший practical порядок обычно такой. Сначала смотри на **общую сводку**: хватает ли device resources в принципе и нет ли очевидного перекоса, например unusually high LUT usage при скромном числе FF, или чрезмерного потребления BRAM/DSP. Потом переходи к **hierarchy view** и ищи, какие blocks являются крупнейшими consumers. UG906 прямо показывает, что utilization report позволяет раскрывать sub-hierarchy и смотреть, какие блоки потребляют основные RAM/FIFO resources. Это особенно полезно в prototyping, где важно быстро локализовать “тяжелые” места, а не просто видеть общую цифру сверху.

На этом этапе полезно задавать себе очень простые вопросы.  
Почему именно этот block стал biggest LUT consumer?  
Почему memory-heavy part не использует BRAM так, как ожидалось?  
Почему seemingly simple control block съел disproportionate число registers?  
Такие вопросы уже не про tool usage, а про архитектурное чтение synthesis result. И именно в этом главная ценность темы: utilization report становится bridge between RTL intent and actual mapped structure.

### Top-level utilization: что видно сразу

На верхнем уровне utilization report дает быстрый ответ на вопрос, насколько design вообще realistic для выбранного FPGA. Но даже если суммарные проценты выглядят “нормально”, этого недостаточно. Можно иметь приемлемый общий процент LUT и FF, но при этом слишком плотную или плохо распределенную локально логику, что позднее приведет к congestion или routing-driven timing issues. UG906 прямо связывает design analysis с вопросами placement and routing complexity, а complexity report оценивает такие вещи, как average fanout и distribution of leaf cell types для понимания routing pressure.

Поэтому top-level utilization полезно воспринимать как **первая грубая граница**, а не как окончательный verdict. Если design использует, например, умеренное число LUT, но уже сейчас видно множество memory macros, heavy fanout control nets и очень крупные отдельные hierarchies, то реальная сложность implementation может оказаться выше, чем подсказывает один summary percentage.

### Hierarchy-based analysis: где живет настоящая информация

Самый полезный режим для **resource utilization** — это почти всегда **hierarchy view**. Именно там видно, какие modules реально формируют лицо проекта. UG906 отдельно говорит, что utilization report breaks down usage per hierarchy, а в example usage показывает, как по sub-hierarchy можно найти major RAM/FIFO consumers. Для prototyping это особенно ценно, потому что большая часть ранних architectural решений принимается не на уровне “весь design”, а на уровне конкретных subsystems.

В хорошем анализе hierarchy ты обычно ищешь не только biggest blocks, но и **unexpected blocks**. Иногда крупный datapath block и должен быть тяжелым — это нормально. Но если внезапно control wrapper или protocol adapter начинает потреблять аномально много LUT или registers, это уже повод смотреть на coding style, multiplexing structure, packet bookkeeping, FSM expansion или sideband handling. Такие выводы не даны в одной строке документации, но они напрямую следуют из того, что Vivado предоставляет hierarchy-level resource decomposition именно для инженерной интерпретации structure.

### LUT и FF: не просто количество, а соотношение

В utilization analysis полезно смотреть на **соотношение LUT и FF**, а не только на их absolute counts. Высокий LUT usage при сравнительно скромном FF usage может намекать на heavy combinational structure, сложные mux trees, wide decode logic или недостаточную pipeline regularity. Наоборот, очень высокий FF usage при умеренной combinational logic может говорить о deeply registered architecture, большом количестве status/state storage или о том, что часть behavior выражена через storage-heavy style. UG906 напрямую не формулирует это как универсальную интерпретацию, но в категории timing analysis отдельно советует смотреть на logic depth и cell types, если logic delay dominates path delay. Это делает LUT/FF pattern useful early hint еще на synthesis stage.

Для prototyping это особенно важно, потому что на этой стадии часто нужно решить: продолжать ли текущую microarchitecture, или уже пора менять структуру pipeline, упрощать control path, делить крупные blocks или улучшать inference memory/DSP resources. Utilization сам по себе не отвечает на эти вопросы, но очень рано показывает, где их нужно задавать.

### BRAM, FIFO, URAM: memory footprint как признак архитектуры

Когда в design есть buffering, packet storage, frame storage или queue-based decoupling, memory resource usage становится одним из важнейших сигналов synthesis report. **BRAM/FIFO/URAM utilization** говорит не только о том, сколько памяти ты занял, но и о том, какая strategy buffering реально сложилась после inference. Если block, который по замыслу должен был лечь в block RAM, внезапно расползся по LUTRAM или registers, это уже architectural issue, а не просто cosmetic detail. UG906 utilization report как раз предназначен для анализа таких resource types по hierarchy.

В прототипах это особенно заметно на streaming systems: каждая “небольшая” FIFO, каждая elastic queue и каждый packet buffer по отдельности могут выглядеть безобидно, но в сумме начать определять resource shape всего проекта. Поэтому memory utilization почти всегда стоит читать вместе с hierarchy view: какие именно blocks потребляют RAM resources, и совпадает ли это с архитектурными ожиданиями.

### DSP utilization: когда arithmetic shape уже видна после synthesis

Если design использует multiplies, MAC structures, filters или fixed-point processing, **DSP utilization** — еще один ключевой индикатор. После synthesis уже можно увидеть, насколько clean tool смог отобразить arithmetic в dedicated resources, а не в distributed LUT logic. Для prototyping это важно потому, что early mismatch между intended arithmetic architecture и actual inferred resource mix быстро приводит к неверным ожиданиям по timing, power и scaling. UG906 utilization report включает breakdown by resource type, и этого уже достаточно, чтобы на раннем этапе понять, ложится ли math-part проекта в intended hardware pattern.

Здесь полезно мыслить не только категорией “много DSP или мало DSP”, а вопросом:  
**используются ли DSP там, где они действительно должны использоваться?**  
Если нет, проблема часто уже не в place/route, а в RTL style, widths, signedness, operator grouping или synthesis assumptions. Это inference, но полностью соответствует смыслу post-synthesis structural reading, который AMD рекомендует делать после synthesis.

### Control sets: маленькая цифра с большими последствиями

Отдельно очень важно смотреть на **control sets**. В Vivado control set — это уникальная комбинация **clock**, **clock enable** и **set/reset** у sequential elements. UG906 отдельно выделяет **Report Control Sets** и отмечает, что если design exceeds recommended control set limit, стоит оптимизировать control sets с низким BEL-load count и использовать histogram для оценки распределения. AMD также предупреждает, что replicated nets, особенно после synthesis, могут увеличивать routing resource usage и накладываться друг на друга.

Почему это настолько важно для resource utilization? Потому что избыток control sets ухудшает packing регистров в slices и косвенно ухудшает общий QoR. То есть design может “влезать” по LUT/FF totals, но вести себя как structurally fragmented и трудноупаковываемый. Для prototyping это часто один из самых недооцененных факторов: engineer смотрит на общую utilization, но не замечает, что project уже расколот на слишком много уникальных clock/reset/enable patterns.

### Control-set threshold и влияние synthesis settings

UG901 отдельно описывает synthesis option **CONTROL_SET_THRESHOLD**: higher values mean less logic on control signals and more on D-input of flop; lower values mean more control signals and less logic on D-input. Это очень полезный bridge между synthesis settings и utilization shape. Иначе говоря, число и структура control sets — не только следствие RTL, но и отчасти следствие synthesis strategy.

Практический вывод отсюда такой: если utilization analysis показывает неприятную картину по control sets, не всегда нужно сразу переписывать полпроекта. Иногда надо отдельно проверить, что именно навязано RTL architecture, а что усиливается конкретной synthesis setting. Но и обратная крайность опасна: нельзя рассчитывать, что strategy magically исправит fundamentally fragmented control architecture.

### High-fanout nets как скрытая часть utilization story

Формально **high-fanout nets** — это уже отдельный report, но practically они очень тесно связаны с темой resource utilization. UG906 для `report_high_fanout_nets` показывает, что можно анализировать nets с высоким fanout, driver type, load types, clock regions и SLRs. Если control net или enable-like signal имеет huge fanout, это влияет не только на timing, но и на effective physical resource pressure: replication, routing spread, packing constraints и congestion risks.

Поэтому зрелый utilization analysis почти всегда смотрит на two layers together:  
**сколько ресурсов занято** и **какими связями эти ресурсы управляются**.  
Дизайн с умеренным LUT count, но тяжелыми high-fanout control nets может быть намного сложнее в реальной implementation, чем более крупный, но хорошо локализованный design. Это inference, но она напрямую опирается на то, что AMD выделяет high-fanout analysis и complexity analysis как отдельные design-analysis tools.

### Pblocks и SLR: когда важна локальная плотность, а не только общий процент

UG906 прямо указывает, что utilization report можно использовать на уровне **user-defined Pblocks** и **SLRs**. Это очень важный момент для больших devices и более серьезных prototypes. Общая utilization может выглядеть комфортно, но если одна физическая область перегружена, implementation все равно станет тяжелой. Отдельный режим `-pblocks` показывает, какие ресурсы доступны внутри parent Pblock, как они делятся между child Pblocks и non-assigned logic, и это помогает оценивать локальную competition for resources.

Для prototyping это особенно полезно, когда уже есть rough floorplanning assumptions или когда design naturally делится на крупные subsystems. Тогда resource utilization нужно читать не только в терминах “влезает ли в chip”, но и “влезает ли cleanly туда, где ты собираешься это physically собрать”.

### Complexity metrics как следующий шаг после utilization

Иногда сам utilization report показывает только symptom, а не глубину проблемы. В таких случаях полезно переходить к **complexity report**. UG906 говорит, что complexity report дает insight в **Rent Exponent**, average fanout и distribution of leaf cell types, и эти метрики помогают оценить routing complexity и placement challenges. Это особенно ценно, когда utilization percentages еще не пугают, но design уже выглядит structurally difficult.

Практически это означает, что **resource utilization** — не финальная точка анализа, а отправная. Если какая-то hierarchy подозрительно большая или физически нагруженная, следующим логичным вопросом становится не “сколько там LUT”, а “насколько routing-complex эта логика”. И именно здесь utilization analysis начинает работать как часть более широкого synthesis-report workflow.

### Что resource utilization говорит о готовности prototype

В контексте **Prototyping** тема особенно полезна, потому что prototype не обязан быть идеально оптимизированным, но он должен быть **жизнеспособным и масштабируемым**. Если уже после synthesis видно, что design чрезмерно тяжел по одному виду ресурсов, слишком fragmented по control sets, плохо распределен по hierarchy или тяготеет к локальной перегрузке в Pblocks/SLRs, это сильный сигнал, что prototype architecture может быть хрупкой. AMD прямо позиционирует post-synthesis analysis как часть раннего engineering feedback loop.

Полезно думать так:  
**resource utilization — это не отчет о прошлом, а прогноз на будущее implementation.**  
Он еще не говорит всего о final timing, но уже показывает, какие части design likely станут дорогими по packing, routing и closure effort. В prototyping именно такой ранний прогноз особенно ценен, потому что позволяет менять архитектуру до того, как project глубоко обрастет остальной логикой и constraints.

### Типичные ошибки

Самая частая ошибка — смотреть только на общий процент LUT/FF и считать анализ завершенным. UG906 прямо дает инструменты для hierarchy/Pblock/SLR analysis именно потому, что общий summary часто скрывает реальные локальные hotspots.

Вторая ошибка — игнорировать **control sets**. Формально ресурсы еще могут помещаться, но packing and routing quality уже деградируют. AMD отдельно выделяет report_control_sets и дает рекомендации, что делать при превышении рекомендуемых limits.

Третья ошибка — не связывать utilization с high-fanout и complexity. Ресурсный отчет сам по себе не показывает всю будущую routing difficulty, поэтому его полезно читать вместе с **report_high_fanout_nets** и complexity-oriented metrics.

Четвертая ошибка — трактовать heavy resource block как обязательно “плохой”. Большой block может быть абсолютно нормальным, если он архитектурно expected. Важнее искать unexpected consumers и structural mismatches между intent и actual mapping. Это уже инженерный вывод из самой идеи hierarchy-based utilization analysis.

### Практический итог

**Resource utilization** — это полноценная подтема внутри блока **Synthesis report analysis**. Она отвечает не только на вопрос “помещается ли design”, а на более глубокий вопрос: **какую физическую и структурную форму получил RTL после synthesis, и какие будущие implementation-problems уже видны по этой форме**. По официальным материалам AMD, utilization report полезен для анализа по hierarchy, Pblocks и SLR; control sets влияют на packing quality; high-fanout and complexity metrics помогают увидеть routing-related risk еще до implementation.

Если сказать совсем коротко: **resource utilization** — это искусство видеть за числами архитектуру. Не просто “у меня занято 58% LUT”, а “какие именно blocks, control patterns и physical regions делают проект таким, и что это означает для prototype readiness”. Именно с этого обычно и начинается зрелое чтение synthesis reports в Vivado.