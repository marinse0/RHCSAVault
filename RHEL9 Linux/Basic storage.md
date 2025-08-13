# Basic storage

The `/etc/fstab` file is a white-space-delimited file with six fields per line.
```bash
[root@host ~] cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Thu Apr 5 12:05:19 2022
#
# Accessible filesystems, by reference, are maintained under '/dev/disk/'.
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
#
# After editing this file, run 'systemctl daemon-reload' to update systemd
# units generated from this file.
#
UUID=a8063676-44dd-409a-b584-68be2c9f5570   /        xfs   defaults   0 0
UUID=7a20315d-ed8b-4e75-a5b6-24ff9e1f9838   /dbdata  xfs   defaults   0 0
```

- The first field specifies the device. This example uses a UUID to specify the device. File systems create and store the UUID in the partition super block at creation time. Alternatively, you could use the device file, such as `/dev/vdb1`.
- The second field is the directory mount point, from which the block device is accessible in the directory structure. The mount point must exist; if not, create it with the `mkdir` command.
- The third field contains the file-system type, such as `xfs` or `ext4`.
- The fourth field is the comma-separated list of options to apply to the device. `defaults` is a set of commonly used options. The `mount`(8) man page documents the other available options.
- The fifth field is used by the `dump` command to back up the device. Other backup applications do not usually use this field.
- The last field, the `fsck` order field, determines whether to run the `fsck` command at system boot to verify that the file systems are clean. The value in this field indicates the order in which `fsck` should run. For XFS file systems, set this field to `0`, because XFS does not use `fsck` to verify its file-system status. For ext4 file systems, set it to `1` for the root file system, and to `2` for the other ext4 file systems. By using this notation, the `fsck` utility processes the root file system first, and then verifies file systems on separate disks concurrently, and file systems on the same disk in sequence.

**Note**
An incorrect entry in `/etc/fstab` might render the machine non-bootable. Verify that an entry is valid by manually unmounting the new file system and then by using ``mount /_`mountpoint`_`` to read the `/etc/fstab` file, and remount the file system with that entry's mount options. If the `mount` command returns an error, then correct it before rebooting the machine.

Alternatively, use the `findmnt --verify` command to parse the `/etc/fstab` file for partition usability.

When you add or remove an entry in the `/etc/fstab` file, run the `systemctl daemon-reload` command, or reboot the server, to ensure that the `systemd` daemon loads and uses the new configuration.
[root@host ~]# **`systemctl daemon-reload`**

Red Hat recommends the use of UUIDs to persistently mount file systems, because block device names can change in certain scenarios, such as if a cloud provider changes the underlying storage layer of a virtual machine, or if disks are detected in a different order on a system boot. The block device file name might change, but the UUID remains constant in the file-system's super block.

Use the `lsblk --fs` command to scan the block devices that are connected to a machine and retrieve the file-system UUIDs.
```bash
[root@host ~] lsblk --fs
NAME   FSTYPE  FSVER  LABEL    UUID            FSAVAIL FSUSE% MOUNTPOINTS
vda
├─vda1
├─vda2 xfs            boot     49dd...75fdf    312M    37%    /boot
└─vda3 xfs            root     8a90...ce0da    4.8G    48%    /

```
## Create partition - parted
1. Create an `msdos` disk label on the `/dev/vdb` device.
```bash
[root@servera ~] parted /dev/vdb mklabel msdos
Information: You may need to update /etc/fstab.
```

2.  Use `parted` interactive mode to create the partition.
```bash
[root@servera ~] parted /dev/vdb
GNU Parted 3.4
Using /dev/vdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mkpart
Partition type?  primary/extended? primary
File system type?  [ext2]? xfs
Start? 2048s
End? 1001MB
(parted) quit
Information: You may need to update /etc/fstab.
```

Because the partition starts at the 2048 sector, the previous command sets the end position to 1001 MB to get a partition size of 1000 MB (1 GB).
Alternatively, you can perform the same operation with the following non-interactive command:
`parted /dev/vdb mkpart primary xfs 2048s 1001 MB`

3. Verify your work by listing the partitions on the `/dev/vdb` device.
```bash
[root@servera ~] parted /dev/vdb print
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: `msdos`
Disk Flags:
Number  Start   End     Size    Type     File system  Flags
1      1049kB  1001MB  `1000MB  primary`
```

4. Run the `udevadm settle` command. This command waits for the system to register the new partition, and returns when it is done.
```bash
udevadm settle
```

5. Format the new partition with the XFS file system.
```bash
mkfs.xfs /dev/vdb1
```

 6. Create the `/archive` directory.
```bash
mkdir /archive
```
 
 7. Discover the UUID of the `/dev/vdb1` device. The UUID in the output is probably different on your system.
```bash
lsblk --fs /dev/vdb
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
vdb
	└─vdb1 xfs                881e856c-37b1-41e3-b009-ad526e46d987
```
 8. Add an entry to the `/etc/fstab` file. Replace the UUID with the one that you discovered from the previous step.

 9. Update the `systemd` daemon for the system to register the new `/etc/fstab` file configuration.
```bash
systemctl daemon-reload
```
 10. Mount the new file system with the new entry in the `/etc/fstab` file.
