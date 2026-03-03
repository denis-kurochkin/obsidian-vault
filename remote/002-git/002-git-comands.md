## Команды настройки git

Посмотреть все установленные настройки и узнать где именно они заданы.
```
$ git config --list --show-origin 
```

Указать имя и адрес электронной почты.
```
$ git config --global user.name "John Doe"
$ git config --global user.email johndoe@example.com
```

Выбрать текстовый редактор, который будет использоваться.
```
$ git config --global core.editor vscode
```
```
$ git config --global core.editor "'C:/Program Files/Notepad++/notepad++.exe'
-multiInst -notabbar -nosession -noPlugin"
```

Установить имя main для ветки по умолчанию.

```
$ git config --global init.defaultBranch main
```

Проверить используемую конфигурацию.
```
$ git config --list
```
---
## Основные команды git

Переход в существующий каталог.
Linux
```
$ cd /home/user/my_project
```
Win
```
$ cd /Users/user/my_project
```

Инициализация репозитория. Команда создаёт в текущей директории новую поддиректорию с именем .git,содержащую все необходимые файлы репозитория — структуру Git-репозитория.
```
$ git init
```

Клонирование существующего репозитория.
```

```
