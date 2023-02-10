# Размещаем свой RPM в своем репозитории
1) Попасть в систему без пароля несколькими способами
2) Установить систему с LVM, после чего переименовать VG
3) Добавить модуль в initrd

#### 1 Устанавливаем пакеты:
```
[root@otuslinux ~]# yum install -y \
redhat-lsb-core \
wget \
rpmdevtools \
rpm-build \
createrepo \
yum-utils \
gcc
```

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
