# Synchronize Files Between Systems Securely
## rsync
Use the `rsync` command `-n` option for a dry run. A dry run simulates what happens when the command is executed. The dry run shows the changes that the `rsync` command would perform when executing the command. Perform a dry run before the actual `rsync` command operation to ensure that no critical files are overwritten or deleted.

When synchronizing with the `rsync` command, two standard options are the `-v` and `-a` options.

The `rsync` command `-v` or `--verbose` option provides a more detailed output. This option is helpful for troubleshooting and viewing live progress.

The `rsync` command `-a` or `--archive` option enables "archive mode". This option enables recursive copying and turns on many valuable options to preserve most characteristics of the files. Archive mode is the same as specifying the following options:

**Table 4.1. Options Enabled with `rsync -a` (Archive Mode)**

|Option|Description|
|:--|:--|
|`-r`, `--recursive`|Synchronize the whole directory tree recursively|
|`-l`, `--links`|Synchronize symbolic links|
|`-p`, `--perms`|Preserve permissions|
|`-t`, `--times`|Preserve time stamps|
|`-g`, `--group`|Preserve group ownership|
|`-o`, `--owner`|Preserve the owner of the files|
|`-D`, `--devices`|Preserve device files|

Archive mode does not preserve hard links, because it might add significant time to the synchronization. Use the `rsync` command `-H` option to preserve hard links too.

### Note

To include extended attributes when syncing files, add these options to the `rsync` command:
- `-A` to preserve Access Control Lists (ACLs)
- `-X` to preserve SELinux file contexts

You can use the `rsync` command to synchronize the contents of a local file or directory with a file or directory on a remote machine, with either machine as the source. You can also synchronize the contents of two local files or directories on the same machine.

You must be the `root` user on the destination system to preserve file ownership. If the destination is remote, then authenticate as the `root` user. If the destination is local, then you must run the `rsync` command as the `root` user.

In this example, synchronize the local `/var/log` directory to the `/tmp` directory on the `hosta` system:
```bash
[root@host ~] rsync -av /var/log hosta:/tmp
root@hosta password: password
receiving incremental file list
log/
log/README
log/boot.log
_...output omitted..._
sent 9,783 bytes  received 290,576 bytes  85,816.86 bytes/sec
total size is 11,585,690  speedup is 38.57
```

In the same way, the `/var/log` remote directory on the `hosta` machine synchronizes to the `/tmp` directory on the `host` machine:
```bash
[root@host ~] rsync -av hosta:/var/log /tmp
root@hosta password: password
receiving incremental file list
log/boot.log
log/dnf.librepo.log
log/dnf.log
_...output omitted..._
sent 9,783 bytes  received 290,576 bytes  85,816.86 bytes/sec
total size is 11,585,690  speedup is 38.57
```

The following example synchronizes the contents of the `/var/log` directory to the `/tmp` directory on the same machine:
```bash
[user@host ~]$ su -
Password: password
[root@host ~] rsync -av /var/log /tmp
receiving incremental file list
log/
log/README
log/boot.log
_...output omitted..._
log/tuned/tuned.log

sent 11,592,423 bytes  received 779 bytes  23,186,404.00 bytes/sec
total size is 11,586,755  speedup is 1.00
[user@host ~]$
[user@host ~]$ ls /tmp
log  ssh-RLjDdarkKiW1
[user@host ~]$
```

### Important

Correctly specifying a source directory trailing slash is important.
- A source directory **with a trailing slash** synchronizes the contents of the directory _without_ including the directory itself. The contents are synced directly into the destination directory.
- **Without the trailing slash,** the source directory itself will sync to the destination directory. The source directory's contents are in the new subdirectory in the destination.

Bash **Tab**-completion automatically adds a trailing slash to directory names.

In this example, the content of the `/var/log/` directory is synchronized to the `/tmp` directory instead of the `log` directory being created in the `/tmp` directory.
```bash
[root@host ~] rsync -av /var/log/ /tmp
sending incremental file list
./
README
boot.log
_...output omitted..._
tuned/tuned.log

sent 11,592,389 bytes  received 778 bytes  23,186,334.00 bytes/sec
total size is 11,586,755  speedup is 1.00
[root@host ~] ls /tmp
anaconda                  dnf.rpm.log-20190318  private
audit                     dnf.rpm.log-20190324  qemu-ga
boot.log                  dnf.rpm.log-20190331  README
_...output omitted..._

```

