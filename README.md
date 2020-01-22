# Bionic + btrfs
Instructions on how to install Ubuntu Bionic on btrfs subvolumes - may work on further releases

This meant to be a semi-copy-pastable cookbook for setup, worked for me but may not work for you. 

## Story
Came across [this](https://wiki.archlinux.org/index.php/User:Altercation/Bullet_Proof_Arch_Install) excellent guide for Arch. Liked the idea of a fully encrypted system with native dm-crypt and using btrfs features like compression, snapshots and subvolumes. Experimented for a while and got it work on Debian. (turns out Ubuntu can be installed like this as well)

## Assumptions:
- An UEFI-capable PC or laptop with one SSD witout data. Disk will be wiped so secure your data first someplace else, if any.
- Enabled UFEI boot and disabled both Legacy boot and Secure boot in firmware settings
- Wired network (for wireless to work you need to get the wifi firmware package, eg. *firmware-iwlwifi.deb* for an Intel adapter)
- an USB drive with Debian Live on it (I used the XFCE Edition from [here](https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/) )
- You don't mind typing the encryption password twice on system (re)start. *(once for the boot loader to get the kernel image and another for mount the / partition)* 
- You don't want to hibernate the machine. With this setup swap partition content is non-recoverable *(gets re-encrypted with random on every boot)* therefore resuming from hibernation is not possible.

## Set up the environment
First you need to boot the machine from the USB drive. 
Then you need to know the device name you will install to. Chances are it will be either **/dev/sda** if you have SATA disk or **/dev/nvme0n1** in case of PCIe disk.
Open a terminal, **be root** and execute 
`lsblk | grep disk`
to find out; let's assume it is /dev/sda and assign it to a variable and a couple of others.
For MIRROR the easiest is `http://archive.ubuntu.com/ubuntu`; I used a local one
```bash
DRIVE=/dev/sda
MIRROR="http://hu.archive.ubuntu.com/ubuntu"
o=defaults,x-mount.mkdir
o_btrfs=$o,compress=lzo,ssd,noatime
```
Now let's go ahead and wipe that drive!

## Partitioning
Let's create a big enough EFI partition and 8GB worth of swap. (Feel free to alter the values.) All the remaining space will be used for the encrypted system storage. 
```bash
sgdisk --zap-all $DRIVE
sgdisk --clear \
         --new=1:0:+550MiB --typecode=1:ef00 --change-name=1:EFI \
         --new=2:0:+8GiB   --typecode=2:8200 --change-name=2:cryptswap \
         --new=3:0:0       --typecode=3:8300 --change-name=3:cryptsystem \
           $DRIVE

mkfs.fat -F32 -n EFI /dev/disk/by-partlabel/EFI
```

## Create encrypted system space and swap
Have a good password at hand to use for unlocking your drive. 
```bash
cryptsetup luksFormat --type luks1 /dev/disk/by-partlabel/cryptsystem
cryptsetup open /dev/disk/by-partlabel/cryptsystem system
cryptsetup open --type plain --key-file /dev/urandom /dev/disk/by-partlabel/cryptswap swap
mkswap -L swap /dev/mapper/swap
swapon -L swap
```
## Create btrfs subvolumes
```bash
mkfs.btrfs --force --label system /dev/mapper/system
mount -t btrfs LABEL=system /mnt
btrfs subvolume create /mnt/root
btrfs subvolume create /mnt/home
btrfs subvolume create /mnt/snapshots
```
## Mount the subvolumes
with the mount options we declared above:
```bash
umount -R /mnt
mount -t btrfs -o subvol=root,$o_btrfs LABEL=system /mnt
mount -t btrfs -o subvol=home,$o_btrfs LABEL=system /mnt/home
mount -t btrfs -o subvol=snapshots,$o_btrfs LABEL=system /mnt/.snapshots
mkdir -p /mnt/boot/efi
mount -o $o LABEL=EFI /mnt/boot/efi
```
## Install some packages into the live system
```bash
apt-get update && apt-get install vim debootstrap arch-install-scripts
```

## Install base system
```bash
debootstrap --arch amd64 bionic /mnt $MIRROR
```

## Config
Save the actual mount layout to fstab:
```bash
genfstab -L -p /mnt >> /mnt/etc/fstab
```
then edit it:
```bash
vim /mnt/etc/fstab
```
and change the line for swap from this: 
```bash
LABEL=swap          	none      	swap      	defaults  	0 0
```
to this:
```bash
/dev/mapper/cryptswap    	none      	swap      	sw  	0 0
```

## chroot time
Head to the new installation and set up basic things like root password, hostname, time zone. 
Use your own values for the things inside [ ]s: 
```bash
arch-chroot /mnt
passwd
apt-get update && apt-get install locales console-setup tasksel
dpkg-reconfigure locales
dpkg-reconfigure tzdata
dpkg-reconfigure console-setup 
hostnamectl set-hostname [your hostname]
apt-get install linux-image-generic grub-efi-amd64 efibootmgr cryptsetup btrfs-progs sudo vim zsh
adduser --shell /usr/bin/zsh [your username]
usermod -aG sudo [your username]
```
### misc configurations on the new system
Edit /etc/crypttab and add the following:
```bash
system         /dev/disk/by-partlabel/cryptsystem        none    luks
cryptswap        /dev/disk/by-partlabel/cryptswap        /dev/urandom        swap,offset=2048,cipher=aes-xts-plain64,size=256
```

Re-generate initramfs with some tweaks:
```bash
echo -e "CRYPTSETUP=y\n" >> /etc/cryptsetup-initramfs/conf-hook
echo -e "RESUME=none\n" >> /etc/initramfs-tools/conf.d/noresume.conf

update-initramfs -k all -u
```

### Configure and install the boot loader
Edit /ec/default/grub
First add
``` bash
GRUB_ENABLE_CRYPTODISK=y
```
to the end of the file; then alter the **GRUB_CMDLINE** lines like this:
``` bash
GRUB_CMDLINE_LINUX_DEFAULT="net.ifnames=0 biosdevname=0 noresume"
GRUB_CMDLINE_LINUX="quiet"
```
(because I like good old interface names like eth0 and wlan0)

And install it into the EFI partition:
``` bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu --recheck --debug
grub-mkconfig -o /boot/grub/grub.cfg
```

### Configure wired network
Create /etc/netplan/interfaces.yaml and add:
```bash
network:
        version: 2
        renderer: networkd
        ethernets:
                eth0:
                        dhcp4: true
```

At this point the base install is completed, time to exit the chroot and boot into the new system:
```bash
exit
systemctl reboot
```

## Start the new system
If everything went as expected, you will be prompted for the unlock password, then got a grub menu, then asked for the password again mid-boot.
If you mistype the first password then grub will be unable to get it's config and and present you a naked **grub>** prompt.
Start over with Ctrl + Alt + Del.

If you got a **login:** prompt then congratulations: your system is up and running. Go ahead and log in as root.

### Create the first snapshot
To have a restoration point to the very basic state of the system:
```bash
btrfs subvol snapshot / /.snapshots/root_default
```

### Install something useful
I recommend to start with **tasksel** and choose your favourite Desktop Environment from the list.
After that you can start adding more stuff, tweaking, etc. There are many and more guides for those out there to continue on.

Good luck && have fun!
