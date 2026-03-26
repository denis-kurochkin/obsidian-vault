# Lane Structure

## 1. Что такое lane

**Lane** — это одна независимая последовательная линия передачи данных внутри SerDes-интерфейса.

Обычно под lane понимают один полный канал передачи, который включает:

- **TX path** — передающую часть;
- **RX path** — принимающую часть;
- связанные с ними clocking/alignment/control-механизмы.

На уровне платы lane обычно соответствует:

- одной дифференциальной паре на передачу;
- одной дифференциальной паре на приём.

То есть для full-duplex link один lane — это не “один провод”, а один полноценный serial channel.

---

## 2. Зачем нужны несколько lanes

Один lane удобен своей простотой, но его пропускная способность ограничена:

- возможностями transceiver;
- качеством канала;
- допустимым line rate;
- требованиями по BER и SI.

Если нужен больший throughput, часто выгоднее не бесконечно поднимать line rate одного канала, а использовать **несколько параллельных lanes**.

Идея простая:

- один lane даёт некоторую скорость;
- несколько lanes дают суммарную пропускную способность выше.

Пример:

- 1 lane × 10 Gbps = 10 Gbps raw
- 4 lanes × 10 Gbps = 40 Gbps raw

---

## 3. Lane — это базовая единица SerDes-линка

Когда говорят о serial link, часто есть два уровня:

### Уровень lane

Одна отдельная последовательная полоса со своим:

- line rate;
- CDR;
- alignment;
- error status;
- физическим каналом.

### Уровень link

Группа lanes, которые вместе образуют один логический интерфейс.

Это важное различие:  
**lane** — физико-логический кирпичик,  
**link** — более крупная конструкция из одного или нескольких lanes.

---

## 4. Что обычно входит в структуру одного lane

На концептуальном уровне lane включает несколько частей.

## 4.1. User-side interface

Параллельные данные и служебные сигналы со стороны логики FPGA/PCS.

Например:

- data bus;
- control symbols;
- valid/status;
- alignment indicators.

## 4.2. PCS-функции

То, что связано с обработкой символов и структур потока:

- encoding/decoding;
- word alignment;
- comma/block detection;
- elastic buffer;
- lane-specific status.

## 4.3. PMA-функции

Более низкий уровень, ближе к физике:

- serializer/deserializer;
- PLL/CDR;
- equalization;
- polarity handling;
- high-speed TX/RX analog path.

## 4.4. Physical channel

Сама трасса/разъём/кабель/бэкплейн, по которому идёт сигнал.

---

## 5. Lane на TX и lane на RX

На передаче lane делает следующее:

1. принимает параллельные данные;
2. при необходимости кодирует их;
3. сериализует;
4. отправляет в линию с нужным line rate.

На приёме lane:

1. принимает искажённый физический сигнал;
2. восстанавливает timing;
3. выполняет sampling и deserialization;
4. ищет границы символов;
5. декодирует поток;
6. отдаёт параллельные данные выше.

То есть lane — это не просто “кусок трассы”, а полный тракт преобразования между parallel domain и serial domain.

---

## 6. Single-lane и multi-lane link

## Single-lane

Вся логика и весь поток идут через один serial channel.

Плюсы:

- проще bring-up;
- проще debug;
- нет lane alignment между полосами;
- меньше проблем со skew.

Минусы:

- ограниченный throughput.

## Multi-lane

Один логический поток раскладывается по нескольким lanes.

Плюсы:

- выше aggregate bandwidth;
- можно не поднимать слишком сильно rate одного канала.

Минусы:

- нужны lane alignment и deskew;
- выше сложность логики и отладки;
- больше требований к структуре линка.

---

## 7. Что меняется, когда lanes несколько

Как только появляется multi-lane link, возникают дополнительные задачи.

### 7.1. Data distribution

Нужно решить, как раскладывать данные между lanes.

### 7.2. Lane identification

Нужно понимать, какой lane несёт какую часть потока.

### 7.3. Deskew

Из-за разных задержек канала данные из разных lanes приходят не одновременно.

### 7.4. Lane alignment

Нужно выровнять их во времени перед сборкой общего потока.

### 7.5. Link assembly

После выравнивания данные надо снова собрать в единый логический поток.

Это и есть ключевое отличие структуры multi-lane от single-lane.

---

## 8. Lane structure в многополосном интерфейсе

Если смотреть на multi-lane link сверху, обычно структура выглядит так:

```
logical data stream  
   -> striping/distribution across lanes  
      -> lane 0  
      -> lane 1  
      -> lane 2  
      -> lane 3  
   -> physical transmission  
   -> per-lane recovery/alignment  
   -> deskew/reassembly  
   -> logical data stream restored
```

То есть каждый lane физически работает отдельно, но на уровне link они должны снова стать единым интерфейсом.

---

## 9. Lane rate и link throughput

Очень важно не путать:

### Line rate per lane

Скорость одной полосы.

### Aggregate link rate

Суммарная raw скорость всех lanes вместе.

Формально:

aggregate raw rate = lane_count × line_rate_per_lane

Но если есть encoding overhead, то payload throughput ниже.

Поэтому при разговоре о lane structure всегда полезно уточнять:

- сколько lanes;
- какой rate на lane;
- какая effective payload rate у всего link.

---

## 10. Lane skew

Одна из центральных тем multi-lane architecture.

**Lane skew** — это разность во времени прихода данных с разных lanes.

Причины:

- разная длина трасс;
- разные физические свойства каналов;
- отличия в трансиверах;
- разные фазовые условия;
- различия при training/alignment.

Если skew не компенсировать, данные из lanes будут относиться к разным кускам потока, и link соберётся неправильно.

---

## 11. Deskew как часть структуры multi-lane link

Поэтому у multi-lane link почти всегда есть блок **deskew**.

Его задача:

- принять данные из нескольких lanes;
- временно буферизовать их;
- дождаться alignment markers;
- компенсировать различия задержек;
- выдать синхронно выровненный поток.

Практически это значит: в multi-lane link структура одного lane недостаточна сама по себе — нужна ещё логика **между lanes**.

---

## 12. Lane alignment markers

Чтобы multi-lane system могла понять, как выровнять полосы, в поток обычно вставляют специальные структуры:

- alignment markers;
- control symbols;
- training sequences;
- block headers;
- lane markers.

Их задача — дать RX понять, где находятся соответствующие позиции в разных lanes.

Без этого multi-lane reassembly была бы крайне ненадёжной.

---

## 13. Lane independence и lane coupling

Полезно видеть две стороны lane structure.

### Lane independence

Каждый lane имеет собственный:

- физический канал;
- RX recovery;
- error state;
- possibly local CDR behavior.

### Lane coupling

Но на уровне link lanes связаны:

- общим protocol framing;
- alignment rules;
- deskew logic;
- общим состоянием “link up”.

То есть lane не полностью автономен: он одновременно отдельный канал и часть общей системы.

---

## 14. Lane mapping

В multi-lane systems нужно понимать, как логические данные распределяются по lanes.

Возможны варианты:

- round-robin striping;
- fixed byte/word mapping;
- protocol-defined distribution;
- lane-specific control/data mapping.

Это важно при:

- интеграции PCS;
- debug;
- проверке alignment;
- поиске ошибок “данные есть, но порядок неверный”.

---

## 15. Lane polarity and lane swap

На практике структура линка часто должна быть устойчивой к особенностям платы.

### Polarity inversion

Дифференциальная пара lane может быть разведена с инверсией полярности. Многие transceivers умеют это компенсировать.

### Lane swap

Номера lanes на плате могут быть переставлены относительно логической нумерации.

Поэтому в реальных системах lane structure — это не только “идеальная логическая схема”, но и механизм сопоставления:

- physical lane numbering
- logical lane numbering

Это особенно важно в широких multi-lane links.

---

## 16. Lane status

У каждого lane обычно есть собственные признаки состояния, например:

- CDR lock;
- byte/word alignment;
- comma detected;
- disparity/code errors;
- lane ready;
- RX reset done / TX reset done;
- buffer status.

А у всего link сверху уже есть агрегированное состояние:

- all lanes aligned;
- deskew complete;
- channel bonded;
- link ready.

Это полезно помнить: **link up** обычно означает не просто “каждый lane жив”, а “все lanes живы и правильно собраны вместе”.

---

## 17. Lane structure и channel bonding

Во многих FPGA/IP для multi-lane link используется термин **channel bonding**.

Это механизм, который:

- связывает несколько lanes в одну группу;
- выравнивает их;
- гарантирует правильную совместную работу.

С архитектурной точки зрения channel bonding — это и есть важная часть multi-lane structure поверх отдельных lanes.

---

## 18. Lane structure и latency

У single-lane link latency обычно проще:

- serialization;
- propagation;
- recovery;
- deserialization;
- decoding.

У multi-lane link добавляется ещё:

- deskew buffering;
- lane alignment;
- channel bonding logic.

Поэтому multi-lane architecture почти всегда добавляет latency и делает её менее “прозрачной”.

---

## 19. Практический взгляд для FPGA/RTL

Если смотреть глазами FPGA-разработчика, то lane structure влияет на:

- сколько transceiver channels нужно;
- как выбрать их placement;
- какой общий refclk/PLL использовать;
- как устроить user-side width;
- как реализовать lane bonding;
- как делать reset/bring-up;
- как контролировать status каждого lane;
- как диагностировать skew, loss of alignment, broken lane mapping.

То есть lane structure — это одновременно вопрос:

- физической архитектуры линка,
- логической сборки потока,
- и удобства отладки.

---

## 20. Частые ошибки понимания

### Ошибка 1. Думать, что lane — это просто одна “пара проводов”

Физически да, но архитектурно lane включает и TX/RX clocking, и alignment, и status.

### Ошибка 2. Путать lane и link

Link может состоять из одного или нескольких lanes.

### Ошибка 3. Считать, что если каждый lane работает отдельно, то и весь link работает

Для multi-lane этого недостаточно: нужны alignment и reassembly.

### Ошибка 4. Недооценивать skew

В multi-lane это одна из главных практических проблем.

### Ошибка 5. Игнорировать lane mapping/swap/polarity

На реальной плате это часто всплывает сразу.

---

## 21. Мини-чеклист по теме

Когда разбираешь serial interface, полезно сразу уточнить:

1. Сколько lanes у link?
2. Какой line rate на один lane?
3. Single-lane это или multi-lane architecture?
4. Как данные распределяются по lanes?
5. Как выполняется lane alignment?
6. Есть ли deskew / channel bonding?
7. Какие alignment markers используются?
8. Есть ли отдельный status на lane?
9. Как нумерация physical lanes сопоставлена с logical lanes?
10. Что добавляет multi-lane structure к latency и debug complexity?

---

## 22. Короткие правила

### Правило 1

**Lane** — это базовый serial channel, а не просто кусок проводника.

### Правило 2

**Link** может состоять из одного или нескольких lanes.

### Правило 3

В multi-lane system важны не только per-lane TX/RX, но и **deskew + bonding + reassembly**.

### Правило 4

Работа всех lanes по отдельности ещё не означает корректную работу всего link.

### Правило 5

Чем больше lanes, тем выше throughput, но и выше сложность alignment и debug.

---

## 23. Вывод

**Lane structure** — это способ организации serial link на уровне отдельных последовательных каналов и их совместной работы. Один lane — это полноценный TX/RX тракт с собственным физическим и логическим поведением. Несколько lanes позволяют наращивать пропускную способность, но требуют дополнительной логики для alignment, deskew и сборки общего потока.

Самая полезная мысль по теме:

**single-lane link — это просто один serial channel, а multi-lane link — это уже система координации нескольких serial channels.**