# HowToInstallArchBtw (LVM, Home partition + root partition)

Boot into iso<br>
select disk<br>
lsblk<br>
select one of the disks without a number at the end (for nvme don't ask me) (**WARNING: BE SURE TO USE THE CORRECT DISK! DATA _WILL_ BE LOST!**)<br>
in this case we will use /dev/sda **BUT BE SURE TO CHANGE IT TO THE CORRECT DEVICE**<br>
$ESP (EFI SYSTEM PARTITION) is 1<br>
$LLVM (Linux LVM) is 30<br>
**each `-` is a new line**<br>
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
n
-
-
-
t
-
$ESP
-
n
-
-
+500M
-
n
-
-
-
t
-
$LLVM
-
```

check the structure using `p`<br>
$UNKNOWN can be anything<br>
```
/dev/sda1 2048 1026047 10240000 500M EFI System
/dev/sda2 1026048 2050047 10240000 500M Linux filesystem
/dev/sda3 2050048 $UNKNOWN $UNKNOWN $UNKNOWN Linux LVM
```

if it doesn't look like that then run `q` and restart<br>
<br>
now we need to format the boot _drive_<br>
```
mkfs.fat -F32 /dev/sda1
```
formats the ESP to fat32<br>
```
mkfs.ext4 /dev/sda2
```
formats the grub partition to ext4<br>
$VGRP is the name of the Volume group (for example `volgroup0`)<br>
```
pvcreate --dataalignment 1m /dev/sda3
vgcreate $VGRP /dev/sda3
```
lets set some things up<br>
$RTSZ is root partition size<br>
$HMSZ is home partition size<br>
(WARNING: for using 100%FREE (or using all available space which is recommended for HMSZ) use -l instead of -L)<br>
```
lvcreate -L $RTSZ $VGRP -n root_part
lvcreate -L $HMSZ $VGRP -n home_part
```
something related to lvm<br>
```
modprobe dm_mod
vgscan
vgchange -ay
```
lets mount _and_ format our root partition<br>
```
mkfs.ext4 /dev/$VGRP/root_part
mount /dev/$VGRP/root_part /mnt
```
the same but with our home partition<br>
```
mkdir /mnt/home
mkfs.ext4 /dev/$VGRP/home_part
mount /dev/$VGRP/home_part /mnt/home
```
we will create /boot<br>
```
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
```
some f-stab related stuff<br>
```
mkdir /mnt/etc
genfstab -U -p /mnt >> /mnt/etc/fstab
```
well download our base packages<br>
```
pacstrap -i /mnt base
```
answer Y for all question<br>
```
arch-chroot /mnt
```
CHangingROOT to our installation<br>
now we technically installed arch but no linux<br>
anyway lets fix up pacman!<br>
```
vim /etc/pacman.conf
/#ParallelDownloads
x
/#Color
x
:wq
```
now do we want the LTS kernel or the BleedingEdge kernel?<br>
to install the bleedin edge kernel<br>
```
pacman -S linux linux-headers
```
to install the LTS kernel<br>
```
pacman -S linux-lts linux-lts-headers
```
you can install both!<br>
ByTheWay press `ENTER` on all questions here<br>
installing some stuff<br>
```
pacman -S vim base-devel networkmanager lvm2
```
now we will create the initramfs(s)<br>
```
sed -i 's/block filesystems/block lvm2 filesystems/g' /etc/mkinitcpio
```
this will setup our initramfs config<br>
now if we have the bleedin edge kernel<br>
```
mkinitcpio -p linux
```
if we have the lts kernel<br>
```
mkinitcpio -p linux-lts
```
if we have both then execute all of them<br>
lets setup some LOCALES<br>
```
vim /etc/locale.gen
/#en_US.UTF-8
x
:wq
locale-gen
```
lets change the root password<br>
```
passwd
```
lets add a user<br>
$UN is the username<br>
```
useradd -m -g users -G wheel $UN
passwd $UN
```
if we don't have sudo (check if `which sudo` outputs nothing) then<br>
```
pacman -S sudo
```
now lets fix the sudo config<br>
```
visudo
/# %wheel ALL=(ALL:ALL) ALL
x
x
:wq
```
lets setup grub<br>
$BOOTID is the boot entry name (E.G. "grub_uefi", "ARCHBTW" or "Archlinux")<br>
$UUID is the uuid from running `blkid /dev/sda3`<br>
$ESC press the escape key
```
pacman -S grub efibootmgr dosfstools mtools os-prober
mkdir /boot/EFI
mount /dev/sda1 /boot/EFI

grub-install --target=x86_64-efi --bootloader-id=$BOOTID --recheck
mkdir /boot/grub/locale
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
vim /etc/default/grub
/quiet
xxxxx
:wq
sudo systemctl enable NetworkManager
grub-mkconfig -o /boot/grub/grub.cfg
exit
umount /mnt/boot/EFI
umount /mnt/boot
umount /mnt
```
now reboot into arch<br>
```
shutdown now
```
now start your computer<br>
login<br>
and run<br>
```
sudo rm -r /etc/pacman.d/gnupg
sudo pacman-key --init
sudo pacman -S archlinux-keyring --noconfirm
sudo pacman-key --populate archlinux
```
if you have an intel cpu<br>
```
sudo pacman -S --noconfirm --needed intel-ucode
```
for an amd cpu<br>
```
sudo pacman -S --noconfirm --needed amd-ucode
```
install video drives check my archtall/install-after.sh script<br>
in this case we wil use nvidia<br>
```
sudo pacman -S nvidia nvidia-xconfig
```
if we have the lts kernel<br>
```
sudo pacman -S nvidia nvidia-lts nvidia-xconfig
```
now checkout my arch-setup script
