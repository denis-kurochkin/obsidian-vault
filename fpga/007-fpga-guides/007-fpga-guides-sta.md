**Static Timing Analysis (STA)** в контексте FPGA/Vivado и **Prototyping** — это блок про анализ временных путей в design **без симуляции по векторам**. Vivado рассматривает все релевантные timing paths, использует clock definitions и constraints, вычисляет requirement, delay и **slack**, а затем показывает, какие paths укладываются в setup/hold и какие нет. В UG906 timing analysis описан как часть design analysis flow, доступная уже после synthesis, а timing path разбирается через source clock path, data path и destination clock path.

Главная идея блока такая:  
**STA — это основной способ понять, сможет ли FPGA design работать на нужной частоте и с нужными интерфейсами в реальном железе.** Для prototyping это особенно важно, потому что prototype часто быстро растет, clocks и interfaces множатся, и без STA очень легко получить design, который “логически работает”, но уже не имеет надежного временного запаса. UG903 отдельно подчеркивает, что timing constraints формируют timing model проекта, а UG949 рассматривает timing reports как базовый инструмент для оценки design against those constraints.

В practical sense STA в Vivado строится вокруг нескольких опорных понятий:

- **clocks** и их relationships,
- **timing paths** между startpoint и endpoint,
- **max/min delay analysis**,
- **setup/recovery** и **hold/removal** checks,
- **clock skew**, **uncertainty** и связанные effects,
- итоговый **slack** как главный индикатор запаса или нарушения timing.  
    UG906 прямо выделяет эти concepts в разделе Timing Analysis Key Concepts, включая max/min delay analysis, setup/hold, skew, uncertainty и pulse width checks.

Для prototyping ценность этого блока в том, что STA дает очень ранний engineering feedback. Уже после synthesis можно увидеть, где начинаются риски, хотя AMD отдельно оговаривает: post-synthesis timing менее точен, чем implemented timing, потому что net delays еще оценочные и основаны на connectivity/fanout, а не на final placement and routing. То есть STA полезен и рано, и поздно, но interpretation на разных стадиях разная: после synthesis это больше ранний structural forecast, после implementation — уже основа signoff.

Еще одна важная мысль:  
**STA — это не только про “частоту clock”.** В Vivado timing analysis включает не только intra-clock register-to-register paths, но и inter-clock paths, I/O timing, pulse width, bus skew, unconstrained paths и влияние timing exceptions. Это видно даже по структуре **Report Timing Summary**, где есть sections для setup, hold, pulse width, clock summary, check timing, intra-clock, inter-clock, user-ignored и unconstrained paths.

Хороший инженерный взгляд на STA такой:  
RTL описывает, **что** делает логика.  
Constraints описывают, **в каких временных условиях** она должна работать.  
STA проверяет, **совместимы ли эти две модели друг с другом**.  
Именно поэтому этот блок естественно стоит рядом с **Constraints writing**: без корректных clocks, I/O delays, clock groups и exceptions даже хороший STA report будет отражать неверную timing reality. UG903 прямо связывает timing analysis с корректной timing-constraint model.

**Итог:**  
**Static Timing Analysis (STA)** — это обзорный блок про то, как Vivado вычисляет и показывает временную реализуемость design без testbench-driven simulation. Для prototyping это один из ключевых блоков, потому что он рано показывает, где проект уже упирается в clock frequency, inter-clock interaction, I/O timing или constraint quality, и тем самым помогает управлять архитектурой до того, как проблемы станут дорогими в implementation и lab debug.