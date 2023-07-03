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
	      yum install -y mdadm smartmontools hdparm gdisk      # Устанавливаем утилиты для работы с массивами
```
### При создании машины через раз возникает ошибка: при первом запуске диск может именоваться sda, а при следующем sdb. Поэтому добавляем следующие строчки:
              spisok=`sudo lshw -short | grep disk | awk '$4 < 300 {print $2}'`  
              mdadm --create --verbose /dev/md0 -l 5 -n 10 $spisok
### Создаем файл mdadm.conf чтобы массив не распался при перезапуске: 	      
              mkdir /etc/mdadm
              touch /etc/mdadm/mdadm.conf
              chmod 777 /etc/mdadm/mdadm.conf
              echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
              mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
### Создаем партиции: 
	      parted -s /dev/md0 mklabel gpt
              parted /dev/md0 mkpart primary ext4 0% 20%
              parted /dev/md0 mkpart primary ext4 20% 40%
              parted /dev/md0 mkpart primary ext4 40% 60%
              parted /dev/md0 mkpart primary ext4 60% 80%
              parted /dev/md0 mkpart primary ext4 80% 100%
### Цикл для создания файловых систем на созданных партициях:	      
              for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
              mkdir -p /raid/part{1,2,3,4,5}
              for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
  	  SHELL
```
