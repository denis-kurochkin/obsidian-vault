
## Добавление пункта в контекстное меню проводника

1. Найти файл <span style="color:#00ff00">vivado.bat</span>
2. Запомнить путь до файла. Например <span style="color:#00ff00">"A:\Xilinx\2025.1\Vivado\bin\vivado.bat"</span>
3. Открыть редактор реестра ( <span style="color:#00ff00">regedit</span> )
4. Перейти в раздел <span style="color:#00ff00"> HKEY_CLASSES_ROOT\Directory\Background\shell </span>
5. Создать новый ключ, например <span style="color:#00ff00">VivadoTclHere</span>
6. В ( <span style="color:#00ff00">Default</span> ) задать имя пункта меню, например: <span style="color:#00ff00">Open Vivado Tcl Shell here</span>
7. Внутри создать ключ <span style="color:#00ff00">command</span> и в его ( <span style="color:#00ff00">Default</span> ) указать: <span style="color:#00ff00">"A:\Xilinx\2025.1\Vivado\bin\vivado.bat" -mode tcl"</span>
---

## Добавление пункта в "открыть с помощью"

1. Найти файл <span style="color:#00ff00">vivado.bat</span>
2. Запомнить путь до файла. Например <span style="color:#00ff00">"A:\Xilinx\2025.1\Vivado\bin\vivado.bat"</span>
3. Открыть редактор реестра ( <span style="color:#00ff00">regedit</span> )
4. Перейти в раздел <span style="color:#00ff00"> HKEY_CLASSES_ROOT\.tcl </span>
5. Убедиться, что его значение (по умолчанию) — например, <span style="color:#00ff00">TclScript</span> (запомнить это имя).
6. Перейти в раздел <span style="color:#00ff00"> HKEY_CLASSES_ROOT\TclScript\shell </span>
7. Создать новый ключ, например <span style="color:#00ff00">OpenWithVivado</span>
8. В ( <span style="color:#00ff00">Default</span> ) задать имя пункта меню, например: <span style="color:#00ff00">Open with Vivado Tcl Shell</span>
9. Внутри создать ключ <span style="color:#00ff00">command</span> и в его ( <span style="color:#00ff00">Default</span> ) указать: <span style="color:#00ff00">"A:\Xilinx\2025.1\Vivado\bin\vivado.bat" -mode tcl -source "%1"</span>