# Tcl constraint injection в Vivado

## Что это такое

**Tcl constraint injection** — это способ **добавлять, изменять или условно применять ограничения (constraints)** к проекту Vivado **через Tcl-скрипты**, а не только через статический `.xdc` файл.

Проще говоря:
- обычный путь — написать ограничения вручную в `*.xdc`;
- Tcl injection — сформировать или подмешать эти ограничения **программно** во время сборки.

Это особенно полезно, когда:
- проект собирается автоматически;
- есть несколько конфигураций платы или FPGA;
- часть ограничений зависит от состава дизайна;
- IP/иерархия могут меняться;
- нужно аккуратно включать или отключать constraints по условию.
---
## Интуитивная идея

Обычный `.xdc` — это как фиксированный текстовый файл.
**Tcl injection** — это когда ты говоришь Vivado:

> "Перед синтезом или implementation проверь текущий дизайн, найди нужные объекты и только потом создай нужные ограничения."

То есть constraint становится не просто строкой в XDC, а **логикой принятия решения**:
- существует ли нужный cell;
- есть ли нужный pin;
- какой вариант top-модуля сейчас собирается;
- активен ли определенный IP;
- какая ревизия платы используется.
---

## Где это используется

### 1. Условное применение ограничений

Например, ILA может быть включена не во всех сборках.

Если написать в статическом XDC:

```
set_false_path -to [get_pins -of_objects [get_cells ila/inst/ila_core_inst] -filter {REF_PIN_NAME =~ DATA_I*}]
```

а ILA в проекте нет, появятся предупреждения вида:

- `No cells matched ...`
- `No pins matched ...`
- `One or more constraints failed evaluation`

Через Tcl injection можно сначала проверить наличие объекта:


```
set ila_cells [get_cells -quiet ila/inst/ila_core_inst]
if {[llength $ila_cells] > 0} 
{
	set ila_pins [get_pins -quiet -of_objects $ila_cells -filter {REF_PIN_NAME =~ DATA_I* || REF_PIN_NAME =~ TRIGGER_I* || REF_PIN_NAME == TRIG_IN_I}]
	if {[llength $ila_pins] > 0} 
	{
		set_false_path -to $ila_pins
	}
}
```

Это и есть типичный пример **инъекции constraints через Tcl**.

---

### 2. Генерация constraints под разные платы

Один и тот же RTL может собираться под несколько плат.

Например:
- у одной платы сигнал `clk_in` на пине `E3`;
- у другой — на `F7`.

Вместо нескольких почти одинаковых XDC можно сделать Tcl-логику:

```
if {$board_variant eq "revA"} 
{
	set_property PACKAGE_PIN E3 [get_ports clk_in]
} elseif {$board_variant eq "revB"} 
{
	set_property PACKAGE_PIN F7 [get_ports clk_in]
}
set_property IOSTANDARD LVCMOS33 [get_ports clk_in]
```

---
### 3. Подстройка ограничений под иерархию

Если путь к экземпляру IP или блока зависит от конфигурации, статические пути в XDC легко ломаются.

Tcl позволяет:

- искать объекты по шаблону;
- проверять количество найденных объектов;
- применять constraint только если объект найден однозначно.

Пример:

```
set regs [get_cells -hier -quiet *cdc_fifo*/*sync_reg*]
if {[llength $regs] > 0} 
{
	set_property ASYNC_REG TRUE $regs
}
```

---

### 4. Добавление ограничений в автоматизированном flow

Когда проект собирается не руками в GUI, а через `build.tcl`, CI/CD или batch mode, часто удобно:

- открыть проект;
- выбрать top;
- подключить исходники;
- на основе параметров подмешать нужные constraints;
- запустить synth/impl.

То есть constraints становятся частью **сценария сборки**, а не только частью исходников.

---

## Какие бывают формы constraint injection

### Вариант 1. Прямое выполнение constraint-команд в Tcl

Например:

```
set_property PACKAGE_PIN E3 [get_ports clk_in]

create_clock -period 10.000 [get_ports clk_in]

set_false_path -from [get_cells -hier *src_ff*] -to [get_cells -hier *dst_ff*]
```

Это самый прямой способ: Tcl-скрипт сам вызывает constraint-команды.

---

### Вариант 2. Генерация временного XDC файла

Скрипт формирует `.xdc` и добавляет его в проект:

```
set fp [open generated_constraints.xdc w]

puts $fp "create_clock -period 10.000 [get_ports clk_in]"

puts $fp "set_property PACKAGE_PIN E3 [get_ports clk_in]"

close $fp

read_xdc generated_constraints.xdc
```

Полезно, когда нужно:

- сохранить итоговые constraints в файл;
- отлаживать, что именно было сгенерировано;
- передать generated XDC дальше по flow.

---
### Вариант 3. Условное подключение разных XDC

Иногда инъекция — это не генерация отдельных команд, а выбор нужного XDC:

```
if {$debug_build} 
{
	read_xdc debug_constraints.xdc
} 
else 
{
	read_xdc release_constraints.xdc
}
```

Это тоже часто относят к Tcl-based constraint injection.

---
## Когда это действительно полезно

Используй Tcl constraint injection, если:

### Подходит

- проект имеет **несколько конфигураций**;
- есть **debug/release** сборки;
- блоки типа **ILA/VIO/Debug Hub** присутствуют не всегда;
- constraints зависят от **реально существующей иерархии**;
- проект собирается в **batch mode**;
- нужно уменьшить количество ложных warning-ов из-за отсутствующих объектов.

### Не обязательно

- маленький проект;
- один top;
- одна плата;
- ограничения статичны и редко меняются.

В таком случае обычный аккуратный XDC обычно лучше и проще.

---

## Основные плюсы

### 1. Гибкость

Можно применять ограничения только тогда, когда они действительно нужны.

### 2. Меньше ложных warning-ов

Если сначала проверять наличие cell/pin/port, не будет сообщений типа `No cells matched`.

### 3. Удобство автоматизации

Отлично ложится в `build.tcl`, CI и параметризованные сборки.

### 4. Повторное использование

Один и тот же flow можно использовать для нескольких ревизий платы или вариантов прошивки.

### 5. Больше контроля

Можно логировать, какие constraints реально были применены.

---

## Основные минусы

### 1. Сложнее сопровождать

Статический XDC обычно читается проще, чем комбинация XDC + Tcl-логики.

### 2. Легко скрыть проблему

Если скрипт молча "не нашел" нужный объект и просто не применил constraint, можно пропустить реальную ошибку.

Например, путь к экземпляру изменился из-за рефакторинга, а ты думал, что constraint все еще действует.

### 3. Повышается зависимость от имен и иерархии

Если поиск идет по шаблонам, при изменении имен логика может сломаться.

### 4. Сложнее отладка

Иногда непонятно:

- constraint не применился,
    
- объект не был найден,
    
- или он был оптимизирован.
    

---

## Очень важный принцип

**Не превращай Tcl injection в способ замаскировать ошибки.**

Плохой подход:

- скрипт ничего не нашел;
    
- промолчал;
    
- сборка прошла;
    
- timing/CDC/debug constraints фактически не применились.
    

Хороший подход:

- для необязательных объектов — мягкая проверка;
    
- для обязательных объектов — явная ошибка.
    

Пример:

set clk_port [get_ports -quiet clk_in]

if {[llength $clk_port] == 0} {

error "Required port clk_in not found"

}

create_clock -period 10.000 $clk_port

То есть:

- **optional object** → `-quiet` + условие;
    
- **required object** → если не найден, лучше остановить сборку.
    

---

## Практическая классификация: optional vs required

### Optional constraints

Подходят для:

- ILA/VIO;
    
- debug-only false paths;
    
- отладочных clock groups;
    
- ограничений для временных или опциональных IP.
    

### Required constraints

Подходят для:

- основные clocks;
    
- pin assignment внешних интерфейсов;
    
- IOSTANDARD;
    
- обязательные timing exceptions;
    
- критичные CDC-свойства.
    

Для required лучше не делать молчаливый пропуск.

---

## Типовые приемы

## 1. Проверка существования объекта

set objs [get_cells -quiet -hier *my_block*]

if {[llength $objs] > 0} {

# применяем constraint

}

---

## 2. Проверка, что найден ровно один объект

set clk_port [get_ports -quiet clk_in]

if {[llength $clk_port] != 1} {

error "Expected exactly one port clk_in, found [llength $clk_port]"

}

Это полезно, если неоднозначность опаснее, чем отсутствие.

---

## 3. Логирование действий

puts "INFO: Applying debug false path constraints"

puts "INFO: Found [llength $ila_pins] ILA pins"

Очень помогает при разборе логов batch-сборки.

---

## 4. Вынос логики в процедуры

proc apply_debug_constraints {} {

set ila_cells [get_cells -quiet ila/inst/ila_core_inst]

if {[llength $ila_cells] == 0} {

puts "INFO: ILA not found, skipping debug constraints"

return

}

  

set ila_pins [get_pins -quiet -of_objects $ila_cells -filter {REF_PIN_NAME =~ DATA_I* || REF_PIN_NAME =~ TRIGGER_I*}]

if {[llength $ila_pins] > 0} {

set_false_path -to $ila_pins

puts "INFO: Applied false path to [llength $ila_pins] ILA pins"

}

}

  

apply_debug_constraints

Так скрипт становится читабельнее.

---

## 5. Разделение по этапам flow

Нужно помнить: некоторые объекты доступны только на определенной стадии.

Например:

- порты доступны рано;
    
- часть иерархии может меняться после synth;
    
- некоторые net/pin/cell-имена удобнее искать уже после synthesis.
    

Поэтому важно понимать, **в каком контексте исполняется Tcl**:

- до synth;
    
- после synth;
    
- в implementation;
    
- внутри project mode или non-project mode.
    

---

## Связь с XDC

Важно: XDC сам по себе — это по сути тоже Tcl-подобный формат constraint-команд.

Но когда говорят **Tcl constraint injection**, обычно имеют в виду, что:

- ограничения не просто лежат в статическом `.xdc`;
    
- они **добавляются динамически** из управляющего Tcl-скрипта.
    

То есть разница не в синтаксисе команд, а в **способе их подачи в flow**.

---

## Типичный сценарий в Vivado

Пример упрощенного flow:

read_verilog top.v

read_xdc base.xdc

  

if {$debug_build} {

source debug_constraints.tcl

}

  

synth_design -top top -part xczu3eg-sbva484-1-e

opt_design

place_design

route_design

Где `debug_constraints.tcl` может содержать проверку существования ILA и добавление `set_false_path` только при необходимости.

---

## Пример хорошего практического шаблона

proc safe_set_false_path_to_ila {} {

set ila_cells [get_cells -quiet -hier *ila*]

if {[llength $ila_cells] == 0} {

puts "INFO: No ILA cells found, skipping false path injection"

return

}

  

set ila_pins [get_pins -quiet -of_objects $ila_cells \

-filter {REF_PIN_NAME =~ DATA_I* || REF_PIN_NAME =~ TRIGGER_I* || REF_PIN_NAME == TRIG_IN_I}]

  

if {[llength $ila_pins] == 0} {

puts "INFO: ILA cells found, but no matching pins found"

return

}

  

set_false_path -to $ila_pins

puts "INFO: Applied false path to [llength $ila_pins] ILA pins"

}

  

safe_set_false_path_to_ila

Почему это хороший шаблон:

- нет лишних warning-ов;
    
- есть логирование;
    
- optional объект обрабатывается мягко;
    
- поведение предсказуемо.
    

---

## Где чаще всего делают ошибки

### Ошибка 1. Используют слишком хрупкие пути

Например, жестко прошивают полный иерархический путь, который меняется после любого рефакторинга.

### Ошибка 2. Слишком широкие шаблоны

`get_cells -hier *ila*` может найти больше объектов, чем ожидалось.

### Ошибка 3. Молча игнорируют отсутствие обязательного объекта

Это опаснее warning-а.

### Ошибка 4. Не понимают стадию, на которой вызывается скрипт

Команда может выполняться до появления нужных объектов.

### Ошибка 5. Путают debug constraints и функциональные constraints

Например, временно добавили exception для отладки, а потом он остался в release-сборке.

---

## Рекомендации по стилю

1. **Базовые обязательные constraints** держать в обычном XDC.
    
2. **Условные/отладочные/вариантные constraints** выносить в Tcl.
    
3. Для optional-объектов использовать:
    
    - `-quiet`;
        
    - `llength`;
        
    - `puts` для лога.
        
4. Для required-объектов использовать `error`, если объект не найден.
    
5. Не злоупотреблять поиском по слишком широким wildcard.
    
6. Делать скрипты идемпотентными и понятными по логам.
    
7. Явно разделять:
    
    - `base constraints`
        
    - `board constraints`
        
    - `debug constraints`
        
    - `generated constraints`
        

---

## Простая рекомендуемая структура проекта

constraints/

base.xdc

board_revA.xdc

board_revB.xdc

debug_constraints.tcl

generated/

build/

build.tcl

Идея:

- `base.xdc` — обязательная база;
    
- `board_revA.xdc`, `board_revB.xdc` — плата-зависимые статические ограничения;
    
- `debug_constraints.tcl` — условная инъекция;
    
- `generated/` — временно сгенерированные XDC при необходимости.
    

---

## Когда лучше не использовать Tcl injection

Лучше остаться на обычном XDC, если:

- проект небольшой и стабильный;
    
- нет вариантов конфигурации;
    
- constraints статичны;
    
- команде важно максимально простое сопровождение.
    

Иначе можно получить "умный", но трудно читаемый flow.

---

## Главное, что нужно запомнить

**Tcl constraint injection** — это не отдельный специальный механизм Vivado, а **подход**:

> constraints применяются динамически через Tcl-логику, а не только как фиксированный текст в XDC.

Его основная ценность:

- условное применение ограничений;
    
- защита от ложных warning-ов;
    
- удобство автоматизированной сборки;
    
- поддержка нескольких конфигураций проекта.
    

Его главный риск:

- можно незаметно не применить важный constraint и не заметить этого.
    

Поэтому правило такое:

**обязательные constraints — жестко контролировать;**  
**опциональные constraints — применять безопасно и с логированием.**

---

## Краткая памятка

### Что это

Динамическое добавление constraints через Tcl.

### Когда полезно

- debug/release сборки;
    
- опциональная ILA;
    
- разные платы;
    
- batch/CI сборка;
    
- условные timing exceptions.
    

### Плюсы

- гибкость;
    
- меньше ложных warning-ов;
    
- удобно автоматизировать.
    

### Минусы

- сложнее поддержка;
    
- можно скрыть реальную проблему.
    

### Золотое правило

- optional → `-quiet` + `if {[llength ...] > 0}`
    
- required → если не найден, `error`
    

---

## Мини-шаблон для повседневного использования

proc require_single_port {name} {

set p [get_ports -quiet $name]

if {[llength $p] != 1} {

error "Required port '$name' not found uniquely"

}

return $p

}

  

proc optional_cells {pattern} {

return [get_cells -quiet -hier $pattern]

}

  

set clk_port [require_single_port clk_in]

create_clock -period 10.000 $clk_port

  

set ila_cells [optional_cells *ila*]

if {[llength $ila_cells] > 0} {

set ila_pins [get_pins -quiet -of_objects $ila_cells -filter {REF_PIN_NAME =~ DATA_I* || REF_PIN_NAME =~ TRIGGER_I*}]

if {[llength $ila_pins] > 0} {

set_false_path -to $ila_pins

puts "INFO: Debug false path applied"

}

}

Этот шаблон уже можно брать как основу и адаптировать под проект.



---