---
date: 2025-01-25
draft: false
tags:
  - linux
  - grub2
title: Grub recovery to boot up non bootable linux instance
icon: simple/gnubash
---

# Grub recovery
If, for some reason, your Linux system won't boot, you can spin up a rescue system.  
This issue might occur after a kernel upgrade or downgrade.

First of all, if the system uses RAID, you need to assemble it:
```bash
mdadm --assemble --scan
```
Next, mount the appropriate disks:
```bash
# lsblk to identify them
mount /dev/md127 /mnt/
mount /dev/md126 /mnt/boot/
# Last md device will be swap, skip that
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
```
Now, for the GRUB commands to work, change the root environment and make it think `/mnt` is the root. This can be achieved with `chroot`:
```bash
chroot /mnt
grub-install
# If grub install fails use: on all devices
grub-install /dev/nvme0n1 && grub-install /dev/nvme1n1
```
Finally, update GRUB:
```bash
grub-update
```
Depending on your system and the specific issue, you might need to take additional steps to get the system working. This guide serves as a baseline for troubleshooting when systems won't boot.

If the system is still not able to boot, then try to investigate the boot partition

a few note worthy things that could be an issue:  
1) fstab no longer has the correct disk for boot  
2) grub is not installed on the disk install with    
```bash
grub-install /dev/disk0n1
```