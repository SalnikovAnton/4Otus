# Загрузка Linux
1) Попасть в систему без пароля несколькими способами
2) Установить систему с LVM, после чего переименовать VG
3) Добавить модуль в initrd

#### 1 запускаем виртуальную машину и в окне выбора ядра для загрузки нажимаем Е
Переходим на строчку начинающуюся с linux и добавляем init=/bin/sh и нажимаем сtrl-x для загрузки в систему

![Image alt](https://github.com/SalnikovAnton/8Otus/blob/main/1.jpg)

используем команду для переформатирования файловой системы в режим Read-Write

![Image alt](https://github.com/SalnikovAnton/8Otus/blob/main/4.jpg)

rd.break. Переходим на строчку начинающуюся с linux и добавляем rd.break и нажимаем сtrl-x для загрузки в систему и попадаем в emergency mode

![Image alt](https://github.com/SalnikovAnton/8Otus/blob/main/7.JPG)

Наша корневаā файловая система смонтирована опять же в режиме Read-Only, но мы не в ней. Далее вводим команды чтобы попасть в нее и поменять пароль администратора:

![Image alt](https://github.com/SalnikovAnton/8Otus/blob/main/8.JPG)

rw init=/sysroot/bin/sh. Переходим на строчку начинающуюся с linux16 и заменяем ro на rw init=/sysroot/bin/sh и нажимаем сtrl-x для загрузки в систему, но файловая система сразу смонтирована в режим Read-Write.

![Image alt](https://github.com/SalnikovAnton/8Otus/blob/main/5.JPG)

#### 2 текущее состояние системы:
```
[root@lvm ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree  
  VolGroup00   1   3   0 wz--n- <38.97g <27.47g
  vg_var       2   1   0 wz--n-   2.99g   1.12g
```
Приступим к переименованию, нас интересует VolGroup00 :
```
[root@lvm ~]# vgrename VolGroup00 OtusRoot
  Volume group "VolGroup00" successfully renamed to "OtusRoot"
```
Далее правим
```
[root@lvm ~]# vim /etc/fstab
___________
#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/OtusRoot-LogVol01 swap                    swap    defaults        0 0
___________

[root@lvm ~]# vim /etc/default/grub 
___________
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
___________

[root@lvm ~]# vim /boot/grub2/grub.cfg 
___________
...
fi
        linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/OtusRoot-LogVol00 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet 
        initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img
...

```
Пересоздаем initrd image, чтобы он знал новое название Volume Group
```
[root@lvm ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
...
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```
Перезагружаемся и проверяем имя Volume Group
```
[root@lvm ~]# vgs
  VG       #PV #LV #SN Attr   VSize   VFree  
  OtusRoot   1   3   0 wz--n- <38.97g <27.47g
```
#### 3 Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того чтобы добавить свой модуль создаем там папку с именем 01test:
```
[root@lvm ~]# mkdir /usr/lib/dracut/modules.d/01test
```
В нее поместим два скрипта: module-setup.sh - который устанавливает модуль и вызывает скрипт test.sh
```
[root@lvm ~]# cat /usr/lib/dracut/modules.d/01test/module-setup.sh 
#!/bin/bash

check() {
    return 0
}

depends() {
    return 0
}

install() {
    inst_hook cleanup 00 "${moddir}/test.sh"
}
```
сам вызываемый скрипт
```
[root@lvm ~]# cat /usr/lib/dracut/modules.d/01test/test.sh 
#!/bin/bash

exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
cat <<'msgend'

Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
msgend
sleep 10
echo " continuing...."
```
Пересобираем образ initrd
```
[root@lvm ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
...
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```
или
```
[root@lvm ~]# dracut -f -v
```
Можно проверить/посмотреть какие модули загружены в образ
```
[root@lvm ~]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test
```
Перезагружаемся и выключаем опции rghb и quiet и смотрим вывод
![Image alt](https://github.com/SalnikovAnton/8Otus/blob/main/pingvin.png)
