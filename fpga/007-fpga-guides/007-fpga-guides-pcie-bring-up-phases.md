# PCIe bring-up phases внутри PCIe architecture

**PCIe bring-up** — это весь путь от “на плату подали питание” до “host видит FPGA, BAR работает, DMA ходит, interrupt приходит, link стабилен под нагрузкой”.

Важно не смешивать разные уровни успеха:

```
1. Link поднялся до L0.
2. Data Link initialization завершилась.
3. Host увидел устройство.
4. Enumeration прошла.
5. BAR назначены.
6. Driver загрузился.
7. Register access работает.
8. DMA работает.
9. Interrupts работают.
10. Throughput и stability соответствуют ожиданию.
```

Очень частая ошибка — увидеть `L0` и считать, что PCIe уже полностью работает. На самом деле `L0` — это в основном успех lower-level link training. После него еще остаются enumeration, configuration space, BAR, driver, DMA и user logic.

---

# 1. Общая карта bring-up

Полезно думать фазами:

```
Phase 0: Design/IP preparation
Phase 1: Power, clocks, reset, FPGA configuration
Phase 2: PCIe PHY / GT initialization
Phase 3: LTSSM link training
Phase 4: Data Link initialization
Phase 5: Enumeration by Root Complex
Phase 6: BAR assignment and command enable
Phase 7: Basic register access
Phase 8: Driver / software initialization
Phase 9: DMA / interrupts / high-level function
Phase 10: Stability and performance validation
```

Каждая фаза имеет свои симптомы. Если фаза 3 не прошла, нет смысла начинать debug с DMA descriptor. Если фаза 6 не прошла, не надо искать проблему в GT equalization.

---

# 2. Phase 0 — Design/IP preparation

До железа нужно убедиться, что PCIe IP настроен под реальный сценарий.

Проверить:

```
Endpoint или Root Port;
target PCIe generation;
target link width;
правильное FPGA family/package;
правильный PCIe block/GT quad;
BAR sizes;
AXI4-Stream / AXI Bridge / XDMA / QDMA mode;
MSI/MSI-X support;
reference clock settings;
reset polarity;
example design availability.
```

В Vivado лучше первым шагом поднять **PCIe example design**, а не сразу сложную application logic. Так ты отделяешь проблемы PCIe core/board от ошибок своего BAR/DMA/CDC.

---

# 3. Endpoint vs Root Port

Для FPGA add-in card чаще всего FPGA работает как:

```
PCIe Endpoint
```

Host или SoC — это:

```
Root Complex / Root Port
```

Bring-up Endpoint означает:

```
FPGA должна быть обнаружена и перечислена host-ом.
```

Bring-up Root Port означает другое:

```
FPGA/SoC сама должна обнаружить downstream Endpoint-устройства.
```

Эти сценарии похожи на уровне link training, но сильно отличаются по software, enumeration и роли устройства.

---

# 4. Phase 1 — Power, REFCLK, PERST#, FPGA configuration

Низкоуровневый bring-up начинается не с TLP, а с аппаратных условий:

```
power rails stable;
PCIe REFCLK stable;
PERST# корректно удерживается и отпускается;
FPGA успела сконфигурироваться;
PCIe IP готов к link training;
GT reset sequence выполнена.
```

AMD PCIe documentation указывает важное требование старта: `PERST#` должен быть отпущен после power-good, а PCIe port должен быть готов к link training в установленном окне после отпускания `PERST#`; в документации это обсуждается в контексте PCIe boot-time requirement и Tandem Configuration.

---

# 5. Почему FPGA configuration time важен

В отличие от ASIC PCIe Endpoint, FPGA должна сначала загрузить bitstream.

Если host начинает enumeration раньше, чем FPGA PCIe Endpoint готов, устройство может не появиться в системе.

Типичный симптом:

```
после cold boot устройство не видно;
после warm reboot устройство появляется.
```

Это часто указывает на timing между:

```
host boot / PERST# release
FPGA configuration completion
PCIe IP readiness
link training
enumeration
```

Для таких случаев существуют подходы вроде Tandem PCIe / fast configuration, но это уже отдельная тема.

---

# 6. REFCLK

PCIe REFCLK должен быть:

```
правильной частоты;
стабильным;
с допустимым jitter;
подключенным к правильному MGTREFCLK;
совместимым с выбранным PCIe IP/GT quad.
```

Если REFCLK отсутствует или плохой, возможны симптомы:

```
LTSSM не выходит из Detect;
LTSSM циклично ходит Detect/Polling;
link не поднимается выше Gen1;
link падает в Recovery;
link нестабилен под нагрузкой.
```

---

# 7. PERST#

`PERST#` — фундаментальный reset PCIe-устройства.

Практическая логика:

```
PERST# asserted:
    PCIe Endpoint удерживается в reset.

PERST# deasserted:
    Endpoint может начинать link initialization.
```

Для FPGA важно не просто подключить `PERST#`, а правильно связать его с:

```
PCIe IP reset;
GT reset;
user logic reset;
configuration done;
PLL/MMCM lock;
local reset synchronizers.
```

Неправильный reset sequencing может дать ситуацию, где LTSSM иногда стартует, иногда нет.

---

# 8. Phase 2 — PCIe PHY / GT initialization

До LTSSM должны быть готовы transceivers.

Проверять:

```
GT reset done;
PLL/QPLL/CPLL lock;
TX/RX reset done;
user clock stable;
PCIe hard block out of reset;
lane pins соответствуют Vivado constraints;
GT quad выбран правильно.
```

Если GT не готов, LTSSM может не пойти дальше ранних состояний.

Для debug полезно вывести в ILA:

```
gt_reset_done;
phy ready/status;
user_reset;
user_clk;
cfg_phy_link_down;
ltssm_state.
```

---

# 9. Phase 3 — LTSSM link training

Это фаза:

```
Detect -> Polling -> Configuration -> L0
```

Ее цель:

```
найти receiver;
получить lock;
согласовать lanes;
согласовать width;
согласовать speed;
перейти в L0.
```

AMD Vivado Link Debug может сохранять LTSSM transitions, доступные в Hardware Manager; `report_hw_pcie` также дает информацию о PCIe core, LTSSM state visitation и trace data.

---

# 10. Что значит L0 в bring-up

`L0` означает, что link находится в нормальном рабочем состоянии на уровне PCIe link.

Но это еще не значит:

```
host назначил BAR;
driver загружен;
Memory Space Enable включен;
Bus Master Enable включен;
DMA готова;
user logic работает.
```

Поэтому `L0` — это важная контрольная точка, но не финал bring-up.

---

# 11. Если LTSSM не дошел до L0

Это lower-level проблема.

Проверять:

```
REFCLK;
PERST#;
power rails;
lane mapping;
TX/RX swap;
P/N polarity;
AC coupling capacitors;
GT placement;
PCIe IP mode;
slot/root port enable;
retimer/redriver;
signal integrity;
configured max speed/width.
```

Не начинать с:

```
BAR;
driver;
DMA;
interrupt;
Linux application.
```

Пока нет `L0`, Transaction Layer debug почти не имеет смысла.

---

# 12. Если LTSSM циклично уходит в Recovery

`Recovery` может быть нормальным при speed change, но постоянные циклы подозрительны.

Возможные причины:

```
signal integrity;
equalization issue;
REFCLK jitter;
неудачный переход на Gen3/Gen4;
retimer issue;
потеря lock;
нестабильное питание;
проблемы с channel loss.
```

Смотреть не только текущее состояние, а trace:

```
L0 -> Recovery -> L0
L0 -> Recovery -> Detect
Polling -> Recovery -> Polling
```

Trace часто важнее одиночного снимка state.

---

# 13. Phase 4 — Data Link initialization

После physical training должна завершиться Data Link initialization.

Это переход от “physical link есть” к “можно нормально обмениваться protocol traffic”.

AMD configuration status interface показывает информацию о negotiated link width/speed, power state и link-related status; для PCIe core есть `cfg_phy_link_down`, основанный на Physical Layer LTSSM.

В практическом debug полезно различать:

```
physical link up;
DL initialization in progress;
DL initialization completed.
```

Если Data Link initialization не завершилась, enumeration и TLP traffic могут не работать корректно.

---

# 14. Phase 5 — Enumeration by Root Complex

Когда link готов, Root Complex начинает enumeration.

Он читает configuration space Endpoint:

```
Vendor ID;
Device ID;
Class Code;
BAR size;
Capabilities;
MSI/MSI-X capability;
PCIe capability;
link capability/status.
```

На Linux базовая проверка:

```
lspci
```

AMD PCIe debug checklist предлагает сначала проверить, обнаруживается ли PCIe IP через `lspci`; если устройство не обнаружено, попробовать warm reboot и повторить `lspci`, а затем проверить, находится ли LTSSM в `L0`.

---

# 15. Если L0 есть, но `lspci` не видит устройство

Это уже не чистый Physical Layer debug.

Возможные причины:

```
FPGA Endpoint был не готов во время host enumeration;
PERST# timing нарушен;
host не сделал rescan;
configuration space некорректен;
endpoint mode настроен неверно;
BIOS/UEFI отключил port;
slot работает не так, как ожидается;
FPGA загружается слишком долго;
требуется warm reboot/rescan.
```

Практические действия:

```bash
lspci
lspci -vv
```

Иногда на Linux применяют rescan:

```bash
echo 1 | sudo tee /sys/bus/pci/rescan
```

Но rescan не исправит проблему, если link не находится в стабильном рабочем состоянии.

---

# 16. Phase 6 — BAR assignment

Во время enumeration host определяет размер BAR и назначает адреса.

BAR — это окно, через которое software обращается к FPGA.

После успешного назначения можно увидеть:

```bash
lspci -vv
```

И проверить:

```
Region 0;
Region 1;
Memory at ...;
I/O или Memory BAR;
prefetchable / non-prefetchable;
BAR size.
```

Если BAR не назначен или размер неверный, basic register access не заработает.

---

# 17. Command Register: Memory Space Enable

Даже если BAR назначен, host должен включить доступ к memory space.

В PCIe Command Register есть bit:

```
Memory Space Enable
```

Если он не установлен, memory accesses к BAR могут не работать.

AMD configuration status interface перечисляет command/status bits, включая Memory Space Enable для функций устройства.

---

# 18. Command Register: Bus Master Enable

Для DMA особенно важен:

```
Bus Master Enable
```

Если этот bit не включен, Endpoint может быть виден в системе, но не сможет инициировать PCIe transactions как bus master.

AMD debug checklist и материалы по `lspci/setpci` отдельно указывают проверять Command Register bits, включая Bus Master Enable, при проблемах с memory access/DMA.

Практический симптом:

```
lspci видит устройство;
BAR есть;
host пишет registers;
но DMA из FPGA не работает.
```

Одна из первых проверок — Bus Master Enable.

---

# 19. Phase 7 — Basic BAR register access

После enumeration и enable bits нужно проверить самый простой доступ:

```
read FPGA ID register;
write scratch register;
read scratch register back;
read status register;
clear/set control bit.
```

Это лучше, чем сразу запускать DMA.

Минимальный register map:

```
0x00: FPGA_ID
0x04: VERSION
0x08: SCRATCH
0x0C: STATUS
0x10: CONTROL
```

Если scratch register не работает, DMA debug начинать рано.

---

# 20. Если BAR read зависает

Возможные причины:

```
Memory Space Enable не включен;
Memory Read TLP дошел, но FPGA не отправила Completion;
Completion malformed;
AXI/AXIS interface заблокирован;
user logic в reset;
BAR decode неправильный;
read path не возвращает valid;
clock domain crossing внутри BAR path ошибочный.
```

Если используется raw TLP interface, нужно проверить CQ/CC path.

Если используется AXI Bridge/XDMA, проверить AXI read channel и reset/status subsystem.

---

# 21. Если BAR write работает, а BAR read нет

Это важный симптом.

Memory Write — posted transaction:

```
host отправил write и не ждет Completion.
```

Memory Read — non-posted transaction:

```
host отправил read request и ждет Completion.
```

Поэтому write может “работать”, а read зависать из-за отсутствующего или неправильного Completion.

Проверять:

```
приходит ли read request;
формируется ли completion;
правильный ли tag;
правильный ли byte count;
правильный ли lower address;
нет ли backpressure на completion stream.
```

---

# 22. Phase 8 — Driver / software initialization

После basic PCIe visibility начинается software layer.

Driver обычно делает:

```
находит Vendor ID / Device ID;
мапит BAR;
настраивает registers;
выделяет buffers;
настраивает DMA descriptors;
включает interrupts;
проверяет capabilities;
включает Bus Mastering, если нужно.
```

Если устройство видно в `lspci`, но application не работает, проблема может быть уже в driver или user-space flow.

---

# 23. Разделение FPGA issue и driver issue

Полезный принцип:

```
сначала доказать hardware register access;
потом debug driver;
потом DMA.
```

Минимальные проверки:

```
lspci видит устройство;
BAR назначен;
Memory Space Enable включен;
scratch register read/write работает;
interrupt status register работает;
DMA control registers доступны.
```

Если это не работает, driver высокого уровня не спасет.

---

# 24. Phase 9 — DMA bring-up

DMA bring-up — отдельный слой.

Для DMA нужно:

```
Bus Master Enable;
корректные host memory buffers;
правильные physical/IOVA addresses;
descriptor setup;
MPS/MRRS;
outstanding request handling;
completion handling;
interrupt или polling completion;
cache/IOMMU considerations;
AXI/stream side ready.
```

Если используется XDMA/QDMA, часть этого уже реализована IP/driver stack.

Если DMA самописная, нужно понимать TLP RQ/RC/CQ/CC гораздо глубже.

---

# 25. DMA write vs DMA read

DMA write:

```
FPGA -> Host Memory Write
```

Обычно проще, потому что Memory Write — posted transaction.

DMA read:

```
FPGA -> Host Memory Read Request
Host -> FPGA Completion with Data
```

Сложнее, потому что нужно:

```
выдать read request;
отслеживать tag;
принять completions;
собрать данные;
обработать split completions;
соблюдать ordering/credits.
```

Поэтому DMA write часто поднимают раньше, чем DMA read.

---

# 26. Interrupt bring-up

Interrupt проверяется после basic BAR и DMA/status.

Для modern PCIe обычно используются:

```
MSI;
MSI-X.
```

Bring-up interrupt:

```
host включает MSI/MSI-X;
driver настраивает vectors;
FPGA генерирует interrupt request;
host вызывает ISR;
driver читает status;
driver очищает interrupt.
```

Если interrupt не приходит, проверять:

```
MSI/MSI-X capability enabled;driver enabled interrupt;vector programmed;FPGA interrupt pulse/level correct;status bit set;clear-on-write работает;interrupt masked или нет;Bus Master/Memory settings;IOMMU/OS settings.
```

---

# 27. Phase 10 — Stability validation

После “оно заработало один раз” нужно проверить стабильность.

Тесты:

```
cold boot;warm reboot;PCIe rescan;FPGA reconfiguration;driver reload;runtime reset;link retrain;DMA long run;small transfers;large transfers;backpressure;max throughput;interrupt storm;error recovery.
```

Многие PCIe-баги проявляются только при:

```
cold boot;долгой работе;максимальной скорости;runtime reset;переинициализации link;нагрузке на host.
```

---

# 28. Bring-up phases как debug decision tree

Очень полезный flow:

```
1. Есть ли REFCLK/PERST#/power?   Нет -> board/reset/clock debug.2. LTSSM дошел до L0?   Нет -> physical/link training debug.3. Data Link initialization completed?   Нет -> lower-layer link debug.4. Видно ли устройство в lspci?   Нет -> enumeration/configuration timing debug.5. BAR назначен?   Нет -> configuration space/BAR setup debug.6. Memory Space Enable включен?   Нет -> OS/driver/command register debug.7. BAR scratch работает?   Нет -> Transaction/user logic/AXI debug.8. Bus Master Enable включен?   Нет -> DMA не заработает.9. DMA write работает?   Нет -> DMA TX/path/descriptor debug.10. DMA read работает?    Нет -> tags/completions/MRRS/RC path debug.11. Interrupt работает?    Нет -> MSI/MSI-X/driver/status debug.12. Throughput нормальный?    Нет -> credits/MPS/MRRS/outstanding/buffering/performance debug.
```

---

# 29. Что смотреть в Vivado ILA на раннем этапе

Для первых фаз:

```
PERST# synchronized/local;PCIe user reset;PCIe user clock alive;ltssm_state;cfg_phy_link_down;cfg_phy_link_status;cfg_current_speed;cfg_negotiated_width;user_lnk_up;GT reset done;PLL lock;link debug trace.
```

Vivado Hardware Manager PCIe Link Debug показывает core properties, LTSSM State Trace и LTSSM State Diagram with transitions.

---

# 30. Что смотреть в OS

На Linux:

```
lspcilspci -vvlspci -nnlspci -s <bus:dev.fn> -vv
```

Для command register и capability debug иногда используют:

```
setpci
```

Практически важны:

```
Vendor ID / Device ID;BAR regions;Kernel driver in use;LnkSta speed/width;Command register;Memory Space Enable;Bus Master Enable;MSI/MSI-X enabled.
```

---

# 31. Phase mapping to symptoms

|Симптом|Вероятная фаза|
|---|---|
|LTSSM stuck Detect|Power/REFCLK/PERST#/lanes|
|LTSSM stuck Polling|SI/clocking/lane decode|
|LTSSM reaches L0, no `lspci`|Enumeration timing/config|
|Device in `lspci`, no BAR|BAR/configuration issue|
|BAR exists, read hangs|Completion/user logic issue|
|Writes work, reads fail|Non-posted/completion path|
|BAR works, DMA fails|Bus Master/descriptor/DMA path|
|DMA write works, DMA read fails|Read request/completion/tag path|
|Works at Gen1, fails at Gen3|Equalization/SI/speed issue|
|Works after warm reboot only|FPGA config/PERST#/enumeration timing|

---

# 32. Example design as reference point

Bring-up на custom board лучше начинать с example design.

Порядок:

```
1. Сгенерировать PCIe IP.2. Собрать example design.3. Проверить constraints/pins.4. Запрограммировать FPGA.5. Проверить LTSSM/L0.6. Проверить lspci.7. Проверить BAR access/example tests.8. Только потом интегрировать свою логику.
```

Если example design не поднимается, проблема почти точно не в твоей application logic.

Если example design работает, а твой проект нет — искать различия:

```
reset;clocking;constraints;IP parameters;BAR setup;AXI/AXIS handshake;CDC;driver.
```

---

# 33. Cold boot vs warm reboot

PCIe FPGA bugs часто отличаются между cold boot и warm reboot.

```
Cold boot:    power rails ramp;    FPGA configuration;    PERST# timing;    host enumeration window.Warm reboot:    FPGA уже может быть configured;    link может быть ready быстрее;    host повторяет enumeration.
```

Если warm reboot помогает, подозревать:

```
FPGA configuration time;PERST# timing;Tandem requirement;host enumeration too early;reset sequencing.
```

---

# 34. Runtime reset

Отдельно проверить:

```
soft reset через driver;FPGA internal reset;PCIe core reset;DMA reset;user logic reset;link retrain;driver reload.
```

PCIe link может остаться в L0, но user logic быть в reset.

Или наоборот: user logic active, а PCIe core еще не готов.

Поэтому reset sequencing должен учитывать:

```
link ready;DL init complete;user reset released;BAR logic ready;DMA ready;driver initialized.
```

---

# 35. Bring-up state machine внутри FPGA

Хорошая практика — иметь свой system-ready state machine.

Пример:

```
WAIT_FPGA_INITWAIT_PCIE_USER_CLKWAIT_PCIE_LINKWAIT_DL_INITWAIT_CFG_ENABLEWAIT_USER_RESET_DONEREADY
```

И только после `READY` разрешать:

```
DMA start;interrupt generation;stream source enable;external data path enable.
```

---

# 36. Gating user logic

Плохо:

```
assign dma_start_allowed = sw_start;
```

Лучше:

```
assign dma_start_allowed =    sw_start &&    pcie_link_ready &&    cfg_bus_master_enable &&    dma_engine_ready &&    !user_reset;
```

Иначе software может записать start bit, когда PCIe link еще не готов или Bus Master Enable не включен.

---

# 37. BAR logic ready

Если host делает BAR read сразу после enumeration, FPGA должна корректно ответить.

Поэтому BAR register block должен быть готов раньше, чем сложная application logic.

Практический подход:

```
BAR ID/VERSION/STATUS registers доступны сразу после PCIe ready;сложная логика показывает not-ready flags;DMA start запрещен до full system ready.
```

Так driver может прочитать status и понять, что FPGA еще инициализируется.

---

# 38. Status register для bring-up

Очень полезно иметь register:

```
PCIE_STATUS
```

Поля:

```
bit 0: link_up_seenbit 1: dl_init_done_seenbit 2: user_reset_donebit 3: bus_master_enable_seenbit 4: memory_space_enable_seenbit 5: dma_readybit 6: interrupt_enabledbit 7: error_seen
```

Так debug через software становится намного проще.

---

# 39. Error counters

Полезные counters:

```
BAR read count;BAR write count;DMA write request count;DMA read request count;DMA completion count;unexpected completion count;timeout count;interrupt count;reset count;link_down_seen count;
```

Если есть link down / recovery status от PCIe IP, полезно latch-ить такие события.

---

# 40. Bring-up для XDMA/QDMA

Если используется XDMA/QDMA, bring-up обычно делится так:

```
1. PCIe link up.2. Device visible in lspci.3. XDMA/QDMA driver loaded.4. BAR/control registers visible.5. DMA channels detected.6. Host-to-card transfer works.7. Card-to-host transfer works.8. Interrupt/completion path works.9. Throughput/stress tests pass.
```

Если driver не видит DMA channel, это уже не обязательно LTSSM. Проверять нужно IP config, driver compatibility, BAR, interrupts, descriptors и OS view.

---

# 41. Bring-up для raw TLP interface

Если работаешь без XDMA/QDMA, через raw TLP streams, фазы сложнее:

```
CQ receive path;BAR decode;CC completion path;RQ request generation;RC completion receive path;tag management;split completions;flow control;error handling.
```

Тогда basic BAR read/write особенно важен, потому что он проверяет Completer path.

---

# 42. Когда переходить к performance debug

Performance debug имеет смысл только после:

```
link stable;expected speed/width negotiated;BAR access работает;DMA работает корректно;нет protocol errors;нет reset instability.
```

Иначе throughput debug будет смешан с функциональными ошибками.

Performance зависит от:

```
Gen/speed;width;MPS;MRRS;credits;DMA burst size;outstanding requests;host memory behavior;IOMMU;AXI backpressure;buffering.
```

Это лучше разбирать в заметке про PCIe flow control/performance.

---

# 43. Что логировать при bring-up

Для воспроизводимого debug сохранять:

```
FPGA bitstream version;Vivado version;PCIe IP version/config;board revision;host platform;BIOS settings;cold/warm boot;lspci -vv output;LTSSM trace;negotiated speed/width;BAR map;driver version;DMA test result.
```

Иначе легко потерять различия между “работало вчера” и “не работает сегодня”.

---

# 44. Minimal successful bring-up definition

Минимальный успех для PCIe Endpoint FPGA:

```
LTSSM reaches L0;Data Link init complete;device visible in lspci;expected speed/width or acceptable fallback;BAR assigned;Memory Space Enable set;scratch register read/write works.
```

Минимальный успех для DMA design:

```
previous list+ Bus Master Enable set;+ DMA write works;+ DMA read works;+ interrupt/completion works;+ long-run test stable.
```

---

# 45. Частые ошибки bring-up

## Ошибка 1

```
Начинать debug с driver, когда LTSSM не L0.
```

Сначала link.

---

## Ошибка 2

```
Считать, что L0 означает, что DMA должна работать.
```

L0 — только lower-level success.

---

## Ошибка 3

```
Не проверять Memory Space Enable.
```

BAR может быть назначен, но memory access отключен.

---

## Ошибка 4

```
Не проверять Bus Master Enable.
```

DMA может не работать, хотя device виден.

---

## Ошибка 5

```
Сразу интегрировать сложную user logic без example design.
```

Потом непонятно, где ошибка: в плате, IP или RTL.

---

## Ошибка 6

```
Игнорировать cold boot.
```

Warm reboot может маскировать FPGA configuration timing проблемы.

---

## Ошибка 7

```
Не latch-ить link_down/recovery/error events.
```

Краткие сбои могут исчезнуть до того, как их увидит software.

---

## Ошибка 8

```
Разрешать DMA до готовности PCIe/user logic.
```

Нужен system-ready gating.

---

# 46. Практический bring-up checklist

```
Board / hardware:    power rails    REFCLK    PERST#    lane mapping    AC coupling    connector/slotVivado / IP:    Endpoint/Root Port mode    speed/width    BAR config    GT quad    constraints    example designLink:    LTSSM trace    L0 reached    DL init complete    negotiated width    current speedOS:    lspci visible    BAR regions assigned    command register    Memory Space Enable    Bus Master Enable    driver boundFPGA user logic:    reset released    BAR scratch works    DMA ready    interrupts enabled    errors clearStress:    cold boot    warm reboot    driver reload    long DMA    throughput    link stability
```

---

# 47. Главное резюме

PCIe bring-up лучше делать не как хаотичный debug, а как последовательность фаз.

Короткая карта:

```
Power / REFCLK / PERST#        |FPGA configuration / PCIe IP ready        |GT / PHY ready        |LTSSM -> L0        |Data Link init complete        |Host enumeration / lspci        |BAR assignment        |Memory Space Enable / Bus Master Enable        |BAR read/write        |Driver        |DMA / interrupts        |performance / stability
```

Главное правило:

```
Каждая фаза проверяет свой уровень.Не переходи к debug следующего уровня, пока предыдущий не доказан.
```

Практическая формула:

```
Нет L0       -> debug physical/link training.Есть L0, нет lspci -> debug enumeration/config timing.Есть lspci, нет BAR access -> debug command/BAR/TLP/user logic.Есть BAR, нет DMA -> debug Bus Master/descriptor/DMA path.Есть DMA, плохой throughput -> debug credits/MPS/MRRS/buffering/performance.
```