## Basic scripting for build automation в Vivado: небольшая заметка

**Build automation** в контексте Vivado — это запуск типового FPGA-потока не руками через GUI, а через **Tcl-скрипты** и, при необходимости, оболочку вроде `Makefile`, `bash` или CI. В Vivado это естественный путь: и **Project Mode**, и **Non-Project Mode** поддерживают Tcl и batch-запуск, а для non-project mode Tcl вообще является основным способом работы.

### Зачем это нужно

Скриптовая сборка дает несколько практических плюсов:

- повторяемость сборки;
- меньше ручных ошибок;
- удобнее хранить проект в git;
- проще делать nightly build, regression и CI;
- легче воспроизводить сборку на другой машине или для другой платы.

Vivado прямо поддерживает запуск в batch-режиме через команду вида `vivado -mode batch -source script.tcl`, а из IDE можно сгенерировать Tcl-скрипт, который recreates весь проект.

### Два базовых подхода

#### 1. Project Mode automation

Подходит, когда ты работаешь с обычным Vivado project и хочешь автоматизировать стандартные run’ы.  
Типичный сценарий:

- создать/открыть проект;
- добавить RTL, XDC, IP;
- запустить synthesis/implementation;
- дождаться завершения;
- выгрузить bitstream и отчеты.

Для project mode обычно используют команды вроде `launch_runs` и `wait_on_runs`; AMD отдельно предупреждает, что не стоит смешивать project-based `launch_runs` с “атомарными” non-project командами вроде `synth_design` и `opt_design` в одном и том же flow. `wait_on_runs` ждет завершения run, а результат потом проверяют по свойствам run.

#### 2. Non-Project Mode automation

Это более “инженерный” и обычно более удобный для build automation путь.  
В non-project mode ты явно задаешь шаги потока Tcl-командами:

- `read_verilog` / `read_vhdl`
- `read_xdc`
- `read_ip` / `read_bd`
- `synth_design`
- `opt_design`
- `place_design`
- `route_design`
- `write_bitstream`
- `report_*`
- `write_checkpoint`

AMD пишет, что в non-project mode дизайн компилируется через `read_*` команды и обрабатывается в памяти; проект на диск не сохраняется как обычный `.xpr`, а checkpoints и отчеты можно сохранять на нужных этапах.

### Почему non-project mode часто удобнее

Для автоматизации он обычно лучше, потому что:

- flow полностью прозрачен;
- проще контролировать порядок шагов;
- легче встроить в git/CI;
- проще параметризовать сборки по top, part, board, define, strategy;
- checkpoints можно сохранять на ключевых стадиях.

AMD прямо показывает пример non-project Tcl script и рекомендует запускать его в Tcl/batch режиме.

### Что обычно кладут в базовый build script

Минимально полезный Tcl-скрипт обычно делает такие вещи:

- задает `part` и top-level;
- читает исходники и constraints;
- запускает synthesis и implementation;
- пишет отчеты по timing/utilization;
- генерирует `.bit`;
- при желании сохраняет `.dcp` после synth/place/route.

`write_bitstream` работает для уже implemented design, а `write_checkpoint` в non-project flow удобно использовать как точки восстановления и анализа.

### Хорошая практическая структура

Обычно удобно делить автоматизацию на три слоя:

- **Tcl-скрипт Vivado** — знает FPGA-flow;
- **оболочка** (`Makefile`/`bash`) — знает цели вроде `build`, `clean`, `reports`;
- **CI** — знает, когда это запускать.

AMD в материалах по revision control и tutorial-потокам отдельно показывает использование Tcl вместе с Makefile для сборочных целей.

### Что полезно автоматизировать в первую очередь

Для базового уровня обычно достаточно автоматизировать:

- полную сборку bitstream;
- генерацию timing/utilization reports;
- отдельную цель synth-only;
- clean/rebuild;
- сборку для нескольких top/part-конфигураций.

Это уже сильно лучше ручного кликанья.

### Практический совет

Если проект уже собран в GUI, хороший первый шаг — **не переписывать все с нуля**, а сначала:

- сделать `Write Tcl` из Vivado;
- посмотреть, как Vivado описывает проект;
- потом постепенно упростить скрипт до читаемой и reproducible версии.

Vivado поддерживает генерацию Tcl-скрипта для воспроизведения проекта именно для таких сценариев.

---

## Краткое резюме

**Basic scripting for build automation** в Vivado — это перевод сборки FPGA из ручного GUI-flow в **Tcl + batch flow**.  
Для простых задач подходит project mode с `launch_runs`, а для более управляемой и reproducible автоматизации обычно удобнее **non-project mode**, где ты явно управляешь чтением исходников, synthesis, implementation, отчетами и bitstream generation.