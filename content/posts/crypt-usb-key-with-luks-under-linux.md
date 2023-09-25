---
title: "Encrypt your usb key under Linux with LUKS"
date: 2010-05-14
summary: "Learn LUKS to add some security on your USB keys"
tags:
  - linux
---

This morning, I format my usb key and create 2 partitions, a FAT one to share files with most hosts, and a crypted ext2 one for my personal data. The first part is very simple (*fdisk,mkfs.vfat*), so was the crypted part: It's much more simpler than in the past, when you had to patch kernel a lot of times, losetuping the device, try the passkey, etc.

To do that, I followed the ["Howto disk encryption with dm crypt luks and debian"](http://www.hermann-uwe.de/blog/howto-disk-encryption-with-dm-crypt-luks-and-debian) tutorial.

The procedure to me was the following one. Firstly, we create the partitions with fdisk:

```sh
$ fdisk /dev/sdb

Command (m for help): p

Disk /dev/sdb: 8053 MB, 8053063680 bytes
248 heads, 62 sectors/track, 1022 cylinders
Units = cylinders of 15376 * 512 = 7872512 bytes
Disk identifier: 0x000cb690

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         819     6296441    c  W95 FAT32 (LBA)
/dev/sdb2             820        1022     1560664   83  Linux

$ mkfs.vfat -n "voyager" /dev/sdb1
[...]
$ dd if=/dev/urandom of=/dev/sdb2
```

Disk initialization: We create the encrypted layer of the device:

```sh
$ cryptsetup --verbose --verify-passphrase luksFormat /dev/sdb2

WARNING!
========
This will overwrite data on /dev/sdb2 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter LUKS passphrase: 
Verify passphrase: 
Command successful.

$ cryptsetup luksOpen /dev/sdb2 cryptedDevice
Enter passphrase for /dev/sdb2: 
Key slot 0 unlocked.

$ ls -l /dev/mapper/
total 0
crw-rw---- 1 root root  10, 62 2010-03-29 11:56 control
brw-rw---- 1 root disk 253,  3 2010-05-14 13:14 cryptedDevice
```

You'll have a */dev/mapper/cryptedDevice* which is a virtual device, and you'll be able to format it:

```sh
$ mkfs.ext3 -j -m 1 -O dir_index,filetype,sparse_super /dev/mapper/cryptedDevice 
mke2fs 1.41.9 (22-Aug-2009)
(...)

$ mkdir /mnt/cryptedUsbKey ; mount /dev/mapper/cryptedDevice /mnt/cryptedUsbKey
```

At this moment, the device is mounted and fully usable!
To demount and disconnect it:

```sh
$ umount /mnt/cryptedUsbKey
$ cryptsetup luksClose /dev/mapper/cryptedDevice
```

With Ubuntu/Fedora, when a key is encrypted partition is created like this, you just have to unplug/plug the usb key to automaticaly recognize the encrypted paru'll also be able to add/remove passkeys with *luksAddKey* and *luksDelKey*.

See also [Encrypted Device Using LUKS](http://www.saout.de/tikiwiki/tiki-index.php?page=EncryptedDeviceUsingLUKS).
