## Scripted runs в Vivado:

**Scripted runs** в Vivado — это запуск build flow через **Tcl scripts**, а не руками через GUI.  
В FPGA-практике это означает, что synthesis, implementation, reports, checkpoints и bitstream generation выполняются **как код**, а не как последовательность кликов. Vivado поддерживает два основных подхода: **Project Mode** и **Non-Project Mode**. В Project Mode run’ы управляются объектами `run` и командами вроде `launch_runs` и `wait_on_runs`. В Non-Project Mode flow собирается вручную из “atomic” команд вроде `read_*`, `synth_design`, `opt_design`, `place_design`, `route_design` и `write_bitstream`.

---

### 1. Зачем вообще нужны scripted runs

На маленьком проекте GUI-flow кажется удобным. Но как только появляется хотя бы один из следующих факторов, scripted runs становятся почти обязательными:

- один и тот же build нужно повторять много раз;
- проект должен собираться одинаково на разных машинах;
- нужно хранить flow в git;
- хочется запускать synth-only, impl-only, report-only, nightly build;
- нужно собирать несколько variants одного дизайна;
- хочется встроить Vivado в Makefile, CI или regression flow.

В Vivado batch-запуск делается через `vivado -mode batch -source <script.tcl>`, а сам flow может быть либо project-based, либо полностью Tcl-driven в non-project режиме. AMD отдельно описывает Tcl script-based compilation style как штатный способ работы, особенно для Non-Project Mode.

---

## 2. Два разных мира: Project Mode и Non-Project Mode

Это главное различие, которое нужно понять с самого начала.

### Project Mode

В **Project Mode** Vivado создает и хранит `.xpr` project, управляет sources, runs, strategies, run-state и связанными служебными файлами. Для synthesis и implementation используются run objects, которые запускаются через `launch_runs`. Чтобы дождаться завершения, обычно применяют `wait_on_runs`. Команда `create_run` позволяет явно создавать synthesis или implementation run и потом настраивать их свойства через `set_property`.

### Non-Project Mode

В **Non-Project Mode** ты сам управляешь тем, какие files читать и в каком порядке, а также сам вызываешь шаги потока. Vivado читает sources через `read_*` команды, держит дизайн в памяти и дальше выполняет flow вручную через `synth_design`, `opt_design`, `place_design`, `phys_opt_design`, `route_design`, `write_bitstream`, `report_*`, `write_checkpoint`. AMD прямо отмечает, что в этом режиме source files компилируются в том порядке, в котором перечислены `read_*` команды в Tcl script.

---

## 3. Что такое “run” в Project Mode

В Project Mode слово **run** означает не просто “запуск сборки”, а конкретный объект Vivado, который хранит:

- тип run: synthesis или implementation;
- strategy;
- target step;
- status / progress;
- связь с parent run;
- output artifacts.

Именно поэтому в Project Mode ты обычно не вызываешь `synth_design` напрямую. Вместо этого создается или используется существующий run, а потом запускается `launch_runs`. AMD прямо пишет, что в Project Mode synthesis should be launched from an existing synthesis run, а для Non-Project Mode synthesis запускается напрямую через `synth_design`.

Это важный ментальный переход:

- в **Project Mode** ты управляешь **run objects**;
- в **Non-Project Mode** ты управляешь **steps**.

---

## 4. Типовой scripted run в Project Mode

Логика здесь обычно такая:

1. создать или открыть project;
2. добавить sources / constraints / IP;
3. при необходимости настроить run properties;
4. запустить `launch_runs synth_1`;
5. дождаться `wait_on_runs synth_1`;
6. запустить `launch_runs impl_1 -to_step write_bitstream`;
7. дождаться завершения;
8. собрать artifacts и reports.

AMD в tutorial по project-based flow прямо показывает pattern вида:

launch_runs synth_1  
wait_on_run synth_1  
launch_runs impl_1 -to_step write_bitstream

и отдельно отмечает, что implementation run зависит от завершения synthesis run. Команда `wait_on_runs` блокирует выполнение Tcl до завершения run либо с success, либо с error, либо до timeout; при этом сам факт окончания run еще не говорит, что run завершился успешно, поэтому статус обычно потом проверяют через свойства run.

### Что удобно в Project Mode scripted runs

- Vivado сам управляет деталями run-state;
- удобно использовать стандартные `synth_1`, `impl_1`;
- проще автоматизировать flow, похожий на GUI;
- удобно, если команда и так живет в project-centric процессе.

### Где границы Project Mode

Если нужен более прозрачный, reproducible и тонко параметризуемый flow, Project Mode иногда начинает мешать из-за своей “managed” природы. Тогда часто переходят к Non-Project Mode. Это особенно типично для CI, reproducible builds и build systems вокруг Tcl/Make.

---

## 5. Типовой scripted run в Non-Project Mode

Здесь идея другая: вместо больших run objects ты сам явно описываешь pipeline.

Обычно flow выглядит так:

1. `read_verilog` / `read_vhdl` / `read_xdc` / `read_ip` / `read_bd`
2. `synth_design`
3. `opt_design`
4. `place_design`
5. `phys_opt_design` при необходимости
6. `route_design`
7. `write_checkpoint`
8. `report_timing_summary`, `report_utilization` и другие `report_*`
9. `write_bitstream`

AMD прямо перечисляет Non-Project Mode flow через эти команды и подчеркивает, что `place_design` и `route_design` в этом режиме вызываются вручную, а не через `launch_runs`.

### Почему это нравится инженерам

Потому что здесь flow прозрачен.  
Ты видишь не “запустить impl”, а конкретно:

- что было прочитано;
- в каком порядке;
- какие steps выполнялись;
- где сохраняются checkpoints;
- какие reports генерируются;
- что считается output’ом.

Именно поэтому Non-Project Mode очень часто лучше подходит для серьезной automation.

---

## 6. Почему нельзя смешивать два подхода без понимания модели

Это одна из самых частых путаниц.

`launch_runs` относится к **Project Mode** и работает с заранее определенными run objects.  
`synth_design`, `opt_design`, `place_design`, `route_design` — это основа **Non-Project Mode**. AMD прямо пишет, что в Project Mode synthesis запускается из synthesis run, а в Non-Project Mode — напрямую через `synth_design`; аналогично implementation steps в non-project flow выполняются вручную.

Практический вывод:

**не надо строить flow, где часть логики run-centric, а часть — atomic-step-centric, если ты четко не понимаешь, зачем это делаешь.**

В нормальной инженерной практике лучше выбрать одну модель на конкретный build flow:

- либо Project Mode scripted runs,
- либо Non-Project Mode scripted runs.

---

## 7. Что обычно включает хороший scripted run

Полноценный scripted run — это не только “собрать `.bit`”.  
Обычно он должен делать минимум пять вещей.

### 7.1 Deterministic input setup

Скрипт должен явно задавать:

- `part`;
- top module;
- source list;
- XDC list;
- IP / BD sources;
- compile order при необходимости.

Для Non-Project Mode это особенно важно, потому что именно Tcl script определяет порядок `read_*` команд, а значит и build context.

### 7.2 Step control

Нужно понимать, где заканчивается synth, где начинается impl, и какие steps запускаются.  
В Project Mode это `launch_runs` с target step’ами.  
В Non-Project Mode это прямой вызов commands вроде `opt_design`, `place_design`, `route_design`.

### 7.3 Status handling

Scripted run должен не просто запуститься, а уметь:

- ждать завершения;
- проверять success / failure;
- корректно завершаться с ошибкой при bad outcome.

`wait_on_runs` только ждет run termination; результат run надо дополнительно проверять через свойства run. Это важная деталь из UG835.

### 7.4 Reports

Нужны как минимум:

- `report_timing_summary`
- `report_utilization`
- иногда `report_clock_utilization`, `report_power`, `report_drc`

В Non-Project Mode AMD прямо рекомендует вручную генерировать reports внутри Tcl script; в example scripts checkpoints и reports — важная часть flow.

### 7.5 Artifacts

Обычно scripted run должен генерировать:

- `.bit`
- `.dcp`
- `.rpt`
- иногда `.ltx`, `.xsa`, netlists, logs

---

## 8. Checkpoints как часть scripted runs

Одна из самых полезных привычек в scripted flows — использовать **design checkpoints**.

В Non-Project Mode AMD пример script прямо показывает использование checkpoints для сохранения database state на разных стадиях flow. Это очень полезно, потому что позволяет:

- не пересобирать все с нуля при анализе;
- открыть post-synth или post-route состояние;
- отделить synthesis debug от routing debug;
- быстрее разбираться с QoR issues.

Практически полезно сохранять как минимум:

- post-synth checkpoint;
- post-place checkpoint;
- post-route checkpoint.

Для scripted runs это почти всегда хороший стандарт.

---

## 9. Scripted runs и reproducibility

Одна из самых сильных причин писать scripted runs — **reproducibility**.

GUI-flow удобен для исследования, но плохо документирует полную build-intent в одном месте. Tcl script, наоборот, превращает build в явное описание:

- какие файлы читаются;
- в каком порядке;
- с какими settings;
- какие steps выполняются;
- какие outputs считаются официальными.

Vivado также поддерживает создание Tcl script для recreation проекта, то есть GUI-проект можно экспортировать в Tcl и потом сделать из этого более чистый scripted flow. Это очень полезный путь миграции от “ручного” процесса к automation.

---

## 10. Scripted runs и build system вокруг Vivado

Обычно сам Tcl script — это только внутренний слой.  
Вокруг него часто строят внешний automation layer:

- `Makefile`
- `bash` wrapper
- Python launcher
- CI job

Роли обычно делятся так:

### Tcl

Знает Vivado-native действия:

- create/open/read
- synth/impl
- reports
- bitstream
- checkpoints

### External wrapper

Знает build targets:

- `make synth`
- `make impl`
- `make clean`
- `make reports`
- `make bit`

### CI

Знает policy:

- когда запускать;
- какие artifacts сохранять;
- какие thresholds считать fail;
- как публиковать logs.

Именно поэтому scripted runs — это основа build automation, а не вся automation целиком.

---

## 11. Что делает scripted run “хорошим”, а что “хрупким”

### Хороший scripted run

- явно задает inputs;
- не зависит от случайного состояния GUI;
- логично разделяет stages;
- сохраняет reports и checkpoints;
- валится с error code при провале;
- легко читается и параметризуется.

### Хрупкий scripted run

- полагается на скрытое состояние проекта;
- молча игнорирует ошибки;
- не проверяет статус runs;
- не сохраняет ключевые artifacts;
- смешивает Project Mode и Non-Project Mode без ясной модели;
- читает sources неявно или в нестабильном порядке.

Особенно важен последний пункт: в Non-Project Mode порядок `read_*` команд важен, и AMD это отдельно подчеркивает.

---

## 12. Практический пример мышления: когда выбирать какой стиль

### Выбирай Project Mode scripted runs, если:

- команда уже живет в `.xpr` project;
- нужен flow, близкий к GUI;
- хочется автоматизировать уже существующий project;
- build infrastructure пока простая.

### Выбирай Non-Project Mode scripted runs, если:

- хочешь более reproducible flow;
- хочешь явный step-by-step Tcl;
- нужен cleaner integration с git/CI;
- проект собирается как code-defined pipeline.

Оба режима официально поддерживаются Vivado. Но по инженерной культуре **Non-Project Mode** обычно лучше подходит для серьезной build automation, а **Project Mode scripted runs** — для мягкого перехода от GUI к automation.

---

## 13. Частые ошибки при scripted runs

### 13.1 Думать, что `wait_on_runs` сам проверяет success

Нет. Он ждет окончания run, но результат run нужно смотреть отдельно. AMD это пишет явно.

### 13.2 Смешивать `launch_runs` и atomic flow без понимания контекста

Это часто порождает script, который формально работает, но концептуально грязный и плохо сопровождается.

### 13.3 Делать scripted run только для bitstream, но без reports

Тогда automation есть, а build observability нет.

### 13.4 Не сохранять checkpoints

Потом любой анализ timing/QoR требует полного rerun.

### 13.5 Оставлять source handling неявным

Для reproducible automation это одна из самых неприятных ошибок.

---

## 14. Минимальный mental model

Полезно держать в голове такую очень простую схему.

### Project Mode

`create/open project -> create/configure runs -> launch_runs -> wait_on_runs -> collect artifacts`

### Non-Project Mode

`read_* -> synth_design -> opt_design -> place_design -> route_design -> reports -> write_bitstream`

Если держать эту разницу четко, scripted runs становятся понятными и предсказуемыми.

---

## 15. Главная инженерная мысль

**Scripted runs — это перевод Vivado build из режима “interactive tool” в режим “deterministic build process”.**

Именно это делает их настолько важными для FPGA engineering.  
Они дают:

- repeatability;
- reviewability;
- parameterization;
- CI-friendliness;
- меньше ручных ошибок;
- лучшее понимание самого flow.

---

## Краткое резюме

**Scripted runs** в Vivado — это Tcl-driven запуск synthesis/implementation flow.  
Есть две основные модели:

- **Project Mode**: работа через run objects, `create_run`, `launch_runs`, `wait_on_runs`;
- **Non-Project Mode**: явное управление steps через `read_*`, `synth_design`, `opt_design`, `place_design`, `route_design`, `write_bitstream`.

Главный практический вывод:

**scripted run должен быть не просто “способом нажать Build из Tcl”, а полным, воспроизводимым описанием build flow, включая status handling, reports и artifacts.**