Source: [Red Hat Certified System Administrator exam | EX200](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam?section=objectives)
RHCSA exam candidates should be able to accomplish the tasks below without assistance. These have been grouped into several categories.

## Understand and use essential tools

- [x] Access a shell prompt and issue commands with correct syntax
- [ ] Use input-output redirection (>, >>, |, 2>, etc.)
- [ ] Use grep and regular expressions to analyze text
- [x] Access remote systems using SSH
- [x] Log in and switch users in multiuser targets
- [x] Archive, compress, unpack, and uncompress files using tar, gzip, and bzip2
- [x] Create and edit text files
- [x] Create, delete, copy, and move files and directories
- [x] Create hard and soft links
- [x] List, set, and change standard ugo/rwx permissions
- [x]  Locate, read, and use system documentation including man, info, and files in /usr/share/doc

## Create simple shell scripts

- [ ] Conditionally execute code (use of: if, test, [], etc.)
- [ ] Use Looping constructs (for, etc.) to process file, command line input
- [ ] Process script inputs ($1, $2, etc.)
- [ ] Processing output of shell commands within a script

## Operate running systems

- [x] Boot, reboot, and shut down a system normally
- [ ] Boot systems into different targets manually
- [ ] Interrupt the boot process in order to gain access to a system
- [x] Identify CPU/memory intensive processes and kill processes
- [ ] Adjust process scheduling
- [x] Manage tuning profiles
- [x] Locate and interpret system log files and journals
- [ ] Preserve system journals
- [ ] Start, stop, and check the status of network services
- [x] Securely transfer files between systems

## Configure local storage

- [x] List, create, delete partitions on MBR and GPT disks
- [x] Create and remove physical volumes
- [x] Assign physical volumes to volume groups
- [x] Create and delete logical volumes
- [x] Configure systems to mount file systems at boot by universally unique ID (UUID) or label
- [x] Add new partitions and logical volumes, and swap to a system non-destructively

## Create and configure file systems

- [x] Create, mount, unmount, and use vfat, ext4, and xfs file systems
- [ ] Mount and unmount network file systems using NFS
- [ ] Configure autofs
- [x] Extend existing logical volumes
- [ ] Create and configure set-GID directories for collaboration
- [x] Diagnose and correct file permission problems

## Deploy, configure, and maintain systems

- [x] Schedule tasks using at and cron
- [x] Start and stop services and configure services to start automatically at boot
- [ ] Configure systems to boot into a specific target automatically
- [x] Configure time service clients
- [ ] Install and update software packages from Red Hat Network, a remote repository, or from the local file system
- [ ] Modify the system bootloader

## Manage basic networking

- [x] Configure IPv4 and IPv6 addresses
- [x] Configure hostname resolution
- [ ] Configure network services to start automatically at boot
- [x] Restrict network access using firewall-cmd/firewall

## Manage users and groups

- [x] Create, delete, and modify local user accounts
- [ ] Change passwords and adjust password aging for local user accounts
- [x] Create, delete, and modify local groups and group memberships
- [ ] Configure superuser access

## Manage security

- [x] Configure firewall settings using firewall-cmd/firewalld
- [x] Manage default file permissions
- [x] Configure key-based authentication for SSH
- [x] Set enforcing and permissive modes for SELinux
- [x] List and identify SELinux file and process context
- [x] Restore default file contexts
- [x] Manage SELinux port labels
- [ ] Use boolean settings to modify system SELinux settings
- [ ] Diagnose and address routine SELinux policy violations

## Manage containers

- [x] Find and retrieve container images from a remote registry
- [x] Inspect container images
- [ ] Perform container management using commands such as podman and skopeo
- [x] Perform basic container management such as running, starting, stopping, and listing running containers
- [ ] Run a service inside a container
- [x] Configure a container to start automatically as a systemd service
- [x] Attach persistent storage to a container

As with all Red Hat performance-based exams, configurations must persist after reboot without intervention.