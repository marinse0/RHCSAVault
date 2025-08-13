# Containers, podman
## Manage Containers as System Services

### Manage Small Container Environments with systemd Units

As a regular user, you can create a `systemd` unit to configure your rootless containers. You can use this configuration to manage your container as a regular system service with the `systemctl` command.

To discuss the topics in this lecture, imagine the following scenario.

As a system administrator, you are tasked to configure the `webserver1` container that is based on the `http24` container image to start at system boot. You must also mount the `/app-artifacts` directory for the web server content and map the 8080 port from the local machine to the container. Configure the container to start and stop with `systemctl` commands.

### Requirements for systemd User Services

As a regular user, you can enable a service with the `systemctl` command. The service starts when you open a session (graphical interface, text console, or SSH), and it stops when you close the last session. This behavior differs from a system service, which starts when the system boots and stops when the system shuts down.

By default, when you create a user account with the `useradd` command, the system uses the next available ID from the regular user ID range. The system also reserves a range of IDs for the user's containers in the `/etc/subuid` file. If you create a user account with the `useradd` command `--system` option, then the system does not reserve a range for the user containers. As a consequence, you cannot start rootless containers with system accounts.

You decide to create a dedicated user account to manage containers. You use the `useradd` command to create the `appdev-adm` user, and use `redhat` as the password.

You then use the `su` command to switch to the `appdev-adm` user, and you start to use the `podman` command.
```bash
[user@host ~]$ su appdev-adm
Password: redhat
[appdev-adm@host ~]$ podman info
ERRO[0000] XDG_RUNTIME_DIR directory "/run/user/1000" is not owned by the current user
[appdev-adm@host ~]$
```

Podman is a stateless utility and requires a full login session. Podman must be used within an SSH session, and cannot be used in a `sudo` or an `su` shell. So you exit the `su` shell and **log in to the machine via SSH**.

You then configure the container registry and authenticate with your credentials. You run the `http` container with the following command.
```bash
[appdev-adm@host ~]$ podman run -d --name webserver1 -p 8080:8080 -v \
~/app-artifacts:/var/www/html:Z registry.access.redhat.com/ubi8/httpd-24
cde4a3d8c9563fd50cc39de8a4873dcf15a7e881ba4548d5646760eae7a35d81
[appdev-adm@host ~]$ podman ps
CONTAINER ID  IMAGE                                            COMMAND               CREATED        STATUS            PORTS                   NAMES
cde4a3d8c956  registry.access.redhat.com/ubi8/httpd-24:latest  /usr/bin/run-http...  4 seconds ago  Up 5 seconds ago  0.0.0.0:8080->8080/tcp  webserver1
```

**Note**  
Remember to provide the right access to the directory that you mount from the host file system to the container. For any error when running a container, you can use the `podman container logs` command for troubleshooting.

### Create systemd User Files for Containers

You can manually define `systemd` services in the `~/.config/systemd/user/` directory. The file syntax for user services is the same as for the system services files. For more details, review the `systemd.unit`(5) and `systemd.service`(5) man pages.

Use the `podman generate systemd` command to generate `systemd` service files for an existing container. The `podman generate systemd` command uses a container as a model to create the configuration file.

The `podman generate systemd` command `--new` option instructs the `podman` utility to configure the `systemd` service to create the container when the service starts, and to delete the container when the service stops.

### Important

Without the `--new` option, the `podman` utility configures the service unit file to start and stop the existing container without deleting it.

You use the `podman generate systemd` command with the `--name` option to display the `systemd` service file that is modeled for the `webserver1` container.
```bash
[appdev-adm@host ~]$ podman generate systemd --name webserver1
_...output omitted..._
ExecStart=/usr/bin/podman start webserver1 # [1]
ExecStop=/usr/bin/podman stop -t 10 webserver1 # [2]
ExecStopPost=/usr/bin/podman stop -t 10 webserver1
_...output omitted..._
```

|       |                                                                                                                                                                        |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| - [1] | On start, the `systemd` daemon executes the `podman start` command to start the existing container.                                                                    |
| - [2] | On stop, the `systemd` daemon executes the `podman stop` command to stop the container. Notice that the `systemd` daemon does not delete the container on this action. |

You then use the previous command with the addition of the `--new` option to compare the `systemd` configuration.
```bash
[appdev-adm@host ~]$ podman generate systemd --name webserver1 --new
_...output omitted..._
ExecStartPre=/bin/rm -f %t/%n.ctr-id
ExecStart=/usr/bin/podman run --cidfile=%t/%n.ctr-id --cgroups=no-conmon --rm --sdnotify=conmon --replace -d --name webserver1 -p 8080:8080 -v /home/appdev-adm/app-artifacts:/var/www/html:Z registry.access.redhat.com/ubi8/httpd-24 # [1]
ExecStop=/usr/bin/podman stop --ignore --cidfile=%t/%n.ctr-id # [2]
ExecStopPost=/usr/bin/podman rm -f --ignore --cidfile=%t/%n.ctr-id # [3]
_...output omitted..._
```

|       |                                                                                                                                                                                                                 |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| - [1] | On starting, the `systemd` daemon executes the `podman run` command to create and then start a new container. This action uses the `podman run` command `--rm` option, which removes the container on stopping. |
| - [2] | On stopping, `systemd` executes the `podman stop` command to stop the container.                                                                                                                                |
| - [3] | After `systemd` stops the container, `systemd` removes it by using the `podman rm -f` command.                                                                                                                  |

You verify the output of the `podman generate systemd` command, and run the previous command with the `--files` option to create the `systemd` user file in the current directory. Because the `webserver1` container uses persistent storage, you choose to use the `podman generate systemd` command with the `--new` option. You then create the `~/.config/systemd/user/` directory and move the file to this location.
```bash
[appdev-adm@host ~]$ podman generate systemd --name webserver1 --new --files
/home/appdev-adm/container-webserver1.service
[appdev-adm@host ~]$ mkdir -p ~/.config/systemd/user/
[appdev-adm@host ~]$ mv container-webserver1.service ~/.config/systemd/user/
```

### Manage systemd User Files for Containers

Now that you created the `systemd` user file, you can use the `systemctl` command `--user` option to manage the `webserver1` container.

First, you reload the `systemd` daemon to make the `systemctl` command aware of the new user file. You use the `systemctl --user start` command to start the `webserver1` container. Use the name of the generated `systemd` user file for the container.
```bash
[appdev-adm@host ~]$ systemctl --user daemon-reload
[appdev-adm@host ~]$ systemctl --user start container-webserver1.service
[appdev-adm@host ~]$ systemctl --user status container-webserver1.service
● container-webserver1.service - Podman container-webserver1.service
     Loaded: loaded (/home/appdev-adm/.config/systemd/user/container-webserver1.service; disabled; vendor preset: disabled)
     Active: active (running) since Thu 2022-04-28 21:22:26 EDT; 18s ago
       Docs: man:podman-generate-systemd(1)
    Process: 31560 ExecStartPre=/bin/rm -f /run/user/1003/container-webserver1.service.ctr-id (code=exited, status=0/SUCCESS)
   Main PID: 31600 (conmon)
_...output omitted..._
[appdev-adm@host ~]$ podman ps
CONTAINER ID  IMAGE                                            COMMAND               CREATED         STATUS             PORTS                   NAMES
18eb00f42324  registry.access.redhat.com/ubi8/httpd-24:latest  /usr/bin/run-http...  28 seconds ago  Up 29 seconds ago  0.0.0.0:8080->8080/tcp  webserver1
Created symlink /home/appdev-adm/.config/systemd/user/default.target.wants/container-webserver1.service → /home/appdev-adm/.config/systemd/user/container-webserver1.service.
```

### Important

When you configure a container with the `systemd` daemon, the daemon monitors the container status and restarts the container if it fails. Do not use the `podman` command to start or stop these containers. Doing so might interfere with the `systemd` daemon monitoring.

The following table summarizes the directories and commands that are used between `systemd` system and user services.

**Table 13.3. Comparing System and User Services**

|                                            |                 |                                                                |
| ------------------------------------------ | --------------- | -------------------------------------------------------------- |
| Storing custom unit files                  | System services | `/etc/systemd/system/unit.service`                             |
|                                            | User services   | `~/.config/systemd/user/unit.service`                          |
| Reloading unit files                       | System services | # `systemctl daemon-reload`                                    |
|                                            | User services   | `$ systemctl --user daemon-reload`                             |
| Starting and stopping a service            | System services | `# systemctl start\stop UNIT`                                  |
|                                            | User services   | `$ systemctl --user start\stop UNIT`                           |
| Starting a service when the machine starts | System services | `# systemctl enable UNIT                                       |
|                                            | User services   | $ `loginctl enable-linger`<br>`$ systemctl --user enable UNIT` |
|                                            |                 |                                                                |

### Configure Containers to Start at System Boot

The `systemd` service stops the container after a certain time if the user logs out from the system. This behavior occurs because the `systemd` service unit was created with the `--user` option, which starts a service at user login and stops it at user logout.

You can change this default behavior, and force your enabled services to start with the server and stop during the shutdown, by running the `loginctl enable-linger` command.

You use the `loginctl` command to configure the `systemd` user service to persist after the last user session of the configured service closes. You then verify the successful configuration with the `loginctl show-user` command.
```bash
[user@host ~]$ loginctl show-user appdev-adm
_...output omitted..._
`Linger=no`
[user@host ~]$ loginctl enable-linger
[user@host ~]$ loginctl show-user appdev-adm
_...output omitted..._
`Linger=yes`
```

To revert the operation, use the `loginctl disable-linger` command.

### Manage Containers as Root with systemd

You can also configure containers to run as root and manage them with `systemd` service files. One advantage of this approach is that you can configure the service files to work the same as common `systemd` unit files, rather than as a particular user.

The procedure to set the service file as root is similar to the previously outlined procedure for rootless containers, with the following exceptions:
- Do not create a dedicated user for container management.  
- The service file must be in the `/etc/systemd/system` directory instead of in the `~/.config/systemd/user` directory.
- You manage the containers with the `systemctl` command without the `--user` option.
- Do not run the `loginctl enable-linger` command as the `root` user.

For a demonstration, see the YouTube video from the Red Hat Videos channel that is listed in the References at the end of this section.
#### References

`loginctl`(1), `systemd.unit`(5), `systemd.service`(5), `subuid`(5), and `podman-generate-systemd`(1) man pages

[Managing Containers in Podman with Systemd Unit Files](https://www.youtube.com/watch?v=AGkM2jGT61Y)
## Manage Container Storage and Network Resources

### Environment Variables for Containers

Some container images enable passing environment variables to customize the container at creation time. You can use environment variables to set parameters to the container to tailor for your environment without the need to create your own custom image. Usually, you would not modify the container image, because it would add layers to the image, which might be harder to maintain.

You use the `podman run -d registry.lab.example.com/rhel8/mariadb-105` command to run a containerized database, but you notice that the container fails to start.
```bash
[user@host ~]$ podman run -d registry.lab.example.com/rhel8/mariadb-105 \
--name db01
20751a03897f14764fb0e7c58c74564258595026124179de4456d26c49c435ad
[user@host ~]$ podman ps -a
CONTAINER ID  IMAGE                                              COMMAND         CREATED         STATUS                     PORTS       NAMES
20751a03897f  registry.lab.example.com/rhel8/mariadb-105:latest  run-mysqld      29 seconds ago  Exited (1) 29 seconds ago              db01
```
You use the `podman container logs` command to investigate the reason of the container status.
```bash
[user@host ~]$ podman container logs db01
_...output omitted..._
You must either specify the following environment variables:
  MYSQL_USER (regex: '^[a-zA-Z0-9_]+$')
  MYSQL_PASSWORD (regex: '^[a-zA-Z0-9_~!@#$%^&*()-=<>,.?;:|]+$')
  MYSQL_DATABASE (regex: '^[a-zA-Z0-9_]+$')
Or the following environment variable:
  MYSQL_ROOT_PASSWORD (regex: '^[a-zA-Z0-9_~!@#$%^&*()-=<>,.?;:|]+$')
Or both.
_...output omitted..._
```
From the preceding output, you determine that the container did not continue to run, because the required environment variables were not passed to the container. So you inspect the `mariadb-105` container image to find more information about the environment variables to customize the container.
```bash
[user@host ~]$ skopeo inspect docker://registry.lab.example.com/rhel8/mariadb-105
_...output omitted..._
        "name": "rhel8/mariadb-105",
        "release": "40.1647451927",
        "summary": "MariaDB 10.5 SQL database server",
        **`"url": "https://access.redhat.com/containers/#/registry.access.redhat.com/rhel8/mariadb-105/`**
images/1-40.1647451927",
        **`"usage": "podman run -d -e MYSQL_USER=user -e MYSQL_PASSWORD=pass -e MYSQL_DATABASE=db -p 3306:3306 rhel8/mariadb-105",`**
        "vcs-ref": "c04193b96a119e176ada62d779bd44a0e0edf7a6",
        "vcs-type": "git",
        "vendor": "Red Hat, Inc.",
_...output omitted..._
```
The `usage` label from the output provides an example of how to run the image. The `url` label points to a web page in the Red Hat Container Catalog that documents environment variables and other information about how to use the container image.

The documentation for this image shows that the container uses the 3306 port for the database service. The documentation also shows that the following environment variables are available to configure the database service:

**Table 13.2. Environment Variables for the `mariadb` Image**

| Variable              | Description                              |
| :-------------------- | :--------------------------------------- |
| `MYSQL_USER`          | Username for the MySQL account to create |
| `MYSQL_PASSWORD`      | Password for the user account            |
| `MYSQL_DATABASE`      | Database name                            |
| `MYSQL_ROOT_PASSWORD` | Password for the root user (optional)    |

After examining the available environment variables for the image, you use the `podman run` command `-e` option to pass environment variables to the container, and use the `podman ps` command to verify that it is running.
```bash
[user@host ~]$ podman run -d --name db01 \
-e MYSQL_USER=student \
-e MYSQL_PASSWORD=student \
-e MYSQL_DATABASE=dev_data \
-e MYSQL_ROOT_PASSWORD=redhat \
registry.lab.example.com/rhel8/mariadb-105
[user@host ~]$ podman ps
CONTAINER ID  IMAGE                                              COMMAND     CREATED        STATUS            PORTS       NAMES
4b8f01be7fd6  registry.lab.example.com/rhel8/mariadb-105:latest  run-mysqld  6 seconds ago  Up 6 seconds ago              db01
```

### Container Persistent Storage

By default, when you run a container, all of the content uses the container-based image. Given the ephemeral nature of container images, all of the new data that the user or the application writes is lost after removing the container.

To persist data, you can use host file-system content in the container with the `--volume (-﻿v)` option. You must consider file-system level permissions when you use this volume type in a container.

In the MariaDB container image, the `mysql` user must own the `/var/lib/mysql` directory, the same as if MariaDB was running on the host machine. The directory to mount into the container must have `mysql` as the user and group owner (or the UID and GID of the `mysql` user, if MariaDB is not installed on the host machine). If you run a container as the `root` user, then the UIDs and GIDs on your host machine match the UIDs and GIDs inside the container.

The UID and GID matching configuration does not occur the same way in a rootless container. In a rootless container, the user has root access from within the container, because Podman launches a container inside the user namespace.

You can use the `podman unshare` command to run a command inside the user namespace. To obtain the UID mapping for your user namespace, use the `podman unshare cat` command.
```bash
[user@host ~]$ podman unshare cat /proc/self/uid_map
         0       1000          1
         1     100000      65536
[user@host ~]$ podman unshare cat /proc/self/gid_map
         0       1000          1
         1     100000      65536

```
The preceding output shows that in the container, the root user (UID and GID of 0) maps to your user (UID and GID of `1000`) on the host machine. In the container, the UID and GID of 1 maps to the UID and GID of 100000 on the host machine. Every UID and GID after 1 increments by 1. For example, the UID and GID of 30 inside a container maps to the UID and GID of 100029 on the host machine.

You use the `podman exec` command to view the `mysql` user UID and GID inside the container that is running with ephemeral storage.

```bash
[user@host ~]$ podman exec -it db01 grep mysql /etc/passwd
mysql:x:27:27:MySQL Server:/var/lib/mysql:/sbin/nologin
```
You decide to mount the `/home/user/db_data` directory into the `db01` container to provide persistent storage on the `/var/lib/mysql` directory of the container. You then create the `/home/user/db_data` directory, and use the `podman unshare` command to set the user namespace UID and GID of `27` as the owner of the directory.
```bash
[user@host ~]$ mkdir /home/user/db_data
[user@host ~]$ podman unshare chown 27:27 /home/user/db_data
```

The UID and GID of `27` in the container maps to the UID and GID of `100026` on the host machine. You can verify the mapping by viewing the ownership of the `/home/user/db_data` directory with the `ls` command.
```bash
[user@host ~]$ ls -l /home/user/
total 0
drwxrwxr-x. 3  100026  100026 18 May  5 14:37 db_data
_...output omitted..._
```

Now that the correct file-system level permissions are set, you use the `podman run` command `-v` option to mount the directory.
```bash
[user@host ~]$ podman run -d --name db01 \
-e MYSQL_USER=student \
-e MYSQL_PASSWORD=student \
-e MYSQL_DATABASE=dev_data \
-e MYSQL_ROOT_PASSWORD=redhat \
-v /home/user/db_data:/var/lib/mysql \
registry.lab.example.com/rhel8/mariadb-105
```

You notice that the `db01` container is not running.
```bash
[user@host ~]$ podman ps -a
CONTAINER ID  IMAGE                                              COMMAND         CREATED         STATUS                     PORTS       NAMES
dfdc20cf9a7e  registry.lab.example.com/rhel8/mariadb-105:latest  run-mysqld      29 seconds ago  Exited (1) 29 seconds ago              db01
```

The `podman container logs` command shows a permission error for the `/var/lib/mysql/db_data` directory.
```bash
[user@host ~]$ podman container logs db01
_...output omitted..._
---> 16:41:25     Initializing database ...
---> 16:41:25     Running mysql_install_db ...
mkdir: cannot create directory '/var/lib/mysql/db_data': Permission denied
Fatal error Can't create database directory '/var/lib/mysql/db_data'
```

This error happens because of the incorrect SELinux context that is set on the `/home/user/db_data` directory on the host machine.

##### SELinux Contexts for Container Storage

You must set the `container_file_t` SELinux context type before you can mount the directory as persistent storage to a container. If the directory does not have the `container_file_t` SELinux context, then the container cannot access the directory. You can append the `Z` option to the argument of the `podman run` command `-v` option to automatically set the SELinux context on the directory.

So you use the `podman run -v /home/user/db_data:/var/lib/mysql:Z` command to set the SELinux context for the `/home/user/db_data` directory when you mount it as persistent storage for the `/var/lib/mysql` directory.
```bash
[user@host ~]$ podman run -d --name db01 \
-e MYSQL_USER=student \
-e MYSQL_PASSWORD=student \
-e MYSQL_DATABASE=dev_data \
-e MYSQL_ROOT_PASSWORD=redhat \
-v /home/user/db_data:/var/lib/mysql:Z \
registry.lab.example.com/rhel8/mariadb-105
```

You then verify that the correct SELinux context is set on the `/home/user/db_data` directory with the `ls` command `-Z` option.
```bash
[user@host ~]$ ls -Z /home/user/
system_u:object_r:container_file_t:s0:c81,c1009 db_data
_...output omitted..._
```

### Assign a Port Mapping to Containers

To provide network access to containers, clients must connect to ports on the container host that pass the network traffic through to ports in the container. When you map a network port on the container host to a port in the container, the container receives network traffic that is sent to the host network port.

For example, you can map the 13306 port on the container host to the 3306 port on the container for communication with the MariaDB container. Therefore, traffic that is sent to the container host port 13306 would be received by MariaDB that is running in the container.

You use the `podman run` command `-p` option to set a port mapping from the `13306` port from the container host to the `3306` port on the `db01` container.
```bash
[user@host ~]$ podman run -d --name db01 \
-e MYSQL_USER=student \
-e MYSQL_PASSWORD=student \
-e MYSQL_DATABASE=dev_data \
-e MYSQL_ROOT_PASSWORD=redhat \
-v /home/user/db_data:/var/lib/mysql:Z \
-p 13306:3306 \
registry.lab.example.com/rhel8/mariadb-105
```

Use the `podman port` command `-a` option to show all container port mappings in use. You can also use the `podman port db01` command to show the mapped ports for the `db01` container.
```bash
[user@host ~]$ podman port -a
1c22fd905120	3306/tcp -> 0.0.0.0:13306
[user@host ~]$ podman port db01
3306/tcp -> 0.0.0.0:13306
```

You use the `firewall-cmd` command to allow port `13306` traffic into the container host machine to redirect to the container.
```bash
[root@host ~] firewall-cmd --add-port=13306/tcp --permanent
[root@host ~] firewall-cmd --reload
```

### Important

A rootless container cannot open a privileged port (ports below `1024`) on the container. That is, the `podman run -p 80:8080` command does not normally work for a running rootless container. To map a port on the container host below `1024` to a container port, you must run Podman as root or otherwise adjust the system.

You can map a port above `1024` on the container host to a privileged port on the container, even if you are running a rootless container. The `8080:80` mapping works if the container provides service listening on port `80`.

### DNS Configuration in a Container

Podman v4.0 supports two network back ends for containers, Netavark and CNI. Starting with RHEL 9, systems use Netavark by default. To verify which network back end is used, run the following `podman info` command.
```bash
[user@host ~]$ podman info --format {{.Host.NetworkBackend}}
netavark

```
#### Note

The `container-tools` meta-package includes the `netavark` and `aardvark-dns` packages. If Podman was installed as a stand-alone package, or if the `container-tools` meta-package was installed later, then the result of the previous command might be `cni`. To change the network back end, set the following configuration in the `/usr/share/containers/containers.conf` file:
```bash
[network]
_...output omitted..._
network_backend = "netavark"
```

Existing containers on the host that use the default Podman network cannot resolve each other's hostnames, because DNS is not enabled on the default network.

Use the `podman network create` command to create a DNS-enabled network. You use the `podman network create` command to create the network called `db_net`, and specify the subnet as `10.87.0.0/16` and the gateway as `10.87.0.1`.
```bash
[user@host ~]$ podman network create --gateway 10.87.0.1 \ --subnet 10.87.0.0/16 db_net
db_net
```

If you do not specify the `--gateway` or `--subnet` options, then they are created with the default values.

The `podman network inspect` command displays information about a specific network. You use the `podman network inspect` command to verify that the gateway and subnet were correctly set and that the new `db_net` network is DNS-enabled.

```bash
[user@host ~]$ podman network inspect db_net
[
     {
          "name": "db_net",
_...output omitted..._
          "subnets": [
               {
                    "subnet": "10.87.0.0/16",
                    "gateway": "10.87.0.1"
               }
          ],
_...output omitted..._
          "dns_enabled": true,
_...output omitted..._
```

You can add the DNS-enabled `db_net` network to a new container with the `podman run` command `--network` option. You use the `podman run` command `--network` option to create the `db01` and `client01` containers that are connected to the `db_net` network.
```bash
[user@host ~]$ podman run -d --name db01 \
-e MYSQL_USER=student \
-e MYSQL_PASSWORD=student \
-e MYSQL_DATABASE=dev_data \
-e MYSQL_ROOT_PASSWORD=redhat \
-v /home/user/db_data:/var/lib/mysql:Z \
-p 13306:3306 \
--network db_net \
registry.lab.example.com/rhel8/mariadb-105
[user@host ~]$ podman run -d --name client01 \
--network db_net \
registry.lab.example.com/ubi8/ubi:latest \
sleep infinity

```

Because containers are designed to have only the minimum required packages, the containers might not have the required utilities to test communication, such as the `ping` and `ip` commands. You can install these utilities in the container by using the `podman exec` command.
```bash
[user@host ~]$ podman exec -it db01 dnf install -y iputils iproute
_...output omitted..._
[user@host ~]$ podman exec -it client01 dnf install -y iputils iproute
_...output omitted..._
```

The containers can now ping each other by container name. You test the DNS resolution with the `podman exec` command. The names resolve to IPs within the subnet that was manually set for the `db_net` network.
```bash
[user@host ~]$ podman exec -it db01 ping -c3 client01
PING client01.dns.podman (10.87.0.4) 56(84) bytes of data.
64 bytes from 10.87.0.4 (10.87.0.4): icmp_seq=1 ttl=64 time=0.049 ms
_...output omitted..._
--- client01.dns.podman ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2007ms
rtt min/avg/max/mdev = 0.049/0.060/0.072/0.013 ms

[user@host ~]$ podman exec -it client01 ping -c3 db01
PING db01.dns.podman (10.87.0.3) 56(84) bytes of data.
64 bytes from 10.87.0.3 (10.87.0.3): icmp_seq=1 ttl=64 time=0.021 ms
_...output omitted..._
--- db01.dns.podman ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2047ms
rtt min/avg/max/mdev = 0.021/0.040/0.050/0.013 ms
```


You verify that the IP addresses in each container match the DNS resolution with the `podman exec` command.
```bash
[user@host ~]$ podman exec -it db01 ip a | grep 10.8
    inet 10.87.0.3/16 brd 10.87.255.255 scope global eth0
    inet 10.87.0.4/16 brd 10.87.255.255 scope global eth0
[user@host ~]$ podman exec -it client01 ip a | grep 10.8
    inet 10.87.0.3/16 brd 10.87.255.255 scope global eth0
    inet 10.87.0.4/16 brd 10.87.255.255 scope global eth0
```

#### Multiple Networks to a Single Container

Multiple networks can be connected to a container at the same time to help to separate different types of traffic.

You use the `podman network create` command to create the `backend` network.
```bash
[user@host ~]$ podman network create backend
```

You then use the `podman network ls` command to view all the Podman networks.
```bash
[user@host ~]$ podman network ls
NETWORK ID    NAME        DRIVER
a7fea510a6d1  backend     bridge
fe680efc5276  db01        bridge
2f259bab93aa  podman      bridge
```


The subnet and gateway were not specified with the `podman network create` command `--﻿gateway` and `--subnet` options.

You use the `podman network inspect` command to obtain the IP information of the `backend` network.
```bash
[user@host ~]$ podman network inspect backend
[
     {
          "name": "backend",
_...output omitted..._
          "subnets": [
               {
                    "subnet": "10.89.1.0/24",
                    "gateway": "10.89.1.1"
_...output omitted..._
```

You can use the `podman network connect` command to connect additional networks to a container when it is running. You use the `podman network connect` command to connect the `backend` network to the `db01` and `client01` containers.
```bash
[user@host ~]$ podman network connect backend db01
[user@host ~]$ podman network connect backend client01
```

#### Important

If a network is not specified with the `podman run` command, then the container connects to the default network. The default network uses the `slirp4netns` network mode, and the networks that you create with the `podman network create` command use the _bridge_ network mode. If you try to connect a bridge network to a container by using the `slirp4netns` network mode, then the command fails:

Error: "slirp4netns" is not supported: invalid network mode

You use the `podman inspect` command to verify that both networks are connected to each container and to display the IP information.
```bash
[user@host ~]$ podman inspect db01
_...output omitted..._
                    "backend": {
                         "EndpointID": "",
                         "Gateway": "10.89.1.1",
                         "IPAddress": "10.89.1.4",
_...output omitted..._
                    },
                    "db_net": {
                         "EndpointID": "",
                         "Gateway": "10.87.0.1",
                         "IPAddress": "10.87.0.3",
_...output omitted..._
[user@host ~]$ podman inspect client01
_...output omitted..._
                    "backend": {
                         "EndpointID": "",
                         "Gateway": "10.89.1.1",
                         "IPAddress": "10.89.1.5",
_...output omitted..._
                    },
                    "db_net": {
                         "EndpointID": "",
                         "Gateway": "10.87.0.1",
                         "IPAddress": "10.87.0.4",
_...output omitted..._
```

The `client01` container can now communicate with the `db01` container on both networks. You use the `podman exec` command to ping both networks on the `db01` container from the `client01` container.
```bash
[user@host ~]$ podman exec -it client01 ping -c3 10.89.1.4 | grep 'packet loss'
3 packets transmitted, 3 received, 0% packet loss, time 2052ms
[user@host ~]$ podman exec -it client01 ping -c3 10.87.0.3 | grep 'packet loss'
3 packets transmitted, 3 received, 0% packet loss, time 2054ms
```

#### References

`podman`(1), `podman-exec`(1), `podman-info`(1), `podman-network`(1), `podman-network-create`(1), `podman-network-inspect`(1), `podman-network-ls`(1), `podman-port`(1), `podman-run`(1), and `podman-unshare`(1) man pages

## Image Versioning and Tags

Because container images package software, developers often consider images as deployment artifacts. To keep images up to date, developers often map the image versions to the versions of the packaged software. Keeping images up to date also means updating the OS libraries within the image to receive improvements and security fixes.

One way to version images relative to their packaged software product is to use semantic versioning. Semantic version numbers form a string with the format `MAJOR.MINOR.PATCH` meaning:

- MAJOR: backward incompatible changes
- MINOR: backward compatible changes
- PATCH: bug fixes

Because versioning has no enforced structure, it is up to the image maintainers to follow good versioning practices. This is one reason why you should use trusted image registries and repositories.

Image versions can be used in the image name or in the image tag. An image tag is a string that you specify after the image name. Also, the same image can have multiple tags.

`[<image repository>/<namespace>/]<image name>[:<tag>]`

Using a tag in Podman is optional. When you do not specify a tag in a Podman command, Podman uses the `latest` tag by default.

```bash
[user@host ~]$ podman pull quay.io/argoproj/argocd
Trying to pull `quay.io/argoproj/argocd:latest`...
_...output omitted..._
```

Using the `latest` tag is considered a bad practice. Because the `latest` tag also represents the latest version of the image, it can include backwards-incompatible changes and cause containers that use the image to break.

To create additional tags for local images, use the `podman image tag` command.

```bash
[user@host ~]$ podman image tag LOCAL_IMAGE:TAG LOCAL_IMAGE:NEW_TAG
```

### Pulling Images

To search for images in different image registries, use a web browser to navigate to the registry URL and use the web UI.

Alternatively, use the `podman search` command to search for images in all the registries present in the `unqualified-search-registries` list in your `registries.conf` file. This enables you to search multiple registries.

```bash
[user@host ~]$ podman search nginx
NAME                                              DESCRIPTION
registry.fedoraproject.org/f29/nginx
...output omitted...
registry.access.redhat.com/ubi8/nginx-118       Platform for running nginx 1.18 or building...
...output omitted...
docker.io/library/nginx                         Official build of Nginx.
...output omitted...
quay.io/linuxserver.io/baseimage-alpine-nginx
...output omitted...
```

To retrieve an image, run ``podman image pull _`IMAGE_NAME`_``, which downloads the image from a registry. Alternatively, ``podman pull _`IMAGE_NAME`_`` provides the same functionality.

```bash
[user@host ~]$ podman pull registry.redhat.io/rhel8/mariadb-103:1
```

When you pull an image as a non-root user, Podman stores container images in the `~/.local/share/containers` directory.

To find which images your user has available locally, use the `podman image ls` or `podman images` command.

```
[user@host ~]$ podman image ls
REPOSITORY                                TAG    IMAGE ID     CREATED     SIZE
registry.redhat.io/rhel9/python-39        1-52   d336a3191d35 3 weeks ago 995 MB
registry.access.redhat.com/ubi8/python-39 latest 6b7a42c9d513 4 weeks ago 894 MB
_...output omitted..._
```

If an image is pulled by the root user, then it is stored in the `/var/lib/containers` directory. This image is only listed when `podman image ls` is run as root.

### Building Images

You can also build an image from a Containerfile, which describes the steps used to build an image. Run the `podman build --file CONTAINERFILE --tag IMAGE_REFERENCE` to build a container image.

For example, to build an image that you can later push to Red Hat Quay.io, execute the following command:

```
[user@host ~]$ podman build --file Containerfile \ 
--tag quay.io/YOUR_QUAY_USER/IMAGE_NAME:TAG
```

### Pushing Images

After you build an image, share it by pushing it to a remote registry. To push an image, you must be logged in to the registry. Run the `podman login REGISTRY` to log in to the specified registry. Then, you can use the `podman push IMAGE` command to push a local image to the remote registry.

For example, to push an image to Quay.io, run the following command:

```bash
[user@host ~]$ podman push quay.io/YOUR_QUAY_USER/IMAGE_NAME:TAG
Getting image source signatures
Copying blob fb3154998920 done
_...output omitted..._
Writing manifest to image destination
Storing signatures
```

### Inspecting Images

The `podman image inspect` command provides useful information about a locally available image in your system.

The following example usage shows information about a `mariadb` image.

```
[user@host ~]$ podman image inspect registry.redhat.io/rhel8/mariadb-103:1
 [
   {
    "Id": "6683...98ea",
    _...output omitted..._
    "Config": {
      "User": "27", # ![1]
      "ExposedPorts": { # ![2]
           "3306/tcp": {}
      },
      "Env": [ # ![3]
           "PATH=/opt/app-root/src/bin:/opt/app-root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            _...output omitted..._
          ],
      "Entrypoint": [ # ![4]
           "container-entrypoint"
      ],
      "Cmd": [ # ![5]
           "run-mysqld"
      ],
      "WorkingDir": "/opt/app-root/src", # ![6]
      "Labels": { # ![7]
         _...output omitted..._
         "release": "177.1654147959",
         "summary": "MariaDB 10.3 SQL database server",
         "url": "https://access.redhat.com/containers/#/registry.access.redhat.com/rhel8/mariadb-103/images/1-177.1654147959",
        _...output omitted..._
    "Architecture": "amd64", # ![8]
    "Os": "linux",
    "Size": 573593952,
    ...output omitted...
```

|     |                                                                |
| --- | -------------------------------------------------------------- |
| 1   | The default user for the image.                                |
| 2   | The port that the application exposes.                         |
| 3   | The environment variables used by the image.                   |
| 4   | The entrypoint, a command that runs when the container starts. |
| 5   | The command that the `container-entrypoint` script runs.       |
| 6   | The working directory for the commands in the image.           |
| 7   | Labels providing extra metadata.                               |
| 8   | The architecture where this image can be used.                 |

The output from `podman inspect` is verbose, which makes it hard to find information. To select a specific part of the output use the Go templating feature in Podman by providing the `--format` option. Use the `podman inspect` output keys preceded by dots and surrounded by double curly braces.

```
--format="{{.Key.Subkey}}"
```

For example, the following command extracts the `CMD` instruction from the `rhel8/mariadb-103` image:

```
[user@host ~]$ podman image \   inspect registry.redhat.io/rhel8/mariadb-103:1 \   --format="{{.Config.Cmd}}"
[run-mysqld]
```

To inspect a remote image, you can use the Skopeo tool.

### Export and Import File Systems

The `podman export` command exports a container's filesystem to a .tar file on your local machine. This command creates a snapshot of an existing container, so you can use it later. For example, if you inadvertently make changes to your container's filesystem and do not know how to fix it, then use the snapshot to return to a known starting point. By default, the `podman export` command writes to the standard output (STDOUT). To redirect the output to a file use the `--output` or `-o` option, specifying the name for the archive to create, and the container name or ID to export as arguments.

```
[user@host ~]$ podman export -o mytarfile.tar fb601b05cd3b
```

To import a .tar file containing a container file system, and save the file system as a container image, use the `podman import` command. The `podman import` command requires the image name and tag as arguments.

```
[user@host ~]$ podman import mytarfle.tar httpdcustom:2.4
Getting image source signatures
Copying blob 47662b708e31 done  |
Copying config 9af04983ef done  |
Writing manifest to image destination
sha256:9af0...4c8f
```

After importing a file system, you can verify the creation of the container image by using the `podman images` command.

```
[user@host ~]$ podman images
REPOSITORY                        TAG      IMAGE ID      CREATED         SIZE
localhost/httpdcustom             2.4      9af04983ef93  18 minutes ago  305 MB
registry.../rhscl/httpd-24-rhel7  latest   699f5c8b7fd3  2 months ago    330 MB
```

## Create Images with Containerfiles

### Creating Images with Containerfiles

A _Containerfile_ lists a set of instructions that a container runtime uses to build a container image. You can use the image to create any number of containers.

Each instruction causes a change that is captured in a resulting image layer. These layers are stacked together to form the resulting container image.

### Choosing a Base Image

When you build an image, podman executes the instructions in the Containerfile and applies the changes on top of a container _base image_. A base image is the image from which your Containerfile and its resulting image is built.

The base image you choose determines the Linux distribution and its components, such as:

- Package manager
- Init system
- Filesystem layout
- Preinstalled dependencies and runtimes

The base image can also influence other factors, such as image size, vendor support, and processor compatibility.

Red Hat provides container images intended as a common starting point for containers known as _universal base images (UBI)_. These UBIs come in four variants: `standard`, `init`, `minimal`, and `micro`. Additionally, Red Hat also provides UBI-based images that include popular runtimes, such as Python and Node.js.

These UBIs use Red Hat Enterprise Linux (RHEL) at their core and are available from the Red Hat Container Catalog. The main differences are as follows:

- Standard - This is the primary UBI, which includes DNF, systemd, and utilities such as `gzip` and `tar`.
- Init - Simplifies running multiple applications within a single container by managing them with systemd.
- Minimal - This image is smaller than the `init` image and still provides nice-to-have features. This image uses the `microdnf` minimal package manager instead of the full-sized version of DNF.
- Micro - This is the smallest available UBI because it only includes the bare minimum number of packages. For example, this image does not include a package manager.

### Containerfile Instructions

Containerfiles use a small domain-specific language (DSL) consisting of basic instructions for crafting container images. The following are the most common instructions.

`FROM`
Sets the base image for the resulting container image. Takes the name of the base image as an argument.

`WORKDIR`
Sets the current working directory within the container. Instructions that follow the `WORKDIR` instruction run within this directory.

`COPY` and `ADD`
Copy files from the build host into the file system of the resulting container image. Relative paths use the host current working directory, known as the build context. Both instructions use the working directory within the container as defined by the `WORKDIR` instruction.

The `ADD` instruction adds the following functionality:
- Copying files from URLs.
- Unpacking `tar` archives in the destination image.

Because the `ADD` instruction adds functionality that might not be obvious, developers tend to prefer the `COPY` instruction for copying local files into the container image.

`RUN`
Runs a command in the container and commits the resulting state of the container to a new layer within the image.

`ENTRYPOINT`
Sets the executable to run when the container is started.

`CMD`
Runs a command when the container is started. This command is passed to the executable defined by `ENTRYPOINT`. Base images define a default `ENTRYPOINT`, which is usually a shell executable, such as Bash.

__Note__  Neither `ENTRYPOINT` nor `CMD` run when building a container image. Podman executes them when you start a container from the image.

`USER`
Changes the active user within the container. Instructions that follow the `USER` instruction run as this user, including the `CMD` instruction. It is a good practice to define a different user other than `root` for security reasons.

`LABEL`
Adds a key-value pair to the metadata of the image for organization and image selection.

`EXPOSE`
Adds a port to the image metadata indicating that an application within the container binds to this port. This instruction does not bind the port on the host and is for documentation purposes.

`ENV`
Defines environment variables that are available in the container. You can declare multiple `ENV` instructions within the Containerfile. You can use the `env` command inside the container to view each of the environment variables.

`ARG`
Defines build-time variables, typically to make a customizable container build. Developers commonly configure the `ENV` instructions by using the `ARG` instruction. This is useful for preserving the build-time variables for runtime.

`VOLUME`
Defines where to store data outside of the container. The value configures the path where Podman mounts persistent volume inside of the container. You can define more than one path to create multiple volumes.

Each Containerfile instruction runs in an independent container by using an intermediate image built from every previous command. This means each instruction is independent from other instructions in the Containerfile. The following is an example Containerfile for building a simple Apache web server container:

```
# This is a comment line ![1]
FROM        registry.redhat.io/ubi8/ubi:8.6 ![2]
LABEL       description="This is a custom httpd container image" ![3]
RUN         yum install -y httpd ![4]
EXPOSE      80 ![5]
ENV         LogLevel "info" ![6]
ADD         http://someserver.com/filename.pdf /var/www/html ![7]
COPY        ./src/   /var/www/html/ ![8]
USER        apache ![9]
ENTRYPOINT  ["/usr/sbin/httpd"] ![10]
CMD         ["-D", "FOREGROUND"] ![11]

```


|     |                                                                                                                                       |
| --- | ------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Lines that begin with a hash, or pound, sign (`#`) are comments.                                                                      |
| 2   | The `FROM` instruction declares that the new container image extends the `registry.redhat.io/ubi8/ubi:8.6` container base image.      |
| 3   | The `LABEL` instruction adds metadata to the image.                                                                                   |
| 4   | `RUN` executes commands in a new layer on top of the current image. The shell that is used to execute commands is `/bin/sh`.          |
| 5   | `EXPOSE` configures metadata indicating that the container listens on the specified network port at runtime.                          |
| 6   | `ENV` is responsible for defining environment variables that are available in the container.                                          |
| 7   | The `ADD` instruction copies files or directories from a local or remote source and adds them to the container's file system.         |
| 8   | `COPY` copies files from a path relative to the working directory and adds them to the container's file system.                       |
| 9   | `USER` specifies the username or the UID to use when running the container image for the `RUN`, `CMD`, and `ENTRYPOINT` instructions. |
| 10  | `ENTRYPOINT` specifies the default command to execute when users instantiate the image as a container.                                |
| 11  | `CMD` provides the default arguments for the `ENTRYPOINT` instruction.                                                                |
| 12  | `ENTRYPOINT` specifies the default command to execute when users instantiate the image as a container.                                |
| 13  | `CMD` provides the default arguments for the `ENTRYPOINT` instruction.                                                                |

### Container Image Tags

When building a container image, you can specify an image name to later identify the image. The image name is a string composed of letters, numbers, and some special characters.

An _image tag_ comes after the image name and is delimited by a colon (`:`). When you omit an image tag, podman uses the default `latest` tag.

__Note__ 
It is a best practice to specify tags in addition to `latest` tag. Because `latest` is the default tag applied to new images, references that use the `latest` tag can change unintentionally.

The full name of the image includes both a name and an optional image tag.

For example, in the full image name `my-app:1.0`, the name is `my-app` and the tag is `1.0`.

## Build Images with Advanced Containerfile Instructions

### Advanced Container file Instructions

You can package your application in a container with the application runtime dependencies. However, developers commonly create containers that have a number of downsides, for example:

- The container is bound to a specific environment, and requires a rebuild before deploying to a production environment.
- The container contains developer tools, such as a debugger, text editors, or compilers.
- The container contains build-time dependencies that are not necessary at runtime.
- The container generates a large volume of files stored on the copy-on-write file system, which limits the performance of the application.

This section focuses on advanced container patterns that help you limit previously mentioned issues, such as:

- Reducing image storage footprint by using the multistage container build pattern.
- Customizing container runtime with environment variables.
- Using volumes to decrease the container size and increase the performance of writing files.

### The ENV Instruction

The `ENV` instruction lets you specify environment dependent configuration, for example, hostnames, ports or usernames. The containerized application can use the environment variables at runtime.

To include an environment variable, use the `key=value` format. The following example declares a `DB_HOST` variable with the database hostname.

```
ENV DB_HOST="database.example.com"
```

Then you can retrieve the environment variable in your application. The following example is a Python script that retrieves the `DB_HOST` environment variable.

```
from os import environ

DB_HOST = environ.get('DB_HOST')

# Connect to the database at DB_HOST...
```

### The ARG Instruction

Use the `ARG` instruction to define build-time variables, typically to make a customizable container build.

You can optionally configure a default build-time variable value if the developer does not provide it at build time. Use the following syntax to define a build-time variable:

```
ARG key[=default value]
```

When you build the container image, use the `--build-arg` flag to set the value, such as `podman build --build-arg key=example-value` for the preceding example.

Consider the following Containerfile example:

```
ARG VERSION="1.16.8" \
    BIN_DIR=/usr/local/bin/

RUN curl "https://dl.example.io/${VERSION}/example-linux-amd64" \
        -o ${BIN_DIR}/example
```

If you do not provide the `--build-arg` flag, then Podman uses the default values during the build process. However, you can change the values during the build process without changing the Containerfile by using the `ARG` instruction.

Developers commonly configure the `ENV` instructions by using the `ARG` instruction. This is useful for preserving the build-time variables for runtime.

Consider the following Containerfile example:

```
ARG VERSION="1.16.8" \
    BIN_DIR=/usr/local/bin/

ENV VERSION=${VERSION} \
    BIN_DIR=${BIN_DIR}

RUN curl "https://dl.example.io/${VERSION}/example-linux-amd64" \
        -o ${BIN_DIR}/example
```

In the preceding example, the `ENV` instruction uses the value of the `ARG` instruction to configure the run-time environment variables.

If you do not configure the `ARG` default values but you must configure the `ENV` default values, then use the `${VARIABLE:-DEFAULT_VALUE}` syntax, such as:

```
ARG VERSION \
    BIN_DIR

ENV VERSION=${VERSION:-1.16.8} \
    BIN_DIR=${BIN_DIR:-/usr/local/bin/}

RUN curl "https://dl.example.io/${VERSION}/example-linux-amd64" \
        -o ${BIN_DIR}/example
```

### The VOLUME Instruction

Use the `VOLUME` instruction to persistently store data. The value is the path where Podman mounts a persistent volume inside of the container. The `VOLUME` instruction accepts more than one path to create multiple volumes.

For example, the following Containerfile creates a volume to store PostgreSQL data.

```
FROM registry.redhat.io/rhel9/postgresql-13:1

VOLUME /var/lib/pgsql/data

```
In the previous Containerfile, if you add instructions that update `/var/lib/pgsql/data` after the VOLUME instruction, then those instructions are ignored.

To retrieve the local directory used by a volume, you can use the ``podman inspect _`VOLUME_ID`_`` command.

```
[user@host ~]$ **``podman inspect _`VOLUME_ID`_``**
[
    {
        `"Name": "VOLUME_ID",`
        "Driver": "local",
        `"Mountpoint": "/home/your-name/.local/share/containers/storage/volumes/VOLUME_ID/_data"`,
        _...output omitted..._
        `"Anonymous": true`
    }
]
```

The `Mountpoint` field gives you the absolute path to the directory where the volume exists on your host file system. Volumes created from the `VOLUME` instruction have a random ID in the `Name` field and are considered `Anonymous` volumes.

To remove unused volumes, use the `podman volume prune` command.

```
[user@host ~]$ **`podman volume prune`**
WARNING! This will remove all volumes not used by at least one container. The following volumes will be removed:
8c0c...bcb8c
Are you sure you want to continue? [y/N] **`y`**
8c0c...bcb8c
```

You can create a _named_ volume by using the `podman volume create` command.

```
[user@host ~]$ podman volume create _`VOLUME_NAME`_`
VOLUME_NAME
```

You can also format the `podman volume ls` command to include the `Mountpoint` field of every volume. Persistent data storage with volumes is covered in depth later in the course.

```
[user@host ~]$ podman volume ls \  --format="{{.Name}}\t{{.Mountpoint}}"
0a8c...82c2  /home/your-name/.local/share/containers/storage/volumes/0a8c...82c2/_data
252d...b2ed  /home/your-name/.local/share/containers/storage/volumes/252d...b2ed/_data
```

### The ENTRYPOINT and CMD Instructions

The `ENTRYPOINT` and `CMD` instructions specify the command to execute when the container starts. A valid Containerfile must have at least one of these instructions.

The `ENTRYPOINT` instruction defines an executable, or command, that is always part of the container execution. This means that additional arguments are passed to the provided command.

Consider the following example:

```
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5
ENTRYPOINT ["echo", "Hello"]
```

Running a container from the previous Containerfile with no arguments prints "Hello".

```
[user@host ~]$ podman run my-image
Hello
```

If you provide the container with the "Red" and "Hat" arguments, then it prints "Hello Red Hat".

```
[user@host ~]$ podman run my-image Red Hat
Hello Red Hat
```

If you only use the `CMD` instruction, then passing arguments to the container overrides the command provided in the `CMD` instruction.

```
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5
CMD ["echo", "Hello", "Red Hat"]
```

Running a container from the previous Containerfile with no arguments prints `Hello Red Hat`.

```
[user@host ~]$ podman run my-image
Hello Red Hat
```

Running a container with the argument `whoami` overrides the `echo` command.

```
[user@host ~]$ podman run my-image whoami
root
```

When a Containerfile specifies both `ENTRYPOINT` and `CMD` then `CMD` changes its behavior. In this case the values provided to `CMD` are passed as default arguments to the `ENTRYPOINT`.

```
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5
ENTRYPOINT ["echo", "Hello"]
CMD ["Red", "Hat"]
```

The previous example prints `Hello Red Hat` when you run the container.

```
[user@host ~]$ podman run my-image
Hello Red Hat
```

If you provide the argument `Podman` to the container, then it prints `Hello Podman`.

```
[user@host ~]$ podman run my-image Podman
Hello Podman
```


The `ENTRYPOINT` and `CMD` instructions have two formats for executing commands:

**Text array**

The executable takes the form of a text array, such as:

ENTRYPOINT ["executable", "param1", ... "paramN"]

In this form you must provide the full path to the executable.

**String form**

The command and parameters are written in a text form, such as:

CMD executable param1 ... paramN

The string form wraps the executable in a shell command such as `sh -c "executable param1 …​ paramN"`. This is useful when you require shell processing, for example for variable substitution.

You might need to change the container entrypoint at runtime, for example for troubleshooting. Consider the following Containerfile:

```
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5

LABEL GREETING="World"

ENTRYPOINT echo Hello "${GREETING}"
```

When you run the container, you notice unexpected output:

```
[user@host ~]$ **`podman run my-image`**
Hello
```

You can execute the `env` command to print the environment variables in the container. However, the preceding container uses the `ENTRYPOINT` instruction, which means that you cannot change the `echo` command by adding an argument to the container execution:

```
[user@host ~]$ **`podman run my-image env`**
Hello

```
In that case, you can overwrite the entrypoint by using the `podman run --entrypoint` command.

```
[user@host ~]$ **`podman run --entrypoint env my-image`**
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TERM=xterm
container=oci
HOME=/root
HOSTNAME=ba00a7663a93
```

Consequently, you verify that the `GREETING` environment variable is not set. This is because the developer that created the Containerfile used the `LABEL` instruction instead of the `ENV` instruction.

### Podman Secrets

A secret is a blob of sensitive information required by containers at runtime. This can be usernames, passwords, or keys. For example, if your containerized application requires credentials for connecting to a database, then you must store those credentials as a secret. After creating the secret, you must instruct the application container to make the credentials available to the application when it starts. To manage this process, use the `podman secret` subcommands.

#### Creating Podman Secrets

With Podman, you can create secrets by using either a file, or by passing the sensitive information to the standard input (STDIN). To create a secret from a file, run the `podman secret create` subcommand specifying the name of the file containing the sensitive information, and the name of the secret to create as arguments. The output of the `podman secret create` subcommand displays the secret ID.

[user@host ~]$ **`echo "R3d4ht123" > dbsecretfile`**
[user@host ~]$ **`podman secret create dbsecret dbsecretfile`**
9c2400836ee16ed07d86a3122

Run the `podman secret create` subcommand specifying the secret name and `-` as arguments. The `-` argument instructs podman to read the sensitive information from standard input.

[user@host ~]$ **`printf "R3d4ht123" | podman secret create dbsecret2 -`**
875a1e46fa64639756968c644

The `podman secret create` command supports different drivers to store the secrets. You can use the `--driver` or `-d` option to specify one of the supported secret drivers:

`file` (default)

Stores the secret in a read-protected file

`pass`

Stores the secret in a GPG-encrypted file.

`shell`

Manages the secret storage by using a custom script.

To list stored secrets, use the `podman secret ls` subcommand.

[user@host ~]$ **`podman secret ls`**
ID                         NAME         DRIVER      CREATED             UPDATED
875a1e46fa64639756968c644  dbsecret2    file        About a minute ago  About a minute ago
9c2400836ee16ed07d86a3122  dbsecret     file        14 minutes ago      14 minutes

When you no longer need a secret, you can remove it by running the `podman secret rm` command, and providing the name of the secret as argument.

[user@host ~]$ **`podman secret rm dbsecret2`**
875a1e46fa64639756968c644

#### Running Containers With Podman Secrets

To make secrets available for use to a container, execute the `podman run` command with the `--secret` option, and specify the name of the secret as parameter. You can use the `--secret` option multiple times for multiple secrets.

[user@host ~]$ **`podman run -it --secret dbsecret \ --name myapp registry.access.redhat.com/ubi8/ubi /bin/bash`**
[root@f9fedddbc81a /]# **`cat /run/secrets/dbsecret`**
R3d4ht123

When you use secrets, Podman retrieves the secret and places it on a tmpfs volume and then mounts the volume inside the container in the `/run/secret` directory as a file based on the name of the secret.

To prevent secrets from being stored in an image, neither the `podman commit` nor `podman export` commands copy the secret data to an image or .tar file.

### Multistage Builds

A multistage build uses multiple `FROM` instructions to create multiple independent container build processes, also called _stages_. Every stage can use a different base image and you can copy files between stages. The resulting container image consists of the last stage.

Multistage builds can reduce the image size by only keeping the necessary runtime dependencies. For example, consider the following example, where a Node application is containerized.

```
FROM registry.redhat.io/ubi8/nodejs-14:1

WORKDIR /app
COPY . .

RUN npm install
RUN npm run build

RUN serve build
```

The `npm install` command installs the required NPM packages, which includes packages that are only needed at build-time. The `npm run build` command uses the packages to create an optimized production-ready application build. Then, the container uses an HTTP server to expose the application by using the `serve` command.

If you build an image from the previous Containerfile, then the image contains both the build-time and runtime dependencies, which increases the image size. The resulting image also contains the Node.js runtime, which is not used at container runtime but might increase the attack surface of the container.

To avoid this issue, you can define two stages:
- **First stage:** Build the application.
- **Second stage:** Copy and serve the static files by using an HTTP server, such as NGINX or Apache Server.

The following example implements the two-stage build process.

```yaml
# First stage
FROM registry.access.redhat.com/ubi8/nodejs-14:1 as builder  # ![1]
COPY ./ /opt/app-root/src/
RUN npm install
RUN npm run build  #  ![2]

# Second stage
FROM registry.access.redhat.com/ubi8/nginx-120  # ![3]
COPY --from=builder /opt/app-root/src/ /usr/share/nginx/html  # ![4]
```

|     |                                                                                                                                                  |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Define the first stage with an alias. The second stage uses the `builder` alias to reference this stage.                                         |
| 2   | Build the application.                                                                                                                           |
| 3   | Define the second stage without an alias. It uses the `ubi8/nginx-120` base image to serve the production-ready version of the application.      |
| 4   | Copy the application files to a directory in the final image. The `--from` flag indicates that Podman copies the files from the `builder` stage. |

### Examine Container Data Layers

Container images use a copy-on-write (COW), layered file system. When you create a Containerfile, the `RUN`, `COPY`, and `ADD` instructions create _layers_ (sometimes referred to as _blobs_).

The layered COW file system ensures that a container remains immutable. When you start a container, Podman creates and mounts a thin, ephemeral, writable layer on top of the container image layers. When you delete the container, Podman deletes the writable thin layer. This means that all container layers stay identical, except for the thin writable layer.

#### Cache Image Layers

Because all container layers are identical, multiple containers can share the layers. This means Podman can cache layers, and build only those layers that are modified or not cached.

You can use caching to decrease build time. For example, consider a Node.js application Containerfile:

_...content omitted..._
COPY . /app/
_...content omitted..._

The previous instruction copies every file and directory in the Containerfile directory to the `/app` directory in the container. This means that any changes to the application result in the rebuilding of every layer after the `COPY` layer.

You can change the instructions as follows:

_...content omitted..._
COPY package.json /app/
RUN npm ci --production
COPY src ./src
_...content omitted..._

This means that if you change your application source code in the `src` directory and rebuild your container image, then the dependency layer is cached and skipped, which reduces the build time. Podman rebuilds the dependency layer only when you change the `package.json` file.

#### Reduce Image Layers

You can reduce the number of container image layers, for example, by chaining instructions. Consider the following `RUN` instructions:

RUN mkdir /etc/gitea
RUN chown root:gitea /etc/gitea
RUN chmod 770 /etc/gitea

You can reduce the number of layers to one by chaining the commands. In Linux, you can chain commands by using the double ampersand (`&&`). You can also use the backslash character (`\`) to break a long command into multiple lines.

Consequently, the resulting `RUN` command looks as follows:

RUN mkdir /etc/gitea && \
    chown root:gitea /etc/gitea && \
    chmod 770 /etc/gitea

The advantage of chaining commands is that you create less container image layers, which typically results in smaller images.

However, chained commands are more difficult to debug and cache.

You can also create Containerfiles that do not use chained commands, and configure Podman to squash the layers. Use the `--squash` option to squash layers declared in the Containerfile. Alternatively, use the `--squash-all` option to also squash the layers from the parent image.

For example, consider the following three builds of one Containerfile.

```
[user@host ~]$ podman build -t localhost/not-squashed .
_...output omitted..._
[user@host ~]$ podman build --squash -t localhost/squashed .
_...output omitted..._
[user@host ~]$ podman build --squash-all -t localhost/squashed-all .
_...output omitted..._
[user@host ~]$ podman images --format="{{.Names}}\t{{.Size}}"
[localhost/not-squashed:latest]  419 MB  # ![1]
[localhost/squashed:latest]      419 MB  # ![2]
[localhost/squashed-all:latest]  394 MB  # ![3]

```

|     |                                                                                                          |
| --- | -------------------------------------------------------------------------------------------------------- |
| 1   | The base image size is 419MB.                                                                            |
| 2   | When Podman squashed the image layers, the image size stayed the same but the number of layers is lower. |
| 3   | When Podman squashed the layers of the container image and its parent image, the size reduced by 25MB.   |

Developers commonly reduce the number of layers by using multistage builds in combination with chaining some commands.
