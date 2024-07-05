# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.
  config.vm.define "Debian12" do |srv|

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
    srv.vm.box = "/home/max/vagrant/images/debian12"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  #  config.vm.network "private_network", ip: "192.168.33.10"
  # config.vm.network "private_network", ip: "192.168.33.10", adapter: "5", type: "ip"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.
  # system("virsh net-create /home/max/vagrant/vagrant-libvirt-mgmt.xml")
  config.vm.provider "libvirt" do |lv|
    lv.memory = "2048"
    lv.cpus = "2"
    lv.title = "Debian12"
    lv.description = "Виртуальная машина на базе дистрибутива Debian Linux"
    lv.management_network_name = "vagrant-libvirt-mgmt"
    lv.management_network_address = "192.168.121.0/24"
    lv.management_network_keep = "true"
    lv.management_network_mac = "52:54:00:27:28:83"
    lv.storage :file, :size => '20G', :device => 'vdb', :allow_existing => false
    lv.storage :file, :size => '4G', :device => 'vdc', :allow_existing => false
    lv.storage :file, :size => '4G', :device => 'vdd', :allow_existing => false
    lv.storage :file, :size => '1G', :device => 'vde', :allow_existing => false
    lv.storage :file, :size => '1G', :device => 'vdf', :allow_existing => false
  end
  config.vm.provision "shell", inline: <<-SHELL
    brd='*************************************************************'
    echo "$brd"
    echo 'Установим пакет rsync'
    echo "$brd"
  # apt update
    apt install -y rsync arch-install-scripts
    echo "$brd"
    echo 'Создадим первый физический том'
    echo "$brd"
    
    pvcreate /dev/vdb
    echo "$brd"
    echo 'Создадим группу томов'
    echo "$brd"
    
    vgcreate data /dev/vdb
    echo "$brd"
    echo 'Посмотрим информацию о только что созданной группе томов'
    echo "$brd"
    
    vgdisplay data
    
    echo "$brd"
    echo 'Выведем информацию о том, какие диски входит в VG'
    echo "$brd"
    
    vgdisplay -v data | grep 'PV Name'
    
    echo "$brd"
    echo 'Создадим первый логический том'
    echo "$brd"

    lvcreate -L8G -n large data

    echo "$brd"
    echo 'Создадим второй логический том'
    echo "$brd"   

    lvcreate -L100M -n small data

    echo "$brd"
    echo 'Создадим файловую систему на первом томе'
    echo "$brd"    

    mkfs.ext4 /dev/mapper/data-large

    echo "$brd"
    echo 'Смонтируем том'
    echo "$brd"
    
    mount --mkdir /dev/mapper/data-large /data/

    echo "$brd"
    echo 'Создадим второй физический том'
    echo "$brd"
    
    pvcreate /dev/vdc

    echo "$brd"
    echo 'Расширим группу томов за счёт второго физического тома'
    echo "$brd"
    
    vgextend data /dev/vdc

    echo "$brd"
    echo 'Посмотрим информацию о группе томов'
    echo "$brd"
    
    vgdisplay data
    
    echo "$brd"
    echo 'Выведем информацию о том, какие диски входит в VG'
    echo "$brd"
    
    vgdisplay -v data | grep 'PV Name'
    
    echo "$brd"
    echo 'Информация о логическом томе'
    echo "$brd"
    
    lvdisplay /dev/mapper/data-large
    
    echo "$brd"
    echo 'Краткий вывод информации о группах томов и о логических томах'
    echo "$brd"

    vgs && lvs
    
    echo "$brd"
    echo 'Информация о монтируемом томе в каталог /data'
    echo "$brd"

    mount | grep /data
    
    echo "$brd"
    echo 'Создадим большой файл размером 8Gb и проверим наличие места на диске'
    echo 'dd if=/dev/zero of=/data/test.log bs=1M count=8000 status=progress'
    echo 'df -Th /data/'
    echo "$brd"

    dd if=/dev/zero of=/data/test.log bs=1M count=8000 status=progress
    df -Th /data/
    
    echo "$brd"
    echo 'Расширим том large'
    echo "$brd"

    lvextend -L12G /dev/mapper/data-large

    echo "$brd"
    echo 'Выведем информацию о томе'
    echo "$brd"
    
    lvs /dev/mapper/data-large
    
    echo "$brd"
    echo 'И о свободном месте на файловой системе'
    echo "$brd"
    
    df -Th /data
    
    echo "$brd"
    echo 'Увеличим размер фс'
    echo "$brd"

    resize2fs /dev/mapper/data-large

    echo "$brd"
    echo 'Снова выведем информацию о свободном месте'
    echo "$brd"
    
    df -Th /data
    
    echo "$brd"
    echo 'Уменьшим файловую систему и том, на котором она создана'
    echo "$brd"

    umount /data/
    e2fsck -fy /dev/mapper/data-large
    resize2fs /dev/mapper/data-large 10G
    lvreduce -y /dev/mapper/data-large -L 10G

    echo "$brd"
    echo 'Снова смонтируем фс и сверим её размер и размер тома, на котором она создана'
    echo "$brd"

    mount /dev/mapper/data-large /data/
    df -Th /data/    
    lvs /dev/mapper/data-large
    
    echo "$brd"
    echo 'Создадим снимок тома large'
    echo "$brd"

    lvcreate -L 500M --snapshot -n large-snap /dev/mapper/data-large

    echo "$brd"
    echo 'Выведем содержимое /data'
    echo "$brd"
    
    ls -l /data
    
    echo "$brd"
    echo 'Удалим файл test.log и выведем содержимое /data'
    echo "$brd"

    rm /data/test.log
    ls -l /data
    
    echo "$brd"
    echo 'Отмонтируем /data и восстановим том large из снимка'
    echo "$brd"

    umount /data
    lvconvert --merge /dev/mapper/data-large--snap

    echo "$brd"
    echo 'Снова смонтируем том и проверим содержимое /data'
    echo "$brd"

    mount /dev/mapper/data-large /data
    ls -l /data    
    umount /data

    echo "$brd"
    echo 'Удалим все логические тома в группе data'
    echo "$brd"

    lvremove -y data

    echo "$brd" 
    echo 'Удалим физический том из группы data'
    echo "$brd"

    vgreduce -y data /dev/vdc

    echo "$brd"
    echo "$brd"
    echo 'ПЕРЕНОС КАТАЛОГОВ /VAR, /HOME НА LVM RAID'
    echo "$brd"
    echo "$brd"
    echo 'Создадим новые физические тома для логических томов lvvar и lvhome'
    echo "$brd"
    
    pvcreate /dev/vdd /dev/vde
    
    echo "$brd"
    echo 'Расширим системную группу томов vg-system, здесь устройства vdc и vdd будут использоваться для создания массива, а vde для возможности делать снимки томов'
    echo "$brd"
    
    vgextend vg-system /dev/vdc /dev/vdd /dev/vde

    echo "$brd"
    echo 'Создадим логические тома lvvar и lvhome тип RAID1 на физических устройствах /dev/vdc и /dev/vdd'
    echo "$brd"

    lvcreate --type raid1 --mirrors 1 -l 50%PVS -n lvvar vg-system /dev/vdc /dev/vdd
    lvcreate --type raid1 --mirrors 1 -l 100%PVS -n lvhome vg-system /dev/vdc /dev/vdd
    
    echo "$brd"
    echo 'Создадим ФС на томе, смонтируем его во временную директорию и скопируем содержимое каталогов /var и /home на эти тома'
    echo "$brd"
    
    mkfs.ext4 /dev/mapper/vg--system-lvvar
    mkfs.ext4 /dev/mapper/vg--system-lvhome
    mount --mkdir /dev/mapper/vg--system-lvvar /lvvar
    mount --mkdir /dev/mapper/vg--system-lvhome /lvhome
    rsync -a /var/ /lvvar
    rsync -a /home/ /lvhome

    echo "$brd"
    echo 'Отредактируем /etc/fstab для последующего монтирования тома lvvar'
    echo "$brd"
    
    sed -i 's/vg--system-lvolhome/vg--system-lvhome/' /etc/fstab
    genfstab -L / | grep "lvvar" >> /etc/fstab 

    echo "$brd"
    echo 'Создадим новый логический том lvroot для временного размещения корневой файловой системы'
    echo "$brd"

    lvcreate -l+100%FREE -n lvroot -y data

    echo "$brd"
    echo 'Сделаем дамп действующей корневой файловой системы во временную'
    echo "$brd"

    dd if=/dev/mapper/vg--system-lvolroot of=/dev/mapper/data-lvroot bs=4M

    echo "$brd"
    echo 'Смонтируем том lvroot в каталог /lvroot'
    echo "$brd"

    mount --mkdir /dev/mapper/data-lvroot /lvroot
    
    echo "$brd"
    echo 'Исправим точку монтирования корневой ФС в файле fstab этой ФС'
    echo "$brd"

    sed -i 's/vg--system-lvolroot/data-lvroot/' /lvroot/etc/fstab
    grep "data-lvroot" /lvroot/etc/fstab
    SHELL
  # Enable provisioning with a shell script. Additional provisioners such as
  # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
  end

  # config.vm.define "ArchLinux1" do |srv1|
  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
    # srv1.vm.box = "/home/max/vagrant/images/debian12"
  # config.vm.provider "libvirt" do |lv|
  #   lv.memory = "1024"
  #   lv.cpu = "2"
  # end
  # end
end
