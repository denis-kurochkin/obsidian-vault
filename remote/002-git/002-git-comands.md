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


