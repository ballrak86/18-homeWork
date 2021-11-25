## Описание файлов в директории
logFileFull.log - полный лог выполнения  

Vagrant_folder - все что понадобится для поднятия VM и краткое описание файлов в ней  
Vagrantfile - вагрант файл  
setup_pxe.sh - скрипт подготовки PXE сервера  

## Описание как запустить виртуальную машину (кратко)
Перед запуском ВМ нужно скачать iso образ, размер которого болеее 10 ГБ.
```
mkdir ./.vagrant/
wget http://ftp.mgts.by/pub/CentOS/8.5.2111/isos/x86_64/CentOS-8.5.2111-x86_64-dvd1.iso -P ./.vagrant/
sed -i "s|FullWayToISO|$PWD/.vagrant/CentOS-8.5.2111-x86_64-dvd1.iso|" ./Vagrantfile
```
Если не хочется качать 10 ГБ, можно воспользоваться файлом ниже. Но он не подходит для автоинсталяции.
```
wget http://ftp.mgts.by/pub/CentOS/8.5.2111/isos/x86_64/CentOS-8.5.2111-x86_64-boot.iso -P ./.vagrant/
```
Далее запускаем ВМ
```
vagrant up
```
И наблюдаем как устанавливается ОС автоматически. Если хотим установку вручном режиме, то нужно успеть выбрать в меню клиента нужный пункт  

## VagrantFile только важное
Подключаем наш скачанный iso файл к ВМ. FullWayToISO будет заменен на полный путь до iso
```
vb.customize [
      'storageattach', :id,
      '--storagectl', 'IDE',
      '--port', 1,
      '--device', 0,
      '--type', 'dvddrive',
      '--medium', "FullWayToISO"
]
```

## setup_pxe.sh
Модифицировал имеющийся скрипт  
Устанавливаем все что нужно
```
#!/bin/bash
echo Install PXE server
yum -y install epel-release
yum -y install dhcp-server
yum -y install tftp-server
yum -y install httpd
firewall-cmd --add-service=tftp,dhcp,http
# disable selinux or permissive
setenforce 0
#
```
Настраиваем dhcp
```
centos_version=8.5.2111

cat >/etc/dhcp/dhcpd.conf <<EOF
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;
subnet 10.0.0.0 netmask 255.255.255.0 {
        #option routers 10.0.0.254;
        range 10.0.0.100 10.0.0.120;
        class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          next-server 10.0.0.20;
          if option architecture-type = 00:07 {
            filename "uefi/shim.efi";
            } else {
            filename "pxelinux/pxelinux.0";
          }
        }
}
EOF
systemctl start dhcpd
systemctl enable dhcpd
```
Готовим tftp
```
systemctl start tftp.service
systemctl enable tftp.service
yum -y install syslinux-tftpboot.noarch
mkdir /var/lib/tftpboot/pxelinux
cp /tftpboot/pxelinux.0 /var/lib/tftpboot/pxelinux
cp /tftpboot/libutil.c32 /var/lib/tftpboot/pxelinux
cp /tftpboot/menu.c32 /var/lib/tftpboot/pxelinux
cp /tftpboot/libmenu.c32 /var/lib/tftpboot/pxelinux
cp /tftpboot/ldlinux.c32 /var/lib/tftpboot/pxelinux
cp /tftpboot/vesamenu.c32 /var/lib/tftpboot/pxelinux
```
Подготавливаем меню PXE и скачиваем нужные файлы
```
mkdir /var/lib/tftpboot/pxelinux/pxelinux.cfg

cat >/var/lib/tftpboot/pxelinux/pxelinux.cfg/default <<EOF
default menu
prompt 0
timeout 600
MENU TITLE Demo PXE setup
LABEL linux
  menu label ^Install system
  kernel images/CentOS-8/vmlinuz
  initrd images/CentOS-8/initrd.img
  append ip=enp0s3:dhcp inst.repo=http://10.0.0.20/centos8-install
LABEL linux-auto
  menu label ^Auto install system
  menu default
  kernel images/CentOS-8/vmlinuz
  initrd images/CentOS-8/initrd.img
  append ip=enp0s3:dhcp inst.ks=http://10.0.0.20/cfg/ks.cfg inst.repo=http://10.0.0.20/centos8-install
LABEL vesa
  menu label Install system with ^basic video driver
  kernel images/CentOS-8/vmlinuz
  append initrd=images/CentOS-8/initrd.img ip=dhcp inst.xdriver=vesa nomodeset
LABEL rescue
  menu label ^Rescue installed system
  kernel images/CentOS-8/vmlinuz
  append initrd=images/CentOS-8/initrd.img rescue
LABEL local
  menu label Boot from ^local drive
  localboot 0xffff
EOF

mkdir -p /var/lib/tftpboot/pxelinux/images/CentOS-8/
curl -O http://ftp.mgts.by/pub/CentOS/$centos_version/BaseOS/x86_64/os/images/pxeboot/initrd.img
curl -O http://ftp.mgts.by/pub/CentOS/$centos_version/BaseOS/x86_64/os/images/pxeboot/vmlinuz
cp {vmlinuz,initrd.img} /var/lib/tftpboot/pxelinux/images/CentOS-8/
```
Ранее подключенный iso монтируем в нужную директорию
```
mkdir /var/www/html/centos8-install
devIso=$(blkid | cut -f1 -d':' | grep -v sda)
mount $devIso /var/www/html/centos8-install
systemctl enable httpd.service
systemctl start httpd.service
```
Подготовка kickstarter файла
```
autoinstall(){
  # to speedup replace URL with closest mirror
  mkdir /var/www/html/cfg
cat > /var/www/html/cfg/ks.cfg <<EOF
#version=RHEL8
ignoredisk --only-use=sda
autopart --type=lvm
# Partition clearing information
clearpart --all --initlabel --drives=sda
# Use graphical install
graphical
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8
#repo
#url --url=http://ftp.mgts.by/pub/CentOS/${centos_version}/BaseOS/x86_64/os/
# Network information
network  --bootproto=dhcp --device=enp0s3 --ipv6=auto --activate
network  --bootproto=dhcp --device=enp0s8 --onboot=off --ipv6=auto --activate
network  --hostname=localhost.localdomain
# Root password
rootpw --iscrypted $6$g4WYvaAf1mNKnqjY$w2MtZxP/Yj6MYQOhPXS2rJlYT200DcBQC5KGWQ8gG32zASYYLUzoONIYVdRAr4tu/GbtB48.dkif.1f25pqeh.
# Run the Setup Agent on first boot
firstboot --enable
# Do not configure the X Window System
skipx
# System services
services --enabled="chronyd"
# System timezone
timezone America/New_York --isUtc
user --name=vagrant --password=vagrant
%packages
@^minimal-environment
kexec-tools
%end
%addon com_redhat_kdump --enable --reserve-mb='auto'
%end
%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
EOF
chown -R vagrant.vagrant /var/www/html/cfg
}
# uncomment to enable automatic installation
autoinstall
```

📚Домашнее задание разработано для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)