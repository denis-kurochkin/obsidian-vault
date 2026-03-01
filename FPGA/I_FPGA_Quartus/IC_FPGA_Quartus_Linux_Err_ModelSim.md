Решение проблемы с запуском Modelsim-altera для Quartus 20.1 на Linux.

1. Проверить путь до папки
     
     Tools → Options → EDA Tool Options
     intelFPGA_lite/20.1/modelsim_ase/linuxaloem
     
2. Убедиться что нет переменных окружения
     
     echo $LM_LICENSE_FILE
     echo $MGLS_LICENSE_FILE
      
     Если есть, то удалить:
     
     unset LM_LICENSE_FILE
     unset MGLS_LICENSE_FILE
     
3. Проверить запускается ли Modelsim
    
4. Если нет, установить библиотеки:
     
     sudo dpkg --add-architecture i386
     sudo apt update
     sudo apt install libc6:i386 
     sudo apt install libxext6:i386
     sudo apt install libxft2:i386 
     sudo apt install libxrender1:i386 
     sudo apt install libfreetype6:i386 
     sudo apt install libexpat1:i386