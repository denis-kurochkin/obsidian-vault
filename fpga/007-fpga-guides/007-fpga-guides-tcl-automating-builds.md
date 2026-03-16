# Tcl automating builds в Vivado

## Что это такое

**Tcl automating builds** — это автоматизация процесса сборки FPGA-проекта через Tcl-скрипты вместо ручного запуска шагов в GUI Vivado.

Проще говоря, это подход, при котором ты не нажимаешь последовательно:

- создать проект;
    
- добавить исходники;
    
- добавить constraints;
    
- сгенерировать IP;
    
- запустить synthesis;
    
- запустить implementation;
    
- получить bitstream;
    
- выгрузить hardware handoff;

а описываешь все это в Tcl-сценарии и запускаешь одной командой.

Идея очень простая:

> сборка должна быть не набором ручных действий, а повторяемой процедурой.

---

## Зачем это нужно

Автоматизация сборки через Tcl нужна, чтобы:

- исключить ручные ошибки;
    
- одинаково собирать проект на разных машинах;
    
- ускорить повторные сборки;
    
- запускать Vivado в batch mode;
    
- интегрировать сборку в CI/CD;
    
- поддерживать несколько конфигураций проекта;
    
- хранить build flow в Git как текст, а не как набор GUI-действий.

Для командной разработки это почти всегда полезно.

---

## Интуитивная идея

Без автоматизации проект часто существует в виде:

- папки Vivado;
    
- набора ручных настроек;
    
- неявных шагов, которые помнит только один разработчик.

С Tcl automation проект превращается в:

- исходники;
    
- constraints;
    
- IP descriptions;
    
- build scripts;
    
- понятную последовательность шагов.

То есть сборка становится частью кода проекта.

---

## Что обычно автоматизируют

В реальном Vivado flow через Tcl часто автоматизируют не один, а сразу несколько уровней.

### 1. Создание проекта

- `create_project`
    
- задание `part`
    
- задание `board_part` при необходимости
    
- настройка project properties

### 2. Подключение исходников

- `add_files`
    
- добавление RTL
    
- добавление simulation files
    
- добавление constraints
    
- `update_compile_order`

### 3. Настройка top и filesets

- `set_property top ...`
    
- выбор fileset
    
- настройка sim top

### 4. Работа с IP

- добавление `.xci`
    
- генерация IP outputs
    
- export user files
    
- create IP runs

### 5. Сборка

- запуск `synth_1`
    
- ожидание завершения
    
- запуск `impl_1`
    
- выполнение до `write_bitstream`

### 6. Генерация артефактов

- bitstream
    
- `.ltx`
    
- `.hwh`
    
- `.xsa`
    
- отчеты по timing/utilization/power

### 7. Проверки и логирование

- проверка входных файлов
    
- вывод ключевых параметров
    
- анализ статуса runs
    
- явное завершение с ошибкой при сбое

---

## Важная мысль

Автоматизация сборки — это не просто команда `launch_runs`.

Настоящий build automation — это когда скрипт:

- знает, **что** собирать;
    
- знает, **как** это создать;
    
- знает, **в каком порядке** выполнять шаги;
    
- умеет проверять ошибки;
    
- дает воспроизводимый результат.

---

## Типичная структура автоматизированного flow

Обычно хороший flow разделяют на несколько логических частей.

### Уровень 1. Подготовка окружения

- определить `origin_dir`
    
- определить `build_dir`
    
- задать `part`
    
- задать `top`
    
- проверить версию Vivado

### Уровень 2. Создание или открытие проекта

- создать новый проект;
    
- либо удалить старый build dir и пересоздать;
    
- либо открыть существующий project mode проект.

### Уровень 3. Наполнение проекта

- добавить RTL;
    
- добавить XDC;
    
- добавить IP;
    
- установить properties.

### Уровень 4. Запуск сборки

- synthesis;
    
- implementation;
    
- bitstream.

### Уровень 5. Сбор результатов

- скопировать артефакты в понятное место;
    
- вывести путь к результатам;
    
- при необходимости сформировать summary.

---

## Project mode и automation

В `project mode` автоматизация обычно строится вокруг команд:

```
create_project
add_files
set_property
launch_runs
wait_on_run
open_run
```

Это удобно, если:

- команда привыкла к GUI;
    
- нужно иногда открывать проект руками;
    
- используются стандартные project runs;
    
- важна совместимость с привычным flow Vivado.

Пример:

```
create_project my_proj $build_dir -part xc7a100tfgg484-2
add_files $rtl_files
add_files -fileset constrs_1 $xdc_file
set_property top top [current_fileset]
update_compile_order -fileset sources_1
launch_runs synth_1 -jobs 8
wait_on_run synth_1
launch_runs impl_1 -to_step write_bitstream -jobs 8
wait_on_run impl_1
```

---

## Non-project mode и automation

В `non-project mode` автоматизация часто выглядит еще прямолинейнее:

```
read_verilog top.v
read_verilog core.v
read_xdc top.xdc
synth_design -top top -part xc7a100tfgg484-2
opt_design
place_design
route_design
write_bitstream -force top.bit
```

Плюсы:

- меньше скрытого project state;
    
- flow более явный;
    
- удобно для CI и полностью скриптовой сборки.

Минусы:

- не всем привычно;
    
- некоторые команды/интеграции удобнее в project mode.

---

## Что дает автоматизация на практике

### 1. Одна команда вместо ручной последовательности

Например:

vivado -mode batch -source tcl/build.tcl

### 2. Повторяемость

Один и тот же flow можно запускать сколько угодно раз.

### 3. Меньше человеческих ошибок

Не нужно каждый раз вспоминать:

- какой top выбрать;
    
- какой XDC подключить;
    
- какие IP regenerate;
    
- какой run запускать.

### 4. Легче поддерживать разные конфигурации

Например:

- `debug` / `release`
    
- разные FPGA
    
- разные платы
    
- разные top modules

### 5. Проще делать ночные и CI-сборки

Автоматизация особенно ценна, когда сборка идет без GUI.

---

## Что чаще всего автоматизируют плохо

## 1. Все пишут в один огромный Tcl-файл

Когда в одном `build.tcl` смешано все:

- создание проекта;
    
- IP generation;
    
- constraints injection;
    
- synth/impl;
    
- packaging;
    
- debug-параметры;
    
- board-specific ветвления.

Такой скрипт быстро становится хрупким.

Лучше разделять по ролям.

---

## 2. Скрипт зависит от уже существующего состояния

Например:

- предполагает, что проект уже открыт;
    
- ожидает, что IP уже сгенерированы;
    
- рассчитывает на существование `.runs`;
    
- требует, чтобы кто-то до этого вручную нажал `Generate Output Products`.

Это плохая автоматизация, потому что она не самодостаточна.

---

## 3. Нет обработки ошибок

Например, скрипт запускает runs, но не проверяет:

- завершились ли они успешно;
    
- существует ли bitstream;
    
- нашлись ли входные файлы.

Тогда скрипт может формально дойти до конца, но не дать корректного результата.

---

## 4. Жестко зашиты локальные пути

Плохой пример:

set src_dir A:/work/proj/src

Это сразу делает automation непереносимой.

---

## 5. Смешиваются обязательные и временные шаги

Например, в один и тот же build flow без условий включены:

- debug ILA;
    
- экспериментальные constraints;
    
- release packaging.

В результате сложно понять, какой build считается эталонным.

---

## Хорошая архитектура build automation

Обычно полезно разделять flow на несколько Tcl-файлов.

Пример структуры:

```
tcl/
	common.tcl
	create_project.tcl
	add_sources.tcl
	ip.tcl
	constraints.tcl
	build.tcl
	reports.tcl
```

Роли могут быть такими:

- `common.tcl` — общие процедуры;
    
- `create_project.tcl` — создание проекта и базовые свойства;
    
- `add_sources.tcl` — RTL/XDC/filesets;
    
- `ip.tcl` — IP generation;
    
- `constraints.tcl` — условные Tcl constraints;
    
- `build.tcl` — orchestration всего flow;
    
- `reports.tcl` — генерация отчетов.

---

## Основные строительные блоки

## 1. Определение корня проекта

```
set origin_dir [file normalize [file join [file dirname [info script]] ..]]
```

Это один из самых важных элементов переносимого build automation.

---

## 2. Выделение build directory

```
set build_dir [file join $origin_dir build vivado]
file mkdir $build_dir
```

Так становится понятно, где исходники, а где сборочные артефакты.

---

## 3. Проверка обязательных файлов

proc require_file {path} 
{
	if {![file exists $path]} 
	{
		error "Required file not found: $path"
	}
}

---

## 4. Логирование параметров

```
puts "INFO: Origin dir = $origin_dir"

puts "INFO: Build dir = $build_dir"

puts "INFO: Top = $top_name"

puts "INFO: Part = $part_name"
```

---

## 5. Явный запуск runs

```
launch_runs synth_1 -jobs 8

wait_on_run synth_1

launch_runs impl_1 -to_step write_bitstream -jobs 8

wait_on_run impl_1
```

---

## 6. Проверка результата

```
open_run impl_1

set bitfile [get_property BITSTREAM.FILE [current_design]]

if {$bitfile eq "" || ![file exists $bitfile]} 
{
	error "Bitstream was not generated"
}

puts "INFO: Bitstream = $bitfile"
```

Это намного лучше, чем просто предполагать, что все прошло успешно.

---

## 7. Вынос повторяющихся действий в процедуры

```
proc run_and_wait {run_name args} 
{
	launch_runs $run_name {*}$args
	wait_on_run $run_name
}
```

Или даже с проверкой статуса.

---

## Пример минимального build.tcl для project mode

```
set origin_dir [file normalize [file join [file dirname [info script]] ..]]
set build_dir [file join $origin_dir build project]
set part_name xc7a100tfgg484-2
set top_name top
  
proc require_file {path} 
{
	if {![file exists $path]} 
	{
		error "Required file not found: $path"
	}
}
  
puts "INFO: Origin dir = $origin_dir"
puts "INFO: Build dir = $build_dir"
puts "INFO: Part = $part_name"
puts "INFO: Top = $top_name"
  
if {[file exists $build_dir]} 
{
	file delete -force $build_dir
}
file mkdir $build_dir
  
create_project my_proj $build_dir -part $part_name
  
set rtl_files [list \
	[file join $origin_dir src top.v] \
	[file join $origin_dir src core.v] \
	]
  
foreach f $rtl_files 
{
	require_file $f
	add_files $f
}
  
set xdc_file [file join $origin_dir constraints top.xdc]
require_file $xdc_file
add_files -fileset constrs_1 $xdc_file
  
set_property top $top_name [current_fileset]
update_compile_order -fileset sources_1
  
launch_runs synth_1 -jobs 8
wait_on_run synth_1
  
launch_runs impl_1 -to_step write_bitstream -jobs 8
wait_on_run impl_1
  
open_run impl_1
set bitfile [get_property BITSTREAM.FILE [current_design]]

if {$bitfile eq "" || ![file exists $bitfile]} 
{
	error "Bitstream was not generated"
}
  
puts "INFO: Build finished successfully"
puts "INFO: Bitstream = $bitfile"
```

Это уже базовый пример автоматизированной сборки.

---

## Пример common-процедур

```
proc require_file {path} 
{
	if {![file exists $path]} 
	{
		error "Required file not found: $path"
	}
}
 
proc info_msg {msg} 
{
	puts "INFO: $msg"
}

proc fail {msg} 
{
	error "ERROR: $msg"
}
  
proc check_run_succeeded {run_name} 
{
	set status [get_property STATUS [get_runs $run_name]]
	if {![string match "*Complete*" $status]} 
	{
		error "Run $run_name did not complete successfully. Status: $status"
	}
}

```

Такие утилиты быстро окупаются, когда flow становится больше.

---

## Автоматизация IP как часть build flow

Очень часто build automation ломается именно на IP.

Хороший подход:

- хранить `.xci`;
    
- добавлять их в проект Tcl-скриптом;
    
- при необходимости создавать runs;
    
- запускать генерацию автоматически.

Упрощенный пример:

```
set xci [file join $origin_dir ip my_ip my_ip.xci]
require_file $xci
add_files $xci
export_ip_user_files -of_objects [get_files $xci] -no_script -sync -force -quiet
create_ip_run -force [get_files $xci]
launch_runs my_ip_synth_1
wait_on_run my_ip_synth_1
```

Но конкретная схема зависит от того, как именно организованы IP в проекте.

---

## Автоматизация разных конфигураций

Build automation особенно полезна, когда есть варианты сборки.

Например:

- `debug_build`
    
- `release_build`
    
- `board_revA`
    
- `board_revB`
    
- `top=adc_top` или `top=aurora_top`

Тогда скрипт может принимать параметры:

```
set build_type debug
set board_variant revA
```

И условно:

- подключать нужный XDC;
    
- включать ILA;
    
- выбирать top;
    
- выбирать набор исходников.

Главное — чтобы эта логика была прозрачной и не скрывала обязательные вещи.

---

## Batch mode

Один из главных практических смыслов Tcl automation — запуск Vivado без GUI:

vivado -mode batch -source tcl/build.tcl

Это важно, потому что именно batch mode делает сборку:

- удобной для серверов;
    
- пригодной для CI;
    
- одинаковой для всей команды;
    
- независимой от ручного нажатия кнопок.

---

## CI/CD и автоматизация сборки

Когда build flow оформлен в Tcl, его гораздо проще встроить в:

- Jenkins;
    
- GitLab CI;
    
- GitHub Actions;
    
- локальные build scripts.

Тогда pipeline может:

- запускать Vivado batch;
    
- собирать bitstream;
    
- сохранять артефакты;
    
- публиковать отчеты;
    
- падать при timing failure или build error.

То есть Tcl — это естественный интерфейс между FPGA-проектом и автоматизированной системой сборки.

---

## Важное различие: automate build vs automate everything

Не обязательно с первого дня автоматизировать вообще все.

Хороший путь развития такой:

### Этап 1

Автоматизировать базовую сборку:

- create project;
    
- add files;
    
- synth/impl;
    
- bitstream.

### Этап 2

Добавить:

- IP generation;
    
- параметры build variants;
    
- отчеты;
    
- packaging.

### Этап 3

Добавить:

- CI integration;
    
- regression checks;
    
- quality gates.

Так проще, чем сразу писать гигантский универсальный framework.

---

## Частые ошибки

### Ошибка 1. Скрипт можно запустить только из одной конкретной директории

Это лечится вычислением `origin_dir`.

### Ошибка 2. Скрипт не удаляет старое состояние и использует устаревшие артефакты

Нужно явно определить стратегию:

- либо clean build;
    
- либо incremental flow;
    
- но не оставлять это неявным.

### Ошибка 3. Скрипт не проверяет статус runs

Это одна из самых неприятных ошибок.

### Ошибка 4. В логах ничего не понятно

Минимальное логирование очень помогает.

### Ошибка 5. Параметры build variants разбросаны по коду

Лучше централизовать их в одном месте.

### Ошибка 6. Скрипт зависит от GUI-настроек, которые нигде не описаны

Это делает automation неполной.

---

## Рекомендуемые правила

1. Делать build flow самодостаточным.
    
2. Все пути вычислять относительно корня проекта.
    
3. Разделять create-project, add-sources, ip, build, reports.
    
4. Проверять обязательные входные файлы.
    
5. Проверять успешность synth/impl.
    
6. Явно логировать ключевые параметры.
    
7. Явно отделять debug и release flow.
    
8. Держать build artifacts вне исходников.
    
9. Не полагаться на случайное состояние `.runs`.
    
10. Делать batch запуск основным сценарием, а не дополнительным.

---

## Когда автоматизация особенно полезна

Особенно оправдано делать Tcl build automation, если:

- проект живет долго;
    
- в команде больше одного разработчика;
    
- есть несколько вариантов сборки;
    
- используются IP и BD;
    
- нужно регулярно выпускать bitstream;
    
- планируется CI/CD.

---

## Когда можно начать с минимального варианта

Если проект небольшой, можно начать с простого `build.tcl`, который:

- создает проект;
    
- добавляет RTL и XDC;
    
- задает top;
    
- запускает synth и impl;
    
- проверяет наличие bitstream.

Даже такой минимальный уровень уже сильно лучше полностью ручного flow.

---

## Главное, что нужно запомнить

**Tcl automating builds** в Vivado — это превращение сборки из ручной последовательности GUI-действий в формализованный, повторяемый, проверяемый Tcl flow.

Ценность этого подхода:

- меньше ручных ошибок;
    
- одинаковая сборка у всех;
    
- удобный batch mode;
    
- база для reproducibility и CI/CD.
    

Главный риск:

- написать слишком хрупкий скрипт, который зависит от скрытого состояния и плохо обрабатывает ошибки.
    

Поэтому хороший build automation должен быть:

- явным;
    
- переносимым;
    
- проверяемым;
    
- логируемым;
    
- самодостаточным.
    

---

## Краткая памятка

### Что это

Автоматизация сборки Vivado-проекта через Tcl.

### Что обычно автоматизируют

- create_project
    
- add_files
    
- top/filesets
    
- IP generation
    
- synth/impl
    
- bitstream
    
- reports
    

### Плюсы

- повторяемость
    
- меньше ручных ошибок
    
- batch mode
    
- CI/CD
    

### Риски

- хрупкие скрипты
    
- зависимость от старого project state
    
- отсутствие проверки ошибок
    

### Золотое правило

**build script должен уметь собрать проект с нуля и явно сообщить, если что-то пошло не так.**

---

## Мини-шаблон для повседневного использования

set origin_dir [file normalize [file join [file dirname [info script]] ..]]

set build_dir [file join $origin_dir build project]

set part_name xc7a100tfgg484-2

set top_name top

  

proc require_file {path} {

if {![file exists $path]} {

error "Required file not found: $path"

}

}

  

proc check_run_succeeded {run_name} {

set status [get_property STATUS [get_runs $run_name]]

if {![string match "*Complete*" $status]} {

error "Run $run_name failed. Status: $status"

}

}

  

if {[file exists $build_dir]} {

file delete -force $build_dir

}

file mkdir $build_dir

  

create_project my_proj $build_dir -part $part_name

  

set rtl_files [list \

[file join $origin_dir src top.v] \

[file join $origin_dir src core.v] \

]

  

foreach f $rtl_files {

require_file $f

add_files $f

}

  

set xdc_file [file join $origin_dir constraints top.xdc]

require_file $xdc_file

add_files -fileset constrs_1 $xdc_file

  

set_property top $top_name [current_fileset]

update_compile_order -fileset sources_1

  

launch_runs synth_1 -jobs 8

wait_on_run synth_1

check_run_succeeded synth_1

  

launch_runs impl_1 -to_step write_bitstream -jobs 8

wait_on_run impl_1

check_run_succeeded impl_1

  

open_run impl_1

set bitfile [get_property BITSTREAM.FILE [current_design]]

if {$bitfile eq "" || ![file exists $bitfile]} {

error "Bitstream was not generated"

}

  

puts "INFO: Build successful"

puts "INFO: Bitstream = $bitfile"

Это хороший стартовый шаблон, который уже можно адаптировать под реальный проект.