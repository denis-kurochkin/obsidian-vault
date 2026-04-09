## Build reproducibility в Vivado (FPGA): подробная заметка

**Build reproducibility** — это способность получить **один и тот же результат сборки** (или как минимум предсказуемо эквивалентный результат) при повторном запуске build’а:

- на другой машине
- в другое время
- другим человеком
- в CI

В FPGA это особенно важно, потому что flow включает **heuristics, placement, routing**, и без контроля результат может отличаться даже при одинаковом RTL.

---

# 1. Что значит “reproducible build” в FPGA

Важно различать два уровня reproducibility:

## 1.1 Bit-for-bit reproducibility

- `.bit` файл **полностью идентичен побитово**
- крайне сложно добиться в FPGA (из-за randomness, tool heuristics)

## 1.2 Functional reproducibility (реалистичная цель)

- timing met одинаково
- utilization близкий
- path distribution похожий
- поведение дизайна одинаковое

👉 В реальных проектах обычно стремятся именно к **functional reproducibility**, а не к абсолютной битовой идентичности.

---

# 2. Почему reproducibility — это проблема в FPGA

FPGA build flow НЕ детерминирован по природе.

Причины:

### 2.1 Placement и routing используют heuristics

- tool принимает решения не строго алгоритмически, а эвристически
- small changes → different placement → different timing

### 2.2 Random seeds

- разные run’ы могут использовать разные initial seeds
- это влияет на placement и routing

### 2.3 Environment differences

- разные версии Vivado
- разные OS / paths
- разные library versions
- разные machine resources

### 2.4 Hidden state (особенно в Project Mode)

- GUI project может хранить internal state
- incremental runs могут влиять на результат

👉 Поэтому без контроля build может быть:

- “works on my machine”
- но ломаться в CI или у коллеги

---

# 3. Основная идея build reproducibility

Главная идея:

> **Build должен быть полностью описан кодом, а не состоянием среды.**

То есть:

- не “я открыл проект и нажал Run”
- а “этот Tcl script полностью описывает, как собрать дизайн”

---

# 4. Основные компоненты reproducible build

## 4.1 Deterministic inputs

Все входы должны быть явно заданы:

- RTL files
- XDC constraints
- IP configuration
- block design
- simulation models (если влияют)
- defines / generics

👉 Никаких “оно где-то лежит в проекте”.

---

## 4.2 Явный build flow (scripted)

Build должен выполняться через:

- Tcl script (Vivado)
- batch mode (`vivado -mode batch`)

А не через GUI.

Почему:

- GUI хранит состояние
- Tcl фиксирует порядок действий

---

## 4.3 Фиксация tool version

ОЧЕНЬ важный момент.

Vivado version должен быть:

- зафиксирован (например, 2023.2)
- одинаковый у всех

Почему:

- разные версии дают разные результаты synthesis/implementation
- даже minor changes могут менять timing

👉 В serious проектах version фиксируют как часть build system.

---

## 4.4 Controlled source order

Особенно в Non-Project Mode:

read_verilog file1.v  
read_verilog file2.v

Порядок **имеет значение**.

👉 Он должен быть:

- явным
- стабильным
- не зависеть от файловой системы

---

## 4.5 No hidden state

Плохой подход:

- использовать существующий `.xpr`
- rely на GUI state
- incremental builds без контроля

Хороший подход:

- build “с нуля”
- clean environment
- script fully defines flow

---

## 4.6 Fixed constraints

Constraints (XDC) — часть reproducibility:

- clock definitions
- false paths
- multicycle paths
- IO constraints

Если constraints меняются → build уже другой.

---

## 4.7 Controlled randomness (seeds)

Vivado использует randomness в implementation.

👉 Для reproducibility:

- фиксировать seed (если нужно)
- или хотя бы понимать, что seed влияет на результат

Это особенно важно при debugging QoR.

---

## 4.8 Artifacts как часть build

Reproducible build должен генерировать:

- `.bit`
- `.dcp`
- reports (`timing`, `utilization`)
- logs

И эти artifacts должны:

- сохраняться
- сравниваться
- быть частью CI

---

# 5. Project Mode vs Non-Project Mode

## Project Mode

Проблемы:

- hidden state
- run history
- incremental behavior
- GUI influence

Можно сделать reproducible, но сложнее.

## Non-Project Mode

Плюсы:

- полный контроль через Tcl
- нет hidden state
- легче сделать deterministic flow

👉 Поэтому для reproducibility обычно предпочтителен **Non-Project Mode**.

---

# 6. Основные угрозы reproducibility

Вот список того, что чаще всего ломает reproducibility.

## 6.1 “ручные правки в GUI”

- добавили файл руками
- поменяли constraint
- изменили strategy

👉 и забыли отразить это в script

---

## 6.2 Разные версии Vivado

- у одного 2022.2
- у другого 2023.1

👉 результаты могут сильно отличаться

---

## 6.3 Нестабильный file list

- wildcard’ы (`*.v`)
- порядок файлов меняется
- OS влияет

---

## 6.4 Неявные зависимости

- IP не regenerate
- BD не обновлен
- external scripts

---

## 6.5 Incremental build без контроля

- reuse старого run
- reuse checkpoint
- случайное состояние

---

## 6.6 Environment differences

- PATH
- licenses
- machine performance
- temp directories

---

# 7. Практические техники обеспечения reproducibility

## 7.1 One-command build

Должна существовать команда:

make build

или

./build.sh

которая:

- делает clean
- запускает Vivado
- генерирует artifacts

---

## 7.2 Clean build philosophy

Каждый build должен уметь:

- стартовать с нуля
- не зависеть от предыдущих run’ов

---

## 7.3 Version control всего

В git должны быть:

- RTL
- XDC
- Tcl scripts
- IP config (xci)
- BD scripts
- build scripts

Не должны быть:

- generated artifacts (обычно)

---

## 7.4 Generated vs source separation

Четко разделять:

- source (git)
- build outputs (build dir)

---

## 7.5 Logging и reports

Каждый build должен сохранять:

- timing report
- utilization report
- logs

Чтобы можно было:

- сравнить builds
- понять regression

---

## 7.6 Checkpoints

Использовать `.dcp`:

- post-synth
- post-place
- post-route

Это помогает:

- debug reproducibility issues
- сравнивать состояния

---

## 7.7 Parameterization

Build должен позволять:

- менять top
- менять part
- менять defines

👉 без изменения самого скрипта

---

# 8. Reproducibility vs QoR

Интересный момент:

Иногда хочется:

- reproducible build
- и максимально лучший QoR

Но:

- разные seeds → разные результаты
- один run может быть лучше другого

👉 поэтому есть два режима:

## Stable mode

- фиксированные seeds
- reproducible результат

## Exploration mode

- разные seeds
- поиск лучшего QoR

Хороший flow поддерживает оба.

---

# 9. Reproducibility и CI

В CI reproducibility становится критичной.

CI должен:

- запускать build с нуля
- использовать фиксированную toolchain
- проверять timing
- сохранять artifacts
- сравнивать с предыдущими build’ами

👉 если build не reproducible → CI теряет смысл

---

# 10. Mental model

Полезно думать так:

> Build — это функция  
> **output = f(RTL, constraints, tool, script, environment)**

Reproducibility означает:

- все аргументы функции известны
- все аргументы контролируются
- функция детерминирована (или максимально близка к этому)

---

# 11. Признаки хорошей reproducibility

- любой разработчик может собрать проект одной командой
- CI дает тот же результат
- timing стабилен между сборками
- нет “магических” GUI-настроек
- build не зависит от предыдущего состояния
- изменения в QoR объяснимы

---

# 12. Признаки плохой reproducibility

- “у меня работает, у тебя нет”
- timing скачет без изменения RTL
- GUI и script дают разные результаты
- невозможно повторить старый build
- нет четкого build entry point
- изменения появляются “сами собой”

---

# 13. Главная инженерная мысль

**Reproducibility — это не свойство Vivado.  
Это свойство твоего build flow.**

Vivado допускает недетерминизм,  
но именно ты решаешь:

- насколько контролируем build
- насколько он описан
- насколько он воспроизводим

---

## Краткое резюме

**Build reproducibility** — это способность повторять FPGA build с одинаковым результатом.  
Она достигается через:

- Tcl-based scripted flow
- фиксированные inputs
- контроль tool version
- отсутствие hidden state
- управление randomness
- сохранение artifacts

Главный принцип:

> **Build должен быть полностью описан кодом и не зависеть от ручных действий.**