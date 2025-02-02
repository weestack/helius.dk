---
date: 2025-01-25
draft: false
tags:
  - linux
  - mdadm
title: Grub recovery to boot up non bootable linux instance
icon: simple/linux
---

# mdadm replace disk
When working with dedicated hardware, there are times when a disk may fail or become unstable. This is where RAID comes into play, allowing data to be duplicated across disks using different strategies to maintain redundancy and minimize downtime.

Use the following commands to list all disks and identify the RAID setup:
```bash
# Use lsblk to list all disks
# Check raid details if in doubt
mdadm --detail /dev/md0
mdadm --detail /dev/md1
mdadm --detail /dev/md2
```

Take note of the faulty disks and use diagnostic tools such as smartctl to confirm the disk's health.

Mark the faulty disk as such in the RAID array:
```bash
mdadm --manage --set-faulty /dev/md0 /dev/nvme1n1p1
mdadm --manage --set-faulty /dev/md1 /dev/nvme1n1p2
mdadm --manage --set-faulty /dev/md2 /dev/nvme1n1p3
```

Finally, remove the faulty disk from the RAID:
```bash
mdadm /dev/md0 --remove /dev/nvme1n1p1
mdadm /dev/md1 --remove /dev/nvme1n1p2
mdadm /dev/md2 --remove /dev/nvme1n1p3
```

Now that the faulty disk has been removed, the RAID is in a degraded state. Proceed with formatting the replacement disk.

Write zeroes to the new disk to ensure it's ready for use:
```bash
dd if=/dev/zero of=/dev/nvme0n1 bs=1M status=progress
```

Clone the partition table from a functioning disk to the new disk:
```bash
sfdisk -d /dev/nvme0n1 | sfdisk /dev/nvme1n1
```

Add the new disk to the RAID and allow the synchronization to complete. Depending on the size and speed of the RAID, this process may take several hours:
```bash
mdadm /dev/md0 -a /dev/nvme1n1p1
mdadm /dev/md1 -a /dev/nvme1n1p2
mdadm /dev/md2 -a /dev/nvme1n1p3
```

Use the following command to monitor the status of the RAID synchronization in real-time:
```bash
watch -n 1 'cat /proc/mdstat'
```