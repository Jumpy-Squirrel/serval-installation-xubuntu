# Installing XUbuntu 16.04 LTS with Full System Encryption on serval

Recently, I bought a [Serval WS](https://system76.com/laptops/serval) laptop from [System 76](https://system76.com/).

It comes with Ubuntu pre-installed, which works fine out of the box, but I wanted to make some changes:
1. full disk encryption
2. no swap space
3. I prefer Xubuntu over Ubuntu
4. full control over the installation (I never use a system I did not install myself)

This repository contains my notes and some ansible automation scripts from the installation so I can repeat it,
should I ever need to.

I MAKE NO CLAIM AS TO THE USEFULNESS OF THIS INFORMATION FOR ANY PARTICULAR PURPOSE. 

USE AT YOUR OWN RISK.

## Initial installation

### Preparing the install media

Boot some CD based linux distribution.

Download [XUbuntu 16.04.3 LTS](https://xubuntu.org/download/) (amd64 version) from one of the mirrors 
and verify the signature.

dd the image to a USB stick (use the main device, not one of the partitions)

*Tip:* `lsblk` will show a list of all block devices

### Imaging the current SSD

Boot the USB stick on the serval (hold f2 to enter bios, f7 to get a boot menu)

*Tip:* You can set the display resolution under Settings &rightarrow; Settings Editor 
&rightarrow; xsettings &rightarrow; DPI.  

The system has a gpt partition table, which you can edit using `gparted` (graphical tool) or `parted` (command line).
On my system, the only SSD is `/dev/nvme0n1`. 

I made images of the EFI partition and the main system partition as a backup. Basically, mount a file system from 
another server, then 

```
dd if=/dev/nvme0n1p1 | bzip > /mnt/somewhere/origsys.img.dd.nvme0n1p1.dd.bz2
dd if=/dev/nvme0n1p2 | bzip > /mnt/somewhere/origsys.img.dd.nvme0n1p2.dd.bz2
```

No need to make an image of the swap partition, of course.

I also created a tarball of the original installed system, that can come in handy in case I needed 
to recover some firmware files.

### Partitioning and setting up the encrypted volume

Keeping the first partition (EFI boot partition) intact, I changed the partition table to this, not setting a 
filesystem type on the partition that is to contain the encrypted volume.

```
(parted) select /dev/nvme0n1
Using /dev/nvme0n1
(parted) print
Model: Unknown (unknown)
Disk /dev/nvme0n1: 500GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                  Flags
 1      1049kB  538MB   537MB   fat32        EFI System Partition  boot, esp
 2      538MB   1612MB  1074MB  ext2         boot
 3      1612MB  500GB   498GB                everything
```

The next steps come from [this article on askubuntu](https://askubuntu.com/questions/623814/install-ubuntu-15-04-with-full-disk-encryption-but-without-swap-partition).

```
cryptsetup --key-size 512 luksFormat /dev/nvme0n1p3
cryptsetup luksOpen /dev/nvme0n1p3 crypted
```

Now set up LVM (xubuntu installer didn't let me pick /dev/mapper/crypted for / directly, so I needed LVM)

```
pvcreate /dev/mapper/crypted
vgcreate vg /dev/mapper/crypted
lvcreate -l 100%FREE vg -n root
``` 

Manually format the partitions (xubuntu installer hung on formatting /boot for me)

```
mkfs -t ext2 /dev/nvme0n1p2
mkfs -t ext4 /dev/mapper/vg-root
```

### Run Installation

Run the installer, choose "Something else", set up partitions 

- `/dev/mapper/vg-root` for `/` with ext4 
- `/dev/nvme0n1p2` for `/boot` with ext2

Install the boot loader into `/dev/nvme0n1p2` (just to be safe in case install runs
in bios mode, the choice is ignored in EFI mode anyway).

When the installer finishes, click "Continue Testing". 

### chroot into the fresh installation

Now we chroot into the installed system, mounting everything that is needed. 

See [this helpful article from askubuntu](https://askubuntu.com/questions/976894/install-package-to-ubuntu-16-04-installation-while-booted-into-live-cd)

```
mount /dev/nvme0n1p2 /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot/efi
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run
chroot /mnt
```

Now we can make changes to the installed system even though we actually booted the live system.

### Configure driver repo and install drivers

See the relevant [System76 support article](http://support.system76.com/articles/system76-driver/). 
Obviously, this part will only work for one of their products.

```
apt-add-repository -ys ppa:system76-dev/stable
apt-get update
apt-get install -y system76-driver
```

### Install nvidia firmware package

??? not sure if this step is needed ???

```
apt-get install nouveau-firmware
```

### Configure encryption

Create `/etc/crypttab` inside the installed system:

```
echo "crypted UUID=`blkid -o value /dev/nvme0n1p3|head -1` none luks" | tee /etc/crypttab
```

### Update all packages on the system

This will install updated firmware packages to the system, and update many of the other packages.

```
apt-get upgrade
```

### Configure grub2

Now configure grub permanently as you like using `vi /etc/default/grub`. 

See 
[this helpful article](https://wiki.ubuntuusers.de/GRUB_2/Konfiguration/) that describes all the options.
I made these changes:

- change `quiet splash` to `nosplash noplymouth`
- comment out `#GRUB_HIDDEN_TIMEOUT=0`
- add `GRUB_TIMEOUT_STYLE=menu`

Run `update-grub2` to activate your changes.

### Regenerate initramfs

This follows some of the steps in
[this helpful article](http://thesimplecomputer.info/full-disk-encryption-with-ubuntu)

Create a new initramfs that contains the correct setup and firmware files, so we can boot.

```
update-initramfs -u
```

### Reboot

Now unmount and shut down, removing the installation media. Don't forget to press Enter
when the xubuntu installer prompts for it, or it will not shut down.

```
exit
umount /mnt/dev
umount /mnt/run
umount /mnt/dev
umount /mnt/proc
umount /mnt/boot/efi
umount /mnt/boot
umount /mnt
```

## Initial System Configuration

### Switch to NVidia driver

This gives you some measure of fan control, although the fan still never fully turns off.

### Touchpad config

Script to stop annoying touchpad tap-to-click:

```
synclient MaxTapTime=0
```

Has to be run each session.

### Install ansible

```
apt-get install software-properties-common
apt-add-repository ppa:ansible/ansible
apt-get update
apt-get install ansible
```

in `/etc/ansible/hosts` put:

```
[devlaptops]
localhost ansible_connection=local
```

## Now use the ansible files

You can run any of the provided ansible playbooks using

```
ansible-playbook x.yml
```
