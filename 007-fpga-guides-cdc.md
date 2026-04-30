## Clock Domain Crossing (CDC) в FPGA / Vivado / Digital Design / RTL

**Clock Domain Crossing**, или **CDC**, — это тема про безопасную передачу сигналов между разными тактовыми доменами. В FPGA это одна из самых важных областей RTL-проектирования, потому что ошибка CDC может не проявляться в simulation, может не ловиться обычным timing analysis и при этом вызывать редкие, трудноуловимые сбои в реальном железе.

Простой пример:

```verilog
always @(posedge clk_a)    
	data_a <= new_value;

always @(posedge clk_b)    
	data_b <= data_a;
```

Если `clk_a` и `clk_b` не синхронны друг другу, такой код потенциально опасен. Сигнал `data_a` может измениться слишком близко к фронту `clk_b`, и триггер в домене `clk_b` может попасть в **metastability**.

---
# 1. Что такое clock domain

**Clock domain** — это область логики, которая работает от одного clock signal.

Например:

```verilog
always @(posedge clk_100m)    
	reg_a <= signal_a;
```

и

```verilog
always @(posedge clk_156m)    
	reg_b <= signal_b;
```

Это два разных clock domains:

```
clk_100m domain
clk_156m domain
```

Даже если оба clock генерируются внутри одного FPGA, например через MMCM/PLL, они могут считаться разными доменами, если между ними нет гарантированной фазовой связи или если Vivado не может безопасно проанализировать timing между ними.

---

# 2. Почему CDC опасен

Главная проблема — **metastability**.

Триггер ожидает, что входной сигнал будет стабилен до и после фронта clock. Это описывается через:

```
setup timehold time
```

Если сигнал из другого clock domain меняется рядом с фронтом принимающего clock, setup/hold могут быть нарушены.

В результате выход триггера может:

```
0
1
или временно находиться в неопределенном аналоговом состоянии
```

Это состояние называется **metastable state**.

Обычно metastability сама “рассасывается” за некоторое время, но проблема в том, что следующий триггер или логика могут успеть считать неправильное значение.

---

# 3. Почему simulation обычно не показывает CDC-проблемы

Обычная RTL simulation работает с цифровыми значениями:

```
0
1
X
Z
```

Но metastability — это физический аналоговый эффект внутри триггера. RTL simulation не моделирует реальное поведение триггера при нарушении setup/hold.

Поэтому такой код может идеально работать в simulation:

```verilog
always @(posedge clk_b)    
	sync_signal <= async_signal;
```

но нестабильно работать в FPGA.

---

# 4. Основные типы CDC-передач

CDC бывает разным. Нельзя применять один и тот же способ для всех случаев.

Основные варианты:

```
1-bit level signal
1-bit pulse
multi-bit data bus
valid/data handshake
streaming data
FIFO crossing
reset crossing
```

Для каждого типа нужен свой правильный метод.

---

# 5. Передача одного статического сигнала

Если нужно передать один медленно меняющийся 1-bit signal из одного домена в другой, обычно используют **two-flop synchronizer**.

Пример:

```verilog
reg sync_ff1;
reg sync_ff2;

always @(posedge clk_dst or negedge rst_dst_n) 
begin    
	if (!rst_dst_n) 
	begin        
		sync_ff1 <= 1'b0;        
		sync_ff2 <= 1'b0;    
	end 
	else 
	begin        
		sync_ff1 <= async_signal;        
		sync_ff2 <= sync_ff1;    
	end
end
assign synced_signal = sync_ff2;
```

Смысл:

```
async_signal -> FF1 -> FF2 -> safe signal
```

Первый триггер может попасть в metastability, но у него есть почти целый период `clk_dst`, чтобы стабилизироваться до того, как значение попадет во второй триггер.

Важно: такой метод подходит для **однобитного level signal**, но не подходит напрямую для multi-bit bus.

---

# 6. Почему нельзя просто синхронизировать каждый бит шины

Плохой пример:

```verilog
reg [7:0] sync1;
reg [7:0] sync2;

always @(posedge clk_b) 
begin    
	sync1 <= data_from_clk_a;    
	sync2 <= sync1;
end
```

На первый взгляд кажется, что это обычный two-flop synchronizer, только для 8 бит. Но это опасно.

Причина: разные биты шины могут захватиться в разные такты. Например, передавалось значение:

```
8'h3F
```

потом стало:

```
8'h40
```

На принимающей стороне можно временно получить мусорное значение, например:

```
8'h00
8'h7F
8'h30
```

или другое неконсистентное состояние.

Для multi-bit data нужно использовать другие методы:

```
handshake
toggle synchronizer
Gray code
async FIFO
dual-clock FIFO
```

---

# 7. Передача pulse между доменами

Однотактный импульс в одном домене может быть потерян в другом домене.

Например:

```
clk_a:  pulse длится 1 такт clk_a
clk_b:  медленнее или не совпадает по фазе
```

Если фронт `clk_b` не попадет внутрь pulse, принимающий домен вообще его не увидит.

Плохой вариант:

```verilog
always @(posedge clk_b)    
	pulse_b <= pulse_a;
```

Правильные варианты:

```
pulse stretching
toggle synchronizer
request/acknowledge handshake
async FIFO
```

Один из популярных способов — **toggle synchronizer**.

В исходном домене при событии меняется состояние toggle-бита:

```verilog
always @(posedge clk_a or negedge rst_a_n) 
begin    
	if (!rst_a_n)        
		toggle_a <= 1'b0;    
	else if (event_a)        
		toggle_a <= ~toggle_a;
	end
```

В принимающем домене этот toggle синхронизируется и превращается обратно в pulse:

```verilog
reg sync1;
reg sync2;
reg sync2_d;

always @(posedge clk_b or negedge rst_b_n) 
begin    
	if (!rst_b_n) 
	begin        
		sync1   <= 1'b0;        
		sync2   <= 1'b0;        
		sync2_d <= 1'b0;    
	end 
	else 
	begin        
		sync1   <= toggle_a;        
		sync2   <= sync1;        
		sync2_d <= sync2;    
	end
end
assign pulse_b = sync2 ^ sync2_d;
```

Так принимающий домен ловит не сам короткий импульс, а факт изменения состояния.

---

# 8. Передача data bus через handshake

Если нужно передать несколько бит данных из одного домена в другой, можно использовать **handshake protocol**.

Идея:

```
source domain выставляет data и requestdestination domain принимает data и отвечает acknowledgesource domain держит data стабильными до acknowledge
```

Упрощенно:

```
clk_a domain:    data_a stable    req_a = 1clk_b domain:    видит req    захватывает data    ack_b = 1clk_a domain:    видит ack    снимает req
```

Такой подход подходит, когда данные передаются не каждый такт, а отдельными словами или командами.

Главное правило:

**data bus должен оставаться стабильным достаточно долго**, пока принимающий домен не подтвердит прием.

---

# 9. Передача потока данных через async FIFO

Если нужно передавать поток данных между clock domains, лучший стандартный способ — **asynchronous FIFO** или **dual-clock FIFO**.

Например:

```
write clock  = clk_aread clock   = clk_bwrite data   = data_aread data    = data_b
```

В Vivado для этого можно использовать:

```
FIFO Generator IPXPM_FIFO_ASYNCсобственный async FIFO
```

Для большинства практических FPGA-проектов удобно использовать **Xilinx XPM**:

```
xpm_fifo_async
```

Плюсы:

```
готовый проверенный CDC-механизмкорректная обработка full/emptyподдержка разных ширинпонятная интеграция в Vivado
```

Async FIFO обычно нужен для:

```
AXI-Stream между clock domainsпередачи ADC/DAC данныхSFP/PCIe/user logic crossingпередачи пакетовбуферизации между разными частотами
```

---

# 10. CDC и reset

Reset тоже может быть CDC-проблемой.

Особенно опасен **asynchronous reset deassertion**.

Например:

```
always @(posedge clk or negedge rst_n) begin    if (!rst_n)        reg_a <= 1'b0;    else        reg_a <= next_value;end
```

Асинхронный reset может быть нормально применен, но его снятие должно быть безопасным относительно clock.

Часто используют правило:

```
asynchronous assertsynchronous deassert
```

То есть reset можно активировать асинхронно, но отпускать его нужно через синхронизатор в каждом clock domain.

Пример:

```
reg [1:0] rst_sync;always @(posedge clk or negedge arst_n) begin    if (!arst_n)        rst_sync <= 2'b00;    else        rst_sync <= {rst_sync[0], 1'b1};endassign rst_n = rst_sync[1];
```

---

# 11. CDC и Vivado timing constraints

CDC тесно связан с constraints.

Если два clock domain асинхронны, Vivado не должен пытаться делать обычный setup/hold timing analysis между ними как между синхронными доменами.

Для этого используют:

```
set_clock_groups -asynchronous \    -group [get_clocks clk_a] \    -group [get_clocks clk_b]
```

Или иногда:

```
set_false_path -from [get_clocks clk_a] -to [get_clocks clk_b]set_false_path -from [get_clocks clk_b] -to [get_clocks clk_a]
```

Но важно понимать:

**constraint не делает CDC безопасным.**

Constraint только говорит Vivado, как анализировать или не анализировать timing. А безопасность CDC обеспечивается RTL-архитектурой:

```
synchronizerhandshakeasync FIFOGray codereset synchronizer
```

---

# 12. CDC и Vivado reports

В Vivado есть команды и отчеты, которые помогают анализировать CDC.

Например:

```
report_cdc
```

Эта команда ищет потенциально опасные пересечения clock domains.

Типичные проблемы, которые может показать CDC report:

```
одиночный async signal без синхронизатораmulti-bit crossing без handshake/FIFOнеправильный reset crossingCDC через combinational logicодин и тот же async signal используется в нескольких местах
```

Но `report_cdc` не заменяет понимание архитектуры. Он помогает найти подозрительные места, но разработчик должен понимать, какой CDC-механизм применен и почему он корректен.

---

# 13. Частые ошибки CDC

## Ошибка 1: синхронизировать multi-bit bus через два регистра

```
always @(posedge clk_b) begin    data_sync1 <= data_a;    data_sync2 <= data_sync1;end
```

Это не гарантирует целостность значения.

---

## Ошибка 2: передавать короткий pulse напрямую

```
always @(posedge clk_b)    pulse_b <= pulse_a;
```

Pulse может быть потерян.

---

## Ошибка 3: считать, что simulation все проверит

CDC-проблемы часто не видны в обычной RTL simulation.

---

## Ошибка 4: поставить `set_false_path` и думать, что проблема решена

`set_false_path` убирает timing check, но не создает синхронизатор.

---

## Ошибка 5: использовать async reset без синхронного release

Reset может отпускаться в разные моменты относительно разных clock domains и создавать нестабильный старт логики.

---

# 14. Практическое правило выбора CDC-метода

Можно использовать такую таблицу:

|Что передаем|Рекомендуемый способ|
|---|---|
|1-bit медленный level signal|two-flop synchronizer|
|1-bit короткий pulse|toggle synchronizer или handshake|
|multi-bit control word|handshake|
|счетчик между доменами|Gray code|
|поток данных|async FIFO|
|AXI-Stream|AXIS async FIFO / FIFO Generator / XPM FIFO|
|reset|reset synchronizer per domain|
|status flag|synchronizer, но с учетом semantics|
|data + valid|handshake или FIFO|

---

# 15. Минимальный набор CDC-блоков в своем проекте

В хорошей FPGA-практике полезно иметь свою маленькую библиотеку CDC primitives:

```
cdc_sync_bitcdc_sync_pulsecdc_togglecdc_handshakecdc_bus_handshakecdc_reset_synccdc_async_fifo wrapper
```

Например:

```
rtl/common/cdc/    cdc_sync_bit.v    cdc_pulse.v    cdc_handshake.v    cdc_reset_sync.v    cdc_fifo_async.v
```

Тогда на верхнем уровне проекта CDC становится явно видимым:

```
cdc_sync_bit u_link_up_sync (...);cdc_pulse    u_start_pulse_sync (...);cdc_fifo_async u_axis_cdc_fifo (...);
```

Это сильно упрощает сопровождение проекта.

---

# 16. CDC в контексте Vivado-проекта

В Vivado-проекте CDC нужно контролировать на нескольких уровнях:

```
RTL architectureconstraintssimulationimplementation reportshardware debug
```

Практический flow может быть таким:

```
1. На уровне архитектуры определить clock domains.2. Нарисовать связи между доменами.3. Для каждой связи выбрать CDC method.4. В RTL использовать явные CDC-модули.5. В XDC правильно описать clocks и clock groups.6. После synthesis/implementation запускать report_cdc.7. Проверять, что CDC warnings либо исправлены, либо осознанно объяснены.8. В hardware debug проверять уже функциональные проблемы, а не искать CDC вслепую.
```

---

# 17. Главное резюме

CDC — это не просто “передать сигнал между двумя clock”.

Это отдельная область RTL-дизайна, где нужно учитывать:

```
metastabilityпотерю pulseцелостность multi-bit datareset crossingправильные constraintsVivado CDC reportsархитектурную читаемость проекта
```

Главная мысль:

**Для каждого пересечения clock domains должен быть явно выбран и реализован CDC-механизм.**

Нельзя просто подключать сигналы между разными clock domains напрямую.