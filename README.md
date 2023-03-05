# Домашнее задание к занятию «Файловые системы»

### Цель задания

В результате выполнения задания вы: 

* научитесь работать с инструментами разметки жёстких дисков, виртуальных разделов — RAID-массивами и логическими томами, конфигурациями файловых систем. Основная задача — понять, какие слои абстракций могут нас отделять от файловой системы до железа. Обычно инженер инфраструктуры не сталкивается напрямую с настройкой LVM или RAID, но иметь понимание, как это работает, необходимо;
* создадите нештатную ситуацию работы жёстких дисков и поймёте, как система RAID обеспечивает отказоустойчивую работу.


### Чеклист готовности к домашнему заданию

1. Убедитесь, что у вас на новой виртуальной машине  установлены утилиты: `mdadm`, `fdisk`, `sfdisk`, `mkfs`, `lsblk`, `wget` (шаг 3 в задании).  
2. Воспользуйтесь пакетным менеджером apt для установки необходимых инструментов.


### Дополнительные материалы для выполнения задания

1. Разряженные файлы — [sparse](https://ru.wikipedia.org/wiki/%D0%A0%D0%B0%D0%B7%D1%80%D0%B5%D0%B6%D1%91%D0%BD%D0%BD%D1%8B%D0%B9_%D1%84%D0%B0%D0%B9%D0%BB).
2. [Подробный анализ производительности RAID](https://www.baarf.dk/BAARF/0.Millsap1996.08.21-VLDB.pdf), страницы 3–19.
3. [RAID5 write hole](https://www.intel.com/content/www/us/en/support/articles/000057368/memory-and-storage.html).


------

## Задание

1. Узнайте о [sparse-файлах](https://ru.wikipedia.org/wiki/%D0%A0%D0%B0%D0%B7%D1%80%D0%B5%D0%B6%D1%91%D0%BD%D0%BD%D1%8B%D0%B9_%D1%84%D0%B0%D0%B9%D0%BB) (разряженных).


#Ответ:
Узнал. Применяются для хранения контейнеров, резервных копий, файлов в файлообменных сетях.

2. Могут ли файлы, являющиеся жёсткой ссылкой на один объект, иметь разные права доступа и владельца? Почему?


 #Ответ: Нет, не могут. Из-за того что hardlink это ссылка на тот же самый файл и имеет тот же inode - права и владелец будут одни. права доступа выдаются на inode


3. Сделайте `vagrant destroy` на имеющийся инстанс Ubuntu. Замените содержимое Vagrantfile следующим:
```
Сделал.
vagrant@vagrant:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0 55.4M  1 loop /snap/core18/2128
loop1                       7:1    0 32.3M  1 loop /snap/snapd/12704
loop2                       7:2    0 70.3M  1 loop /snap/lxd/21029
loop3                       7:3    0 55.5M  1 loop /snap/core18/2284
loop4                       7:4    0 43.4M  1 loop /snap/snapd/14549
sda                         8:0    0   64G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   63G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0 31.5G  0 lvm  /
sdb                         8:16   0  2.5G  0 disk
sdc                         8:32   0  2.5G  0 disk
```
4. Используя ```fdisk```, разбейте первый диск на 2 раздела: 2 Гб, оставшееся пространство.
```
root@vagrant:~# fdisk -l /dev/sdb
Disk /dev/sdb: 2.51 GiB, 2684354560 bytes, 5242880 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x610becf6

Device     Boot   Start     End Sectors  Size Id Type
/dev/sdb1          2048 4196351 4194304    2G 83 Linux
/dev/sdb2       4196352 5242879 1046528  511M 83 Linux
```
5. Используя ```sfdisk```, перенесите данную таблицу разделов на второй диск.
```
root@vagrant:~# sfdisk -d /dev/sdb | sfdisk --force /dev/sdc
Checking that no-one is using this disk right now ... OK
```
6. Соберите mdadm RAID1 на паре разделов 2 Гб.
```
root@vagrant:~# mdadm --create --verbose /dev/md0 --level 1 -n 2 /dev/sdb1 /dev/sdc1
```
7. Соберите mdadm RAID0 на второй паре маленьких разделов.
```
mdadm --create --verbose /dev/md1 --level 0 -n 2 /dev/sdb2 /dev/sdc2
```
```
echo "DEVICE partitions" >> /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```
8. Создайте 2 независимых PV на получившихся md-устройствах.
```
root@vagrant:~# pvcreate /dev/md0
root@vagrant:~# pvcreate /dev/md1
```
9. Создайте общую volume-group на этих двух PV.
```
vgcreate vg0 /dev/md0 /dev/md1
```
10. Создайте LV размером 100 Мб, указав его расположение на PV с RAID0.
```
root@vagrant:~# lvcreate -n lv-test -L100M vg0 /dev/md1
```
11. Создайте mkfs.ext4 ФС на получившемся LV.
```
root@vagrant:~# mkfs.ext4 /dev/vg0/lv-test

root@vagrant:~# blkid | grep vg0
/dev/mapper/vg0-lv--test: UUID="e7dd4eee-52ad-41ca-82df-158dd5336627" TYPE="ext4"
```
12. Смонтируйте этот раздел в любую директорию, например, /tmp/new.
```
root@vagrant:~# mkdir /tmp/new

root@vagrant:~# tail -n1 /etc/fstab
UUID=e7dd4eee-52ad-41ca-82df-158dd5336627 /tmp/new ext4 defaults 0 1

root@vagrant:~# mount -a
```
13. Поместите туда тестовый файл, например ```wget https://mirror.yandex.ru/ubuntu/ls-lR.gz -O /tmp/new/test.gz```.
```
root@vagrant:~# ls /tmp/new
lost+found  test.gz
```
14. Прикрепите вывод lsblk
```
root@vagrant:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
loop0                       7:0    0 55.4M  1 loop  /snap/core18/2128
loop1                       7:1    0 32.3M  1 loop  /snap/snapd/12704
loop2                       7:2    0 70.3M  1 loop  /snap/lxd/21029
loop3                       7:3    0 55.5M  1 loop  /snap/core18/2284
loop4                       7:4    0 43.4M  1 loop  /snap/snapd/14549
loop5                       7:5    0 61.9M  1 loop  /snap/core20/1270
loop6                       7:6    0 67.2M  1 loop  /snap/lxd/21835
sda                         8:0    0   64G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part  /boot
└─sda3                      8:3    0   63G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0 31.5G  0 lvm   /
sdb                         8:16   0  2.5G  0 disk
├─sdb1                      8:17   0    2G  0 part
│ └─md0                     9:0    0    2G  0 raid1
└─sdb2                      8:18   0  511M  0 part
  └─md1                     9:1    0 1018M  0 raid0
    └─vg0-lv--test        253:1    0  100M  0 lvm   /tmp/new
sdc                         8:32   0  2.5G  0 disk
├─sdc1                      8:33   0    2G  0 part
│ └─md0                     9:0    0    2G  0 raid1
└─sdc2                      8:34   0  511M  0 part
  └─md1                     9:1    0 1018M  0 raid0
    └─vg0-lv--test        253:1    0  100M  0 lvm   /tmp/new
```
15. Протестируйте целостность файла:
```
root@vagrant:~# gzip -t /tmp/new/test.gz
root@vagrant:~# echo $?
0
```
```
Такой же вывод - 0
```
16. Используя pvmove, переместите содержимое PV с RAID0 на RAID1.
```
root@vagrant:~# pvmove /dev/md1
  /dev/md1: Moved: 32.00%
  /dev/md1: Moved: 100.00%
root@vagrant:~#
root@vagrant:~#
root@vagrant:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
loop0                       7:0    0 55.4M  1 loop  /snap/core18/2128
loop1                       7:1    0 32.3M  1 loop  /snap/snapd/12704
loop2                       7:2    0 70.3M  1 loop  /snap/lxd/21029
loop3                       7:3    0 55.5M  1 loop  /snap/core18/2284
loop4                       7:4    0 43.4M  1 loop  /snap/snapd/14549
loop5                       7:5    0 61.9M  1 loop  /snap/core20/1270
loop6                       7:6    0 67.2M  1 loop  /snap/lxd/21835
sda                         8:0    0   64G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part  /boot
└─sda3                      8:3    0   63G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0 31.5G  0 lvm   /
sdb                         8:16   0  2.5G  0 disk
├─sdb1                      8:17   0    2G  0 part
│ └─md0                     9:0    0    2G  0 raid1
│   └─vg0-lv--test        253:1    0  100M  0 lvm   /tmp/new
└─sdb2                      8:18   0  511M  0 part
  └─md1                     9:1    0 1018M  0 raid0
sdc                         8:32   0  2.5G  0 disk
├─sdc1                      8:33   0    2G  0 part
│ └─md0                     9:0    0    2G  0 raid1
│   └─vg0-lv--test        253:1    0  100M  0 lvm   /tmp/new
└─sdc2                      8:34   0  511M  0 part
  └─md1                     9:1    0 1018M  0 raid0
```
17. Сделайте ```--fail``` на устройство в вашем RAID1 md.
```
root@vagrant:~# mdadm /dev/md0 --fail /dev/sdb1
mdadm: set /dev/sdb1 faulty in /dev/md0
```
18. Подтвердите выводом ```dmesg```, что RAID1 работает в деградированном состоянии.
```
root@vagrant:~# dmesg -T | grep md0
[Tue Jan 25 11:40:32 2022] md/raid1:md0: not clean -- starting background reconstruction
[Tue Jan 25 11:40:32 2022] md/raid1:md0: active with 2 out of 2 mirrors
[Tue Jan 25 11:40:32 2022] md0: detected capacity change from 0 to 2144337920
[Tue Jan 25 11:40:32 2022] md: resync of RAID array md0
[Tue Jan 25 11:40:42 2022] md: md0: resync done.
[Tue Jan 25 12:16:55 2022] md/raid1:md0: Disk failure on sdb1, disabling device.
                           md/raid1:md0: Operation continuing on 1 devices.
```
19. Протестируйте целостность файла, несмотря на "сбойный" диск он должен продолжать быть доступен:
```
root@vagrant:~# gzip -t /tmp/new/test.gz
root@vagrant:~# echo $?
0
```
```
Такой же вывод - 0
```
20. Погасите тестовый хост, ```vagrant destroy```.
```
done!
```

 
*В качестве решения пришлите ответы на вопросы и опишите, как они были получены.*

----

### Правила приёма домашнего задания

В личном кабинете отправлена ссылка на .md-файл в вашем репозитории.


### Критерии оценки

Зачёт:

* выполнены все задания;
* ответы даны в развёрнутой форме;
* приложены соответствующие скриншоты и файлы проекта;
* в выполненных заданиях нет противоречий и нарушения логики.

На доработку:

* задание выполнено частично или не выполнено вообще;
* в логике выполнения заданий есть противоречия и существенные недостатки. 
