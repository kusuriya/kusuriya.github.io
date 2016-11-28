---
layout: default
title: UEFI on OpenBSD
date: 2015-10-12 03:58:00 -0000
tags: 
    - OpenBSD
    - UEFI
---
# {{ page.title }}  

After tons of hardwork a group of dedicated hackers have finally put out a UEFI boot loader for OpenBSD. While the installer does not yet deal with EFI boot blocks they are easily installed manually.

<!--more-->

* So to manually install it you will have to start with enabling UEFI mode in your firmware setup screen (BIOS Setup screen)
* With the latest snapshot written to a flash drive boot the flash drive in UEFI mode to see if it will work for you.
* if it worked for you congratulations now you want to set up your partitions: fdisk -i sd0
* if you want to use softraid here is where you would set the disk labels for that: printf "a\n\n\n\nRAID\nw\nq\n" | disklabel -E sd0
* do the softraid setup: bioctl -c C -l /dev/sd0a
* Setup the UEFI partition
  * format the partition: newfs_msdos /dev/sd0i
  * mount the EFI part: /mnt/efi
  * Make the EFI directory structure: mkdir /mnt/efi/EFI;mkdir /mnt/efi/EFI/BOOT
  * copy BOOTX64.EFI and BOOTIA32.EFI to /mnt/efi/EFI/BOOT
* Proceed with the OpenBSD install using the OpenBSD install script, from here the script will do everything else right and provide you with a disk that can be booted under MBR and the work you just finished will let it boot under UEFI.

A big thanks goes out to Toby Slight, I adapted the instructions from what he posted on misc@
