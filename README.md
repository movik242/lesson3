# lesson3
При выполнении задания использовал образ centos/7 - v. 1804.2

**1. Уменьшить том под / до 8G**

Настраиваю vagrantfile согласно методичке, добавляю 4 диска 10G, 2G, 1G, 1G соотвественно.

При развертывании VM в системе появляется папка /vargant, удаляю её, так как из-за нее не работали последующие команды xfsdump, устанавливаю xfsdump
```
rm -rf /vagrant/*
```
```
rmdir /vagrant
```
```
yum install xfsdump
```
Подготавливаю временный том для раздела /
```
pvcreate /dev/sdb
```
Создаю LVM
```
vgcreate vg_root /dev/sdb
```
Выделяю под том весь диск
```
lvcreate -n lv_root -l +100%FREE /dev/vg_root
```
Создаю на нем файловую систему и монтирую
```
mkfs.xfs /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt
```
Копирую все данные с / раздела в /mnt
```
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Монтирую каталоги
```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
```
Перехожу в скопированный root командой chroot и правлю grub
```
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
```
Правлю initramfs
```
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done
```
Редактирую файл /boot/grub2/grub.cfg меняя rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root
Перезагружаюсь и работаю в системе с нового диска

![Снимок экрана 2023-10-07 173351](https://github.com/movik242/lesson3/assets/143793993/8474249d-d14c-49b9-a837-11388005056d)

Удаляю старый LV размером в 40G и создаю новый на 8G
```
lvremove /dev/VolGroup00/LogVol00
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
```
Устанавливаю файловую систему, копирую данный и через chroot настраиваю новый раздел /
```
mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done
```
**2.Выделить том под /var - сделать в mirror**

Создаю зеркало на свободных дисках не выходя из chroot
```
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
```
![Снимок экрана 2023-10-07 175100](https://github.com/movik242/lesson3/assets/143793993/a3897ba8-e69c-4089-a794-4594813549bc)

Создаю на нем ФС и копирую туда /var
```
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
```
Делаю копию /var
```
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```
Монтирую новый каталог var и правлю /etc/fstab
```
umount /mnt
mount /dev/vg_var/lv_var /var
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
После перезагрузки и удаления временного тома /dev/vg_root получилось следующее

![Снимок экрана 2023-10-07 231833](https://github.com/movik242/lesson3/assets/143793993/ac7c1db4-c672-4006-9e91-f9ae641def74)

**3.Выделить том под /home**

Выделяю под том /home 2G из VG - VolGroup00
```
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol_Home
```
Делаю по аналогии с /var
```
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
Генерирую файлы в /home/
```
touch /home/file{1..20}
```
**4./home - сделать том для снапшотов**

Снимаю снапшот
```
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
```
![Снимок экрана 2023-10-07 232926](https://github.com/movik242/lesson3/assets/143793993/7737c35f-456c-4302-869f-42021df57502)

Удаляю часть файлов
```
rm -f /home/file{11..20}
```
Восстанавливаюсь из снапшота
```
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home
```
**5.Прописать монтирование в fstab. Попробовать с разньми опциями и разными файловыми системами (на выбор)**

Согласно заданию создаю, аналогично /LogVol_Home, который в xfs, /LogVol_Home2 размером 2G   
```
lvcreate -n LogVol_Home2 -L 2G /dev/VolGroup00
mkfs.ext4 /dev/VolGroup00/LogVol_Home2
mount /dev/VolGroup00/LogVol_Home2 /opt/
cp -aR /home/* /opt/
umount /opt
mount /dev/VolGroup00/LogVol_Home2 /opt/
echo "`blkid | grep Home2 | awk '{print $2}'` /opt ext4 defaults 0 0" >> /etc/fstab
```
Разница в файловой системе видна по занимаемому месту одними и теми же файлами

![Снимок экрана 2023-10-08 090411](https://github.com/movik242/lesson3/assets/143793993/73a698d9-f65d-4c63-aac3-e99ed6369db1)

**Вывод**

При выполнении ДЗ возникли значительные трудности при выполнении xfsdump, так как мне мешал каталог /vargant и chroot все время выпадал с ошибкой. Получилось раза с пятого)
