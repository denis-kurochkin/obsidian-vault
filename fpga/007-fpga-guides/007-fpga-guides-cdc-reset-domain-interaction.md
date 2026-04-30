# Reset-domain interaction внутри CDC

**Reset-domain interaction** — это тема о том, как сигналы сброса взаимодействуют с несколькими clock domains.

В FPGA-проектах reset часто воспринимают как простой служебный сигнал:

```
reset = 1 -> всё сброситьreset = 0 -> всё работает
```

Но в multi-clock design reset сам становится междоменным сигналом. Он может пересекать clock domains, отпускаться в неподходящий момент, приходить в разные части схемы с разной задержкой и ломать даже правильно сделанные CDC-структуры.

Главная идея:

> Reset должен быть спроектирован так же осознанно, как clocking и CDC.

---

# 1. Почему reset связан с CDC

Обычный CDC говорит:

```
сигнал из clk_a domain нельзя напрямую использовать в clk_b domain
```

Reset часто нарушает это правило, потому что один reset signal подключают сразу во многие domains:

```
external_reset_n
        |
        +--> sys_clk domain
        +--> aurora_clk domain
        +--> pcie_clk domain
        +--> axi_clk domain
```

Если reset асинхронный, то его снятие может произойти в любой момент относительно каждого clock.

Именно снятие reset, то есть **reset deassertion**, обычно опаснее, чем его активация.

---

# 2. Reset Domain Crossing / RDC

Для reset часто используют отдельный термин:

```
RDC = Reset Domain Crossing
```

CDC — это crossing обычных data/control signals между clock domains.

RDC — это проблемы, связанные с reset-сигналами:

```
асинхронное снятие reset
разные reset domains внутри одного clock domain
reset одного блока влияет на другой блок без согласования
одна часть логики вышла из reset раньше другой
reset пересекает clock domain как обычный control signal
```

RDC может проявляться очень редко, например только при включении питания, при перезапуске IP, при потере PLL lock или при частичном reset одного subsystem.

---

# 3. Assert и deassert

Для reset важно разделять два действия:

```
assert reset    — активировать сброс
deassert reset  — отпустить сброс
```

Например для active-low reset:

```
rst_n = 0 -> reset asserted
rst_n = 1 -> reset deasserted
```

Обычно опасен именно **deassertion**.

Почему?

При assert reset логика принудительно переводится в известное состояние.

При deassert reset логика начинает работать. Если reset отпущен близко к фронту clock, регистры могут выйти из reset неконсистентно.

---

# 4. Asynchronous assert, synchronous deassert

Очень распространенное правило:

```
asynchronous assert
synchronous deassert
```

Смысл:

```
reset можно активировать немедленно, независимо от clock;
но отпускать reset нужно синхронно с clock каждого domain.
```

То есть глобальный reset может прийти асинхронно, но внутри каждого clock domain должен быть свой локальный reset synchronizer.

Структура:

```
global_async_reset_n
        |
        +--> reset_sync for sys_clk
        +--> reset_sync for aurora_clk
        +--> reset_sync for pcie_clk
```

---

# 5. Reset synchronizer

Типовой reset synchronizer для active-low external reset:

```verilog
module reset_sync #(
    parameter integer STAGES = 2
)(
    input  wire clk,
    input  wire arst_n,
    output wire srst_n
);

    (* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)
    reg [STAGES-1:0] sync_ff = {STAGES{1'b0}};

    always @(posedge clk or negedge arst_n) begin
        if (!arst_n)
            sync_ff <= {STAGES{1'b0}};
        else
            sync_ff <= {sync_ff[STAGES-2:0], 1'b1};
    end

    assign srst_n = sync_ff[STAGES-1];

endmodule
```

Поведение:

```
arst_n упал в 0:
    srst_n быстро становится 0 асинхронно

arst_n поднялся в 1:
    srst_n станет 1 только через STAGES тактов clk
```

Так reset отпускается синхронно с `clk`.

---

# 6. Почему каждому clock domain нужен свой reset synchronizer

Плохо:

```
external_rst_n -> reset_sync on sys_clk -> rst_n_sys
rst_n_sys используется также в aurora_clk domain
```

Это ошибка.

`rst_n_sys` уже синхронен к `sys_clk`, но не синхронен к `aurora_clk`.

Правильно:

```
external_rst_n -> reset_sync(sys_clk)    -> rst_n_sys
external_rst_n -> reset_sync(aurora_clk) -> rst_n_aurora
external_rst_n -> reset_sync(pcie_clk)   -> rst_n_pcie
```

Правило:

> Reset должен отпускаться синхронно в том clock domain, где он используется.

---

# 7. Один reset — много локальных reset

В multi-clock FPGA-проекте глобальный reset обычно не используют напрямую.

Лучше делать так:

```
raw reset source
    |
    v
reset controller / conditioning
    |
    +--> local reset for clk_a
    +--> local reset for clk_b
    +--> local reset for clk_c
```

Например:

```
board_reset_n
pll_locked
configuration_done
power_good
```

объединяются в reset controller, а затем синхронизируются отдельно в каждый domain.

---

# 8. Reset source тоже может быть асинхронным

Источники reset:

```
кнопка reset
power-on reset
FPGA startup reset
PLL/MMCM locked
external supervisor
software reset bit
watchdog reset
IP error reset
link down reset
```

Некоторые из них асинхронны к вашей логике.

Например, `pll_locked` может измениться не синхронно к user clock.

Значит, даже если reset формируется внутри FPGA, он может быть CDC/RDC-сигналом.

---

# 9. PLL/MMCM locked как часть reset

Частый pattern:

```verilog
assign global_arst_n = board_rst_n & pll_locked;
```

Затем:

```verilog
reset_sync u_rst_sys (
    .clk    (sys_clk),
    .arst_n (global_arst_n),
    .srst_n (sys_rst_n)
);
```

Это нормально как идея, но важно:

```
pll_locked может дрожать при старте;
clock может быть нестабилен до locked;
reset synchronizer работает корректно только когда clock уже тикает;
после locked часто нужен дополнительный delay перед release.
```

Иногда делают reset controller, который ждет несколько тактов после `locked`.

---

# 10. Задержка после PLL lock

Пример:

```
PLL locked стал 1
подождать 16/32/64 такта sys_clk
только потом отпустить reset
```

Зачем:

```
дать clocking resources стабилизироваться;
дать IP-блокам корректно стартовать;
избежать мгновенного release на первом же фронте после locked;
сделать поведение при старте более устойчивым.
```

Пример логики:

```verilog
reg [5:0] rst_cnt = 6'd0;
reg       rst_ready = 1'b0;

always @(posedge clk or negedge arst_n) begin
    if (!arst_n) begin
        rst_cnt   <= 6'd0;
        rst_ready <= 1'b0;
    end else if (!rst_ready) begin
        rst_cnt <= rst_cnt + 1'b1;
        if (&rst_cnt)
            rst_ready <= 1'b1;
    end
end

assign local_rst_n = rst_ready;
```

На практике этот delay часто объединяют с reset synchronizer.

---

# 11. Synchronous reset

Synchronous reset используется только внутри clocked block:

```verilog
always @(posedge clk) begin
    if (rst)
        q <= 1'b0;
    else
        q <= d;
end
```

Плюсы:

```
reset release автоматически синхронен к clk;
STA видит reset path как обычную data/control path;
нет recovery/removal проблем асинхронного reset.
```

Минусы:

```
reset сработает только если clock работает;
для некоторых IP/primitive нужен именно async reset;
может увеличивать logic на data path;
не всегда подходит для глобального аварийного reset.
```

---

# 12. Asynchronous reset

Asynchronous reset:

```verilog
always @(posedge clk or posedge rst) begin
    if (rst)
        q <= 1'b0;
    else
        q <= d;
end
```

Плюсы:

```
может сбросить регистр без clock;
удобен для power-on/global reset;
часто используется в IP;
быстро приводит схему в безопасное состояние.
```

Минусы:

```
опасное асинхронное снятие reset;
нужны recovery/removal timing checks;
сложнее анализ RDC;
может создавать reset tree с большими fanout;
может ухудшать timing/placement.
```

Практический компромисс:

```
external/global reset — async assert
внутренний local reset — sync deassert
```

---

# 13. Recovery и removal

Для асинхронного reset у триггера есть ограничения:

```
recovery time
removal time
```

Это похоже по смыслу на setup/hold, но относится к reset.

Если reset отпускается слишком близко к фронту clock, триггер может начать работать некорректно.

Reset synchronizer решает именно эту проблему для deassertion.

---

# 14. Почему reset deassertion может ломать state machine

Представим FSM:

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        state <= IDLE;
    else
        state <= next_state;
end
```

Если разные регистры FSM выйдут из reset не в один и тот же такт, FSM может попасть в невозможное состояние.

Например one-hot FSM:

```
RESET: 0001
после некорректного release:
       0000
или   0011
```

Это может привести к зависанию.

---

# 15. Reset skew внутри одного clock domain

Даже внутри одного clock domain reset может прийти к разным регистрам с разной задержкой.

Для synchronous reset это обычно анализируется как обычный timing path.

Для asynchronous reset важны recovery/removal и физическое распространение reset.

Если reset имеет огромный fanout:

```
rst_n -> тысячи регистров
```

могут появиться проблемы:

```
fanout слишком большой;
маршрутизация reset тяжелая;
разное время прихода reset;
ухудшение timing;
сложный debug при старте.
```

---

# 16. Reset fanout management

Для больших FPGA-проектов reset часто нужно структурировать.

Плохо:

```
один глобальный rst_n идет на весь дизайн
```

Лучше:

```
reset controller
    |
    +--> rst_n_sys_ctrl
    +--> rst_n_sys_datapath
    +--> rst_n_sfp
    +--> rst_n_pcie
    +--> rst_n_ddr
```

Плюсы:

```
меньше fanout на один net;
проще sequencing;
проще debug;
можно сбрасывать subsystem отдельно;
меньше случайных RDC-связей.
```

---

# 17. Reset как control signal

Reset часто фактически является control signal.

Например:

```
software пишет bit reset_rx = 1
```

Этот bit может формироваться в AXI-Lite clock domain, а сбрасывать блок в SFP clock domain.

Это CDC:

```
axi_clk domain reset command -> sfp_clk domain reset action
```

Нельзя просто подключать software reset bit напрямую к async reset другого домена.

Лучше:

```
software reset request
    -> CDC synchronizer/handshake
    -> local reset controller in target domain
```

---

# 18. Reset request vs reset signal

Полезно разделять:

```
reset request — команда “нужно сбросить блок”
reset signal  — локальный сигнал сброса регистров
```

Пример:

```
AXI register bit reset_rx_req
        |
        v
CDC pulse/level into rx_clk domain
        |
        v
rx reset controller
        |
        v
rx_rst_n
```

Так reset формируется уже в target clock domain.

Это чище, чем тянуть AXI-сигнал как reset по всему проекту.

---

# 19. Reset одного домена влияет на другой

Опасная ситуация:

```
clk_a domain сброшен
clk_b domain продолжает работать
```

Если `clk_b` ожидает valid/ready/ack от `clk_a`, он может зависнуть.

Или наоборот:

```
clk_b domain сброшен
clk_a domain продолжает писать в FIFO/handshake
```

Возможны:

```
потерянные request;
зависший busy;
некорректный ack;
FIFO overflow;
FSM waiting forever.
```

Поэтому reset interaction нужно проектировать как protocol event.

---

# 20. Reset и handshake CDC

Handshake имеет состояния в двух domains:

```
req в source domain
ack в destination domain
```

Если один domain сбросился, а другой нет, protocol может оказаться в невозможном состоянии.

Например:

```
source держит req = 1
destination reset cleared ack/state
source ждет ack, которого уже не будет
```

Решение:

```
оба направления handshake должны иметь reset-aware behavior;
после reset protocol возвращается в idle;
source должен уметь обнаружить reset/busy clear;
или reset должен распространяться sequenced образом.
```

---

# 21. Reset и toggle synchronizer

Toggle synchronizer хранит состояние toggle-bit в source и destination.

Если один domain сбросить, а другой нет:

```
source toggle = 1
destination sync state = 0
```

после reset может появиться ложный pulse или потеряться следующий event.

Поэтому для toggle CDC нужно определить:

```
что происходит при reset source;
что происходит при reset destination;
разрешены ли события во время reset;
нужно ли сбрасывать оба домена вместе;
нужно ли после reset делать resync/idle period.
```

---

# 22. Reset и async FIFO

Async FIFO имеет два clock domains:

```
write side resetread side reset
```

В vendor FIFO обычно есть конкретные требования:

```
как долго держать reset;
какие clocks должны тикать;
когда valid flags становятся корректными;
есть ли wr_rst_busy / rd_rst_busy;
можно ли писать/читать во время reset busy.
```

Практическое правило:

```
после reset async FIFO не писать, пока write side busy;
не читать, пока read side busy;
не считать flags валидными раньше, чем FIFO вышел из reset.
```

---

# 23. XPM FIFO reset busy

У Xilinx XPM FIFO часто есть сигналы типа:

```
wr_rst_busy
rd_rst_busy
```

Их нужно учитывать.

Например:

```verilog
assign fifo_wr_en_safe = src_valid && src_ready && !wr_rst_busy;
assign fifo_rd_en_safe = dst_ready && dst_valid && !rd_rst_busy;
```

Или через ready/valid:

```verilog
assign s_axis_tready = !fifo_full && !wr_rst_busy;
assign m_axis_tvalid = !fifo_empty && !rd_rst_busy;
```

Идея:

```
не взаимодействовать с FIFO, пока соответствующая сторона в reset/busy состоянии.
```

---

# 24. Reset и valid/ready stream

При reset stream-интерфейс должен возвращаться в корректное idle-состояние.

Обычно:

```
source после reset:
    valid = 0

destination после reset:
    ready может быть 0 или 1, но должно быть определено
```

Плохой случай:

```
source выходит из reset и сразу valid=1 со старым data
destination еще в reset или FIFO busy
```

Решение:

```
valid сбрасывать;data считать значимым только при valid;ready блокировать до окончания local reset;при CDC FIFO учитывать reset busy.
```

---

# 25. Reset и data coherency

Reset может нарушить coherency между data и control.

Пример:

```
data path не сброшенcontrol path сброшен
```

или наоборот:

```
control FSM стартовалаdata pipeline еще содержит старые данные
```

Поэтому при reset нужно понимать:

```
какие регистры действительно должны сбрасываться;какие pipeline registers могут оставаться без reset;как сбросить valid-биты;как очистить FIFO/buffers;как downstream узнает, что старые данные недействительны.
```

Часто достаточно сбрасывать `valid`, а не весь `data`.

---

# 26. Не все регистры нужно сбрасывать

В FPGA не всегда полезно сбрасывать каждый data-регистр.

Например pipeline:

```
always @(posedge clk) begin    data_pipe <= next_data;end
```

можно не сбрасывать, если рядом есть `valid_pipe`, который сбрасывается:

```
always @(posedge clk) begin    if (rst)        valid_pipe <= 1'b0;    else        valid_pipe <= next_valid;end
```

После reset data может быть мусором, но `valid = 0`, значит он не используется.

Плюсы:

```
меньше reset fanout;лучше timing;меньше routing congestion;проще synthesis/retiming;меньше reset-related проблем.
```

---

# 27. Reset может мешать retiming/optimization

Большое количество reset-ов на data path может ограничивать оптимизации.

Например:

```
retiming сложнее;SRL/BRAM inference может ухудшиться;DSP pipeline mapping может стать хуже;fanout reset-net может ухудшить timing.
```

Поэтому хороший стиль:

```
сбрасывать control/valid/state;не сбрасывать большие data pipelines без необходимости.
```

---

# 28. Initial values в FPGA

В Xilinx FPGA регистры могут иметь initial value после конфигурации:

```
reg flag = 1'b0;
```

Это часто синтезируется как начальное состояние при загрузке bitstream.

Но initial value — не полная замена reset.

Разница:

```
initial value работает при конфигурации FPGA;reset может понадобиться во время работы;partial subsystem reset требует отдельной логики;после PLL relock/init values сами по себе не применятся.
```

Практически:

```
для power-up можно использовать init values;для runtime reset нужен reset controller.
```

---

# 29. Reset synchronizer без reset?

Для обычных CDC synchronizers иногда используют только initial value и не дают reset.

Например:

```
(* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)reg [1:0] sync_ff = 2'b00;always @(posedge clk_dst) begin    sync_ff <= {sync_ff[0], async_in};end
```

Это может быть нормальным для некоторых status signals.

Плюсы:

```
меньше reset fanout;нет reset deassertion issue внутри synchronizer;простая CDC-структура.
```

Минусы:

```
нет runtime reset этого synchronizer;нужно понимать power-up/init behavior;не подходит, если system reset должен принудительно очистить output.
```

---

# 30. Нужно ли сбрасывать synchronizer stages?

Единого ответа нет.

Для reset synchronizer — да, сама схема использует async reset.

Для обычного bit synchronizer — зависит от signal semantics.

Если output synchronizer должен быть известным во время reset, можно сбрасывать.

Но reset на synchronizer может сам стать дополнительным RDC-фактором, если он не относится к destination domain.

Практическое правило:

```
если сбрасываешь CDC synchronizer,reset должен быть корректным для destination clock domain.
```

---

# 31. Reset release order

В multi-clock system часто важен порядок отпускания reset.

Например:

```
1. clocks stable2. MMCM/PLL locked3. low-level PHY reset released4. IP core reset released5. FIFO reset released6. user logic reset released7. software-visible ready flag set
```

Если отпустить user logic раньше IP core, логика может начать читать невалидные статусы.

Если отпустить writer раньше FIFO read side, FIFO может переполниться.

Если отпустить receiver позже transmitter без backpressure, данные могут потеряться.

---

# 32. Reset controller

Для сложных проектов полезен отдельный reset controller.

Он делает:

```
собирает reset sources;учитывает pll_locked;ждет стабилизацию clocks;создает local resets;управляет sequencing;выдает ready/status;обрабатывает software reset requests;не дает блокам стартовать до готовности зависимостей.
```

Пример структуры:

```
reset_controller    inputs:        board_rst_n        pll_locked        sw_reset_req        ip_error    outputs:        sys_rst_n        sfp_rst_n        fifo_rst        user_logic_enable        system_ready
```

---

# 33. Clock отсутствует — reset не отпустится

Если используется synchronous deassertion, то для отпускания reset нужны clock edges.

Если clock не работает:

```
reset synchronizer не сдвигаетсяlocal reset остается asserted
```

Это обычно хорошо.

Но важно понимать:

```
если домен без clock, он не выйдет из reset;если другой домен ждет от него ack, может зависнуть;reset sequencing должен учитывать clock availability.
```

---

# 34. Clock остановился во время работы

Если clock domain может останавливаться, reset interaction усложняется.

Например:

```
clk_b остановленclk_a продолжает посылать данные
```

Если reset `clk_b` не может корректно отработать без clock, read side FIFO может не выйти из reset.

Архитектура должна определить:

```
можно ли останавливать clock;кто останавливает upstream;какие ready/valid становятся 0;как происходит restart;нужно ли сбрасывать оба домена.
```

---

# 35. Reset и external interfaces

Некоторые внешние интерфейсы имеют свои требования reset sequencing:

```
DDRPCIeGT/AuroraEthernet PHYADC/DACSPI/I2C devices
```

Для них важно:

```
когда clock stable;когда released reset;когда calibration done;когда link up;когда user logic может начать передачу.
```

В таких случаях reset — часть протокола инициализации, а не просто один провод.

---

# 36. Reset и AXI

AXI-интерфейсы обычно имеют reset, связанный с соответствующим `ACLK`.

Например:

```
S_AXI_ACLKS_AXI_ARESETN
```

`ARESETN` должен быть синхронно отпущен относительно `ACLK`.

Если AXI-Lite domain управляет reset другого domain, это уже CDC/RDC:

```
AXI register write -> reset request -> target clock domain reset controller
```

Нельзя напрямую использовать AXI register bit как reset другого clock domain без синхронизации.

---

# 37. Reset и IP cores в Vivado

У Vivado IP часто есть свои reset inputs:

```
aresetnresets_axi_aresetnm_axis_aresetngt_resetuser_reset
```

Важно читать semantics конкретного IP:

```
active high или active low;sync или async reset;к какому clock относится;минимальная длительность reset;есть ли reset done/busy;нужно ли ждать locked/link_up/calib_done;можно ли reset только часть IP.
```

Ошибка в reset polarity или clock association может выглядеть как “рандомный CDC bug”.

---

# 38. Reset polarity

В больших проектах легко перепутать:

```
rstrst_naresetnresetreset_n
```

Хорошая практика:

```
суффикс _n только для active-low;локальные reset naming по domain;не смешивать active-high и active-low без явных инверсий;инверсии делать в одном месте, а не по всему проекту.
```

Пример:

```
wire sys_rst;     // active-highwire sys_rst_n;   // active-low
```

Лучше не иметь оба без необходимости.

---

# 39. Naming по reset domain

Полезные имена:

```
sys_rst_naxi_rst_nsfp_user_rst_npcie_user_rst_nrx_datapath_rst_ntx_datapath_rst_n
```

Плохие имена:

```
resetrstrst1reset_sync
```

По имени должно быть понятно:

```
какая polarity;какой clock domain;для какого subsystem.
```

---

# 40. Simulation reset scenarios

В testbench стоит проверять не только power-up reset.

Полезные сценарии:

```
reset при старте;reset одного domain во время работы;reset обоих domains одновременно;reset source быстрее destination;reset destination быстрее source;PLL lock drop/relock;FIFO reset во время active stream;software reset во время pending handshake;reset при backpressure.
```

Многие reset-domain bugs проявляются только при runtime reset, а не при обычном старте.

---

# 41. Assertions для reset

Даже простые assertions помогают.

Например:

```
не писать в FIFO во время reset busy;не читать из FIFO во время reset busy;valid должен быть 0 во время reset;после reset FSM должна быть в IDLE;request не должен висеть бесконечно после reset;system_ready не должен стать 1 до ready всех подсистем.
```

На SystemVerilog это можно оформить как SVA, но даже обычные testbench checks полезны.

---

# 42. ILA/debug reset issues

В железе reset-problems часто выглядят как:

```
иногда не стартует после power-up;после soft reset блок зависает;после перезапуска link данные не идут;FIFO показывает empty/full неправильно;один раз из десяти система не выходит из IDLE;после потери lock recovery не работает.
```

Для ILA полезно смотреть:

```
локальный reset конкретного domain;reset request;reset busy/done;pll_locked;ip_ready/link_up;FSM state;FIFO full/empty/rst_busy;valid/ready around reset.
```

Лучше иметь ILA в том же clock domain, где анализируются сигналы.

---

# 43. Vivado report_cdc и reset

`report_cdc` может подсвечивать reset-related crossing или асинхронные управляющие сигналы.

Но Vivado не всегда понимает полную reset-sequencing архитектуру.

Поэтому reset нужно проверять не только отчетами, но и review:

```
откуда пришел reset;к какому clock он синхронен;где он используется;что будет, если один domain reset, а другой нет;что будет при повторном reset во время работы.
```

---

# 44. Практический reset flow для multi-clock FPGA

Хороший порядок:

```
1. Выписать все clock domains.2. Выписать все reset sources.3. Определить, какие reset sources асинхронны.4. Сделать reset controller.5. Для каждого clock domain сделать local reset synchronizer.6. Не использовать reset одного domain в другом domain.7. Для software reset делать reset request CDC.8. Для FIFO/IP учитывать reset busy/done.9. Определить reset release order.10. Проверить runtime reset scenarios в simulation.11. Проверить report_cdc/report_methodology.12. Проверить reset/status в ILA на железе.
```

---

# 45. Частые ошибки

## Ошибка 1

```
Один synchronized reset используется в нескольких clock domains.
```

Если reset синхронизирован к `clk_a`, он не синхронен к `clk_b`.

---

## Ошибка 2

```
Software reset bit напрямую подключен к reset другого domain.
```

Это CDC/RDC. Нужен request crossing и local reset generation.

---

## Ошибка 3

```
Не учитывать reset busy у FIFO/IP.
```

После reset нельзя сразу писать/читать, если IP еще занят внутренним reset.

---

## Ошибка 4

```
Считать, что reset исправит CDC.
```

Reset не делает междоменную передачу безопасной.

---

## Ошибка 5

```
Сбрасывать все data-регистры без необходимости.
```

Это увеличивает fanout и может ухудшить timing. Часто достаточно сбрасывать valid/control.

---

## Ошибка 6

```
Не проверять runtime reset.
```

Power-up reset прошел, но soft reset во время работы может ломать protocol.

---

## Ошибка 7

```
Отпускать user logic раньше, чем готов IP/clock/FIFO.
```

Нужно sequencing или ready gating.

---

## Ошибка 8

```
Reset deassertion напрямую от async source.
```

Снятие reset должно быть синхронным в каждом domain.

---

# 46. Практическая таблица

|Ситуация|Рекомендуемый подход|
|---|---|
|Внешний reset pin|async assert, sync deassert per domain|
|PLL/MMCM locked|использовать как reset source, затем local sync/delay|
|Software reset другого domain|CDC request → local reset controller|
|Async FIFO reset|соблюдать reset busy/done, не читать/писать во время reset|
|AXI reset|reset синхронен к соответствующему ACLK|
|Data pipeline|часто не сбрасывать data, сбрасывать valid|
|FSM/control|сбрасывать в известное состояние|
|Несколько domains|отдельный local reset для каждого clock|
|Clock может остановиться|учитывать, что sync reset release требует clock edges|
|Runtime reset|проверять protocol recovery|

---

# 47. Минимальный практический шаблон

Для проекта с двумя доменами:

```
raw sources:    board_rst_n    pll_locked    sw_reset_reqsys_clk domain:    reset_sync -> sys_rst_n    sys reset controllersfp_clk domain:    reset_sync -> sfp_rst_n    sfp reset controllercross-domain reset command:    sw_reset_req -> cdc_pulse/handshake -> target reset controller
```

То есть:

```
не тянуть reset напрямую через весь дизайн;не использовать reset одного domain в другом;не смешивать reset request и reset signal.
```

---

# 48. Главное резюме

Reset-domain interaction — это один из самых недооцененных источников ошибок в multi-clock FPGA design.

Главные правила:

```
1. Reset тоже может быть CDC/RDC-сигналом.2. Опаснее всего reset deassertion.3. Хорошая практика: async assert, synchronous deassert.4. Каждый clock domain должен иметь свой local reset.5. Reset, синхронизированный к одному clock, нельзя использовать в другом clock.6. Software reset другого domain нужно передавать как request через CDC.7. Async FIFO/IP reset требует учета busy/done/status.8. Не все data-регистры нужно сбрасывать; часто достаточно valid/control.9. Reset release order должен быть спроектирован.10. Runtime reset нужно проверять отдельно, а не только power-up.
```

Короткая формула:

```
global reset source        -> reset conditioning/controller        -> per-clock-domain reset synchronizer        -> local reset logic        -> ready/status sequencing
```

И самая важная мысль:

```
Reset — это не просто “очистить регистры”.Reset — это часть архитектуры запуска, восстановления и взаимодействия clock domains.
```