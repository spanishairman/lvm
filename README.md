### Дисковая подсистема. lvm
#### Подготовка окружения
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

#### Особенности работы Vagrant, установленного из официального репозитория OpenSuse Leap 15.5, с дополнительными дисковыми устройствами в гостевой ОС
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

#### Provisioning. 
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
