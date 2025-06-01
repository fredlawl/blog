+++
date = '2025-05-31T09:56:16-05:00'
draft = false
title = 'Bootable ISO'
categories = ["learning"]
tags = ["qemu", "iso"]
next = true
toc = true
+++

## Background

A buddy of mine is building a customized Linux OS using
[Buildroot](buildroot.org). This sounded like a neat project, so I asked him
to keep me posted on his progress. His ultimate goal is to distribute his
OS to some hardware, but he wants to test in emulation first. This makes sense
to me. He uses [xcg-ng](https://xcp-ng.org/) virtual machines, and apparently
xcg-ng and wanted to boot from [Optical Disk Image (ISO)](https://en.wikipedia.org/wiki/Optical_disc_image)

There is a problem. Booting is hard, and it is hard
for me to help him without going through this process myself.

I build filesystems and kernels very often, but I never made a bootable ISO
image before. Typically I use [QEMU](https://www.qemu.org/) and do the
following for testing kernels:

```sh
qemu-system-x86_64 -enable-kvm -kernel ./bzImage -initrd ./rootfs.cpio -nographic -append 'console=ttyS0' -m 4G
```

This isn't good enough for distribution. To do that, I'd usually build a raw
image with partitions and distribute that. I'm fairly certain ISO works the same
way, and that it's possible to convert a raw image into a ISO image.

## Preparation

I wont go into using Buildroot, but I ended up with this structure:

```sh
➜  iso tree -h images
[   72]  images
├── [  13M]  bzImage
├── [    6]  efi-part
│   └── [    8]  EFI
│       └── [   38]  BOOT
│           ├── [ 608K]  bootx64.efi
│           └── [  117]  grub.cfg
├── [  96M]  rootfs.cpio
└── [ 104M]  rootfs.tar

4 directories, 5 files
```

The above QEMU command successfully boots this, and that's a good first sign
I have a working system.

I decided to include [GRUB](https://www.gnu.org/software/grub/) bootloader as
part of the build, and I noticed something interesting:

_efi-part/EFI/BOOT/grub.cfg_

```
set default="0"
set timeout="5"

menuentry "Buildroot" {
 linux /boot/bzImage root=/dev/sda1 rootwait console=tty1
}
```

The GRUB config seemed... barren... It's expecting that our root filesystem
exists in the /dev/sda1 device. In our QEMU command, I use `initrd` as
the filesystem that's loaded into ram. But our entry doesn't want that.

Next, I've never seen the `rootwait` option before. So I looked it up in the
[Admin guide - kernel parameters](https://www.kernel.org/doc/html/v6.12/admin-guide/kernel-parameters.html):

```txt
rootwait [KNL] Wait (indefinitely) for root device to show up.
         Useful for devices that are detected asynchronously
         (e.g. USB and MMC devices).
```

I can test the rootwait functionality with QEMU, but I don't have much
experience toying with hotplug devices, and I didn't have much luck with
setting up a usb-storage device to do so. It's safe to ignore this option.

Before making a bootable ISO, let's make a bootable image first.

### Bootable disk

The setup I want here is:

1. Partition 1: Ext4 1G
2. Partition 2: UEFI  10M
3. Partition 3: /boot ~1G

Modern systems use UEFI for boot, and should be able to detect the boot partition
regardless of location, and our menu expects our first device to be /dev/sda1. No
MBR for me!

```sh
➜  iso qemu-img create rootfs.img 2G

➜  iso fdisk rootfs.img

Welcome to fdisk (util-linux 2.40.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x2dd10ff3.

Command (m for help): g
Created a new GPT disklabel (GUID: E2E0BA42-2265-42CE-A1AA-FB9BCA8AD8ED).

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-4194270, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4194270, default 4192255): +1G

Created a new partition 1 of type 'Linux filesystem' and of size 1 GiB.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): L
Type of partition 1 is unchanged: Linux filesystem.

Command (m for help): n
Partition number (2-128, default 2):
First sector (2099200-4194270, default 2099200):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2099200-4194270, default 4192255): +10M

Created a new partition 2 of type 'Linux filesystem' and of size 10 MiB.

Command (m for help): i
Partition number (1,2, default 2): 2

         Device: rootfs.img2
          Start: 2099200
            End: 2119679
        Sectors: 20480
           Size: 10M
           Type: Linux filesystem
      Type-UUID: 0FC63DAF-8483-4772-8E79-3D69D8477DE4
           UUID: E251FA43-7073-4EB5-850F-3F96573E7CAD

Command (m for help): t
Partition number (1,2, default 2): 2
Partition type or alias (type L to list all): L
Partition type or alias (type L to list all): 1

Changed type of partition 'Linux filesystem' to 'EFI System'.

Command (m for help): n
Partition number (3-128, default 3):
First sector (2119680-4194270, default 2119680):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2119680-4194270, default 4192255):

Created a new partition 3 of type 'Linux filesystem' and of size 1012 MiB.

Command (m for help): t
Partition number (1-3, default 3): 3
Partition type or alias (type L to list all): L
Partition type or alias (type L to list all): 142

Changed type of partition 'Linux filesystem' to 'Linux extended boot'.

Command (m for help): p
Disk rootfs.img: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: E2E0BA42-2265-42CE-A1AA-FB9BCA8AD8ED

Device       Start     End Sectors  Size Type
rootfs.img1     2048 2099199 2097152    1G Linux filesystem
rootfs.img2  2099200 2119679   20480   10M EFI System
rootfs.img3  2119680 4192255 2072576 1012M Linux extended boot

Command (m for help): w
The partition table has been altered.
```

Then format & populate the partitions:

```sh
➜  iso mkdir -p boot efi fs
➜  iso sudo mkfs.ext4 /dev/loop0p1
mke2fs 1.47.2 (1-Jan-2025)
Discarding device blocks: done
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: 19096c35-e1f0-4ec6-9fd2-2e73956ffbfc
Superblock backups stored on blocks:
 32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

➜  iso sudo mkfs.ext4 /dev/loop0p3
mke2fs 1.47.2 (1-Jan-2025)
Discarding device blocks: done
Creating filesystem with 259072 4k blocks and 64768 inodes
Filesystem UUID: 35db06b8-5ab6-4785-9f97-adae766a6aa0
Superblock backups stored on blocks:
 32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

➜  iso sudo mkfs.vfat /dev/loop0p2
mkfs.fat 4.2 (2021-01-31)

➜  iso sudo mount /dev/loop0p1 fs
➜  iso sudo mount /dev/loop0p2 efi
➜  iso sudo mount /dev/loop0p3 boot

➜  iso sudo tar -xpvf images/rootfs.tar -C ./fs
...

➜  iso sudo cp -r images/efi-part/* efi
➜  iso sudo mkdir -p boot/boot
➜  iso sudo cp images/bzImage boot/boot
```

Buildroot certainly has options to make this process seamless. However, I used
nearly all defaults when making the image. This means I need to modify the GRUB
boot entry to work with my kernel and disk setup.

QEMU is setup on ttyS0, I don't have VGA enabled, therefore I need to set the
correct console output. GRUB also doesn't know what the boot partition is,
so I set the root to be that partition.

_efi/EFI/BOOT/grub.cfg_

```txt
set default="0"
set timeout="5"
set root=(hd0,gpt3)

menuentry "Buildroot" {
 linux /boot/bzImage root=/dev/sda1 console=ttyS0
}
```

```sh
➜  iso sudo umount ./fs ./efi ./boot
➜  iso sudo losetup -D
```

With QEMU:

```sh
qemu-system-x86_64 \
    -enable-kvm \
    -m 4G \
    -nographic \
    -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
    -device virtio-scsi-pci,id=scsi \
    -drive id=img,if=none,format=raw,file=rootfs.img \
    -device scsi-hd,drive=img
```

BOOOT!

The big thing I added here is a `bios` option to ensure QEMU uses UEFI
BIOS firmware to boot our disk.

## ISO

I made an assumption before starting this process that it should be trivial
to convert a disk image into a ISO. That was bold of me. I didn't even do the
research first. The formats are completely incompatible and is a rabbit hole of
a mess to sift through.

Most people are interested in extracting information from ISO's than creating
bootable ISOs. Fortunately, [Debian](https://wiki.debian.org/RepackBootableISO#Determine_those_options_which_need_to_be_adapted_on_amd64_or_i386)
saved me!

```sh
➜  iso mkdir -p testiso

➜  iso sudo cp images/bzImage testiso/boot

➜  iso sudo cp -r images/efi-part/* testiso/

➜  iso cat testiso/EFI/BOOT/grub.cfg
set default="0"
set timeout="5"

menuentry "Buildroot" {
 linux /boot/bzImage root=/dev/sr0 console=ttyS0
}

➜  iso sudo xorriso -as mkisofs \
    -iso-level 3 \
    -r -V "TEST" \
    -J -joliet-long \
    -no-emul-boot \
    -o testiso.iso \
    testiso

➜  iso qemu-system-x86_64 \
    -enable-kvm \
    -m 4G \
    -nographic \
    -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
    -device virtio-scsi-pci \
    -drive id=cd,if=none,file=testiso.iso,media=cdrom \
    -device scsi-cd,drive=cd
```

BOOOT!

What's interesting about bootable ISOs is that they're read-only. When I live
boot, there's not much I can do. This makes live ISOs very good for recovery,
but you're on your own to mount filesystems onto disks to have some RW
capabilities.

## Lessons learned

While a fun experiment, I feel it's better for my buddy to not worry about
distributing his OS via ISO. At least not without some installer to go with it.

Best to write a disk image to a USB stick. The reverse
is true too, ISO can be byte-for-byte written to USB. USB the disks are
bootable and writable, after all.

This is exactly what [Unraid](https://unraid.net/?srsltid=AfmBOopbUtS-EiWq8iOhUCvlaxOKd9oCnR8LF-WrqYwn_vJKwAy8NIQX)
does. They ship [Slackware](http://www.slackware.com/) linux.

Here's how we can boot from USB in QEMU:

```sh
qemu-system-x86_64 \
    -enable-kvm \
    -m 4G \
    -nographic \
    -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \
    -drive id=img,if=none,format=raw,file=rootfs.img \
    -usb \
    -device nec-usb-xhci,id=xhci \
    -device usb-storage,bus=xhci.0,drive=img
```

BOOOT!

All that said, making a bootable ISO is still good because now I know
how to make media in which I can run debugging tooling from.

I'm still on the fence of which approach is easier for me. Maybe ISO with
more practice...
