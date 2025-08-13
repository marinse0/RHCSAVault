# Installation
## Troubleshoot the Installation

During a Red Hat Enterprise Linux 9 installation, Anaconda provides two virtual consoles. The first virtual console has five windows that the `tmux` software terminal multiplexer supplies. You can access that console by using **Ctrl**+**Alt**+**F1**. The second virtual console, which is displayed by default, shows the Anaconda graphical interface. You can access it by using **Ctrl**+**Alt**+**F6**.

The `tmux` terminal provides a shell prompt in the second window in the first virtual console. You can use the terminal to enter commands to inspect and troubleshoot the system while the installation continues. The other windows provide diagnostic messages, logs, and additional information.

The following table lists the keystroke combinations to access the virtual consoles and the `tmux` terminal windows. In the `tmux` terminal, the keyboard shortcuts are performed in two actions: press and release **Ctrl**+**B**, and then press the number key of the window to access. In the `tmux` terminal, you can also use **Alt**+**Tab** to rotate the current focus between the windows.

|Key sequence|contents|
|:--|:--|
|**Ctrl**+**Alt**+**F1**|Access the `tmux` terminal multiplexer.|
|**Ctrl**+**B** **1**|In the `tmux` terminal, access the main information page for the installation process.|
|**Ctrl**+**B** **2**|In the `tmux` terminal, provide a root shell. Anaconda stores the installation log files in the `/tmp` directory.|
|**Ctrl**+**B** **3**|In the `tmux` terminal, display the contents of the `/tmp/anaconda.log` file.|
|**Ctrl**+**B** **4**|In the `tmux` terminal, display the contents of the `/tmp/storage.log` file.|
|**Ctrl**+**B** **5**|In the `tmux` terminal, display the contents of the `/tmp/program.log` file.|
|**Ctrl**+**Alt**+**F6**|Access the Anaconda graphical interface.|

### Note

For compatibility with earlier Red Hat Enterprise Linux versions, the virtual consoles from **Ctrl**+**Alt**+**F2** through **Ctrl**+**Alt**+**F5** also present root shells during installation.

## Automate Installation with Kickstart

### Objectives

Explain Kickstart concepts and architecture, create a Kickstart file with the Kickstart Generator website, modify an existing Kickstart file with a text editor and check its syntax with `ksvalidator`, publish a Kickstart file to the installer, and install Kickstart on the network.

### Introduction to Kickstart

The _Kickstart_ feature of Red Hat Enterprise Linux automates system installations. You can use Kickstart text files to configure disk partitioning, network interfaces, package selection, and customize the installation. The Anaconda installer uses Kickstart files for a complete installation without user interaction. The Kickstart feature is similar and uses an unattended installation answer file for Microsoft Windows.

Kickstart files begin with a list of commands that define how to install the target machine. The installer ignores comment lines, which start with the number sign (`#`) character. Additional sections begin with a directive, which start with the percentage sign (`%`) character, and end on a line with the `&end` directive.

The `%packages` section specifies which software to include on installation. Specify individual packages by name, without versions. The at sign (`@`) character denotes package groups (either by group or ID), and the `@^` characters denote environment groups (groups of package groups). Lastly, use the `@module:stream/profile` syntax to denote module streams.

Groups have mandatory, default, and optional components. Normally, Kickstart installs mandatory and default components. To exclude a package or a package group from the installation, precede it with a hyphen (`-`) character. Excluded packages or package groups might still install if they are mandatory dependencies of other requested packages.

A Kickstart configuration file typically includes one or more `%pre` and `%post` sections, which contain scripts that further configure the system. The `%pre` scripts execute before any disk partitioning is done. Typically, you use `%pre` scripts to initialize a storage or network device that the remainder of the installation requires. The `%post` scripts execute after the initial installation is complete. Scripts within the `%pre` and `%post` sections can use any available interpreter on the system, including Bash or Python. Avoid the use of a `%pre` section, because any errors that occur within it might be difficult to diagnose.

Lastly, you can specify as many sections as you need, in any order. For example, you can have two `%post` sections, and they are interpreted in order of appearance.

### Note

The RHEL Image Builder is an alternative installation method to Kickstart files. Rather than a text file that provides installation instructions, Image Builder creates an image with all the required system changes. The RHEL Image Builder can create images for public clouds such as Amazon Web Services and Microsoft Azure, or for private clouds such as OpenStack or VMware. See this section's references for more information about the RHEL Image Builder.

#### Installation Commands

The following Kickstart commands configure the installation source and method:

- `url`: Specifies the URL that points to the installation media. 
```
url --url="http://classroom.example.com/content/rhel9.0/x86_64/dvd/"
```
- `repo`: Specifies where to find additional packages for installation. This option must point to a valid `DNF` repository.
```
repo --name="appstream" --baseurl=http://classroom.example.com/content/rhel9.0/x86_64/dvd/AppStream/
```
- `text`: Forces a text mode installation.
- `vnc`: Enables the VNC viewer so you can access the graphical installation remotely over VNC.
```
vnc --password=redhat
```

#### Device and Partition Commands

The following Kickstart commands configure devices and partitioning schemes:

- `clearpart`: Removes partitions from the system before creating partitions.
```
clearpart --all --drives=vda,vdb
```
- `part`: Specifies the size, format, and name of a partition. Required unless the `autopart` or `mount` commands are present.
```
part /home --fstype=ext4 --label=homes --size=4096 --maxsize=8192 --grow
```
- `autopart`: Automatically creates a root partition, a swap partition, and an appropriate boot partition for the architecture. On large enough drives (50 GB+), this command also creates a `/home` partition.
- `ignoredisk`: Prevents Anaconda from modifying disks, and is useful alongside the `autopart` command.
```
ignoredisk --drives=sdc
```
- `bootloader`: Defines where to install the bootloader. Required.
```
bootloader --location=mbr --boot-drive=sda
```
- `volgroup`, `logvol`: Creates LVM volume groups and logical volumes.
```
part pv.01 --size=8192
volgroup myvg pv.01
logvol / --vgname=myvg --fstype=xfs --size=2048 --name=rootvol --grow
logvol /var --vgname=myvg --fstype=xfs --size=4096 --name=varvol
```
- `zerombr`: Initialize disks whose formatting is unrecognized.
#### Network Commands

The following Kickstart commands configure networking-related features:

- `network`: Configures network information for the target system. Activates network devices in the installer environment.
```
network --device=eth0 --bootproto=dhcp
```
- `firewall`: Defines the firewall configuration for the target system.
```
firewall --enabled --service=ssh,http
```

#### Location and Security Commands

The following Kickstart commands configure security, language, and region settings:

- `lang`: Sets the language to use on the installed system. Required.
```
lang en_US
```
- `keyboard`: Sets the system keyboard type. Required.
```
keyboard --vckeymap=us
```
- `timezone`: Defines the time zone and whether the hardware clock uses UTC. Required.
```
timezone --utc Europe/Amsterdam
```
- `timesource`: Enables or disables NTP. If you enable NTP, then you must specify NTP servers or pools.
```
timesource --ntp-server classroom.example.com
```
- `authselect`: Sets up authentication options. Options that the `authselect` command recognizes are valid for this command. See authselect(8).
- `rootpw`: Defines the initial `root` user password. Required.
```
rootpw --plaintext redhat
or
rootpw --iscrypted $6$KUnFfrTzO8jv.PiH$YlBbOtXBkWzoMuRfb0.SpbQ....XDR1UuchoMG1
```
- `selinux`: Sets the SELinux mode for the installed system.
```
selinux --enforcing
```
- `services`: Modifies the default set of services to run under the default `systemd` target.
```
services --disabled=network,iptables,ip6tables --enabled=NetworkManager,firewalld
```
- `group`, `user`: Creates a local group or user on the system.
```
group --name=admins --gid=10001
user --name=jdoe --gecos="John Doe" --groups=admins --password=changeme --&#xFEFF;plaintext
```

#### Miscellaneous Commands

The following Kickstart commands configure logging the host power state on completion:

- `logging`: This command defines how Anaconda handles logging during the installation.
```
logging --host=loghost.example.com
```
- `firstboot`: If enabled, then the Set up Agent starts the first time that the system boots. This command requires the `initial-setup` package.
```
firstboot --disabled
```
- `reboot`, `poweroff`, `halt`: Specify the final action when the installation completes. The default setting is the `halt` option.

### Note

Most Kickstart commands have multiple available options. Review the _Kickstart Commands and Options_ guide in this section's references for more information.

#### Example Kickstart File

In the following example, the first part of the Kickstart file consists of the installation commands, such as disk partitioning and installation source.
```
#version=RHEL9

# Define system bootloader options
bootloader --append="console=ttyS0 console=ttyS0,115200n8 no_timer_check net.ifnames=0  crashkernel=auto" --location=mbr --timeout=1 --boot-drive=vda

# Clear and partition disks
clearpart --all --initlabel
ignoredisk --only-use=vda
zerombr
part / --fstype="xfs" --ondisk=vda --size=10000

# Define installation options
text
repo --name="appstream" --baseurl="http://classroom.example.com/content/rhel9.0/x86_64/dvd/AppStream/"
url --url="http://classroom.example.com/content/rhel9.0/x86_64/dvd/"

# Configure keyboard and language settings
keyboard --vckeymap=us
lang en_US

# Set a root password, authselect profile, and selinux policy
rootpw --plaintext redhat
authselect select sssd
selinux --enforcing
firstboot --disable

# Enable and disable system services
services --disabled="kdump,rhsmcertd" --enabled="sshd,rngd,chronyd"

# Configure the system timezone and NTP server
timezone America/New_York --utc
timesource --ntp-server classroom.example.com
```

The second part of a Kickstart file contains the `%packages` section, with details of which packages and package groups to install, and which packages not to install.
```
%packages

@core
chrony
cloud-init
dracut-config-generic
dracut-norescue
firewalld
grub2
kernel
rsync
tar
-plymouth

%end
```

The last part of the Kickstart file contains a `%post` installation script.
```
%post

echo "This system was deployed using Kickstart on $(date)" > /etc/motd

%end
```

You can also specify a Python script with the `--interpreter` option.
```
%post --interpreter="/usr/libexec/platform-python"

print("This line of text is printed with python")

%end
```

### Note

In a Kickstart file, missing required values cause the installer to interactively prompt for an answer or to abort the installation entirely.

### Kickstart Installation Steps

Use the following steps to automate the installation of Red Hat Enterprise Linux with the Kickstart feature:
1. Create a Kickstart file.
2. Publish the Kickstart file so that the Anaconda installer can access it.
3. Boot the Anaconda installer and point it to the Kickstart file.

#### Create a Kickstart File

You can use either of these methods to create a Kickstart file:
- Use the Kickstart Generator website.
- Use a text editor.

The Kickstart Generator site at ``[https://access.redhat.com/labs/kickstartconfig`](https://access.redhat.com/labs/kickstartconfig%60)`` presents dialog boxes for user inputs, and creates a Kickstart configuration file with the user's choices. Each dialog box corresponds to the configurable items in the Anaconda installer.

|   |
|---|
|![](https://rol.redhat.com/rol/static/static_file_cache/rh134-9.0/kickstart/kickstart-generator.png)|

Figure 12.2: The Red Hat Customer Portal Kickstart Generator

Creating a Kickstart file from scratch is complex, so first try to edit an existing file. Every installation creates a `/root/anaconda-ks.cfg` file that contains the Kickstart directives that are used in the installation. This file is a good starting point to create a Kickstart file.

The `ksvalidator` utility checks for syntax errors in a Kickstart file. It ensures that keywords and options are correctly used, but it does not validate URL paths, individual packages, groups, nor any part of `%post` or `%pre` scripts. For example, if the `firewall --disabled` directive is misspelled, then the `ksvalidator` command might produce one of the following errors:
```
[user@host ~]$ **`ksvalidator /tmp/anaconda-ks.cfg`**
The following problem occurred on line 10 of the kickstart file:

Unknown command: frewall

[user@host ~]$ **`ksvalidator /tmp/anaconda-ks.cfg`**
The following problem occurred on line 10 of the kickstart file:

no such option: --dsabled
```

The `ksverdiff` utility displays syntax differences between different operating system versions. For example, the following command displays the Kickstart syntax changes between RHEL 8 and RHEL 9:
```
[user@host ~]$ **`ksverdiff -f RHEL8 -t RHEL9`**
The following commands were removed in RHEL9:
device deviceprobe dmraid install multipath

The following commands were deprecated in RHEL9:
autostep btrfs method

The following commands were added in RHEL9:
timesource
_...output omitted..._
```

The `pykickstart` package provides the `ksvalidator` and `ksverdiff` utilities.

#### Publish the Kickstart File to Anaconda
Provide the Kickstart file to the installer by placing it in one of these locations:
- A network server that is available at installation time by using FTP, HTTP, or NFS.
- An available USB disk or CD-ROM.
- A local hard disk on the system.

The installer must access the Kickstart file to begin an automated installation. Usually, the file is made available via an FTP, web, or NFS server. Network servers help with Kickstart file maintenance, because changes can be made once, and then immediately be used for future installations.

By providing Kickstart files on USB or CD-ROM, is also convenient. The Kickstart file can be embedded in the boot media that starts the installation. However, when the Kickstart file is changed, you must generate new installation media.

Providing the Kickstart file on a local disk enables a quick rebuild of a system.

#### Boot Anaconda and Point to the Kickstart File

After a Kickstart method is chosen, the installer is told where to locate the Kickstart file by passing the ``inst.ks=_`LOCATION`_`` parameter to the installation kernel.

Consider the following examples:
```
- inst.ks=http://server/dir/file
- inst.ks=ftp://server/dir/file
- inst.ks=nfs:server:/dir/file
- inst.ks=hd:device:/dir/file
- inst.ks=cdrom:device
```

|   |
|---|
|![](https://rol.redhat.com/rol/static/static_file_cache/rh134-9.0/kickstart/boot-kickstart.png)|

Figure 12.3: Specifying the Kickstart file location during installation

For virtual machine installations by using the Virtual Machine Manager or `virt-manager`, the Kickstart URL can be specified in a box under URL Options. When installing physical machines, boot by using installation media, and press the **Tab** key to interrupt the boot process. Add an ``inst.ks=_`LOCATION`_`` parameter to the installation kernel.