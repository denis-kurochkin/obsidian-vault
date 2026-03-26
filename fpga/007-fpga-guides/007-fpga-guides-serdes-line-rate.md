# Line Rate

## 1. Что такое line rate

**Line rate** — это скорость передачи данных **по физической последовательной линии**.

Обычно она измеряется в:

- **bit/s**
- **Mb/s**
- **Gb/s**

Например:

- 1.25 Gbps
- 3.125 Gbps
- 6.25 Gbps
- 10.3125 Gbps

По сути line rate отвечает на вопрос:

**с какой скоростью биты идут по serial line**

---

## 2. Что именно считается “линией”

В SerDes под line обычно понимают одну последовательную передачу по одному lane.

Если интерфейс дифференциальный, физически это обычно одна дифференциальная пара, но логически это **один serial channel / one lane**.

Поэтому line rate обычно задают **на один lane**, а не на весь многополосный интерфейс сразу.

Например:

- 4 lanes × 10 Gbps  
    не означает “line rate = 40 Gbps” для одной линии;  
    это означает:
- **line rate per lane = 10 Gbps**
- aggregate raw rate = 40 Gbps

---

## 3. Line rate и user data rate — не одно и то же

Это самое важное различие.

**Line rate** — это сырая скорость битов на линии.  
**User data rate** — это скорость полезных данных, которую видит логика выше.

Они часто отличаются, потому что в serial link обычно есть накладные расходы:

- line coding;
- block headers;
- scrambling overhead;
- alignment symbols;
- training sequences;
- idle symbols;
- control characters;
- иногда FEC.

То есть на линии передаются не только payload bits.

---

## 4. Пример: 8b/10b

Если используется **8b/10b**, то каждые 8 бит полезных данных превращаются в 10 бит на линии.

Значит:

user data rate = line rate × 8 / 10

Пример:

- line rate = 6.25 Gbps

Тогда максимальная скорость полезных данных:

- 6.25 × 0.8 = **5.0 Gbps**

Остальные 20% уходят на кодирование.

---

## 5. Пример: 64b/66b

Для **64b/66b** overhead намного меньше.

Формула:

user data rate = line rate × 64 / 66

Если:

- line rate = 10.3125 Gbps

то полезная скорость:

- 10.3125 × 64 / 66 = **10.0 Gbps**

Именно поэтому 10Gb Ethernet использует line rate выше 10 Gbps: часть полосы уходит на кодирование.

---

## 6. Полезная формула

В общем виде:

effective payload rate = line rate × coding_efficiency

где:

- `coding_efficiency = useful_bits / transmitted_bits`

Примеры:

- 8b/10b → 0.8
- 64b/66b → 64/66 ≈ 0.9697

Если есть ещё дополнительные служебные символы или FEC, реальная эффективная скорость будет ниже.

---

## 7. Line rate и baud rate

Эти термины часто путают.

## Для NRZ

При обычной двухуровневой передаче **NRZ**:

- 1 symbol = 1 bit

Тогда:

- **baud rate = line rate**

Пример:

- 10 Gbaud = 10 Gbps

## Для PAM4

При **PAM4** один символ несёт 2 бита.

Тогда:

- line rate в битах больше baud rate в символах

Пример:

- 28 Gbaud PAM4
- line rate = 56 Gb/s

Поэтому line rate — это битовая скорость, а baud rate — скорость символов.

---

## 8. Почему line rate важен

Line rate — один из главных параметров любого SerDes, потому что от него зависят:

- требования к каналу;
- качество сигнала;
- допустимые потери;
- сложность equalization;
- требования к CDR;
- величина unit interval;
- джиттерный бюджет;
- achievable BER;
- выбор transceiver mode;
- reference clock configuration;
- ширина user-side интерфейса.

Чем выше line rate, тем жёстче почти все физические требования.

---

## 9. Unit Interval напрямую связан с line rate

**Unit Interval (UI)** — длительность одного передаваемого бита.

Формула:

UI = 1 / line rate

Примеры:

- 1 Gbps → UI = 1 ns
- 10 Gbps → UI = 100 ps
- 25 Gbps → UI = 40 ps

Это очень важная интуиция: с ростом line rate битовый интервал становится всё короче, а значит:

- меньше временной запас на sampling;
- выше чувствительность к джиттеру;
- сильнее влияние канала.

---

## 10. Line rate и user clock

На user-side логика обычно не работает на битовой частоте линии.  
Иначе при multi-gigabit скоростях это было бы практически невозможно.

Вместо этого SerDes использует:

- высокую serial bit rate на line-side;
- более широкие параллельные слова на user-side.

Связь примерно такая:

user clock × user width = raw internal data rate

С поправкой на конкретную внутреннюю архитектуру transceiver и encoding.

Например, если линия работает на 10 Gbps, внутри user-side данные могут идти как:

- 20 бит × 500 MHz
- 40 бит × 250 MHz
- 64 бит × соответствующая частота

То есть line rate напрямую влияет на выбор внутренней ширины и clocking architecture.

---

## 11. Line rate per lane и aggregate rate

Для многополосных интерфейсов важно различать:

### Per-lane line rate

Скорость одной полосы.

### Aggregate raw rate

Суммарная скорость всех lanes до учёта overhead.

Формула:

aggregate raw rate = line rate per lane × number of lanes

Если:

- 4 lanes
- 12.5 Gbps per lane

то raw aggregate rate:

- 50 Gbps

Но payload throughput всё равно надо считать с учётом кодирования, alignment и протокольных накладных расходов.

---

## 12. Почему нельзя смотреть только на line rate

Высокий line rate сам по себе не означает высокий полезный throughput.

Нужно учитывать:

- encoding overhead;
- protocol overhead;
- idle/control symbols;
- FEC;
- training;
- lane utilization;
- packet structure;
- backpressure or framing inefficiency.

Например, два линка могут иметь близкий line rate, но разную полезную производительность из-за разных схем кодирования и протокольных расходов.

---

## 13. Как выбирают line rate

Line rate обычно определяется не “как удобно”, а исходя из:

- требований стандарта;
- доступного transceiver режима;
- reference clock;
- допустимой ширины user interface;
- пропускной способности;
- качества канала;
- длины трассы/кабеля;
- BER target;
- возможностей PLL/CDR;
- ограничений платы и коннекторов.

То есть это компромисс между:

- desired throughput
- physical feasibility
- supported hardware modes

---

## 14. Line rate и reference clock

В transceiver-системах line rate обычно связан с **reference clock** через PLL.

Упрощённо:

line rate = refclk × multiplier / divider

Точная формула зависит от конкретного transceiver, но идея именно такая:

- есть опорный такт;
- PLL умножает/делит его;
- на выходе получается serial clocking, задающий line rate.

Поэтому при настройке SerDes line rate почти всегда рассматривают вместе с:

- refclk frequency;
- PLL mode;
- divider settings.

---

## 15. Пример инженерного мышления

Допустим, нужно передавать **8 Gbps полезных данных**.

### Вариант с 8b/10b

Требуемый raw line rate:

8 / 0.8 = 10 Gbps

### Вариант с 64b/66b

Требуемый raw line rate:

8 × 66 / 64 = 8.25 Gbps

Вывод:

- при одинаковом payload coding choice сильно влияет на необходимый line rate;
- а значит и на требования к каналу и transceiver.

---

## 16. Практический смысл для FPGA-инженера

Для RTL/FPGA разработчика line rate важен не только как цифра “в гигабитах”, а как параметр, который влияет на:

- конфигурацию transceiver IP;
- выбор refclk;
- data width on fabric side;
- latency;
- reset/bring-up procedure;
- выбор encoding;
- пропускную способность тракта;
- требования к PCB и SI.

Иначе говоря: line rate — это точка, где сходятся и логика, и физика.

---

## 17. Частые ошибки

### Ошибка 1. Путать line rate с payload rate

Это, пожалуй, самая частая ошибка.

### Ошибка 2. Не учитывать overhead кодирования

Например, считать, что 6.25 Gbps line rate = 6.25 Gbps полезных данных.

### Ошибка 3. Путать total link throughput и per-lane rate

Особенно в multi-lane системах.

### Ошибка 4. Путать baud rate и bit rate

Особенно когда речь начинает идти не только про NRZ.

### Ошибка 5. Считать, что line rate — только цифровой параметр

На деле он напрямую влияет на SI, jitter budget и требования к каналу.

---

## 18. Мини-чеклист по теме

Когда смотришь на serial link, полезно сразу уточнять:

1. Какой **line rate per lane**?
2. Сколько lanes?
3. Какое line coding используется?
4. Какой payload rate после overhead?
5. Это NRZ или PAM4?
6. Какой UI при этой скорости?
7. Какой refclk нужен для такого режима?
8. Поддерживает ли transceiver этот line rate?
9. Потянет ли его канал по loss/jitter/BER?
10. Какая user-side width нужна внутри FPGA?

---

## 19. Короткие правила

### Правило 1

**Line rate** — это скорость битов на физической serial line.

### Правило 2

**Payload rate** почти всегда меньше line rate.

### Правило 3

Для multi-lane links всегда разделяй:

- per-lane rate
- aggregate raw rate
- effective payload throughput

### Правило 4

Чем выше line rate, тем меньше UI и тем жёстче требования к каналу.

### Правило 5

Line rate нельзя рассматривать отдельно от coding, PLL/refclk и физики канала.

---

## 20. Вывод

**Line rate** — это базовый параметр SerDes-линка, который описывает, с какой скоростью биты передаются по одной последовательной линии. Это не то же самое, что полезная скорость данных, потому что часть полосы обычно уходит на кодирование и служебные структуры.

Практически line rate нужно всегда рассматривать вместе с четырьмя вещами:

- **encoding efficiency**
- **per-lane vs aggregate throughput**
- **UI / signal integrity impact**
- **clocking and transceiver configuration**