## Automation of regression в FPGA (Vivado): подробная заметка

**Automation of regression** — это система, которая автоматически проверяет, что после изменений в RTL/constraints/IP дизайн **не сломался** и **не ухудшился**:

- функционально (simulation / testbench)
- по timing (setup/hold)
- по utilization (LUT/FF/BRAM/DSP)
- по QoR (Fmax, slack, congestion)

Проще говоря:

> **каждое изменение в проекте автоматически проверяется набором тестов и метрик.**

В FPGA это особенно важно, потому что:

- изменения часто влияют не только на логику, но и на physical implementation
- timing может “поехать” даже при небольших правках
- ошибки могут быть не очевидны без системной проверки

---

# 1. Что такое regression в FPGA-контексте

В software regression — это просто запуск тестов.

В FPGA regression шире:

## 1.1 Functional regression

- simulation (RTL / post-synth / post-route)
- testbench
- protocol checks

## 1.2 Implementation regression

- synthesis + implementation
- timing closure
- QoR анализ

## 1.3 System-level regression

- несколько конфигураций (variants)
- разные top-level
- разные параметры

👉 Поэтому regression в FPGA = **simulation + build + analysis**

---

# 2. Зачем нужна automation of regression

Без автоматизации:

- разработчик проверяет только “свою часть”
- timing может сломаться незаметно
- старые баги возвращаются
- разные люди получают разные результаты

С автоматизацией:

- каждый commit проверяется
- проблемы ловятся рано
- QoR контролируется
- проект становится масштабируемым

---

# 3. Основная идея regression automation

Главная идея:

> **Каждое изменение автоматически запускает полный набор проверок.**

Pipeline обычно выглядит так:

commit → trigger → build → simulate → analyze → report → pass/fail

---

# 4. Основные компоненты regression системы

## 4.1 Test suite

Набор тестов:

- directed tests
- random tests
- corner cases
- protocol scenarios

Важно:

- тесты должны быть **автоматически запускаемыми**
- не требовать ручного вмешательства

---

## 4.2 Build automation (Vivado)

Regression невозможен без scripted runs:

- Tcl scripts
- batch mode
- reproducible build

👉 без этого невозможно запускать builds автоматически

---

## 4.3 Metrics collection

Нужно собирать:

### Timing

- worst negative slack (WNS)
- total negative slack (TNS)
- Fmax

### Utilization

- LUT
- FF
- BRAM
- DSP

### QoR

- congestion
- routing quality

---

## 4.4 Pass/Fail criteria

Regression должен давать ответ:

- PASS
- FAIL

Примеры критериев:

- simulation passed
- no timing violations
- utilization < threshold
- slack ≥ 0

---

## 4.5 Reporting

Результаты должны быть:

- сохранены
- сравнимы
- читаемы

Например:

- отчеты в текст/HTML
- summary таблицы
- diff между builds

---

## 4.6 CI integration

Regression обычно запускается через CI:

- GitHub Actions
- GitLab CI
- Jenkins

CI отвечает за:

- запуск
- логирование
- хранение artifacts
- уведомления

---

# 5. Типы regression в FPGA

## 5.1 Pre-commit (fast regression)

- быстро
- simulation only
- иногда synth-only

Цель:

- быстро отловить грубые ошибки

---

## 5.2 Nightly regression

- полный build
- полный набор тестов
- timing analysis

Цель:

- проверить стабильность проекта

---

## 5.3 Release regression

- максимальный coverage
- все конфигурации
- полный QoR анализ

---

## 5.4 QoR regression

Специальный тип:

- отслеживает timing/utilization изменения
- не обязательно проверяет функциональность

---

# 6. Regression pipeline (практическая структура)

Типичный pipeline:

### Шаг 1. Checkout

- получить код из git

### Шаг 2. Build

- запустить Vivado Tcl script

### Шаг 3. Simulation

- запустить testbench
- собрать результаты

### Шаг 4. Reports

- timing
- utilization
- другие метрики

### Шаг 5. Analysis

- сравнить с baseline
- проверить thresholds

### Шаг 6. Decision

- PASS / FAIL

---

# 7. Regression и baseline

Очень важное понятие — **baseline**.

Baseline — это:

- предыдущий “хороший” build
- reference для сравнения

Сравнивают:

- timing (WNS)
- utilization
- Fmax

Пример:

baseline WNS = 0.5 ns  
current WNS = 0.1 ns → warning  
current WNS = -0.2 ns → FAIL

---

# 8. Regression для timing

Timing regression — отдельная большая тема.

Что проверяют:

- slack ≥ 0
- no new critical paths
- нет деградации QoR

Важно:

- timing может флуктуировать
- нужно учитывать variability

👉 иногда используют:

- thresholds
- допустимые отклонения

---

# 9. Regression для simulation

Simulation regression должна:

- запускать все тесты автоматически
- собирать результаты
- выявлять:
    - assertion failures
    - protocol violations
    - mismatches

Хорошая практика:

- каждый тест → PASS/FAIL
- summary по всем тестам

---

# 10. Regression и parameterized builds

В FPGA часто есть:

- разные `top`
- разные `part`
- разные `define`
- разные configurations

Regression должен уметь:

- запускать build для всех вариантов
- проверять каждый отдельно

---

# 11. Regression и reproducibility

Automation невозможна без reproducibility.

Если build:

- нестабилен
- зависит от среды

то regression будет:

- flaky
- unreliable

👉 поэтому:

- фиксировать tool version
- использовать scripted runs
- избегать hidden state

---

# 12. Regression и performance trade-offs

Есть конфликт:

- хочется быстро
- хочется полно

Решение:

|Тип|Скорость|Coverage|
|---|---|---|
|pre-commit|fast|low|
|nightly|medium|high|
|release|slow|full|

---

# 13. Частые ошибки

## 13.1 Нет автоматического запуска

Regression “есть”, но его запускают вручную.

## 13.2 Нет четких PASS/FAIL

Есть отчеты, но нет решения.

## 13.3 Нет baseline

Невозможно понять, стало лучше или хуже.

## 13.4 Игнор timing regression

Проверяют только simulation.

## 13.5 Слишком медленный regression

Никто не запускает.

## 13.6 Flaky builds

Результаты нестабильны.

---

# 14. Хорошая regression система

Признаки:

- запускается автоматически (CI)
- полностью scripted
- reproducible
- дает PASS/FAIL
- хранит историю
- показывает деградации
- масштабируется

---

# 15. Mental model

Думай так:

> Regression — это “safety net” проекта

Любое изменение:

- проходит через сетку проверок
- либо проходит
- либо ловится

---

# 16. Практическая минимальная система

Минимум, который уже полезен:

- Tcl build script
- simulation script
- timing report check
- simple CI job
- PASS/FAIL

Даже такой уровень уже сильно повышает качество проекта.

---

# 17. Главная инженерная мысль

**Regression automation — это не “дополнительная опция”.  
Это основа стабильного FPGA-проекта.**

Без нее:

- ошибки накапливаются
- timing ломается незаметно
- проект становится хрупким

С ней:

- изменения контролируемы
- качество растет
- масштабирование возможно

---

## Краткое резюме

**Automation of regression** — это система, которая автоматически проверяет дизайн после каждого изменения:

- simulation
- synthesis/implementation
- timing
- QoR

Основные элементы:

- scripted runs
- test suite
- metrics
- baseline
- CI integration

Главный принцип:

> **каждое изменение должно автоматически проверяться и давать четкий PASS/FAIL.**