# Locate Files on the System
To search for files by file name, use the `find` command ``-name _`FILENAME`_`` option to return the path of files that match _FILENAME_ exactly. For example, to search for the `sshd_config` files in the root `/` directory, run the following command:

[root@host ~]# **`find / -name sshd_config`**
## Find Files Based on Ownership or Permission
[developer@host ~]$ **`find -user developer`** or also **`find -uid 1000`**
[developer@host ~]$ **`find -group developer`** or also **`find -gid 1000`**
[root@host ~]# **`find / -user root -group mail`** - file owner and group owner are different

The `find` command `-perm` option looks for files with a particular permission set. The octal values define the permissions with `4`, `2`, and `1` for read, write, and execute. Permissions are preceded with a `/` or `-` sign to control the search results.

Octal permission that is preceded by the `/` sign **matches files where at least one permission is set for user, group, or other** for that permission set. A file with the `r--r--r--` permissions does not match the `/222` permission but matches the `rw-r--r--` permission.   
A `-` sign before the permission means that **all three parts of the permissions must match**. For the previous example, files with the `rw-rw-rw-` permissions match. You can also use the `find` command `-perm` option with the symbolic method for permissions.

For example, the following commands match any file in the `/home` directory for which the owning user has read, write, and execute permissions, and members of the owning group have read and write permissions, and others have read-only access. Both commands are equivalent; the first one uses the octal method for permissions, whereas the second one uses the symbolic methods.
[root@host ~]# **`find /home -perm 764`**
[root@host ~]# **`find /home -perm u=rwx,g=rw,o=r`**

The `find` command `-ls` option is convenient when searching files by permissions, because it provides information for the files that includes their permissions.
[root@host ~]# **`find /home -perm 764 -ls`**
 26207447   0 `-rwxrw-r--`   1 user  user   0 May 10 04:29 /home/user/file1

To search for files for which the user has at least write and execute permissions, and the group has at least write permission, and others have at least read permission, run the following command:
[root@host ~]# **`find /home -perm -324`**
[root@host ~]# **`find /home -perm -u=wx,g=w,o=r`**


To search for files for which the user has read permissions, or the group has at least read permissions, or others have at least write permission, run the following command:
[root@host ~]# **`find /home -perm /442`**
[root@host ~]# **`find /home -perm /u=r,g=r,o=w`**

When used with `/` or `-` signs, the `0` value works as a wildcard, because it means any permission.

To search for any file in the `/home/developer` directory for which others have at least read access on the `host` machine, run the following command:
[developer@host ~]$ **`find -perm -004`**
[developer@host ~]$ **`find -perm -o=r`**

To search for all files in the `/home/developer` directory where others have write permission, run the following command:
[developer@host ~]$ **`find -perm -002`**
[developer@host ~]$ **`find -perm -o=w`**

## Find Files Based on Size
The `find` command `-size` option is followed by a numeric value, and the unit looks up files that match a specified size. Use the following list for the units with the `find` command `-size` option:
- For kilobytes, use the `k` unit with `k` always in lowercase.
- For megabytes, use the `M` unit with `M` always in uppercase.
- For gigabytes, use the `G` unit with `G` always in uppercase.

You can use the plus `+` and minus `-` characters to include files that are larger and smaller than the given size, respectively. The following example shows a search for files with an exact size of 10 megabytes:
[developer@host ~]$ **`find -size 10M`

To search for files with a size of more than 10 gigabytes:
[developer@host ~]$ **`find -size +10G`**

To search for files with a size of less than 10 kilobytes:
[developer@host ~]$ **`find -size -10k`**

## Find Files Based on Modification Time
The `find` command `-mmin` option, followed by the time in minutes, searches for all files with content that changed `n` minutes ago. The file's time stamp is rounded down and supports fractional values with the `+n` and `-n` range.

To search for all files with content that changed 120 minutes ago:
[root@host ~]# **`find / -mmin 120`**
The `+` modifier in front of the minutes finds all files  that changed more than `n` minutes ago. 
The `-` modifier searches for all files that changed less than `n` minutes ago. 

## Find Files Based on File Type
The `find` command `-type` option limits the search scope to a given file type. Use the following flags to limit the search scope:
- For regular files, use the `f` flag.
- For directories, use the `d` flag.
- For soft links, use the `l` flag.
- For block devices, use the `b` flag.
[root@host ~]# **`find /etc -type d`