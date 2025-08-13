# Default Permissions and File Access

**Effects of Special Permissions on Files and Directories**
- The `suid`, `sgid`, and `sticky` special permissions provide additional access-related features to files.

| Permission     | Effect on files                                                                  | Effect on directories                                                                                                                          |
| :------------- | :------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------- |
| `u+s` (suid)   | File executes as the user that owns the file, not as the user that ran the file. | No effect.                                                                                                                                     |
| `g+s` (sgid)   | File executes as the group that owns the file.                                   | Files that are created in the directory have a group owner to match the group owner of the directory.                                          |
| `o+t` (sticky) | No effect.                                                                       | Users with write access to the directory can remove only files that they own; they cannot remove or force saves to files that other users own. |
In a long listing, you can identify the `setuid` permissions by a lowercase `s` character in the place where you would normally expect the `x` character (owner execute permissions). If the owner does not have execute permissions, then this character is replaced by an uppercase `S` character.

The `setgid` special permission on a directory means that created files in the directory inherit their group ownership from the directory, rather than inheriting group ownership from the creating user. This feature is commonly used on group collaborative directories to automatically change a file from the default private group to the shared group, or if a specific group should always own files in a directory.  
If `setgid` is set on an executable file, then commands run as the group that owns that file, rather than as the user that ran the command.  
In a long listing, you can identify the `setgid` permissions by a lowercase `s` character in the place where you would normally expect the `x` character (group execute permissions). If the group does not have execute permissions, then this character is replaced by an uppercase `S` character.  

The `sticky bit`for a directory sets a special restriction on deletion of files. Only the owner of the file (and the `root` user) can delete files within the directory.
In a long listing, you can identify the sticky permissions by a lowercase `t` character in the place where you would normally expect the `x` character (other execute permissions). If other does not have execute permissions, then this character is replaced by an uppercase `T` character.  

**umask**
The default umask values for Bash are defined in the `/etc/login.defs` file and might be affected by settings in the `/etc/profile` and `/etc/bashrc` files, files in `/etc/profile.d`, or your account's shell initialization files.