# Synchronizers внутри CDC

**Synchronizer** — это специальный RTL-блок, который безопасно вводит сигнал из одного clock domain в другой.

В контексте CDC это один из базовых building blocks. Он не “устраняет” физическую metastability полностью, но резко снижает вероятность того, что metastable-состояние попадет дальше в пользовательскую логику.

Главная идея:

```
async / foreign clock domain signal
        |
        v  
	synchronizer in destination clock domain
        |
        v
safe signal for dst logic
```

Важно: **synchronizer всегда принадлежит принимающему clock domain**. То есть если сигнал идет из `clk_a` в `clk_b`, то synchronizer работает на `clk_b`.

---

# 1. Где synchronizer находится в CDC-блоке

Представим два clock domains:

```
clk_a domain                         clk_b domain

+-------------+                      +----------------+
| source RTL  | ---- signal_a -----> | synchronizer   |
+-------------+                      +----------------+
                                             |
                                             v
                                      +--------------+
                                      | dst RTL      |
                                      +--------------+
```

Сигнал `signal_a` был сформирован в домене `clk_a`.

Дальше он попадает в домен `clk_b`.

Без synchronizer принимающий триггер может захватить сигнал в момент его изменения. Это может привести к metastability.

Synchronizer ставится первым элементом в принимающем домене.

---

# 2. Что synchronizer решает

Synchronizer помогает решить проблему **metastability propagation**.

Если первый триггер принимающего домена попал в metastable state, ему дают дополнительное время стабилизироваться до того, как его значение попадет в обычную логику.

Классический вариант:

```
async_signal -> FF1 -> FF2 -> synced_signal                    clk_dst
```

`FF1` может стать metastable.

`FF2` обычно уже получает стабилизированное значение.

---

# 3. Что synchronizer не решает

Это очень важная часть.

Synchronizer **не является универсальным CDC-решением**.

Он не гарантирует:

```
1. что короткий pulse будет пойман;
2. что multi-bit bus будет принят целостно;
3. что несколько связанных сигналов сохранят взаимную синхронность;
4. что событие не потеряется, если события идут слишком часто;
5. что data/valid-протокол будет корректным сам по себе;
6. что reset будет безопасно снят во всех доменах.
```

Synchronizer решает только одну базовую задачу:

> безопасно ввести один асинхронный или чужой по clock сигнал в destination domain.

Все остальное зависит от выбранного CDC-протокола.

---

# 4. Самый базовый вариант: two-flop synchronizer

Это классический synchronizer для одного 1-bit level signal.

```verilog
module cdc_sync_bit #(
    parameter integer STAGES = 2,
    parameter INIT_VALUE = 1'b0
)(
    input  wire clk_dst,
    input  wire async_in,
    output wire sync_out
);

    (* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)
    reg [STAGES-1:0] sync_ff = {STAGES{INIT_VALUE}};

    always @(posedge clk_dst) begin
        sync_ff <= {sync_ff[STAGES-2:0], async_in};
    end

    assign sync_out = sync_ff[STAGES-1];

endmodule
```

Использование:

```verilog
wire link_up_clk_b;

cdc_sync_bit #(
    .STAGES(2),
    .INIT_VALUE(1'b0)
) u_link_up_sync (
    .clk_dst  (clk_b),
    .async_in (link_up_clk_a),
    .sync_out (link_up_clk_b)
);
```

---

# 5. Почему именно два триггера

Один триггер недостаточен.

Плохой вариант:

```verilog
always @(posedge clk_dst) 
begin    
	signal_dst <= signal_async;
end
```

Проблема в том, что `signal_dst` может стать metastable, а дальше этот сигнал сразу используется пользовательской логикой.

Лучше:

```verilog
always @(posedge clk_dst) begin  
	sync_ff1 <= signal_async;  
	sync_ff2 <= sync_ff1;  
end
```

Теперь metastability первого триггера имеет время на разрешение до следующего фронта `clk_dst`.

Смысл:

```
FF1 принимает риск metastability
FF2 защищает остальную логику
```

---

# 6. Два или три stages?

Обычно используют:

```
2 stages — стандартный вариант;
3 stages — если нужна повышенная надежность;
4+ stages — редко, обычно для особых случаев.
```

Пример трехступенчатого synchronizer:

```
async_signal -> FF1 -> FF2 -> FF3 -> synced_signal
```

Чем больше stages, тем ниже вероятность выхода metastability наружу, но тем больше latency.

Latency примерно:

```
2-stage synchronizer: 2 такта clk_dst
3-stage synchronizer: 3 такта clk_dst
```

На практике для большинства FPGA-control сигналов хватает двух stages. Три stages используют, например, если:

```
destination clock очень быстрый;
событие очень частое;
система должна иметь повышенную надежность;
сигнал критичен для безопасности состояния автомата;
есть требование от CDC methodology.
```

Подробный расчет надежности лучше вынести в отдельную тему **MTBF concept**, чтобы не смешивать физическую оценку надежности с RTL-паттернами.

---

# 7. Для каких сигналов подходит обычный bit synchronizer

Обычный `cdc_sync_bit` подходит для сигналов типа **level**.

Примеры:

```
status flag
link_up
pll_locked
calibration_done
mode_enable
error_flag
external button after filtering/debounce
configuration bit
```

Главное условие:

> сигнал должен держать свое значение достаточно долго, чтобы принимающий домен успел его увидеть.

Например, если `link_up` стал `1` и остается `1`, то two-flop synchronizer подходит.

Если же сигнал является коротким pulse на один такт исходного clock, обычный bit synchronizer может его потерять.

---

# 8. Почему короткий pulse нельзя просто синхронизировать

Допустим, в домене `clk_a` есть импульс:

```
clk_a domain:

pulse_a: ___|‾|___
```

Он длится один такт `clk_a`.

Если `clk_b` медленнее или его фронты неудачно расположены, принимающий домен может вообще не попасть фронтом clock внутрь этого pulse.

Плохой вариант:

```verilog
cdc_sync_bit u_pulse_sync (
    .clk_dst  (clk_b),
    .async_in (pulse_a),
    .sync_out (pulse_b_level)
);
```

После такого synchronizer может быть:

```
pulse пойман
pulse задержан
pulse растянут
pulse полностью потерян
```

То есть результат зависит от соотношения clock и фаз.

Для pulse нужны специальные synchronizers.

---

# 9. Toggle synchronizer для передачи события

Один из наиболее полезных способов передать событие между clock domains — **toggle synchronizer**.

Идея:

В source domain короткий event превращается не в pulse, а в изменение состояния бита.

```
event happened -> toggle bit changes 0->1 or 1->0
```

Destination domain синхронизирует toggle-бит и обнаруживает изменение.

Схема:

```
clk_a domain                         clk_b domain

event_a
   |
   v
toggle_a  ----------------------->  sync FF1 -> sync FF2 -> edge detect
                                                        |
                                                        v
                                                    pulse_b
```

---

## 9.1 Source side

```verilog
reg toggle_a = 1'b0;

always @(posedge clk_a) begin
    if (event_a) begin
        toggle_a <= ~toggle_a;
    end
end
```

Каждое событие меняет `toggle_a`.

---

## 9.2 Destination side

```verilog
(* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)
reg [1:0] toggle_b_sync = 2'b00;

reg toggle_b_d = 1'b0;

always @(posedge clk_b) begin
    toggle_b_sync <= {toggle_b_sync[0], toggle_a};
    toggle_b_d    <= toggle_b_sync[1];
end

assign pulse_b = toggle_b_sync[1] ^ toggle_b_d;
```

`pulse_b` будет длиться один такт `clk_b`.

---

## 9.3 Полный модуль toggle synchronizer

```verilog
module cdc_pulse_toggle (
    input  wire clk_src,
    input  wire pulse_src,

    input  wire clk_dst,
    output wire pulse_dst
);

    reg toggle_src = 1'b0;

    always @(posedge clk_src) begin
        if (pulse_src) begin
            toggle_src <= ~toggle_src;
        end
    end

    (* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)
    reg [1:0] toggle_dst_sync = 2'b00;

    reg toggle_dst_d = 1'b0;

    always @(posedge clk_dst) begin
        toggle_dst_sync <= {toggle_dst_sync[0], toggle_src};
        toggle_dst_d    <= toggle_dst_sync[1];
    end

    assign pulse_dst = toggle_dst_sync[1] ^ toggle_dst_d;

endmodule
```

Использование:

```verilog
wire start_pulse_clk_b;

cdc_pulse_toggle u_start_pulse_cdc (
    .clk_src   (clk_a),
    .pulse_src (start_pulse_clk_a),

    .clk_dst   (clk_b),
    .pulse_dst (start_pulse_clk_b)
);
```

---

# 10. Ограничение toggle synchronizer

Toggle synchronizer не должен получать события слишком часто.

Почему?

Если source domain успеет сделать два toggle до того, как destination domain увидел изменение, принимающая сторона может не заметить события.

Пример:

```
toggle_a: 0 -> 1 -> 0
```

Если `clk_b` был слишком медленный, он может увидеть только:

```
0 ... 0
```

И решить, что ничего не произошло.

Поэтому toggle synchronizer подходит, когда:

```
события редкие;
между событиями есть достаточная пауза;
потеря события недопустима, но частота событий ограничена архитектурой.
```

Если события могут идти часто, нужен handshake или FIFO.

---

# 11. Pulse stretching

Иногда короткий pulse растягивают в source domain.

Например:

```
pulse_src на 1 такт -> stretched_pulse_src на N тактов
```

И затем синхронизируют как level.

Но это менее надежный метод, если нет строгого знания частот и фаз.

Чтобы pulse stretching был безопасен, stretched pulse должен быть достаточно длинным относительно `clk_dst`.

Примерно:

```
дольше, чем несколько периодов clk_dst
```

Но если частоты могут меняться, если clock unrelated, если destination clock может останавливаться, если есть clock gating, то лучше использовать handshake.

Pulse stretching можно использовать для простых случаев, но как универсальный CDC pattern лучше предпочитать:

```
toggle synchronizer
request/acknowledge handshake
async FIFO
```

---

# 12. Synchronizer для нескольких независимых битов

Иногда нужно синхронизировать несколько **независимых** single-bit сигналов.

Например:

```
status[0] = link_up
status[1] = error_seen
status[2] = pll_locked
```

Если эти биты не образуют одно целостное значение, их можно синхронизировать массивом bit synchronizers.

```verilog
module cdc_sync_bus_independent #(
    parameter integer WIDTH  = 8,
    parameter integer STAGES = 2
)(
    input  wire             clk_dst,
    input  wire [WIDTH-1:0] async_in,
    output wire [WIDTH-1:0] sync_out
);

    genvar i;

    generate
        for (i = 0; i < WIDTH; i = i + 1) begin : g_sync_bit
            cdc_sync_bit #(
                .STAGES(STAGES),
                .INIT_VALUE(1'b0)
            ) u_sync_bit (
                .clk_dst  (clk_dst),
                .async_in (async_in[i]),
                .sync_out (sync_out[i])
            );
        end
    endgenerate

endmodule
```

Но это **не способ передавать data bus**.

---

# 13. Почему нельзя синхронизировать data bus побитно

Плохой вариант:

```verilog
reg [7:0] sync1;
reg [7:0] sync2;

always @(posedge clk_dst) begin
    sync1 <= data_src;
    sync2 <= sync1;
end
```

Такой код выглядит как обычный synchronizer, но для data bus он опасен.

Причина: разные биты могут быть захвачены в разные моменты.

Например, source меняет значение:

```
8'h0F -> 8'h10
```

На destination side можно временно получить некорректное промежуточное значение:

```
8'h00
8'h1F
8'h08
8'h17
```

Любую комбинацию старых и новых битов.

Если эти 8 бит — независимые флаги, это может быть допустимо.

Если эти 8 бит — одно число, адрес, команда, счетчик, код состояния или data word, это недопустимо.

---

# 14. Array synchronizer: когда можно, когда нельзя

Можно:

```
несвязанные status flags;
независимые enable bits;
редко меняющиеся configuration bits, если нет требования atomic update;
debug/status information, где временно смешанное значение не опасно.
```

Нельзя:

```
data bus;
address bus;
counter value;
state encoding;
packet length;
command word;
AXI data;
valid + data как независимые signals;
one-hot state без специальной защиты.
```

Для multi-bit значений нужны другие CDC-паттерны:

```
handshake synchronizer
mux-based data synchronizer
Gray-code synchronizer
async FIFO
```

Async FIFO лучше рассматривать отдельно, потому что это уже полноценная CDC-структура с write/read pointers, full/empty и memory.

---

# 15. Mux-based / handshake synchronizer для data word

Если нужно передать одно слово данных из `clk_src` в `clk_dst`, но это не поток каждый такт, часто используют handshake.

Идея:

```
clk_src domain:
    1. зафиксировать data_src в регистре;
    2. держать data stable;
    3. отправить request;

clk_dst domain:
    4. синхронизировать request;
    5. захватить data;
    6. отправить acknowledge;

clk_src domain:
    7. получить acknowledge;
    8. разрешить следующую передачу.
```

Структура:

```
clk_src domain                                  clk_dst domain

data_reg_src  ------------------------------->  capture register
req_src      ---> synchronizer -------------->  req_dst
ack_src      <--- synchronizer <--------------  ack_dst
```

Главное отличие от побитного synchronizer:

> data bus не пытаются “метастабильно синхронизировать” побитно. Его держат стабильным достаточно долго, а управляющие события синхронизируют отдельно.

Упрощенный протокол:

```
source:
    если есть новое слово и CDC не занят:
        data_hold <= new_data
        req <= 1

destination:
    когда увидел req:
        data_dst <= data_hold
        ack <= 1

source:
    когда увидел ack:
        req <= 0
```

Это уже не просто synchronizer из двух FF, а CDC protocol. Но внутри него все равно используются обычные bit synchronizers для `req` и `ack`.

---

# 16. Важное правило: source signal лучше регистрировать

Желательно, чтобы сигнал, уходящий в CDC, был зарегистрирован в исходном домене.

Хорошо:

```verilog
always @(posedge clk_src) begin
    status_src_r <= status_next;
end

cdc_sync_bit u_status_sync (
    .clk_dst  (clk_dst),
    .async_in (status_src_r),
    .sync_out (status_dst)
);
```

Плохо:

```verilog
assign status_comb = a & b | c;

cdc_sync_bit u_status_sync (
    .clk_dst  (clk_dst),
    .async_in (status_comb),
    .sync_out (status_dst)
);
```

Почему combinational signal перед synchronizer хуже:

```
1. возможны glitches;
2. фактическая частота переключений выше, чем кажется;
3. destination domain может поймать короткий glitch;
4. CDC-анализ становится менее чистым;
5. сложнее понять реальную semantics сигнала.
```

Правило:

> Перед CDC желательно поставить source register. После CDC использовать только final synchronized output.

---

# 17. Нельзя использовать первый stage synchronizer в логике

Плохой вариант:

```verilog
always @(posedge clk_dst) begin
    sync_ff1 <= async_in;
    sync_ff2 <= sync_ff1;
end

assign some_logic = sync_ff1 & enable_dst;  // плохо
```

`sync_ff1` — это потенциально metastable stage.

Его нельзя использовать нигде, кроме как для подачи на следующий stage.

Правильно:

```verilog
assign some_logic = sync_ff2 & enable_dst;
```

Правило:

```
FF1 output используется только FF2 input.
FF2/FF3 output используется обычной логикой.
```

---

# 18. Нельзя вставлять combinational logic между stages

Плохой вариант:

```verilog
always @(posedge clk_dst) begin
    sync_ff1 <= async_in;
    sync_ff2 <= sync_ff1;
end

assign synced_masked = sync_ff2 & mask;
```

Или:

```
always @(posedge clk_dst) begin    sync_ff1 <= async_in;    sync_ff2 <= sync_ff1 & mask;end
```

Между synchronizer stages не должно быть логики.

Правильно:

```
always @(posedge clk_dst) begin    sync_ff1 <= async_in;    sync_ff2 <= sync_ff1;endassign synced_masked = sync_ff2 & mask;
```

Synchronizer chain должен быть максимально простым:

```
FF -> FF -> FF
```

---

# 19. Fanout от async signal

Плохая практика:

```
async_signal    |    +--> synchronizer A    |    +--> synchronizer B    |    +--> synchronizer C
```

Если один и тот же async signal независимо синхронизируется в нескольких местах одного и того же destination domain, возможны ситуации, когда разные части логики увидят изменение в разные такты.

Лучше:

```
async_signal    |    vsingle synchronizer    |    vsynced_signal    |    +--> logic A    +--> logic B    +--> logic C
```

Правило:

> Один crossing — один synchronizer, затем fanout уже синхронизированного сигнала.

---

# 20. Reconvergence problem

Reconvergence — это когда несколько сигналов пересекают clock domain независимо, а потом снова используются вместе.

Пример:

```
clk_src domain:a_src -----> sync -----> a_dst ----+                                   |                                   +--> combinational decision                                   |b_src -----> sync -----> b_dst ----+
```

Если `a_src` и `b_src` менялись согласованно в source domain, после независимой синхронизации эта связь может нарушиться.

Например, source гарантировал:

```
a_src и b_src никогда не равны 1 одновременно
```

Но после CDC destination может на один такт увидеть:

```
a_dst = 1b_dst = 1
```

или наоборот пропустить нужную комбинацию.

Поэтому связанные сигналы нельзя бездумно синхронизировать отдельно.

Если связь важна, нужны:

```
один encoded signal;Gray code;handshake;FIFO;специально спроектированный protocol.
```

---

# 21. Edge detect после synchronizer

Иногда нужно получить pulse в destination domain при изменении synchronized level.

Например, есть `enable_src`, который редко меняется.

Сначала его синхронизируем:

```
wire enable_dst_level;cdc_sync_bit u_enable_sync (    .clk_dst  (clk_dst),    .async_in (enable_src),    .sync_out (enable_dst_level));
```

Потом уже в destination domain делаем edge detection:

```
reg enable_dst_d = 1'b0;always @(posedge clk_dst) begin    enable_dst_d <= enable_dst_level;endassign enable_rise_dst =  enable_dst_level & ~enable_dst_d;assign enable_fall_dst = ~enable_dst_level &  enable_dst_d;assign enable_edge_dst =  enable_dst_level ^  enable_dst_d;
```

Важно:

> edge detect делается после synchronizer, а не до него.

---

# 22. External async inputs

CDC — это не только пересечение между двумя внутренними clock domains.

Любой внешний сигнал, который не синхронен вашему FPGA clock, тоже является CDC/async input.

Примеры:

```
buttonexternal interruptGPIO inputstatus pin от внешнего чипаUART RX относительно system clockнесинхронный data ready
```

Для простого external status input:

```
pin -> input buffer -> synchronizer -> internal logic
```

Для кнопки обычно:

```
button pin -> synchronizer -> debounce/filter -> edge detect
```

То есть сначала сигнал вводится в clock domain, потом обрабатывается.

---

# 23. Synchronizer и Vivado attributes

В Vivado желательно помечать synchronizer registers атрибутом:

```
(* ASYNC_REG = "TRUE" *)
```

Часто также добавляют:

```
(* SHREG_EXTRACT = "NO" *)
```

Пример:

```
(* ASYNC_REG = "TRUE", SHREG_EXTRACT = "NO" *)reg [1:0] sync_ff = 2'b00;
```

Зачем это нужно:

```
ASYNC_REG:    сообщает Vivado, что эти регистры являются synchronizer chain;    помогает CDC analysis распознавать структуру;    помогает placement держать registers близко;    снижает вероятность нежелательной оптимизации.SHREG_EXTRACT = "NO":    запрещает превращать цепочку регистров в shift register LUT/SRL.
```

Для synchronizer важно, чтобы это были именно flip-flops, а не SRL.

---

# 24. Почему placement важен

Synchronizer — это не просто RTL-шаблон. Его физическая реализация тоже важна.

Хорошо, когда stages synchronizer находятся близко друг к другу.

```
FF1 close to FF2
```

Так уменьшаются routing delays между stages, а значит у metastability первого FF остается больше времени, чтобы разрешиться до входа второго FF.

Атрибут `ASYNC_REG` помогает Vivado правильно относиться к таким цепочкам.

---

# 25. Использование Xilinx XPM CDC primitives

В Vivado можно не писать свои CDC-блоки вручную, а использовать Xilinx XPM primitives.

Полезные CDC primitives:

```
xpm_cdc_single       — single-bit level synchronizerxpm_cdc_array_single — array of independent single-bit synchronizersxpm_cdc_pulse        — pulse transferxpm_cdc_handshake    — handshake-based data transferxpm_cdc_gray         — Gray-code crossingxpm_fifo_async       — asynchronous FIFO
```

Для обычного single-bit signal:

```
xpm_cdc_single #(    .DEST_SYNC_FF   (2),    .INIT_SYNC_FF   (0),    .SIM_ASSERT_CHK (0),    .SRC_INPUT_REG  (1)) u_xpm_cdc_single (    .src_clk  (clk_src),    .src_in   (signal_src),    .dest_clk (clk_dst),    .dest_out (signal_dst));
```

Преимущества XPM:

```
проверенный vendor primitive;Vivado лучше распознает CDC structure;параметры явно описывают intent;удобно для report_cdc;меньше риска написать subtle RTL bug.
```

Но важно понимать, что XPM не отменяет архитектурного выбора. Например, `xpm_cdc_array_single` не превращает data bus в безопасный atomic bus. Это все равно массив независимых bit synchronizers.

---

# 26. Типичная структура своей CDC-библиотеки

Даже если использовать XPM, удобно иметь свои wrapper-модули.

Например:

```
rtl/common/cdc/    cdc_sync_bit.v    cdc_sync_level.v    cdc_sync_pulse.v    cdc_sync_toggle.v    cdc_sync_bus_independent.v    cdc_handshake_word.v
```

На верхнем уровне проекта это выглядит понятно:

```
cdc_sync_bit u_pll_locked_to_user_clk (    .clk_dst  (user_clk),    .async_in (pll_locked),    .sync_out (pll_locked_user_clk));cdc_pulse_toggle u_start_to_processing_clk (    .clk_src   (ctrl_clk),    .pulse_src (start_pulse_ctrl),    .clk_dst   (processing_clk),    .pulse_dst (start_pulse_processing));
```

Так CDC-переходы становятся явно видимыми в RTL.

Это лучше, чем случайные два регистра в разных местах проекта.

---

# 27. Naming convention

Хорошая практика — отражать CDC в именах.

Например:

```
wire start_pulse_ctrl_clk;wire start_pulse_data_clk;wire pll_locked_async;wire pll_locked_user_clk;
```

Или:

```
wire link_up_sfp_clk;wire link_up_sys_clk;
```

Плохой вариант:

```
wire link_up;wire link_up_1;wire link_up_sync;
```

Лучше, когда по имени видно:

```
откуда сигнал пришел;в каком clock domain он сейчас находится;это async/raw signal или уже synchronized signal.
```

---

# 28. Хороший RTL-стиль для synchronizers

Хорошо:

```
// Source domainalways @(posedge clk_src) begin    error_flag_src <= error_condition;end// CDC boundarycdc_sync_bit u_error_flag_cdc (    .clk_dst  (clk_dst),    .async_in (error_flag_src),    .sync_out (error_flag_dst));// Destination domainalways @(posedge clk_dst) begin    if (error_flag_dst) begin        state_dst <= ST_ERROR;    endend
```

Плохо:

```
always @(posedge clk_dst) begin    if (error_condition_from_other_domain) begin        state_dst <= ST_ERROR;    endend
```

В хорошем варианте CDC boundary явно виден.

---

# 29. Synchronizer latency

Synchronizer добавляет задержку.

Для two-flop synchronizer:

```
async_in changes    |    vчерез 1-2 такта clk_dst изменение появляется на sync_out
```

Точнее, из-за произвольного положения изменения относительно `clk_dst`, задержка может быть не строго фиксированной с точки зрения source domain.

Для destination domain обычно воспринимают так:

```
изменение появляется после прохождения через 2 FF stages
```

Это важно для control logic.

Например, если source выставил `start`, destination увидит его не мгновенно, а через несколько тактов своего clock.

---

# 30. Initial value и reset

Synchronizer-регистры часто имеют начальное значение:

```
reg [1:0] sync_ff = 2'b00;
```

В Xilinx FPGA initial value для registers обычно может быть реализован при конфигурации FPGA.

Но вопрос reset для synchronizer — отдельная тема.

Краткое практическое правило:

```
если synchronizer output должен иметь безопасное значение после конфигурации,задайте INIT_VALUE;если используете reset, он должен быть корректен относительно destination clock domain;не смешивайте reset crossing и data/control crossing без явной архитектуры.
```

Подробно это лучше рассматривать в отдельной заметке **reset-domain interaction**, потому что там есть свои CDC-опасности.

---

# 31. Simulation synchronizer не доказывает CDC-безопасность

RTL simulation обычно не моделирует настоящую metastability.

Такой код в simulation может работать идеально:

```
always @(posedge clk_dst) begin    sync_ff1 <= async_in;    sync_ff2 <= sync_ff1;end
```

Но simulation не доказывает, что crossing корректен в железе.

Что можно проверять в simulation:

```
protocol behavior;что pulse не теряется при допустимых интервалах;что source не отправляет новое событие, пока CDC busy;что data stable во время handshake;что destination получает ровно один pulse на одно событие;что reset/init behavior корректен.
```

Что simulation обычно не проверяет:

```
реальную metastability;MTBF;физическое placement;реальные setup/hold нарушения между unrelated clocks.
```

---

# 32. Полезные проверки для pulse/toggle synchronizer

Для toggle synchronizer важно архитектурно гарантировать:

```
source events не приходят слишком часто;между двумя событиями destination успевает увидеть изменение;нет ситуации, где два toggle происходят до sampling в destination domain.
```

В простом виде это можно описать как правило проекта:

```
event_src rate << clk_dst rate
```

Но лучше формулировать точнее:

```
между event_src должно быть не меньше нескольких периодов clk_dst
```

Если это нельзя гарантировать, toggle synchronizer не подходит.

---

# 33. Частые ошибки при использовании synchronizers

## Ошибка 1. Считать synchronizer универсальным решением

```
cdc_sync_bit u_valid_sync (...);cdc_sync_bus_independent u_data_sync (...);
```

Если это `valid + data`, такой подход часто ошибочен.

`valid` может прийти в один такт, а `data` быть смешанным из старого и нового значения.

---

## Ошибка 2. Синхронизировать data bus побитно

```
data_dst[0] <= data_src[0];data_dst[1] <= data_src[1];...
```

Это не гарантирует atomic transfer.

---

## Ошибка 3. Использовать первый stage

```
assign signal_dst = sync_ff1;
```

Первый stage потенциально metastable.

---

## Ошибка 4. Два synchronizer для одного сигнала

```
async_flag -> sync A -> logic Aasync_flag -> sync B -> logic B
```

Разные части логики могут увидеть изменение в разные такты.

---

## Ошибка 5. Логика между stages

```
sync_ff2 <= sync_ff1 & enable;
```

Так делать не надо. Сначала чистый synchronizer, потом логика.

---

## Ошибка 6. Combinational async input

```
assign async_to_dst = src_a & src_b;
```

Такой сигнал может glitch-ить.

Лучше зарегистрировать его в source domain.

---

## Ошибка 7. Синхронизировать pulse как level

Однотактный pulse может быть потерян.

---

## Ошибка 8. Не помечать synchronizer registers

Без `ASYNC_REG` и запрета SRL extraction Vivado может хуже распознать CDC-структуру.

---

# 34. Практическая таблица выбора synchronizer

|Ситуация|Подходящий synchronizer|
|---|---|
|Один медленный 1-bit status|`cdc_sync_bit`, `xpm_cdc_single`|
|Один внешний async input|`cdc_sync_bit`, затем обработка|
|Один редкий pulse/event|toggle synchronizer или `xpm_cdc_pulse`|
|Частые events|handshake или FIFO|
|Несколько независимых flags|array of bit synchronizers|
|Multi-bit data word|handshake/mux-based CDC|
|Multi-bit counter|Gray-code synchronizer|
|AXI-Stream / поток данных|async FIFO|
|Reset release|отдельный reset synchronizer|

---

# 35. Минимальный набор synchronizer-модулей

Для практического проекта полезно иметь хотя бы такие модули:

```
cdc_sync_bit    single-bit level crossingcdc_sync_pulse    event/pulse crossing через togglecdc_sync_bus_independent    массив независимых flagscdc_sync_edge    sync level + edge detect in dst domaincdc_handshake_word    передача одного data word через req/ackcdc_reset_sync    отдельный reset synchronizer
```

При этом `cdc_reset_sync` лучше рассматривать отдельно от обычных data/control synchronizers.

---

# 36. Как читать CDC-код на review

Когда видите crossing между clock domains, задавайте вопросы:

```
1. Что это за сигнал: level, pulse, data bus, counter, stream?2. В каком clock domain он формируется?3. В каком clock domain используется?4. Есть ли source register?5. Есть ли ровно один synchronizer на crossing?6. Есть ли logic между synchronizer stages?7. Используется ли первый stage где-то кроме второго stage?8. Сохраняется ли semantics сигнала после CDC?9. Может ли pulse потеряться?10. Может ли data bus стать неконсистентным?11. Помечены ли registers как ASYNC_REG?12. Распознает ли Vivado этот crossing как корректный CDC pattern?
```

Если на вопрос “что это за CDC method?” нет четкого ответа, значит crossing нужно пересмотреть.

---

# 37. Практический пример: status flag

Source domain:

```
reg error_seen_src = 1'b0;always @(posedge clk_src) begin    if (error_condition_src) begin        error_seen_src <= 1'b1;    endend
```

CDC:

```
wire error_seen_dst;cdc_sync_bit u_error_seen_cdc (    .clk_dst  (clk_dst),    .async_in (error_seen_src),    .sync_out (error_seen_dst));
```

Destination domain:

```
always @(posedge clk_dst) begin    if (error_seen_dst) begin        alarm_dst <= 1'b1;    endend
```

Это хороший случай для обычного synchronizer, потому что `error_seen_src` — это level flag, который после установки остается в `1`.

---

# 38. Практический пример: start pulse

Source domain:

```
wire start_pulse_src;
```

CDC:

```
wire start_pulse_dst;cdc_pulse_toggle u_start_pulse_cdc (    .clk_src   (clk_src),    .pulse_src (start_pulse_src),    .clk_dst   (clk_dst),    .pulse_dst (start_pulse_dst));
```

Destination domain:

```
always @(posedge clk_dst) begin    if (start_pulse_dst) begin        state_dst <= ST_START;    endend
```

Это лучше, чем пытаться напрямую синхронизировать `start_pulse_src`.

---

# 39. Практический пример: неправильный data crossing

Плохо:

```
cdc_sync_bit u_valid_sync (    .clk_dst  (clk_dst),    .async_in (valid_src),    .sync_out (valid_dst));cdc_sync_bus_independent #(    .WIDTH(32)) u_data_sync (    .clk_dst  (clk_dst),    .async_in (data_src),    .sync_out (data_dst));
```

Проблема:

```
valid_dst может соответствовать одному слову,а data_dst может быть смесью старого и нового data_src.
```

Лучше:

```
для одиночных слов — handshake;для потока — async FIFO.
```

---

# 40. Главное резюме

**Synchronizer** — это базовый элемент CDC, но не универсальный протокол передачи данных.

Главные правила:

```
1. Synchronizer ставится в destination clock domain.2. Для 1-bit level signal обычно используют 2 FF.3. Первый FF нельзя использовать в логике.4. Между stages не должно быть combinational logic.5. Source signal желательно регистрировать.6. Один async signal лучше синхронизировать один раз.7. Multi-bit bus нельзя синхронизировать побитно, если нужна целостность.8. Pulse нужно передавать через toggle, stretching с осторожностью, handshake или FIFO.9. В Vivado registers нужно помечать ASYNC_REG.10. XPM CDC primitives часто лучше ручных самописных решений.
```

Короткая формула:

```
level  -> bit synchronizerpulse  -> toggle/pulse synchronizerword   -> handshakestream -> async FIFOreset  -> reset synchronizer
```