# Arch Linux with KDE plasma

![screenshot](docs/screenshot.png)

My personal Arch Linux and KDE Plasma installation.

**Disclaimer:** This is not a beginner guide, please read the official [Arch Linux installation guide](https://wiki.archlinux.org/title/Installation_guide)

## Create a Bootable USB flash drive
Download the latest Arch Linux ISO for x86_64 platform from [Arch Linux Downloads](https://archlinux.org/download/)

Write the ISO image to the USB flash drive using [balenaEtcher](https://etcher.balena.io/)

**Optional:** On Linux `dd` command can be used instead of **balenaEtcher**, first identify the USB flash drive path:
```bash
[user@hostname ~]$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    1  28.9G  0 disk 
└─sda1        8:3    1  28.9G  0 part 
nvme1n1     259:0    0 931.5G  0 disk 
├─nvme1n1p1 259:2    0     1G  0 part /boot/efi
└─nvme1n1p2 259:3    0 930.5G  0 part /
nvme0n1     259:1    0 931.5G  0 disk 
└─nvme0n1p1 259:4    0 931.5G  0 part /mnt/storage
```
Write the ISO image:
```bash
sudo dd if=archlinux-2023.07.01-x86_64.iso of=/dev/sda bs=1M status=progress
```

## Boot from Arch flash drive
While PC is booting hit **F12** and select the Arch flash drive from the boot menu

## Connect to the internet
If you are using an ethernet connection just make sure the ethernet cable is connected and use `ip link` to check if the interface has an IP address.

On WiFi use `iwctl` to connect to your WiFi network.

Test the internet connection by pinging a known server, example
```bash
root@archiso ~ # ping -c 5 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=58 time=17.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=58 time=17.2 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=58 time=17.3 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=58 time=17.2 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=58 time=17.2 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4005ms
rtt min/avg/max/mdev = 17.150/17.262/17.400/0.092 ms
```

## Partition the disks
Use `fdisk` to create a new GPT partition table
```bash
fdisk /dev/nvme0n1
```
Then create the following partitions:
```bash
root@archiso ~ # lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 932.0G  0 disk 
├─nvme0n1p1 259:2    0     1G  0 part
├─nvme0n1p2 259:2    0 128.0G  0 part
└─nvme0n1p3 259:3    0 803.0G  0 part
```
Create the EFI file system
```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

Create the `ext4` Linux file system on the rest
```bash
mkfs.ext4 /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
```

Mount partitions to `/mnt`
```bash
mount /dev/nvme0n1p2 /mnt

mkdir -p /mnt/boot/efi
mkdir /mnt/home

mount /dev/nvme0n1p1 /mnt/boot/efi
mount /dev/nvme0n1p3 /mnt/home
```

**Optional:** If you have other SSD drives create partitions and Linux file system for them or just mount them if they are already partitioned.
```bash
mkdir /mnt/storage
mount /dev/nvme1n1p1 /mnt/storage
```

## Install essential packages
Use `pacstrap` to install the base system and Linux kernel
```bash
pacstrap -K /mnt base base-devel linux linux-headers linux-firmware amd-ucode
```

## Generate fstab
Generate the Linux file system table, use `-U` parameter to identify partitions by their UUID
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

## Change root
Change root to `/mnt`
```bash
arch-chroot /mnt
```

## Configure the timezone
Set your timezone
```bash
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc --utc
```

## Localization
Install `nano` and uncomment the line `LANG=en_US.UTF-8 UTF-8` on `/etc/locale.conf` file and generate localization configuration with `locale-gen`
```bash
pacman -S nano
nano /etc/locale.conf
locale-gen
```
Create the file `/etc/locale.conf` and add the line `LANG=en_US.UTF-8`
```bash
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

## Hostname
Set your desired hostname, replace HOSTNAME with your choice
```bash
echo HOSTNAME > /etc/hostname
```
Edit the `/etc/hosts` file and add the following lines
```
127.0.0.1    localhost
::1          localhost
127.0.1.1    HOSTNAME.localdomain HOSTNAME
```

## Network
Install Network Manager and extra packages for WiFi
```bash
pacman -S networkmanager wpa_supplicant wireless_tools netctl dialog
```
Enable Network Manager
```bash
systemctl enable NetworkManager
```

## Users
Use ``passwd`` to change root account password
```bash
passwd
```
Add your user account, replace USERNAME with your name
```bash
useradd -m -G wheel USERNAME
```
Set the user password
```bash
passwd USERNAME
```
Allow `wheel` group to run administrative commands via `sudo`
```bash
EDITOR=nano visudo
```
Find and uncomment the line
```
%wheel ALL=(ALL) ALL
```

## GRUB Bootloader
Install `grub` and required packages for UEFI
```bash
pacman -S grub efibootmgr dosfstools os-prober mtools
```
Configure `grub` bootloader
```bash
grub-install --target=x86_64-efi --bootloader-id=ARCH --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```
You can now reboot into the ARCH instalation and remove the Arch Linux USB flash drive.
```bash
exit
umount -R /mnt
reboot
```

## SWAP file
Login as `root` and create a 4Gb swap file
```bash
dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```
Add the swap file to the Linux file system table
```bash
nano /etc/fstab
```
Add the lines
```
# swap file
/swapfile none swap sw 0 0    
```

## BASH
Enable BASH autocomplete
```bash
sudo pacman -S bash-completion
```

## XORG and GPU drivers
Login with your user account and install XORG and GPU drivers
```bash
sudo pacman -S xorg mesa
```

## Bluetooth
Install the bluetooth service
```bash
sudo pacman -S bluez bluez-utils
```
Enable the bluetooth service
```bash
sudo systemctl enable bluetooth.service
```

## KDE
Install KDE desktop environment
```bash
sudo pacman -S plasma-meta plasma-wayland-session
```
Install the KDE applications
```bash
sudo pacman -S dolphin ksystemlog partitionmanager ark kate kcalc kdialog konsole print-manager elisa dragon ffmpegthumbs gwenview skanlite okular spectacle krunner packagekit-qt5
```
Enable `ssdm` display manager
```bash
sudo systemctl enable sddm
```

## Audio
Install pipewire
```bash
sudo pacman -S pipewire pipewire-audio pipewire-alsa pipewire-pulse
```

## Printers
Install `cups` service
```bash
sudo pacman -S cups
```
Enable `cups` service
```bash
sudo systemctl enable cups.service
```

## Steam
For Steam you need to enable `multilib` support, edit `/etc/pacman.conf` and uncomment the lines
```
[multilib]
Include = /etc/pacman.d/mirrorlist
```
Install Steam
```bash
sudo pacman -S steam
```
Reboot the computer and login to the KDE Plasma desktop environment.
