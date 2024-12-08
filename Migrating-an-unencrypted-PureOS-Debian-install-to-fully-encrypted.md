The PureOS installer has an important weakness that is shared with many other distributions. When installing with full disk encryption, it creates an unencrypted /boot. This allows attackers to tamper with your linux image and initramfs even without decrypting the root partition, which is a serious weakness. It may allow injecting keyloggers, stealing your encryption passphrase and sending it to the attacker via the network, and many other attacks.

There is no reason today to not have an encrypted /boot. GRUB2 supports decrypting LUKS and LVM since (x year). Before that, there was a legitimate reason to have an unencrypted boot, since that was the only way to make it work, but today it's a security flaw.

I run an encrpted boot on all my personal systems, including Ubuntu, Fedora, Debian and PureOS.

My preferred way is to not have a separate boot partition at all. Just keep /boot inside the root partition. There's absolutely no downsides to this, and the benefit that you can't run out of space on the boot partition which has happened to me numerous times with a separate boot partition.

This guide applies to any Debian-based distribution including Ubuntu and PureOS.


## How to migrate an existing installation to fully encrypted

Note: to do this, you need separate hard drives for the old and new installation, or a disk image (clone) that you can loop-mount and copy the filesystems from.

### 1) Boot to a live linux desktop

I used a PureOS live USB as that's what I used to install the system and it already contains cryptsetup and lvm2.
You can use a different debian(-based) live distro and install cryptsetup and lvm2 yourself.

### 2) determine the sizes of partitions you'll need

#### 2.1) mount the old partitions
Replace oldroot, oldboot with the device names of your old system partitions (fdisk -l to see all disk partitions, commonly /dev/sdX# where X is a letter and # is a number)
```
mkdir /mnt/{oldroot,oldboot,newroot,newboot,newhome}
mount /dev/oldroot /mnt/oldroot
mount /dev/oldboot /mnt/oldboot
```
#### 2.2) delete unneeded files
```
du -axh /mnt/oldroot|sort -hr |less
```
If you're sure you don't need anything in the above output, delete it. DON'T DELETE system files. To be safe, don't delete anything outside your home folder. This isn't strictly necessary, it's just to minimise the amount of data to copy in the next steps.

#### 2.3) determine the new partition sizes
```
du -sh /mnt/oldroot
du -sh /mnt/oldboot
du -sh /mnt/oldroot/home
```
If you're going to make a separate home partition, the newroot minimum size is at least the size of oldroot+oldboot-(oldroot/home) + some spare. For a Gnome desktop I recommend ~25G root to account for extra installed applications.

For a single partition with everything, the minimum size of newroot is oldroot+oldboot plus some spare. This mainly depends on hoy much you intend to store in your /home. I recommend more than 100G for a daily use system.

#### 2.4) swap size
To support suspend to RAM, at least the amount of your installed RAM in the system, or the amount you plan to upgrade to.
You can see your current RAM with ```free -m``` under mem: total.

### 3) prepare the new drive

#### 3.1) secure erase
##### 3.1.1) SATA
Replace /dev/sdX with the new device name you'll be copying to. THIS ERASES ALL THE DATA ON THAT DEVICE. Be sure you've got the correct one (use fdisk -l, perhaps mount it in file manager and look into it, the unmount it)
```
hdparm -I /dev/sdX
hdparm --security-set-pass pass /dev/sdX
hdparm --security-erase pass /dev/sdX
```
##### 3.1.2) NVMe
use nvme sanitize command

#### 3.2) partition
TODO: use fdisk to create a new MBR and one primary partition spanning the whole drive, leave its type to 83.

#### 3.2) luksFormat

TODO -cipher/hash selection, cryptsetup benchmark, security (twofish) vs speed (aes-ni)

#### 3.3) LVM
TODO
-pvcreate, vgcreate
-lvcreate root
-lvcreate home (optional)
-lvcreate swap

### 4) prepare the old filesystem for copying

#### 4.1) shrink old root filesystem
```
umount /dev/oldroot
e2fsck -f /dev/oldroot
resize2fs -M /dev/oldroot
```

### 5) copy to new drive

#### 5.1) copy root
Replace $blocks with the number of 4k blocks resize2fs reported in the previous step.

```
dd if=/dev/oldroot of=/dev/newroot iflag=fullblock bs=4k count=$blocks
resize2fs /dev/newroot
```

#### 5.2) copy boot
```
mount /dev/oldboot /mnt/oldboot
mount /dev/newroot /mnt/newroot
cp -a /mnt/oldboot/* /mnt/newroot/boot/
umount /mnt/oldboot
```

#### 5.3) optional: copy home to separate partition
```
mount /dev/newhome /mnt/newhome
cp -a /mnt/newroot/home/* /mnt/newhome/
rm -rf /mnt/newroot/home/*
umount /mnt/newhome
mount /dev/newhome /mnt/newroot/home
```

### 6) chroot into the new root, update fstab, crypttab

#### 6.1) mount things needed to chroot
```
mount -o bind /dev /mnt/newroot/dev
mount -t proc proc /mnt/newroot/proc
mount -t sysfs sysfs /mnt/newroot/sys
mount -o bind /run /mnt/newroot/run
cp /etc/resolv.conf /mnt/newroot/etc/resolv.conf
```

#### 6.2) do the chroot
```
chroot /mnt/newroot /bin/bash
```
All commands from now on will be executed inside the chroot until you run "exit".

#### 6.3) install needed packages
```
apt-get update
apt-get install cryptsetup lvm2
```

#### 6.4) update fstab, create swap, update /etc/crypttab
```
blkid /dev/newroot
blkid /dev/newhome
mkswap -L pureos-swap /dev/newswap
```

```vim /etc/fstab``` (delete boot entry, update root and swap UUIDs with output from bklid, optional: add new home entry)

```blkid /dev/sdX#```

```vim /etc/crypttab``` add ```crypto-sdX# PARTUUID=... none luks``` if not already there, or update the UUID/PARTUUID with the one from above blkid.

```vim /etc/initramfs-tools/conf.d/cryptsetup```: make sure it contains:
```
CRYPTSETUP=yes
export CRYPTSeTUP
```


#### 6.5) do step 7 now if you prefer

#### 6.6) regenerate initramfs, grub config and install grub
```
update-initramfs -u -k all
add "GRUB_ENABLE_CRYPTODISK=y" to /etc/default/grub
update-grub
grub-install /dev/sdX
```

Now you can reboot and test. If anything goes wrong, reboot to the live USB, mount the new partitions, chroot (as per step 6) and fix what's wrong.


### 7) Unlock root with keyfile in initramfs
To prevent entering a password twice for the same volume (first for GRUB, then for initramfs) you can unlock the root via an embedded keyfile in initramfs.
This is safe, since the initramfs is located on the same encrypted partition that it is unlocking, hence the keyfile is encrypted.

You can do this while still chrooted.

```
mkdir /etc/cryptkeys
chmod 700 /etc/cryptkeys
dd bs=512 count=4 if=/dev/random of=/etc/cryptkeys/sdX#.key iflag=fullblock
chmod 400 /etc/cryptkeys/sdX#.key
chmod 600 /boot/initramfs-linux*
cryptsetup luksAddKey /dev/sdX# /etc/cryptkeys/sdX#.key

vim /etc/cryptsetup-initramfs/conf-hook (set KEYFILE_PATTERN="/etc/cryptkeys/*.key")
vim /etc/initramfs-tools/initramfs.conf (add UMASK=0077)
vim /etc/crypttab (change to "crypto-sdX# PARTUUID=... /etc/cryptkeys/sdX#.key luks")
update-initramfs -u -k all
```

### Sources
- https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#With_a_keyfile_embedded_in_the_initramfs
- https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=839994