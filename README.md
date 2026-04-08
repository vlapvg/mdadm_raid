# ДЗ 2: Работа с mdadm. Тема 2: Дисковая подсистема.  

## Описание домашнего задания   

• Добавить в виртуальную машину несколько дисков

• Собрать RAID-0/1/5/10 на выбор

• Сломать и починить RAID

• Создать GPT таблицу, пять разделов и смонтировать их в системе. 

##   
***Скрипт для создания рейда [тут](https://github.com/vlapvg/mdadm_raid/blob/main/script_raid.sh)***   
***Ниже шаги выполнения ДЗ и основные команды*** 
##
## Добавим в виртуальную машину дополнительно диски   

В настройках VirtualBox добавляем 4 диска по 1GB. 

## Cмотрим, какие блочные устройства у нас есть
    root@april:/home/vlap# lshw -short | grep disk
    /0/100/1.1/0.0.0  /dev/cdrom  disk        CD-ROM
    /0/100/d/0        /dev/sda    disk        26GB VBOX HARDDISK
    /0/100/d/1        /dev/sdb    disk        1073MB VBOX HARDDISK
    /0/100/d/2        /dev/sdc    disk        1073MB VBOX HARDDISK
    /0/100/d/3        /dev/sdd    disk        1073MB VBOX HARDDISK
    /0/100/d/0.0.0    /dev/sde    disk        1073MB VBOX HARDDISK

## Занулим на всякий случай суперблоки

    root@april:/home/vlap# mdadm --zero-superblock --force /dev/sd{b,c,d,e}
    mdadm: Unrecognised md component device - /dev/sdb
    mdadm: Unrecognised md component device - /dev/sdc
    mdadm: Unrecognised md component device - /dev/sdd
    mdadm: Unrecognised md component device - /dev/sde

## Создаём Raid-10

    root@april:/home/vlap# mdadm --create --verbose /dev/md0 -l 10 -n 4 /dev/sd{b,c,d,e}
    mdadm: layout defaults to n2
    mdadm: layout defaults to n2
    mdadm: chunk size defaults to 512K
    mdadm: size set to 1046528K
    mdadm: Defaulting to version 1.2 metadata
    mdadm: array /dev/md0 started.

## Проверим что RAID собрался нормально

    root@april:/home/vlap# cat /proc/mdstat
    Personalities : [raid0] [raid1] [raid4] [raid5] [raid6] [raid10] [linear] 
    md0 : active raid10 sde[3] sdd[2] sdc[1] sdb[0]
        2093056 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
        
    unused devices: <none>  

###
    root@april:/home/vlap# mdadm -D /dev/md0
    /dev/md0:
          ...  
        
            Raid Level : raid10
          ...

        Number   Major   Minor   RaidDevice State
        0       8       16        0      active sync set-A   /dev/sdb
        1       8       32        1      active sync set-B   /dev/sdc
        2       8       48        2      active sync set-A   /dev/sdd
        3       8       64        3      active sync set-B   /dev/sde  

## Сломаем и починим RAID  

“Зафейлим” одно из блочных устройств:

    root@april:/home/vlap# mdadm /dev/md0 --fail /dev/sde
    mdadm: set /dev/sde faulty in /dev/md0  

Посмотрим, как это отразилось на RAID:

    root@april:/home/vlap# cat /proc/mdstat
    Personalities : [raid0] [raid1] [raid4] [raid5] [raid6] [raid10] [linear] 
    md0 : active raid10 sde[3](F) sdd[2] sdc[1] sdb[0]
        2093056 blocks super 1.2 512K chunks 2 near-copies [4/3] [UUU_]
        
    unused devices: <none>

###

    root@april:/home/vlap# mdadm -D /dev/md0
    /dev/md0:
           ...

        Number   Major   Minor   RaidDevice State
        0       8       16        0      active sync set-A   /dev/sdb
        1       8       32        1      active sync set-B   /dev/sdc
        2       8       48        2      active sync set-A   /dev/sdd
        -       0        0        3      removed

        3       8       64        -      faulty   /dev/sde

Удалим “сломанный” диск из массива:  

    root@april:/home/vlap# mdadm /dev/md0 --remove /dev/sde
    mdadm: hot removed /dev/sde from /dev/md0  

"Вставим" новый диск :  

    root@april:/home/vlap# mdadm /dev/md0 --add /dev/sde
    mdadm: added /dev/sde

###

Смотрим процесс ребилда :  


    root@april:/home/vlap# cat /proc/mdstat
    Personalities : [raid0] [raid1] [raid4] [raid5] [raid6] [raid10] [linear] 
    md0 : active raid10 sde[4] sdd[2] sdc[1] sdb[0]
        2093056 blocks super 1.2 512K chunks 2 near-copies [4/3] [UUU_]
        [===================>.]  recovery = 98.9% (1035968/1046528) finish=0.0min speed=207193K/sec
        
    unused devices: <none>  

###

    root@april:/home/vlap# mdadm -D /dev/md0
    /dev/md0:
            ...

        Number   Major   Minor   RaidDevice State
        0       8       16        0      active sync set-A   /dev/sdb
        1       8       32        1      active sync set-B   /dev/sdc
        2       8       48        2      active sync set-A   /dev/sdd
        4       8       64        3      active sync set-B   /dev/sde

## Создадим GPT таблицу, пять разделов и смонтируем их в системе

Создаем раздел GPT на RAID:

    root@april:/home/vlap# parted -s /dev/md0 mklabel gpt

###

Создаем партиции:

    root@april:/home/vlap# parted /dev/md0 mkpart primary ext4 0% 20%
    Information: You may need to update /etc/fstab.

    root@april:/home/vlap# parted /dev/md0 mkpart primary ext4 20% 40%        
    Information: You may need to update /etc/fstab.

    root@april:/home/vlap# parted /dev/md0 mkpart primary ext4 40% 60%        
    Information: You may need to update /etc/fstab.

    root@april:/home/vlap# parted /dev/md0 mkpart primary ext4 60% 80%        
    Information: You may need to update /etc/fstab.

    root@april:/home/vlap# parted /dev/md0 mkpart primary ext4 80% 100%       
    Information: You may need to update /etc/fstab.

###

Cоздаём на этих партициях ФС:

    root@april:/home/vlap# for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
    mke2fs 1.47.0 (5-Feb-2023)
    Creating filesystem with 104448 4k blocks and 104448 inodes
    Filesystem UUID: 0d2e82d3-af01-4c69-a51d-5c3240416dbc
    Superblock backups stored on blocks: 
        32768, 98304

    Allocating group tables: done                            
    Writing inode tables: done                            
    Creating journal (4096 blocks): done
    Writing superblocks and filesystem accounting information: done
    ...   

Смонтируем их по каталогам:

    root@april:/home/vlap# mkdir -p /raid/part{1,2,3,4,5}
    root@april:/home/vlap# for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done

## Создаём конфигурационный файл mdadm.conf  

Проверим информацию:

    root@april:/home/vlap# mdadm --detail --scan --verbose
    ARRAY /dev/md0 level=raid10 num-devices=4 metadata=1.2 UUID=6175f02a:ca566018:75d2e03b:e91657d4
    devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde

Создадим mdadm.conf:

    root@april:/home/vlap# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
    root@april:/home/vlap# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}'
    ARRAY /dev/md0 level=raid10 num-devices=4 metadata=1.2 UUID=6175f02a:ca566018:75d2e03b:e91657d4
    root@april:/home/vlap# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

## Пишем баш скрипт для конфигурации рейда  

Просто обьединяем команды по созданию рейда и mdadm.conf файла в баш скрипт [script_raid.sh](https://github.com/vlapvg/mdadm_raid/blob/main/script_raid.sh))

### Задание выполнено.
