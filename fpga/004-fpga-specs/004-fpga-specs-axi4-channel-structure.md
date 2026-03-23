
# AXI Channel Structure (AXI4)

AXI (Advanced eXtensible Interface) использует **канально-ориентированную архитектуру**, где каждая транзакция разбита на независимые логические каналы. Это ключевое отличие от более старых шин (например, AHB).

---

## Основная идея

AXI не передаёт "транзакцию" как единое целое.  
Вместо этого используются **5 независимых каналов**, каждый со своим handshake.

Это позволяет:
- pipelining
- высокую пропускную способность
- независимость адреса и данных
- поддержку multiple outstanding transactions

---

## 5 каналов AXI4

### Write Path

1. **AW (Write Address Channel)**
   - Передаёт адрес записи и параметры burst
   - Основные сигналы:
     - `AWADDR`
     - `AWLEN`
     - `AWSIZE`
     - `AWBURST`
     - `AWVALID / AWREADY`

2. **W (Write Data Channel)**
   - Передаёт данные
   - Основные сигналы:
     - `WDATA`
     - `WSTRB`
     - `WLAST`
     - `WVALID / WREADY`

3. **B (Write Response Channel)**
   - Ответ от slave
   - Основные сигналы:
     - `BRESP`
     - `BVALID / BREADY`

---

### Read Path

4. **AR (Read Address Channel)**
   - Передаёт адрес чтения
   - Основные сигналы:
     - `ARADDR`
     - `ARLEN`
     - `ARSIZE`
     - `ARBURST`
     - `ARVALID / ARREADY`

5. **R (Read Data Channel)**
   - Передаёт данные чтения
   - Основные сигналы:
     - `RDATA`
     - `RRESP`
     - `RLAST`
     - `RVALID / RREADY`

---

## VALID / READY Handshake

Каждый канал использует независимый handshake:

- `VALID` — данные валидны (producer)
- `READY` — готов принять (consumer)

Передача происходит только если:
VALID && READY

---
### Ключевые правила:
- `VALID` не должен зависеть от `READY`
- данные должны быть стабильны при `VALID=1 && READY=0`
- каналы полностью независимы

---

## Независимость каналов

AXI допускает:
- адрес может быть отправлен раньше данных
- данные могут идти с задержкой
- несколько адресов могут быть "в полёте"

Это даёт:
- latency hiding
- возможность reordering (с ID)
- эффективную работу с памятью (DDR)

---

## Связь каналов (логическая)

Хотя каналы независимы, они логически связаны:

### Write transaction:
1. AW → адрес
2. W → данные (один или несколько beats)
3. B → ответ

### Read transaction:
1. AR → адрес
2. R → данные (один или несколько beats)

---

## Burst и структура передачи

Burst описывается через:
- `LEN` — количество beats
- `SIZE` — размер одного beat
- `BURST` — тип (обычно INCR)

Для write:
- `WLAST` указывает последний beat

Для read:
- `RLAST` указывает последний beat

---

## Практические замечания

- Backpressure (через READY) — ключ к timing closure
- W и AW каналы могут идти независимо → требует буферизации
- AXI interconnect может reorder транзакции (через ID)
- Частая ошибка: связывать READY разных каналов

---

## Краткое резюме

AXI Channel Structure:
- 5 независимых каналов
- VALID/READY handshake на каждом
- полная декомпозиция транзакций
- поддержка pipeline и outstanding операций

👉 Это основа высокой производительности AXI