# Boot Process
## Select the Boot Target

### Describe the Red Hat Enterprise Linux 9 Boot Process

Modern computer systems are complex combinations of hardware and software. Starting from an undefined, powered-down state to a running system with a login prompt requires many pieces of hardware and software to work together. The following list gives a high-level overview of the tasks for a physical `x86_64` system that boots Red Hat Enterprise Linux 9. The list for `x86_64` virtual machines is similar, except that the hypervisor handles some hardware-specific steps in software.

- The machine is powered on. The system firmware, either modern UEFI or earlier BIOS, runs a _Power On Self Test (POST)_ and starts to initialize the hardware.
- The UEFI boot firmware is configured by searching for a bootable device, which searches for or configures the _Master Boot Record (MBR)_ on all disks.
- The system firmware reads a boot loader from disk and then passes control of the system to the boot loader. On a Red Hat Enterprise Linux 9 system, the boot loader is the _GRand Unified Bootloader version 2 (GRUB2)_.

   The `grub2-install` command installs GRUB2 as the boot loader on the disk for BIOS systems. Do not use the `grub2-install` command directly to install the UEFI boot loader. RHEL 9 provides a prebuilt `/boot/efi/EFI/redhat/grubx64.efi` file, which contains the required authentication signatures for a Secure Boot system. Executing `grub2-install` directly on a UEFI system generates a new `grubx64.efi` file without those required signatures. You can restore the correct `grubx64.efi` file from the `grub2-efi` package.

- GRUB2 loads its configuration from the `/boot/grub2/grub.cfg` file for BIOS, and from the `/boot/efi/EFI/redhat/grub.cfg` file for UEFI, and displays a menu to select which kernel to boot.

  GRUB2 is configured by using the `/etc/grub.d/` directory and the `/etc/default/grub` file. The `grub2-mkconfig` command generates the `/boot/grub2/grub.cfg` or `/boot/efi/EFI/redhat/grub.cfg` files for BIOS or UEFI, respectively.

- After you select a kernel, or the timeout expires, the boot loader loads the kernel and _initramfs_ from disk and places them in memory. An `initramfs` image is an archive with the kernel modules for all the required hardware at boot, initialization scripts, and more. In Red Hat Enterprise Linux 9, the `initramfs` image contains a bootable root file system with a running kernel and a `systemd` unit.

   The `initramfs` image is configured by using the `/etc/dracut.conf.d/` directory, the `dracut` command, and the `lsinitrd` command to inspect the `initramfs` file.

- The boot loader hands control over to the kernel, and passes in any specified options on the kernel command line in the boot loader, and the location of the `initramfs` image in memory. 
- The kernel initializes all hardware for which it can find a driver in the `initramfs` image, and then executes the `/sbin/init` script from the `initramfs` image as PID 1. On Red Hat Enterprise Linux 9, the `/sbin/init` script is a link to the `systemd` unit.

  The script is configured by using the kernel `init=` command-line parameter.

- The `systemd` unit from the `initramfs` image executes all units for the `initrd.target` target. This unit includes mounting the root file system on disk to the `/sysroot` directory.

  Configured by using the `/etc/fstab` file.

- The kernel switches (pivots) the root file system from the `initramfs` image to the root file system in the `/sysroot` directory. The `systemd` unit then re-executes itself by using the installed copy of the `systemd` unit on the disk.
- The `systemd` unit looks for a default target, which is either passed in from the kernel command line or is configured on the system. The `systemd` unit then starts (and stops) units to comply with the configuration for that target, and solves dependencies between units automatically. A `systemd` unit is a set of units that the system activates to reach the intended state. These targets typically start a text-based login or a graphical login screen.

  Configured by using the `/etc/systemd/system/default.target` file and the `/etc/systemd/system/` directory.

|   |
|---|
|![](https://rol.redhat.com/rol/static/static_file_cache/rh134-9.0/boot/bios_uefi_boot_process.svg)|

Figure 10.1: Boot process for BIOS-based and UEFI-based systems

### Power Off and Reboot
To power off or reboot a running system from the command line, you can use the `systemctl` command.

The `systemctl poweroff` command stops all running services, unmounts all file systems (or remounts them read-only when they cannot be unmounted), and then powers down the system.

The `systemctl reboot` command stops all running services, unmounts all file systems, and then reboots the system.

You can also use the shorter version of these commands, `poweroff` and `reboot`, which are symbolic links to their `systemctl` equivalents.

### Note
The `systemctl halt` and `halt` commands are also available to stop the system. Unlike the `poweroff` command, these commands do not power off the system; they bring down a system to a point where it is safe to power it off manually.

### Select a Systemd Target
A `systemd` target is a set of `systemd` units that the system must start to reach an intended state. The following table lists the most important targets:

**Table 10.1. Commonly Used Targets**

|Target|Purpose|
|:--|:--|
|`graphical.target`|This target supports multiple users, and provides graphical- and text-based logins.|
|`multi-user.target`|This target supports multiple users, and provides text-based logins only.|
|`rescue.target`|This target provides a single-user system to enable repairing your system.|
|`emergency.target`|This target starts the most minimal system for repairing your system when the `rescue.target` unit fails to start.|


A target can be a part of another target. For example, the `graphical.target` unit includes the `multi-user.target` unit, which in turn depends on the `basic.target` unit and others. You can view these dependencies with the following command:
```bash
[user@host ~]$ systemctl list-dependencies graphical.target | grep target
graphical.target
* └─multi-user.target
*   ├─basic.target
*   │ ├─paths.target
*   │ ├─slices.target
*   │ ├─sockets.target
*   │ ├─sysinit.target
*   │ │ ├─cryptsetup.target
*   | | ├─integritysetup.target
*   │ │ ├─local-fs.target
_...output omitted..._

```

To list the available targets, use the following command:
```bash
[user@host ~]$ systemctl list-units --type=target --all
  UNIT                      LOAD      ACTIVE   SUB    DESCRIPTION
  ---------------------------------------------------------------------------
  basic.target              loaded    active   active Basic System
_...output omitted..._
  cloud-config.target       loaded    active   active Cloud-config availability
  cloud-init.target         loaded    active   active Cloud-init target
  cryptsetup-pre.target     loaded    inactive dead   Local Encrypted Volumes (Pre)
  cryptsetup.target         loaded    active   active Local Encrypted Volumes
_...output omitted..._

```

#### Select a Target at Runtime
On a running system, administrators can switch to a different target by using the `systemctl isolate` command.
```bash
[root@host ~] systemctl isolate multi-user.target
```

Isolating a target stops all services that the target does not require (and its dependencies), and starts any required services that are not yet started.

Not all targets can be isolated. You can isolate only targets where `AllowIsolate=yes` is set in their unit files. For example, you can isolate the graphical target, but not the `cryptsetup` target.
```bash

[user@host ~]$ **`systemctl cat graphical.target`**
# /usr/lib/systemd/system/graphical.target
_...output omitted..._
[Unit]
Description=Graphical Interface
Documentation=man:systemd.special(7)
Requires=multi-user.target
Wants=display-manager.service
Conflicts=rescue.service rescue.target
After=multi-user.target rescue.service rescue.target display-manager.service
`AllowIsolate=yes`
[user@host ~]$ systemctl cat cryptsetup.target
# /usr/lib/systemd/system/cryptsetup.target
_...output omitted..._
[Unit]
Description=Local Encrypted Volumes
Documentation=man:systemd.special(7)
```
#### Set a Default Target
When the system starts, the `systemd` unit activates the `default.target` target. Normally, the default target the `/etc/systemd/system/` directory is a symbolic link to either the `graphical.target` or the `multi-user.target` targets. Instead of editing this symbolic link by hand, the `systemctl` command provides two subcommands to manage this link: `get-default` and `set-default`.
```bash
[root@host ~] systemctl get-default
multi-user.target
[root@host ~] systemctl set-default graphical.target
Removed /etc/systemd/system/default.target.
Created symlink /etc/systemd/system/default.target -> /usr/lib/systemd/system/graphical.target.
[root@host ~] systemctl get-default
graphical.target
```

#### Select a Different Target at Boot Time
To select a different target at boot time, append the ``systemd.unit=_`target`_.target`` option to the kernel command line from the boot loader.

For example, to boot the system into a rescue shell where you can change the system configuration with almost no services running, append the following option to the kernel command line from the boot loader:

`systemd.unit=rescue.target`

This configuration change affects only a single boot, and is a useful tool to troubleshoot the boot process.

To use this method to select a different target, use the following procedure:

1. Boot or reboot the system.
2. Interrupt the boot loader menu countdown by pressing any key (except **Enter**, which would initiate a normal boot).
3. Move the cursor to the kernel entry to start.
4. Press **e** to edit the current entry.
5. Move the cursor to the line that starts with `linux` which is the kernel command line.
6. Append ``systemd.unit=_`target`_.target``, for example, `systemd.unit=emergency.target`.
7. Press **Ctrl**+**x** to boot with these changes.

## Reset the Root Password
### Reset the Root Password from the Boot Loader
One task that every system administrator should be able to accomplish is resetting a lost `root` password. This task is trivial if the administrator is still logged in, either as an unprivileged user but with full `sudo` access, or as `root`. This task becomes slightly more involved when the administrator is not logged in.

Several methods exist to set a new `root` password. A system administrator could, for example, boot the system by using a Live CD, mount the root file system from there, and edit `/etc/shadow`. This section explores a method that does not require the use of external media.

On Red Hat Enterprise Linux 9, the scripts that run from the `initramfs` image can be paused at certain points, to provide a `root` shell, and then continue when that shell exits. This script is mostly meant for debugging, and also to reset a lost `root` password.

Starting from Red Hat Enterprise Linux 9, if you install your system from a DVD, then the default kernel asks for the `root` password when you try to enter maintenance mode. Thus, to reset a lost `root` password, you must use the rescue kernel.

To access that `root` shell, follow these steps:

1. Reboot the system.
2. Interrupt the boot-loader countdown by pressing any key, except **Enter**.
3. Move the cursor to the rescue kernel entry to boot (the entry with the _rescue_ word in its name).
4. Press **e** to edit the selected entry.
5. Move the cursor to the kernel command line (the line that starts with `linux`).
6. Append `rd.break`. With that option, the system breaks just before the system hands control from the `initramfs` image to the actual system.
7. Press **Ctrl**+**x** to boot with the changes.
8. Press **Enter** to perform maintenance when prompted.

At this point, the system presents a `root` shell, and the root file system on the disk is mounted read-only on `/sysroot`. Because troubleshooting often requires modifying the root file system, you must remount the root file system as read/write. The following step shows how the `remount,rw` option to the `mount` command remounts the file system where the new option (`rw`) is set.

### Important

Because the system has not yet enabled SELinux, any file that you create does not have SELinux context. Some tools, such as the `passwd` command, first create a temporary file, and then replace it with the file that is intended for editing, which effectively creates a file without SELinux context. For this reason, when you use the `passwd` command with `rd.break`, the `/etc/shadow` file does not receive SELinux context.

To reset the `root` password, use the following procedure:
```bash
# Remount `/sysroot` as read/write.
sh-5.1 mount -o remount,rw /sysroot

# Switch into a `chroot` jail, where `/sysroot` is treated as the root of the file-system tree.
sh-5.1 chroot /sysroot

# Set a new `root` password.
sh-5.1 passwd root

# Ensure that all unlabeled files, including `/etc/shadow` at this point, get relabeled during boot.
sh-5.1 touch /.autorelabel

# Type `exit` twice. The first command exits the `chroot` jail, and the second command exits the `initramfs` debug shell.
```

At this point, the system continues booting, performs a full SELinux relabeling, and then reboots again.

#### Recovery of a Cloud Image-based System
If your system was installed by deploying and modifying one of the official cloud images instead of using the installer, then some system configuration aspects might differ for the boot process.

The procedure to use the `rd.break` option to get a root shell is similar to the previously outlined procedure, with some minor changes.

If your system was deployed from a Red Hat Enterprise Linux cloud image, then your boot menu does not have a rescue kernel by default. However, you can use the default kernel to enter maintenance mode by using the `rd.break` option without entering the root password.

The kernel prints boot messages and displays the root prompt on the system console. Prebuilt images might have multiple `console=` arguments on the kernel command line in the bootloader. Even though the system sends the kernel messages to all the consoles, the root shell that the `rd.break` option sets up uses the last console that is specified on the command line. If you do not get your prompt, then you might temporarily reorder the `console=` arguments when you edit the kernel command line in the boot loader.

### Inspect Logs
Looking at the logs of previously failed boots can be useful. If the system journals persist across reboots, then you can use the `journalctl` tool to inspect those logs.

Remember that by default, the system journals are kept in the `/run/log/journal` directory, and the journals are cleared when the system reboots. To store journals in the `/﻿var/log/journal` directory, which persists across reboots, set the `Storage` parameter to `persistent` in the `/etc/systemd/journald.conf` file.

```bash
[root@host ~] vim /etc/systemd/journald.conf
_...output omitted..._
[Journal]
Storage=persistent
_...output omitted..._
[root@host ~] systemctl restart systemd-journald.service
```

To inspect the logs of a previous boot, use the `journalctl` command `-b` option. Without any arguments, the `journalctl` command `-b` option displays only messages since the last boot. With a negative number as an argument, it displays the logs of previous boots.
```
[root@host ~]# journalctl -b -1 -p err
```

This command shows all messages that are rated as an error or worse from the previous boot.

### Repair Systemd Boot Issues
To troubleshoot service startup issues at boot time, Red Hat Enterprise Linux 8 and later versions provide the following tools:

#### Enable the Early Debug Shell
By enabling the `debug-shell` service with the `systemctl enable debug-shell.service` command, the system spawns a `root` shell on `TTY9` (**Ctrl**+**Alt**+**F9**) early during the boot sequence. This shell is automatically logged in as `root`, so that administrators can debug the system when the operating system is still booting.

**Warning**
Disable the `debug-shell.service` service when you are done debugging, because it leaves an unauthenticated `root` shell open to anyone with local console access.

Alternatively, to activate the debug shell during the boot by using the GRUB2 menu, follow these steps:
1. Reboot the system.
2. Interrupt the boot-loader countdown by pressing any key, except **Enter**.
3. Move the cursor to the kernel entry to boot.
4. Press **e** to edit the selected entry.
5. Move the cursor to the kernel command line (the line that starts with `linux`).
6. Append `systemd.debug-shell`. With this parameter, the system boots into the debug shell.
7. Press **Ctrl**+**x** to boot with the changes.

#### Use the Emergency and Rescue Targets
By appending either `systemd.unit=rescue.target` or `systemd.unit=emergency.target` to the kernel command line from the boot loader, the system enters into a rescue or emergency shell instead of starting normally. Both of these shells require the `root` password.

The **emergency target** keeps the root file system mounted read-only, while the **rescue target** waits for the `sysinit.target` unit to complete, so that more of the system is initialized, such as the logging service or the file systems. The root user at this point cannot change `/etc/fstab` until the drive is remounted in a read write state with the `mount -o remount,rw /` command.

Administrators can use these shells to fix any issues that prevent the system from booting normally, for example, a dependency loop between services, or an incorrect entry in `/etc/fstab`. Exiting from these shells continues with the regular boot process.

#### Identify Stuck Jobs
During startup, `systemd` spawns various jobs. If some of these jobs cannot complete, then they block other jobs from running. To inspect the current job list, administrators can use the `systemctl list-jobs` command. Any jobs that are listed as running must complete before the jobs that are listed as waiting can continue.

## Repair File-system Issues at Boot
To access a system that cannot complete booting because of file-system issues, the `systemd` architecture provides an `emergency` boot target, which opens an emergency shell that requires the `root` password for access.

The next example demonstrates the boot process output when the system finds a file-system issue and switches to the `emergency` target:
```bash
_...output omitted..._
[*     ] A start job is running for /dev/vda2 (27s / 1min 30s)
[ TIME ] Timed out waiting for device /dev/vda2.
[DEPEND] Dependency failed for /mnt/mountfolder
[DEPEND] Dependency failed for Local File Systems.
[DEPEND] Dependency failed for Mark need to relabel after reboot.
_...output omitted..._
[  OK  ] Started Emergency Shell.
[  OK  ] Reached target Emergency Mode.
_...output omitted..._
Give root password for maintenance
(or press Control-D to continue):
```

The `systemd` daemon failed to mount the `/dev/vda2` device and timed out. Because the device is not available, the system opens an emergency shell for maintenance access.

To repair file-system issues when your system opens an emergency shell, first locate the errant file system, and then find and repair the fault. Now reload the `systemd` configuration to retry the automatic mounting.

Use the `mount` command to find which file systems are currently mounted by the `systemd` daemon.
```bash
[root@host ~] mount
_...output omitted..._
/dev/vda1 on `/` type xfs (`ro`,relatime,seclabel,attr2,inode64,noquota)
_...output omitted..._
```

If the root file system is mounted with the `ro` (read-only) option, then you cannot edit the `/etc/fstab` file. Temporarily remount the root file system with the `rw` (read/write) option, if necessary, before opening the `/etc/fstab` file. With the remount option, an in-use file system can change its mount parameters without unmounting the file system.
```bash
[root@host ~] mount -o remount,rw /
```

Try to mount all the file systems that are listed in the `/etc/fstab` file by using the `mount --all` option. This option mounts processes on every file-system entry, but skips those file systems that are already mounted. The command displays any errors that occur when mounting a file system.
```
[root@host ~] mount --all
mount: /mnt/mountfolder: mount point does not exist.
```


In this scenario, where the `/mnt/mountfolder` mount directory does not exist, create the `/mnt/mountfolder` directory before reattempting the mount. Other error messages can occur, including typing errors in the entries, or wrong device names or UUIDs.

After you corrected all issues in the `/etc/fstab` file, inform the `systemd` daemon to register the new `/etc/fstab` file by using the `systemctl daemon-reload` command. Then, reattempt mounting all the entries.
```
[root@host ~] systemctl daemon-reload
[root@host ~] mount --all
```
### Note
The `systemd` service processes the `/etc/fstab` file by transforming each entry into a `.mount` type `systemd` unit configuration and then starting the unit as a service. The `daemon-reload` option requests the `systemd` daemon to rebuild and reload all unit configurations.

If the `mount --all` command succeeds without further errors, then the final test is to verify that file-system mounting is successful during a system boot. Reboot the system and wait for the boot to complete normally.
```bash
[root@host ~] systemctl reboot
```

For quick testing in the `/etc/fstab` file, use the `nofail` mount entry option. Using the `nofail` option in an `/etc/fstab` entry enables the system to boot even if that file-system mount is unsuccessful. This option must not be used with production file systems that must always mount. With the `nofail` option, an application could start when its file-system data is missing, with possibly severe consequences.
