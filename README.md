# HowToInstallArchBtw
I use arch btw

# First
## boot into the iso
## select the disk
lsblk<br>
select one of the disks without a number at the end (for nvme don't ask me) (**WARNING: BE SURE TO USE THE COORECT DISK! DATA *WILL* BE LOST!**)<br>
in this case im gonna use /dev/sda **BUT BE SURE TO CHANGE IT TO THE CORRECT DEVICE NAME**<br>
if you want to encrypt the disk (with LUKS ) you can!<br>
replace $EFI_SYSTEM_PARTITION with what L showed for EFI_SYSTEM_PARTITION (in my case with an old arch iso it is 1)<br>
replace $Linux_LVM with what L showed for Linux LVM (in my case with an old arch iso it is 30)<br>
on $CHECK_STRUCTURE<br>
check if everything looks like this: (also $UNKNOWN can be anything)<br>
```
/dev/sdX1 2048 1026047 10240000 500M EFI System
/dev/sdX2 1026048 2050047 10240000 500M Linux filesystem
/dev/sdX3 2050048 $UNKNOWN $UNKNOWN $UNKNOWN Linux LVM
```
if everything looks **LIKE THAT** then continue if not press `q` then `ENTER` and do everything again<br>
**READ THIS: each `-` is a new line**
```
fdisk /dev/sda
-
g
-
n
-
-
+500M
-
t
-
L
-
q
$EFI_SYSTEM_PARTITION
-
n
-
-
+500M
n
-
-
-
t
-
L
-
q
$Linux_LVM
-
p
-
$CHECK_STRUCTURE
w
-
```
now we will format the partitions<br>
```
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
```
now we have some choice do we want to encrypt the disk?<br>
if yes:<br>
PHYSVOL is something (E.G. `lvm`)<br>
VOLGRP is something (E.G. `VOLGRP0`)<br>
```
cryptsetup luksFormat /dev/sda3
cryptsetup open --type luks /dev/sda3 /dev/mapper/$PHYSVOL

pvcreate --dataalignment 1m /dev/mapper/$PHYSVOL
vgcreate $VOLGRP /dev/mapper/$PHYSVOL
```
if no:<br>
PHYSVOL is something (E.G. `lvm`)<br>
VOLGRP is something (E.G. `VOLGRP0`)<br>
```
pvcreate --dataalignment 1m /dev/mapper/$PHYSVOL
vgcreate $VOLGRP /dev/mapper/$PHYSVOL
```

now we have another choice<br>
do we want a seperate home partition?<br>
if yes:<br>
$ROOTSZ is the root fs size<br>
$HOMESZ is the home fs size Recommended: 100%FREE<br>
```
lvcreate -L $ROOTSZ $VOLGRP -n root_part
lvcreate -L $HOMESZ $VOLGRP -n home_part
```
if no:<br>
$ROOTSZ is the root fs size Recommended: 100%FREE<br>
```
lvcreate -L $ROOTSZ $VOLGRP -n root_part
```
now<br>
```
modprobe dm_mod
vgscan
vgchange -ay
```
this will activate lvm<br>
lets mount and format root_part<br>
```
mkfs.ext4 /dev/$VOLGRP/root_part
mount /dev/$VOLGRP/root_part /mnt

mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
```
if we have a seperate home part:<br>
```
mkfs.ext4 /dev/$VOLGRP/home_part
mkdir /mnt/home
mount /dev/$VOLGRP/home_part /mnt/home
```
if we don't:<br>
```
mkdir /mnt/home
```
now<br>
```
mkdir /mnt/etc
genfstab -U -p /mnt >> /mnt/etc/fstab
pacstrap -i /mnt base
```
the pacstrap will ask you Y/n question just press enter for all of them<br>
now we will chroot into our new install<br>
```
arch-chroot /mnt
```
now we will install linux<br>
press enter for the questions<br>
```
pacman -S linux linux-headers
echo 'N'>/VAR_LTS_LINUX_KERNEL
```
if you want lts linux run<br>
```
pacman -S linux-lts linux-lts-headers
echo 'Y'>/VAR_LTS_LINUX_KERNEL
```
but if you want both<br>
```
pacman -S linux linux-headers linux-lts linux-lts-headers
echo 'Y'>/VAR_LTS_LINUX_KERNEL
```
now we will install stuff<br>
```
pacman -S vim base-devel networkmanager lvm2
```
again press enter for everything<br>
if we encrypted our disk:<br>
```
sed -i 's/block filesystems/block encrypt lvm2 filesystems/g' /etc/mkinitcpio.conf
```
if not<br>
```
sed -i 's/block filesystems/block lvm2 filesystems/g' /etc/mkinitcpio.conf
```
this will add the modules now lets create our initramfs<br>
if we installed only linux (not lts)<br>
```
mkinitcpio -p linux
```
if we installed only linux-lts (not bleeding edge)<br>
```
mkinitcpio -p linux-lts
```
if we installed both<br>
```
mkinitcpio -p linux
mkinitcpio -p linux-lts
```
setting up the locale<br>
```
sed -i 's/\#en_US.UTF-8/en_US.UTF-8/g' /etc/locale.gen
locale-gen
```
lets change the root password<br>
```
passwd
```
we will add a user<br>
and change its password<br>
$username is the username<br>
```
useradd -m -g users -G wheel $username
passwd $username
```
now a few automated things i don't feel to unautomate<br>
this will install sudo and edit its config<br>
as well as install grub, efithing etc (this whole install for EFI only btw)
```
if [ ! -f $(which sudo)  ]
then
	pacman -S sudo <<EEOF
EEOF
fi

visudo <<EEOF
/# %wheel ALL=(ALL:ALL) ALL
x
x
:w
:q
EEOF
pacman -S grub efibootmgr dosfstools mtools os-prober <<EEOF



EEOF

mkdir /boot/EFI
mount /dev/sda1 /boot/EFI
```
lets install grub<br>
$BOOTID is the entry name (E.G. "grub_uefi" or "ARCHBTW" or "Archlinux")<br>
```
grub-install --target=x86_64-efi --bootloader-id=$BOOTID --recheck
```
more automated things i don't feel like unautomating<br>
```
if [ ! -d "/boot/grub/locale" ]
then
	mkdir /boot/grub/locale
fi
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```
this creates the grub locale<br>
if we encrypted our disk:<br>
you can get the root $UUID from `blkid /dev/sda3` it will be the Numbers and Letters thing
$root_part is $UUID
$volgrp is $VOLGRP
```
sed -i 's/#GRUB_ENABLE_CRYPTODISK=y/GRUB_ENABLE_CRYPTODISK=y/g' /etc/default/grub
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"/GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=\/dev\/disk\/by-uuid\/'${root_part}':'${volgrp}':allow-discards loglevel=3"/g' /etc/default/grub
```
if we didn't encrypt our disk:<br>
```
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"/GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3"/g' /etc/default/grub
```
make config and exit<br>
```
sudo systemctl enable NetworkManager
grub-mkconfig -o /boot/grub/grub.cfg
exit
```
now we will unmount everything<br>
```
unmount /mnt/boot/EFI
unmount /mnt/boot
unmount /mnt/home
unmount /mnt
```
and reboot into ARCH!<br>
but we have a few things to do when we boot into it<br>
login<br>
```
username
password
```
now we will install a script (cuz i hate unautomating my scripts)<br>
```
curl https://raw.githubusercontent.com/Mr-Death-Force-specialChat/archtall/master/install-after.sh -o install-after.sh
sed -i 's/sed -i \'s\/\\/bash install-after.sh\/\/g' ~\/.bashrc//g' ~/.bashrc
chmod u+x install-after.sh
./install-after.sh
rm ./install-after.sh
```
now reboot<br>
and you will have `KDE` installed on your arch install (i recommend `KDE` for beginners but i like `i3`)<br>
also change your hostname<br>
$host_name is the NEW hostname<br>
```
hostnamectl set-hostname $host_name
```
also `pacman` is the package manager<br>
to install `pacman -S $pkgs` $pkgs contains the package names seperated by ` ` (a space)<br>
```
sudo pacman -S neofetch cmatrix
```
NOW reboot<br>
congrats you got ARCH LINUX! (now install artix...)
# CONGRATS!!! (install artix without reading the manual, not sorry RTFM dudes)
