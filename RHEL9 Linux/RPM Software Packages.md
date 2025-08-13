# RPM Software Packages
## Examine RPM Packages
- `rpm -qa` : List all installed packages.
- ``rpm -qf `FILENAME`` : Determine which package provides _FILENAME_.
- `rpm -q` : List the currently installed package version.
- `rpm -qi`: Get detailed package information.
- `rpm -ql` : List the files that the package installs.
- `rpm -qc` : List only the configuration files that the package installs.
- `rpm -qd` : List only the documentation files that the package installs.
- `rpm -q --scripts` : List the shell scripts that run before or after you install or remove the package.
- `rpm -q --changelog` : List the change log information for the package.
- `rpm -qlp` : List the files that the local package installs.
## Extracting RPM packages

Use the `rpm2cpio` command to extract files from an RPM package file without installing the package.  
The `rpm2cpio` command converts an RPM package to a `cpio` archive. After the RPM package is converted to a `cpio` archive, the `cpio` command can extract a list of files.  
Use the `cpio` command with the `-i` option to extract files from standard input. Use the `-d` option to create subdirectories as needed, starting in the current working directory. Use the `-v` option for verbose output.
[user@host tmp-extract]$ `rpm2cpio httpd-2.4.51-7.el9_0.x86_64.rpm | cpio -idv`
Extract individual files by specifying the path of the file:  
[user@host ~]$ `rpm2cpio httpd-2.4.51-7.el9_0.x86_64.rpm | cpio -id "*/etc/httpd/conf/httpd.conf"`
