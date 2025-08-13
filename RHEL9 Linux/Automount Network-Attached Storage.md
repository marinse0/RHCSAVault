# Automount Network-Attached Storage

## Mount NFS Exports with the Automounter

The _automounter_ is a service (`autofs`) that automatically mounts file systems and NFS exports on demand, and automatically unmounts file systems and NFS exports when the mounted resources are no longer in current use.

The automounter function was created to solve the problem that unprivileged users do not have sufficient permissions to use the mount command. Without use of the mount command, normal users cannot access removable media such as CDs, DVDs, and removable disk drives. Furthermore, if a local or remote file system is not mounted at boot time by using the `/etc/fstab` configuration, then a normal user cannot mount and access those unmounted file systems.

The automounter configuration files are populated with file-system mount information, in a similar way to `/etc/fstab` entries. Although `/etc/fstab` file systems mount during system boot and remain mounted until system shutdown or other intervention, automounter file systems do not necessarily mount during system boot. Instead, automounter-controlled file systems mount on demand, when a user or application attempts to enter the file-system mount point to access files.

## Direct and Indirect Map Use Cases

The automounter supports both direct and indirect mount-point mapping, to handle the two types of demand mounting. A _direct_ mount is when a file system mounts to an unchanging, known mount point location. Almost all the file system mounts that you configured, before learning about the automounter, are examples of direct mounts. A _direct_ mount point exists as a permanent directory, the same as other normal directories.

An _indirect_ mount is when the mount point location is not known until the mount demand occurs. An example of an indirect mount is the configuration for remote-mounted home directories, where a user's home directory includes their username in the directory path. The user's remote file system is mounted to their home directory, only after the automounter learns which user specified to mount their home directory, and determines the mount point location to use. Although _indirect_ mount points appear to exist, the `autofs` service creates them when the mount demand occurs, and deletes them again when the demand ended and the file system is unmounted.

## Configure the Automounter Service

The process to configure an automount has many steps.

First, you must install the `autofs` and `nfs-utils` packages.
```bash
[user@host ~]$ sudo dnf install autofs nfs-utils
```

These packages contain all requirements to use the automounter for NFS exports.

## Create a Master Map

Next, add a master map file to `/etc/auto.master.d`. This file identifies the base directory for mount points, and identifies the mapping file to create the automounts.
```bash
[user@host ~]$ sudo vim /etc/auto.master.d/demo.autofs
```

The name of the master map file is mostly arbitrary (although typically meaningful), and it must have an extension of `.autofs` for the subsystem to recognize it. You can place multiple entries in a single master map file; alternatively, you can create multiple master map files, each with its own logically grouped entries.

Include the following content in the master map entry for indirectly mapped mounts:

/shares  /etc/auto.demo

This entry uses the `/shares` directory as the base for indirect automounts. The `/etc/auto.demo` file contains the mount details. Use an absolute file name. The `auto.demo` file must be created before starting the `autofs` service.

## Create an Indirect Map

Now, create the mapping files. Each mapping file identifies:
- the mount point,
- mount options, 
- source location to mount for a set of automounts.
```bash
[user@host ~]$ sudo vim /etc/auto.demo
```

The mapping file-naming convention is ``/etc/auto.name``, where _name_ reflects the content of the map.
```
work  -rw,sync  serverb:/shares/work
```

The format of an entry is _mount point_, _mount options_, and _source location_. This example shows an indirect mapping entry. Direct maps and indirect maps that use wildcards are covered later in this section.

Known as the _key_ in the man pages, the `autofs` service automatically creates and removes the mount point. In this case, the fully qualified mount point is `/shares/work` (see the master map file). The `autofs` service creates and removes the `/shares` and `/shares/work` directories as needed.

In this example, the local mount point mirrors the server's directory structure. However, this mirroring is not required; the local mount point can have an arbitrary name. The `autofs` service does not enforce a specific naming structure on the client.

Mount options start with a dash character (`-`) and are comma-separated with no white space. The file-system mount options for manual mounting are also available when automounting. In this example, the automounter mounts the export with read/write access (`rw` option), and the server is synchronized immediately during write operations (`sync` option).

Useful automounter-specific options include `-fstype=` and `-strict`. Use `fstype` to specify the file-system type, for example `nfs4` or `xfs`, and use `strict` to treat errors when mounting file systems as fatal.

The source location for NFS exports follows the `host:/pathname` pattern, in this example `serverb:/shares/work`. For this automount to succeed, the NFS server, `serverb`, must _export_ the directory with read/write access, and the user that requests access must have standard Linux file permissions on the directory. If `serverb` exports the directory with read/only access, then the client gets read/only access even if it requested read/write access.

## Wildcards in an Indirect Map

When an NFS server exports multiple subdirectories within a directory, then the automounter can be configured to access any of those subdirectories with a single mapping entry.

Continuing the previous example, if `serverb:/shares` exports two or more subdirectories, and they are accessible with the same mount options, then the content for the `/etc/auto.demo` file might appear as follows:
```
*  -rw,sync  serverb:/shares/&
```

The mount point (or key) is an asterisk character (*), and the subdirectory on the source location is an ampersand character (&). Everything else in the entry is the same.

When a user attempts to access `/shares/work`, the `*` key (which is `work` in this example) replaces the ampersand in the source location and `serverb:/exports/work` is mounted. As with the indirect example, the `autofs` service creates and removes the `work` directory automatically.

## Create a Direct Map

A direct map is used to map an NFS export to an absolute path mount point. Only one direct map file is necessary, and can contain any number of direct maps.

To use directly mapped mount points, the master map file might appear as follows:
```
/-  /etc/auto.direct
```

All direct map entries use `/-` as the base directory. In this case, the mapping file that contains the mount details is `/etc/auto.direct`.

The content for the `/etc/auto.direct` file might appear as follows:
```
/mnt/docs  -rw,sync  serverb:/shares/docs
```

The mount point (or key) is always an absolute path. The rest of the mapping file uses the same structure.

In this example, the `/mnt` directory exists, and the `autofs` service does not manage it. The `autofs` service creates and removes the full `/mnt/docs` directory automatically.

## Start the Automounter Service

Lastly, use the `systemctl` command to start and enable the `autofs` service.
```bash
[user@host ~]$ sudo systemctl enable --now autofs
Created symlink /etc/systemd/system/multi-user.target.wants/autofs.service → /usr/lib/systemd/system/autofs.service.
```

## The Alternative `systemd.automount` Method

The systemd daemon can automatically create unit files for entries in the `/etc/fstab` file that include the `x-systemd.automount` option. Use the `systemctl daemon-reload` command after modifying an entry's mount options, to generate a new unit file, and then use the ``systemctl start _`unit`_.automount`` command to enable that automount configuration.

The naming of the unit is based on its mount location. For example, if the mount point is `/remote/finance`, then the unit file is named `remote-finance.automount`. The systemd daemon mounts the file system when the `/remote/finance` directory is initially accessed.

This method can be simpler than installing and configuring the `autofs` service. However, a `systemd.automount` unit can support only absolute path mount points, similar to `autofs` direct maps.

### Guided Exercise: Automount Network-Attached Storage

In this exercise, you create direct-mapped and indirect-mapped automount-managed mount points that mount NFS file systems.

**Outcomes**

- Install required packages for the automounter.
- Configure direct and indirect automounter maps, with resources from a preconfigured NFSv4 server.
- Describe the difference between direct and indirect automounter maps.

As the `student` user on the `workstation` machine, use the `lab` command to prepare your system for this exercise.

This start script determines whether `servera` and `serverb` are reachable on the network. The script alerts you if those servers are not available. The start script configures `serverb` as an NFSv4 server, sets up permissions, and exports directories. The script also creates users and groups that are needed on both `servera` and `serverb`.

[student@workstation ~]$ **`lab start netstorage-autofs`**

**Instructions**

An internet service provider uses a central server, `serverb`, to host shared directories with important documents that must be available on demand. When users log in to `servera`, they need access to the automounted shared directories.

The following list provides the environment characteristics for completing this exercise:

- The `serverb` machine exports the `/shares/indirect` directory, which in turn contains the `west`, `central`, and `east` subdirectories.
- The `serverb` machine also exports the `/shares/direct/external` directory.
- The `operators` group consists of the `operator1` and `operator2` users. They have read and write access to the `/shares/indirect/west`, `/shares/indirect/central`, and `/﻿shares/indirect/east` exported directories.
- The `contractors` group consists of the `contractor1` and `contractor2` users. They have read and write access to the `/shares/direct/external` exported directory.
- The expected mount points for `servera` are `/external` and `/internal`.
- The `/shares/direct/external` exported directory is automounted on `servera` with a _direct_ map on `/external`.
- The `/shares/indirect/west` exported directory is automounted on `servera` with an _indirect_ map on `/internal/west`.
- The `/shares/indirect/central` exported directory is automounted on `servera` with an _indirect_ map on `/internal/central`.
- The `/shares/indirect/east` exported directory is automounted on `servera` with an _indirect_ map on `/internal/east`.
- All user passwords are set to `redhat`.
- The `nfs-utils` package is already installed.

1. Log in to `servera` and install the required packages.
    
    1. Log in to `servera` as the `student` user and switch to the `root` user.
```        
        [student@workstation ~]$ **`ssh student@servera`**
        _...output omitted..._
        [student@servera ~]$ **`sudo -i`**
        [sudo] password for student: `student`
        [root@servera ~]#
```        
    1. Install the `autofs` package.
```
        
        [root@servera ~]# **`dnf install autofs`**
        _...output omitted..._
        Is this ok [y/N]: **`y`**
        _...output omitted..._
        Complete!
```        
2. Configure an automounter direct map on `servera` with exports from `serverb`. Create the direct map with files that are named `/etc/auto.master.d/direct.autofs` for the master map and `/etc/auto.direct` for the mapping file. Use the `/external` directory as the main mount point on `servera`.
    
    1. Test the NFS server and export before you configure the automounter.

```    
        [root@servera ~]# **`mount -t nfs \`**
        **`serverb.lab.example.com:/shares/direct/external /mnt`**
        [root@servera ~]# **`ls -l /mnt`**
        total 4
        -rw-r--r--. 1 root contractors 22 Apr  7 23:15 README.txt
        [root@servera ~]# **`umount /mnt`**
```        
    2. Create a master map file named `/etc/auto.master.d/direct.autofs`, insert the following content, and save the changes.
```        
        /-	/etc/auto.direct
```       
    3. Create a direct map file named `/etc/auto.direct`, insert the following content, and save the changes.
```        
        /external	-rw,sync,fstype=nfs4	serverb.lab.example.com:/shares/direct/external
```        
3. Configure an automounter indirect map on `servera` with exports from `serverb`. Create the indirect map with files that are named `/etc/auto.master.d/indirect.autofs` for the master map and `/etc/auto.indirect` for the mapping file. Use the `/internal` directory as the main mount point on `servera`.
    
    1. Test the NFS server and export before you configure the automounter.

```        
        [root@servera ~]# **`mount -t nfs \`**
        **`serverb.lab.example.com:/shares/indirect /mnt`**
        [root@servera ~]# **`ls -l /mnt`**
        total 0
        drwxrws---. 2 root operators 24 Apr  7 23:34 central
        drwxrws---. 2 root operators 24 Apr  7 23:34 east
        drwxrws---. 2 root operators 24 Apr  7 23:34 west
        [root@servera ~]# **`umount /mnt`**
```        
    2. Create a master map file named `/etc/auto.master.d/indirect.autofs`, insert the following content, and save the changes.

```
        /internal	/etc/auto.indirect
```        
    3. Create an indirect map file named `/etc/auto.indirect`, insert the following content, and save the changes.
```        
        *	-rw,sync,fstype=nfs4	serverb.lab.example.com:/shares/indirect/&
```        
4. Start the `autofs` service on `servera`, and enable it to start automatically at boot time.
    
    1. Start and enable the `autofs` service on `servera`.
        
        [root@servera ~]# **`systemctl enable --now autofs`**
        Created symlink /etc/systemd/system/multi-user.target.wants/autofs.service → /usr/lib/systemd/system/autofs.service.
        
5. Test the direct automounter map as the `contractor1` user. When done, exit from the `contractor1` user session on `servera`.
    
    1. Switch to the `contractor1` user.
        
        [root@servera ~]# **`su - contractor1`**
        [contractor1@servera ~]$
        
    2. List the `/external` mount point.
        
        [contractor1@servera ~]$ **`ls -l /external`**
        total 4
        -rw-r--r--. 1 root contractors 22 Apr  7 23:34 README.txt
        
    3. Review the content and test the access to the `/external` mount point.
        
        [contractor1@servera ~]$ **`cat /external/README.txt`**
        ###External Folder###
        [contractor1@servera ~]$ **`echo testing-direct > /external/testing.txt`**
        [contractor1@servera ~]$ **`cat /external/testing.txt`**
        testing-direct
        
    4. Exit from the `contractor1` user session.
        
        [contractor1@servera ~]$ **`exit`**
        logout
        [root@servera ~]#
        
6. Test the indirect automounter map as the `operator1` user. When done, log out from `servera`.
    
    1. Switch to the `operator1` user.
        
        [root@servera ~]# **`su - operator1`**
        [operator1@servera ~]$
        
    2. List the `/internal` mount point.
        
        [operator1@servera ~]$ **`ls -l /internal`**
        total 0
        
        ### Note
        
        With an automounter indirect map, you must access each exported subdirectory for them to mount. With an automounter direct map, after you access the mapped mount point, you can immediately view and access the subdirectories and content in the exported directory.
        
    3. Test the `/internal/west` automounter exported directory access.
        
        [operator1@servera ~]$ **`ls -l /internal/west/`**
        total 4
        -rw-r--r--. 1 root operators 18 Apr  7 23:34 README.txt
        [operator1@servera ~]$ **`cat /internal/west/README.txt`**
        ###West Folder###
        [operator1@servera ~]$ **`echo testing-1 > /internal/west/testing-1.txt`**
        [operator1@servera ~]$ **`cat /internal/west/testing-1.txt`**
        testing-1
        [operator1@servera ~]$ **`ls -l /internal`**
        total 0
        drwxrws---. 2 root operators 24 Apr  7 23:34 west
        
    4. Test the `/internal/central` automounter exported directory access.
        
        [operator1@servera ~]$ **`ls -l /internal/central`**
        total 4
        -rw-r--r--. 1 root operators 21 Apr  7 23:34 README.txt
        [operator1@servera ~]$ **`cat /internal/central/README.txt`**
        ###Central Folder###
        [operator1@servera ~]$ **`echo testing-2 > /internal/central/testing-2.txt`**
        [operator1@servera ~]$ **`cat /internal/central/testing-2.txt`**
        testing-2
        [operator1@servera ~]$ **`ls -l /internal`**
        total 0
        drwxrws---. 2 root operators 24 Apr  7 23:34 central
        drwxrws---. 2 root operators 24 Apr  7 23:34 west
        
    5. Test the `/internal/east` automounter exported directory access.
        
        [operator1@servera ~]$ **`ls -l /internal/east`**
        total 4
        -rw-r--r--. 1 root operators 18 Apr  7 23:34 README.txt
        [operator1@servera ~]$ **`cat /internal/east/README.txt`**
        ###East Folder###
        [operator1@servera ~]$ **`echo testing-3 > /internal/east/testing-3.txt`**
        [operator1@servera ~]$ **`cat /internal/east/testing-3.txt`**
        testing-3
        [operator1@servera ~]$ **`ls -l /internal`**
        total 0
        drwxrws---. 2 root operators 24 Apr  7 23:34 central
        drwxrws---. 2 root operators 24 Apr  7 23:34 east
        drwxrws---. 2 root operators 24 Apr  7 23:34 west
        
    6. Test the `/external` automounter exported directory access.
        
        [operator1@servera ~]$ **`ls -l /external`**
        ls: cannot open directory '/external': Permission denied
        
    7. Return to the `workstation` machine as the `student` user.
        
        [operator1@servera ~]$ **`exit`**
        logout
        [root@servera ~]# **`exit`**
        logout
        [student@servera ~]$ **`exit`**
        logout
        Connection to servera closed.
        

**Finish