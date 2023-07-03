## ДИСКОВАЯ ПОДСИСТЕМА 

###             ЦЕЛЬ ДОМАШНЕГО ЗАДАНИЯ:
Научиться собирать RAID-массивы при помощи mdadm 

### ОПИСАНИЕ ДОМАШНЕГО ЗАДАНИЯ
• добавить в Vagrantfile еще дисков

• собрать R0/R5/R10 на выбор

• прописать собранный рейд в конф, чтобы рейд собирался при загрузке

• сломать/починить raid 

• создать GPT раздел и 5 партиций и смонтировать их на диск.

### ДОПОЛНИТЕЛЬНЫЕ ЗАДАНИЯ:
Vagrantfile, который сразу собирает систему с подключенным рейдом

### ДОБАВЛЯЕМ ЕЩЕ ДИСКОВ В VAGRANTFILE:
```
:sata5 => {
:dfile => './sata5.vdi', # Путь, по которому будет создан файл диска
:size => 250, # Размер диска в мегабайтах
:port => 5 # Номер порта, на который будет зацеплен диск
},

```
### СОЗДАЕМ BASH-СКРИПТ СО СБОРКОЙ RAID-5 ВНУТРИ VAGRANTFILE:
```
box.vm.provision "shell", inline: <<-SHELL
	      mkdir -p ~root/.ssh
              cp ~vagrant/.ssh/auth* ~root/.ssh
	      yum install -y mdadm smartmontools hdparm gdisk
              spisok=`sudo lshw -short | grep disk | awk '$4 < 300 {print $2}'`
              mdadm --create --verbose /dev/md0 -l 5 -n 10 $spisok                           #/dev/sd{a,b,c,d,e,f,g,h,i,j}
              mkdir /etc/mdadm
              touch /etc/mdadm/mdadm.conf
              chmod 777 /etc/mdadm/mdadm.conf
              echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
              mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
              parted -s /dev/md0 mklabel gpt
              parted /dev/md0 mkpart primary ext4 0% 20%
              parted /dev/md0 mkpart primary ext4 20% 40%
              parted /dev/md0 mkpart primary ext4 40% 60%
              parted /dev/md0 mkpart primary ext4 60% 80%
              parted /dev/md0 mkpart primary ext4 80% 100%
              for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
              mkdir -p /raid/part{1,2,3,4,5}
              for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
  	  SHELL

```

### ТАКЖЕ МЕНЯЕМ КОНТРОЛЬНУЮ СУММУ:
`
"iso_checksum": "017e6f8924248c204fe649403e0fe6896302a6b3c6b5a69968889758d805df26"
`
### МЕНЯЕМ КОМАНДУ ВЫКЛЮЧЕНИЯ (УЖЕ НЕ ПОМНЮ ЗАЧЕМ)
`
"shutdown_command": "echo 'vagrant' | sudo -S shutdown"
`
### СТАВИМ БОЛЬШЕЕ ЗНАЧЕНИЕ Т.К. PACKER СОБИРАЕТ НЕ БЫСТРО
`
"ssh_timeout": "40m"
`
### ДОБАВЛЯЕМ КОНТРОЛЛЕР С GUEST ADDITIONS
```
"vboxmanage": [
       [
          "storageattach",
          "{{.Name}}",
          "--storagectl",
          "IDE Controller",
          "--port",
          "1",
          "--device",
          "0",
          "--type",
          "dvddrive",
          "--medium",
          "/usr/share/virtualbox/VBoxGuestAdditions.iso"

        ]
```
### НЕМНОГО МЕНЯЕМ (НЕ ПОМНЮ ЗАЧЕМ)        
`
"execute_command": "echo 'vagrant'| {{.Vars}} sudo -S -E bash '{{.Path}}'"
`
### МЕНЯЕМ СКРИПТЫ 
#### В СТРОЧКУ УСТАНОВКИ ЯДРА ДОБАВЛЯЕМ УСТАНОВКУ ЗАГОЛОВОЧНЫХ ФАЙЛОВ, УТИЛИТ И БИБЛИОТЕК(БЕЗ НИХ ОТКАЗЫВАЮТСЯ РАБОТАТЬ ГОСТЕВЫЕ ДОПОЛНЕНИЯ)
`yum --enablerepo elrepo-kernel install --allowerasing kernel-ml kernel-ml-devel kernel-ml-core kernel-ml-headers kernel-ml-tools kernel-ml-tools-libs -y
`
#### ТАКЖЕ УСТАНАВЛИВАЕМ УТИЛИТЫ ДЛЯ УСТАНОВКИ МОДУЛЕЙ ЯДРА
`yum install gcc make perl tar bzip2 -y
`
#### УДАЛЯЕМ МОДУЛИ СТАРОГО ЯДРА
`
yum remove kernel-modules-4.18.0-492.el8.x86_64 kernel-core-4.18.0-492.el8.x86_64 kernel-4.18.0-492.el8.x86_64 -y
`
#### ДОБАВЛЯЕМ ЕЩЕ ОДИН СКРИПТ ДЛЯ УСТАНОВКИ ГОСТЕВЫХ ДОПОЛНЕНИЙ
`
!/bin/bash
`
##### Создание папки для гостевых дополнений
`
sudo mkdir /media/GA
`
##### Монтирование образа гостевых дополнений
`
sudo mount /home/vagrant/VBoxGuestAdditions.iso /media/GA
`
##### Запуск установки гостевых дополнений
`
sudo /media/GA/./VBoxLinuxAdditions.run
`
##### Перезагрузка ВМ
`
shutdown -r now
`
