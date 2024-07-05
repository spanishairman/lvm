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

### Provisioning. 
### Установка пакетов, необходимых для дальнейшей работы
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
### Работа с логическими томами
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
