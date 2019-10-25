# busterfs
Instructions on how to install Debian Buster on btrfs subvolumes - may work on further releases

This meant to be a semi-copy-pastable cookbook for setup, worked for me but may not work for you. 

## Story
Came across [this](https://wiki.archlinux.org/index.php/User:Altercation/Bullet_Proof_Arch_Install) excellent guide for Arch. Liked the idea of fully encrypted system with native dm-crypt AND using btrfs subvolumes. Experimented for a while and got it work on Debian. (turns out Ubuntu can be installed like this as well)

## Assumptions:
- An UEFI-capable PC or laptop with one disk witout data. Disk will be wiped so secure your data first someplace else, if any.
- Enabled UFEI boot and disabled both Legacy boot and Secure boot in firmware settings
- Wired network (for wireless to work you need to get the wifi firmware package, eg. *firmware-iwlwifi.deb* for an Intel adapter)
- an USB drive with Debian Live on it (I used the XFCE Edition from [here](https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/) )

## Set up the environment
First you need to boot the machine from the USB drive. 
Then you need to know the device name you will install to. Chances are it will be either **/dev/sda** if you have SATA disk or **/dev/nvme0n1** in case of PCIe disk.
Open a terminal, be root and execute 
`lsblk | grep disk`
to find out; let's assume it is /dev/sda and assign it to a variable and a couple of others.
For MIRROR the easiest is `http://deb.debian.org/debian`; I used a local one
```bash
DRIVE=/dev/sda
MIRROR="http://ftp.bme.hu/debian"
o=defaults,x-mount.mkdir
o_btrfs=$o,compress=lzo,ssd,noatime
```
Now let's go ahead and wipe that drive!

## PARTITIONIG
```bash
sgdisk --zap-all $DRIVE
sgdisk --clear \
         --new=1:0:+550MiB --typecode=1:ef00 --change-name=1:EFI \
         --new=2:0:+8GiB   --typecode=2:8200 --change-name=2:cryptswap \
         --new=3:0:0       --typecode=3:8300 --change-name=3:cryptsystem \
           $DRIVE

mkfs.fat -F32 -n EFI /dev/disk/by-partlabel/EFI
```

## ENCRYPT + CRYPTSWAP
```bash
cryptsetup luksFormat --type luks1 /dev/disk/by-partlabel/cryptsystem
cryptsetup open /dev/disk/by-partlabel/cryptsystem system
cryptsetup open --type plain --key-file /dev/urandom /dev/disk/by-partlabel/cryptswap swap
mkswap -L swap /dev/mapper/swap
swapon -L swap
```
## BTRFS SUBVOLS
```bash
mkfs.btrfs --force --label system /dev/mapper/system
mount -t btrfs LABEL=system /mnt
btrfs subvolume create /mnt/root
btrfs subvolume create /mnt/home
btrfs subvolume create /mnt/snapshots
```
## MOUNT
```bash
umount -R /mnt
mount -t btrfs -o subvol=root,$o_btrfs LABEL=system /mnt
mount -t btrfs -o subvol=home,$o_btrfs LABEL=system /mnt/home
mount -t btrfs -o subvol=snapshots,$o_btrfs LABEL=system /mnt/.snapshots
mkdir -p /mnt/boot/efi
mount -o $o LABEL=EFI /mnt/boot/efi
```
## PACKAGES to live system
```bash
apt-get update && apt-get install vim debootstrap arch-install-scripts
```

## INSTALL base system
```bash
debootstrap --arch amd64 buster /mnt $MIRROR
```
