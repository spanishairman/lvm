## Дисковая подсистема. lvm
### Подготовка окружения
В нашем примере используется гипервизор Qemu-KVM, библиотека Libvirt. В качестве хостовой системы - OpenSuse Leap 15.5.

Для работы Vagrant с Libvirt установлен пакет vagrant-libvirt:
```
Сведения — пакет vagrant-libvirt:
---------------------------------
Репозиторий            : Основной репозиторий
Имя                    : vagrant-libvirt
Версия                 : 0.10.2-bp155.1.19
Архитектура            : x86_64
Поставщик              : openSUSE
Размер после установки : 658,3 KiB
Установлено            : Да
Состояние              : актуален
Пакет с исходным кодом : vagrant-libvirt-0.10.2-bp155.1.19.src
Адрес источника        : https://github.com/vagrant-libvirt/vagrant-libvirt
Заключение             : Провайдер Vagrant для libvirt
Описание               : 

    This is a Vagrant plugin that adds a Libvirt provider to Vagrant, allowing
    Vagrant to control and provision machines via the Libvirt toolkit.
```
Пакет Vagrant также устанавливаем из репозиториев. Текущая версия для OpenSuse Leap 15.5:
```
max@localhost:~/vagrant/vg3> vagrant -v
Vagrant 2.2.18
```
Образ операционной системы создан заранее, для этого установлен [Debian Linux из официального образа netinst](https://www.debian.org/distrib/netinst)

### Особенности работы Vagrant, установленного из официального репозитория OpenSuse Leap 15.5, с дополнительными дисковыми устройствами в гостевой ОС
Для добавления дисков в гостевой системе, используем следующий код в [Vagrantfile](Vagrantfile):
```
  config.vm.provider "libvirt" do |lv|
    lv.memory = "2048"
    lv.cpus = "2"
    lv.title = "Debian12"
    lv.description = "Виртуальная машина на базе дистрибутива Debian Linux"
    lv.management_network_name = "vagrant-libvirt-mgmt"
    lv.management_network_address = "192.168.121.0/24"
    lv.management_network_keep = "true"
    lv.management_network_mac = "52:54:00:27:28:83"
    lv.storage :file, :size => '1G', :device => 'vdc', :allow_existing => false
    lv.storage :file, :size => '1G', :device => 'vdd', :allow_existing => false
    lv.storage :file, :size => '1G', :device => 'vde', :allow_existing => false
    lv.storage :file, :size => '1G', :device => 'vdf', :allow_existing => false
    lv.storage :file, :size => '1G', :device => 'vdg', :allow_existing => false
  end
```
Здесь мы добавляем в нашу гостевую систему пять дисковых устройств, являющихся файлами в системе хоста - это vdc, vdd, vde, vdf и vdg. В случае, если не задавать явно имена устройств
то Vagrant сам назначит их по латинскому алфавиту с префиксом vd, начиная с первого свободного символа. Например, если в гостевой системе после установки используется устройство
/dev/vda, то первое дополнительное дисковое устройство будет /dev/vdb.

### Provisioning
#### Установка пакетов, необходимых для дальнейшей работы
Нам потребуются два дополнительных пакета - rsync и arch-install-scripts. Первый - для копирования содержимого разделов /var и /home на новые файловые системы, второй - для генерирования содержимого /etc/fstab.
```
  config.vm.provision "shell", inline: <<-SHELL
    brd='*************************************************************'
    echo "$brd"
    echo 'Установим пакет rsync'
    echo "$brd"
  # apt update
    apt install -y rsync arch-install-scripts
```
#### Работа с логическими томами
Добавим физические тома и группы томов
```
    pvcreate /dev/vdb
    vgcreate data /dev/vdb
    vgdisplay data
    Debian12:   --- Volume group ---
    Debian12:   VG Name               data
    Debian12:   System ID
    Debian12:   Format                lvm2
    Debian12:   Metadata Areas        1
    Debian12:   Metadata Sequence No  1
    Debian12:   VG Access             read/write
    Debian12:   VG Status             resizable
    Debian12:   MAX LV                0
    Debian12:   Cur LV                0
    Debian12:   Open LV               0
    Debian12:   Max PV                0
    Debian12:   Cur PV                1
    Debian12:   Act PV                1
    Debian12:   VG Size               <20,00 GiB
    Debian12:   PE Size               4,00 MiB
    Debian12:   Total PE              5119
    Debian12:   Alloc PE / Size       0 / 0
    Debian12:   Free  PE / Size       5119 / <20,00 GiB
    Debian12:   VG UUID               CoAH86-5iyF-iXqW-0n56-w23v-5VWC-VKm013
```
Создадим логические тома
```
    lvcreate -L8G -n large data
    lvcreate -L100M -n small data
```
Создадим файловую систему на первом томе и смонтируем её в каталог /data
```
    mkfs.ext4 /dev/mapper/data-large
    mount --mkdir /dev/mapper/data-large /data/

```
Создадим второй физический том
```
    pvcreate /dev/vdc
```
Расширим группу томов за счёт второго физического тома
```
    vgextend data /dev/vdc
```
Посмотрим информацию о группе томов
```
    Debian12:   --- Volume group ---
    Debian12:   VG Name               data
    Debian12:   System ID
    Debian12:   Format                lvm2
    Debian12:   Metadata Areas        2
    Debian12:   Metadata Sequence No  4
    Debian12:   VG Access             read/write
    Debian12:   VG Status             resizable
    Debian12:   MAX LV                0
    Debian12:   Cur LV                2
    Debian12:   Open LV               1
    Debian12:   Max PV                0
    Debian12:   Cur PV                2
    Debian12:   Act PV                2
    Debian12:   VG Size               23,99 GiB
    Debian12:   PE Size               4,00 MiB
    Debian12:   Total PE              6142
    Debian12:   Alloc PE / Size       2073 / <8,10 GiB
    Debian12:   Free  PE / Size       4069 / 15,89 GiB
    Debian12:   VG UUID               CoAH86-5iyF-iXqW-0n56-w23v-5VWC-VKm013
```
Выведем информацию о том, какие диски входят в VG'
```
    vgdisplay -v data | grep 'PV Name'
    Debian12:   PV Name               /dev/vdb
    Debian12:   PV Name               /dev/vdc
```
Информация о логическом томе
```
    lvdisplay /dev/mapper/data-large
    Debian12:   --- Logical volume ---
    Debian12:   LV Path                /dev/data/large
    Debian12:   LV Name                large
    Debian12:   VG Name                data
    Debian12:   LV UUID                DOOvWu-QPH2-JQjA-iSZE-8U8w-WA9Z-XnAXKE
    Debian12:   LV Write Access        read/write
    Debian12:   LV Creation host, time debian12, 2024-07-05 09:55:30 +0300
    Debian12:   LV Status              available
    Debian12:   # open                 1
    Debian12:   LV Size                8,00 GiB
    Debian12:   Current LE             2048
    Debian12:   Segments               1
    Debian12:   Allocation             inherit
    Debian12:   Read ahead sectors     auto
    Debian12:   - currently set to     256
    Debian12:   Block device           253:3
```
Краткий вывод информации о группах томов и о логических томах
```
    vgs && lvs
    Debian12:   VG        #PV #LV #SN Attr   VSize  VFree
    Debian12:   data        2   2   0 wz--n- 23,99g 15,89g
    Debian12:   vg-system   1   3   0 wz--n- 39,06g     0
    Debian12:   LV       VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    Debian12:   large    data      -wi-ao----   8,00g
    Debian12:   small    data      -wi-a----- 100,00m
    Debian12:   lvolhome vg-system -wi-ao----  16,71g
    Debian12:   lvolroot vg-system -wi-ao----  18,62g
    Debian12:   lvolswap vg-system -wi-ao----   3,72g
```
Информация о монтируемом томе в каталог /data'
```
    mount | grep /data
    Debian12: /dev/mapper/data-large on /data type ext4 (rw,relatime)
```
#### Изменение размера логического тома и файловой системы н нём
Создадим большой файл размером 8Gb и проверим наличие места на диске. Используем команду "dd if=/dev/zero of=/data/test.log bs=1M count=8000 status=progress", для просмотра свободного места - "df -Th /data/"
```
    Debian12: 8232370176 bytes (8,2 GB, 7,7 GiB) copied, 28 s, 291 MB/s
    Debian12: dd: ошибка записи '/data/test.log': На устройстве не осталось свободного места
    Debian12: 7948+0 records in
    Debian12: 7947+0 records out
    Debian12: 8333492224 bytes (8,3 GB, 7,8 GiB) copied, 28,4922 s, 292 MB/s

    Debian12: Файловая система       Тип  Размер Использовано  Дост Использовано% Cмонтировано в
    Debian12: /dev/mapper/data-large ext4   7,8G         7,8G     0          100% /data
```
Расширим том large
```
    lvextend -L12G /dev/mapper/data-large
    Debian12:   Size of logical volume data/large changed from 8,00 GiB (2048 extents) to 12,00 GiB (3072 extents).
    Debian12:   Logical volume data/large successfully resized.
```
Выведем информацию о томе
```
    lvs /dev/mapper/data-large
    Debian12:   LV    VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    Debian12:   large data -wi-ao---- 12,00g
```
И о свободном месте на файловой системе
```
    df -Th /data
    Debian12: Файловая система       Тип  Размер Использовано  Дост Использовано% Cмонтировано в
    Debian12: /dev/mapper/data-large ext4   7,8G         7,8G     0          100% /data
```
Здесь мы видим, что места на файловой системе не осталось, занято 100%, при этом логический том занят не полностью и его размер - 12Гб.

Увеличим размер ФС
```
    resize2fs /dev/mapper/data-large
    Debian12: resize2fs 1.47.0 (5-Feb-2023)
    Debian12: Filesystem at /dev/mapper/data-large is mounted on /data; on-line resizing required
    Debian12: old_desc_blocks = 1, new_desc_blocks = 2
    Debian12: The filesystem on /dev/mapper/data-large is now 3145728 (4k) blocks long.
```
Снова выведем информацию о свободном месте
```
    df -Th /data
    Debian12: Файловая система       Тип  Размер Использовано  Дост Использовано% Cмонтировано в
    Debian12: /dev/mapper/data-large ext4    12G         7,8G  3,4G           70% /data
```
Теперь уменьшим файловую систему и том, на котором она создана
```
    umount /data/

    e2fsck -fy /dev/mapper/data-large
    Debian12: e2fsck 1.47.0 (5-Feb-2023)
    Debian12: Pass 1: Checking inodes, blocks, and sizes
    Debian12: Pass 2: Checking directory structure   
    Debian12: Pass 3: Checking directory connectivity
    Debian12: Pass 4: Checking reference counts
    Debian12: Pass 5: Checking group summary information
    Debian12: /dev/mapper/data-large: 12/786432 files (0.0% non-contiguous), 2110529/3145728 blocks

    resize2fs /dev/mapper/data-large 10G
    Debian12: resize2fs 1.47.0 (5-Feb-2023)
    Debian12: Resizing the filesystem on /dev/mapper/data-large to 2621440 (4k) blocks.
    Debian12: The filesystem on /dev/mapper/data-large is now 2621440 (4k) blocks long.

    lvreduce -y /dev/mapper/data-large -L 10G
    Debian12:   WARNING: Reducing active logical volume to 10,00 GiB.
    Debian12:   THIS MAY DESTROY YOUR DATA (filesystem etc.)
    Debian12:   Size of logical volume data/large changed from 12,00 GiB (3072 extents) to 10,00 GiB (2560 extents).
    Debian12:   Logical volume data/large successfully resized.
```
Снова смонтируем фс и сверим её размер и размер тома, на котором она создана
```
    mount /dev/mapper/data-large /data/

    df -Th /data/
    Debian12: Файловая система       Тип  Размер Использовано  Дост Использовано% Cмонтировано в
    Debian12: /dev/mapper/data-large ext4   9,8G         7,8G  1,6G           84% /data

    lvs /dev/mapper/data-large
    Debian12:   LV    VG   Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    Debian12:   large data -wi-ao---- 10,00g
```
#### Работа со снимками логических томов
Создадим снимок тома large
```
    lvcreate -L 500M --snapshot -n large-snap /dev/mapper/data-large
    Debian12:   Logical volume "large-snap" created.
```
Выведем содержимое /data
```
    ls -l /data
    Debian12: итого 8138196
    Debian12: drwx------ 2 root root      16384 июл  5 09:55 lost+found
    Debian12: -rw-r--r-- 1 root root 8333492224 июл  5 09:55 test.log
```
Удалим файл test.log и выведем содержимое /data
```
    rm /data/test.log
    ls -l /data
    Debian12: итого 16
    Debian12: drwx------ 2 root root 16384 июл  5 09:55 lost+found
```
Отмонтируем /data и восстановим том large из снимка
```
    umount /data
    lvconvert --merge /dev/mapper/data-large--snap
    Debian12:   Merging of volume data/large-snap started.
    Debian12:   data/large: Merged: 100,00%
```
Снова смонтируем том и проверим содержимое /data
```
    mount /dev/mapper/data-large /data/
    df -Th /data/
    Debian12: итого 8138196
    Debian12: drwx------ 2 root root      16384 июл  5 09:55 lost+found
    Debian12: -rw-r--r-- 1 root root 8333492224 июл  5 09:55 test.log

```
Удалим все логические тома в группе data
```
    lvremove -y data
    Debian12:   Logical volume "large" successfully removed.
    Debian12:   Logical volume "small" successfully removed.
```
Удалим физический том из группы data
```
    vgreduce -y data /dev/vdc
    Debian12:   Removed "/dev/vdc" from volume group "data"
```
#### Перенос каталогов /var, /home на lvm raid
Создадим новые физические тома для логических томов lvvar и lvhome
```
    pvcreate /dev/vdd /dev/vde
    Debian12:   Physical volume "/dev/vdd" successfully created.
    Debian12:   Physical volume "/dev/vde" successfully created.
```
Расширим системную группу томов vg-system, здесь устройства vdc и vdd будут использоваться для создания массива, а vde для возможности делать снимки томов
```
    vgextend vg-system /dev/vdc /dev/vdd /dev/vde
    Debian12:   Volume group "vg-system" successfully extended
```
Создадим логические тома lvvar и lvhome тип RAID1 на физических устройствах /dev/vdc и /dev/vdd
```
    lvcreate --type raid1 --mirrors 1 -l 50%PVS -n lvvar vg-system /dev/vdc /dev/vdd
    Debian12:   Logical volume "lvvar" created.
    lvcreate --type raid1 --mirrors 1 -l 100%PVS -n lvhome vg-system /dev/vdc /dev/vdd
    Debian12:   Logical volume "lvhome" created.
```
> Здесь, при создании логического тома lvvar, мы использовали 50% свободного места **физического устройства** (ключ 50%PVS), 
> на котором размещается логический том, т.е. при размере устройств vdc и vdd - 4Gb, логический том lvvar будет ~2Gb. 
> При создании тома lvhome мы уже используем 100% оставшегося свободного места на **физическом устройстве**, т.е. тоже ~2Gb,
> напомню, 50% места перед этим занял том lvvar 

Создадим ФС на томе
```
    mkfs.ext4 /dev/mapper/vg--system-lvvar
    Debian12: mke2fs 1.47.0 (5-Feb-2023)
    Debian12: Discarding device blocks: done         
    Debian12: Creating filesystem with 523264 4k blocks and 130816 inodes
    Debian12: Filesystem UUID: 9d104c46-c198-4823-88cf-98e0f7a28bab
    Debian12: Superblock backups stored on blocks:
    Debian12:   32768, 98304, 163840, 229376, 294912
    Debian12: 
    Debian12: Allocating group tables: done 
    Debian12: Writing inode tables: done 
    Debian12: Creating journal (8192 blocks): done
    Debian12: Writing superblocks and filesystem accounting information: done 
    Debian12:
    mkfs.ext4 /dev/mapper/vg--system-lvhome
    Debian12: mke2fs 1.47.0 (5-Feb-2023)
    Debian12: Discarding device blocks: done         
    Debian12: Creating filesystem with 522240 4k blocks and 130560 inodes
    Debian12: Filesystem UUID: 1dd36192-f8e1-4d90-9cd3-d1f2b63a6b37
    Debian12: Superblock backups stored on blocks:
    Debian12:   32768, 98304, 163840, 229376, 294912
    Debian12: 
    Debian12: Allocating group tables: done 
    Debian12: Writing inode tables: done 
    Debian12: Creating journal (8192 blocks): done
    Debian12: Writing superblocks and filesystem accounting information: done 
    Debian12:
```
Смонтируем его во временную директорию
```
    mount --mkdir /dev/mapper/vg--system-lvvar /lvvar
    mount --mkdir /dev/mapper/vg--system-lvhome /lvhome
```
И скопируем содержимое каталогов /var и /home на эти тома
```
    rsync -a /var/ /lvvar
    rsync -a /home/ /lvhome
```
Отредактируем /etc/fstab для последующего монтирования тома lvvar
```
    sed -i 's/vg--system-lvolhome/vg--system-lvhome/' /etc/fstab
    genfstab -L / | grep "lvvar" >> /etc/fstab
```
Создадим новый логический том lvroot для временного размещения корневой файловой системы
```
    lvcreate -l+100%FREE -n lvroot -y data
    Debian12:   Wiping ext4 signature on /dev/data/lvroot.
    Debian12:   Logical volume "lvroot" created.
```
Сделаем дамп действующей корневой файловой системы во временную
```
    dd if=/dev/mapper/vg--system-lvolroot of=/dev/mapper/data-lvroot bs=4M
```
Смонтируем том lvroot в каталог /lvroot
```
    mount --mkdir /dev/mapper/data-lvroot /lvroot
```
Исправим точку монтирования корневой ФС в файле fstab этой ФС
```
    sed -i 's/vg--system-lvolroot/data-lvroot/' /lvroot/etc/fstab
    grep "data-lvroot" /lvroot/etc/fstab
    Debian12: /dev/mapper/data-lvroot /               ext4    errors=remount-ro 0       1
```
#### Уменьшение размера горневого раздела
Все приведённые выше действия выполнились автоматически с помощью ***config.vm.provision*** файла [Vagrantfile](Vagrantfile)

Далее все действия производим в консоли виртуальной машины, после подключения к ней с помощью команды `vagrant ssh`
```
max@localhost:~/vagrant/vg3> vagrant ssh
Linux debian12 6.1.0-21-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.90-1 (2024-05-03) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon May 20 18:30:50 2024 from 192.168.122.1
```
Сравним файлы ***fstab*** в текущей корневой файловой системе
```
root@debian12:~# cat /etc/fstab 
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/vg--system-lvolroot /               ext4    errors=remount-ro 0       1
# /boot was on /dev/vda1 during installation
UUID=4a52c237-1e75-40f0-944d-d712cd385901 /boot           ext2    defaults        0       2
/dev/mapper/vg--system-lvhome /home           ext4    defaults        0       2
/dev/mapper/vg--system-lvolswap none            swap    sw              0       0
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
# /dev/mapper/vg--system-lvvar UUID=9d104c46-c198-4823-88cf-98e0f7a28bab
/dev/mapper/vg--system-lvvar    /lvvar          ext4            rw,relatime     0 2
```
И во временной - /lvroot
```
root@debian12:~# cat /lvroot/etc/fstab 
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/data-lvroot /               ext4    errors=remount-ro 0       1
# /boot was on /dev/vda1 during installation
UUID=4a52c237-1e75-40f0-944d-d712cd385901 /boot           ext2    defaults        0       2
/dev/mapper/vg--system-lvhome /home           ext4    defaults        0       2
/dev/mapper/vg--system-lvolswap none            swap    sw              0       0
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
# /dev/mapper/vg--system-lvvar UUID=9d104c46-c198-4823-88cf-98e0f7a28bab
/dev/mapper/vg--system-lvvar    /lvvar          ext4            rw,relatime     0 2
```
Нас интересует строка c блочным устройством для монтирования корня файловой системы
```/dev/mapper/vg--system-lvolroot /               ext4    errors=remount-ro 0       1
```
в первом случае и
```/dev/mapper/data-lvroot /               ext4    errors=remount-ro 0       1
```
во втором.
Посмотрим размеры логических томов
```
root@debian12:~# lvs
  LV       VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lvroot   data      -wi-ao---- <20,00g                                                    
  lvhome   vg-system rwi-aor---   1,99g                                    100,00          
  lvolhome vg-system -wi-ao----  16,71g                                                    
  lvolroot vg-system -wi-ao----  18,62g                                                    
  lvolswap vg-system -wi-ao----   3,72g                                                    
  lvvar    vg-system rwi-aor---  <2,00g                                    100,00
```

Проверим точки монтирования устройств LVM
```
root@debian12:~# mount | grep "mapper"
/dev/mapper/vg--system-lvolroot on / type ext4 (rw,relatime,errors=remount-ro)
/dev/mapper/vg--system-lvolhome on /home type ext4 (rw,relatime)
/dev/mapper/vg--system-lvvar on /lvvar type ext4 (rw,relatime)
/dev/mapper/vg--system-lvhome on /lvhome type ext4 (rw,relatime)
/dev/mapper/data-lvroot on /lvroot type ext4 (rw,relatime)
```
А также свободное место на файловых системах
```
root@debian12:~# df -h
Файловая система                Размер Использовано  Дост Использовано% Cмонтировано в
udev                              952M            0  952M            0% /dev
tmpfs                             197M         1,2M  196M            1% /run
/dev/mapper/vg--system-lvolroot    19G         5,0G   13G           29% /
tmpfs                             984M            0  984M            0% /dev/shm
tmpfs                             5,0M            0  5,0M            0% /run/lock
/dev/vda1                         937M         117M  773M           14% /boot
/dev/mapper/vg--system-lvolhome    17G          13M   16G            1% /home
tmpfs                             197M          60K  197M            1% /run/user/113
tmpfs                             197M          52K  197M            1% /run/user/1000
/dev/mapper/vg--system-lvvar      2,0G         406M  1,5G           22% /lvvar
/dev/mapper/vg--system-lvhome     2,0G          13M  1,8G            1% /lvhome
/dev/mapper/data-lvroot            19G         5,0G   13G           29% /lvroot
```
Переходим во временное окружение __chroot__
```
root@debian12:~# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /lvroot/$i; done
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
root@debian12:~# chroot /lvroot/
```
Сохраняем копию конфигурационного файла загрузчика
```
root@debian12:/# cp -p /boot/grub/grub.cfg /boot/grub/grub.cfg.bk
```
Генерируем новый конфиг из временного окружения
```
root@debian12:/# grub-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found background image: /usr/share/images/desktop-base/desktop-grub.png
Found linux image: /boot/vmlinuz-6.1.0-21-amd64
Found initrd image: /boot/initrd.img-6.1.0-21-amd64
Found linux image: /boot/vmlinuz-6.1.0-18-amd64
Found initrd image: /boot/initrd.img-6.1.0-18-amd64
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
done
```
Проверяем, что в новом файле конфигурации __Grub__ пути к устройству с корневой файловой системой обновились
```
root@debian12:/# grep "mapper" /boot/grub/grub.cfg
        linux   /vmlinuz-6.1.0-21-amd64 root=/dev/mapper/data-lvroot ro  quiet
                linux   /vmlinuz-6.1.0-21-amd64 root=/dev/mapper/data-lvroot ro  quiet
                linux   /vmlinuz-6.1.0-21-amd64 root=/dev/mapper/data-lvroot ro single 
                linux   /vmlinuz-6.1.0-18-amd64 root=/dev/mapper/data-lvroot ro  quiet
                linux   /vmlinuz-6.1.0-18-amd64 root=/dev/mapper/data-lvroot ro single
```
Выходим из окружения __chroot__ и перезагружаемся
```
root@debian12:/# exit
exit
root@debian12:~# reboot
```
После перезагрузки, логинимся и проверяем точки монтирования системы
```
root@debian12:~# mount | grep "mapper"
/dev/mapper/data-lvroot on / type ext4 (rw,relatime,errors=remount-ro)
/dev/mapper/vg--system-lvhome on /home type ext4 (rw,relatime)
/dev/mapper/vg--system-lvvar on /lvvar type ext4 (rw,relatime)
```
Видим, что мы загрузились с временной корневой ФС. Так как основная корневая ФС сейчас не смонтирована, куменьшим её до 16Gb
```
root@debian12:~# e2fsck -f /dev/mapper/vg--system-lvolroot
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/mapper/vg--system-lvolroot: 169978/1220608 files (0.2% non-contiguous), 1420720/4882432 blocks
root@debian12:~# resize2fs /dev/mapper/vg--system-lvolroot 16G
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/mapper/vg--system-lvolroot to 4194304 (4k) blocks.
The filesystem on /dev/mapper/vg--system-lvolroot is now 4194304 (4k) blocks long.
```
Теперь можно уменьшит размер логического тома корневой файловой системы
```
root@debian12:~# lvreduce -y -L 16G /dev/mapper/vg--system-lvolroot
  WARNING: Reducing active logical volume to 16,00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
  Size of logical volume vg-system/lvolroot changed from 18,62 GiB (4768 extents) to 16,00 GiB (4096 extents).
  Logical volume vg-system/lvolroot successfully resized.

```
Вернём оригинальный конфиг __Grub__ из резервной копии
```
root@debian12:~# mv /boot/grub/grub.cfg.bk /boot/grub/grub.cfg
```
Проверим пути
```
root@debian12:~# grep "mapper" /boot/grub/grub.cfg
        linux   /vmlinuz-6.1.0-21-amd64 root=/dev/mapper/vg--system-lvolroot ro  quiet
                linux   /vmlinuz-6.1.0-21-amd64 root=/dev/mapper/vg--system-lvolroot ro  quiet
                linux   /vmlinuz-6.1.0-21-amd64 root=/dev/mapper/vg--system-lvolroot ro single 
                linux   /vmlinuz-6.1.0-18-amd64 root=/dev/mapper/vg--system-lvolroot ro  quiet
                linux   /vmlinuz-6.1.0-18-amd64 root=/dev/mapper/vg--system-lvolroot ro single 
```
Перезагружаемся обратно в основную корневую ФС
```
root@debian12:~# reboot 
Broadcast message from root@debian12 on pts/1 (Fri 2024-07-05 10:05:10 MSK):
The system will reboot now!
```
Логинимся и проверяем точки монтирования
```
root@debian12:~# Connection to 192.168.121.10 closed by remote host.
Connection to 192.168.121.10 closed.
max@localhost:~/vagrant/vg3> vagrant ssh
Linux debian12 6.1.0-21-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.90-1 (2024-05-03) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jul  5 09:58:56 2024 from 192.168.121.1
vagrant@debian12:~$ sudo -i
root@debian12:~# mount | grep "mapper"
/dev/mapper/vg--system-lvolroot on / type ext4 (rw,relatime,errors=remount-ro)
/dev/mapper/vg--system-lvhome on /home type ext4 (rw,relatime)
/dev/mapper/vg--system-lvvar on /lvvar type ext4 (rw,relatime)
```
Размеры логических томов
```
root@debian12:~# lvs
  LV       VG        Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lvroot   data      -wi-a----- <20,00g                                                    
  lvhome   vg-system rwi-aor---   1,99g                                    100,00          
  lvolhome vg-system -wi-a-----  16,71g                                                    
  lvolroot vg-system -wi-ao----  16,00g                                                    
  lvolswap vg-system -wi-ao----   3,72g                                                    
  lvvar    vg-system rwi-aor---  <2,00g                                    100,00          
```
Размеры файловых систем
```
root@debian12:~# df -h
Файловая система                Размер Использовано  Дост Использовано% Cмонтировано в
udev                              952M            0  952M            0% /dev
tmpfs                             197M         1,2M  196M            1% /run
/dev/mapper/vg--system-lvolroot    16G         5,0G  9,9G           34% /
tmpfs                             984M            0  984M            0% /dev/shm
tmpfs                             5,0M            0  5,0M            0% /run/lock
/dev/vda1                         937M         117M  773M           14% /boot
/dev/mapper/vg--system-lvhome     2,0G          13M  1,8G            1% /home
/dev/mapper/vg--system-lvvar      2,0G         406M  1,5G           22% /lvvar
tmpfs                             197M          60K  197M            1% /run/user/113
tmpfs                             197M          52K  197M            1% /run/user/1000
```
Видим, что размер логического тома __lvolroot__ и файловой системы уменьшились до 16Gb. Готово! :+1:

К данной работе прилагаю также запись консоли. Для того, чтобы воспроизвести выполненные действия, 
необходимо скачать файлы [screenrecord-2024-07-05.script](screenrecord-2024-07-05.script) и [screenrecord-2024-07-05.time](screenrecord-2024-07-05.time), 
после чего выполнить в каталоге с загруженными файлами команду
```
scriptreplay ./screenrecord-2024-06-07.time ./screenrecord-2024-06-07.script
```

Спасибо за прочтение! :potted_plant:
