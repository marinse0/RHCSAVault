# Basic SELinux Concepts

SELinux policies are security rules that define how specific processes access relevant files, directories, and ports. Every resource entity, such as a file, process, directory, or port, has a label called an _SELinux context_. The context label matches a defined SELinux policy rule to allow a process to access the labeled resource. By default, an SELinux policy does not allow any access unless an explicit rule grants access. When no allow rule is defined, all access is disallowed.

SELinux labels have `user`, `role`, `type`, and `security level` fields. Targeted policy, which is enabled in RHEL by default, defines rules by using the `type` context. Type context names typically end with **`_t`**.

|   |
|---|
|![](https://rol.redhat.com/rol/static/static_file_cache/rh134-9.0/selinux/selinux_context.svg)|
Figure 6.1: SELinux file context

## Policy Access Rule Concepts

For example, a web server process is labeled with the `httpd_t` type context. Web server files and directories in the `/var/www/html/` directory and other locations are labeled with the `httpd_sys_content_t` type context. Temporary files in the `/tmp` and `/var/tmp` directories have the `tmp_t` type contexts as a label. The web server's ports have the `http_port_t` type context as a label.

An Apache web server process runs with the `httpd_t` type context. A policy rule permits the Apache server to access files and directories that are labeled with the `httpd_sys_content_t` type context. By default, files in the `/var/www/html` directory have the `httpd_sys_content_t` type context. A web server policy has by default no `allow` rules for using files that are labeled `tmp_t`, such as in the `/tmp` and `/var/tmp` directories, thus disallowing access. With SELinux enabled, a malicious user who uses a compromised Apache process would still not have access to the `/tmp` directory files.

A MariaDB server process runs with the `mysqld_t` type context. By default, files in the `/data/mysql` directory have the `mysqld_db_t` type context. A MariaDB server can access the `mysqld_db_t` labeled files, but has no rules to allow access to files for other services, such as `httpd_sys_content_t` labeled files.

# Control SELinux File Contexts
## Change the SELinux Context

You can manage the SELinux context on files with the `semanage fcontext`, `restorecon`, and `chcon` commands.

The recommended method to change the context for a file is to create a file context policy by using the `semanage fcontext` command, and then to apply the specified context in the policy to the file by using the `restorecon` command. This method ensures that you can relabel the file to its correct context with the `restorecon` command whenever necessary. The advantage of this method is that you do not need to remember what the context is supposed to be, and you can correct the context on a set of files.

The `chcon` command changes the SELinux context directly on files, without referencing the system's SELinux policy. Although `chcon` is useful for testing and debugging, changing contexts manually with this method is temporary. Although file contexts that you can change manually survive a reboot, they might be replaced if you run `restorecon` to relabel the contents of the file system.

### Important
When an SELinux system _relabel_ occurs, all files on a system are labeled with their policy defaults. When you use `restorecon` on a file, any context that you change manually on the file is replaced if it does not match the rules in the SELinux policy.

The following example creates a directory with a `default_t` SELinux context, which it inherited from the `/` parent directory.
```bash
[root@host ~] mkdir /virtual
[root@host ~] ls -Zd /virtual
unconfined_u:object_r:`default_t`:s0 /virtual
```

The `chcon` command sets the file context of the `/virtual` directory to the `httpd_sys_content_t` type.
```bash
[root@host ~] chcon -t httpd_sys_content_t /virtual
[root@host ~] ls -Zd /virtual
unconfined_u:object_r:`httpd_sys_content_t`:s0 /virtual
```

Running the `restorecon` command resets the context to the default value of `default_t`. Note the `Relabeled` message.
```bash
[root@host ~] restorecon -v /virtual
`Relabeled` /virtual from unconfined_u:object_r:`httpd_sys_content_t`:s0 to unconfined_u:object_r:`default_t`:s0
[root@host ~] ls -Zd /virtual
unconfined_u:object_r:`default_t`:s0 /virtual
```

## Define SELinux Default File Context Policies

The `semanage fcontext` command displays and modifies the policies that determine the default file contexts. You can list all the file context policy rules by running the `semanage fcontext -l` command. These rules use extended regular expression syntax to specify the path and file names.

When viewing policies, the most common extended regular expression is `(/.*)?`, which is usually appended to a directory name. This notation is humorously called _the pirate_, because it looks like a face with an eye patch and a hooked hand next to it.

This syntax is described as "a set of characters that begin with a slash and followed by any number of characters, where the set can either exist or not exist". Stated more simply, this syntax matches the directory itself, even when empty, and also matches almost any file name that is created within that directory.

For example, the following rule specifies that the `/var/www/cgi-bin` directory, and any files in it or in its subdirectories (and in their subdirectories, and so on), have the `system_u:object_r:httpd_sys_script_exec_t:s0` SELinux context, unless a more specific rule overrides this one.

/var/www/cgi-bin(/.*)?  all files  system_u:object_r:httpd_sys_script_exec_t:s0

### Note
The `all files` field option from the previous example is the default file type that `semanage` uses when you do not specify one. This option applies to all file types that you can use with `semanage`; they are the same as the standard file types as in the _Control Access to Files_ chapter in the _Red Hat System Administration I_ (RH124) course. You can get more information from the `semanage-fcontext`(8) man page.

## Basic File Context Operations
The following table is a reference for the `semanage fcontext` command options to add, remove, or list SELinux file context policies.

**Table 6.1. The `semanage fcontext` Command**

|Option|Description|
|:--|:--|
|`-a, --add`|Add a record of the specified object type.|
|`-d, --delete`|Delete a record of the specified object type.|
|`-l, --list`|List records of the specified object type.|

To manage SELinux contexts, install the `policycoreutils` and `policycoreutils-python-utils` packages, which contain the `restorecon` and `semanage` commands.

To reset all files in a directory to the default policy context, first use the `semanage fcontext -l` command to locate and verify that the correct policy exists with the intended file context. Then, use the `restorecon` command on the wildcarded directory name to reset all the files recursively. In the following example, view the file contexts before and after using the `semanage` and `restorecon` commands.

First, verify the SELinux context for the files:
```bash
[root@host ~] ls -Z /var/www/html/file*
unconfined_u:object_r:`user_tmp_t`:s0 `/var/www/html/file1`
unconfined_u:object_r:`httpd_sys_content_t`:s0 `/var/www/html/file2`
```

Then, use the `semanage fcontext -l` command to list the default SELinux file contexts:
```bash
[root@host ~] semanage fcontext -l
_...output omitted..._
/var/www(/.*)?       all files    system_u:object_r:httpd_sys_content_t:s0
_...output omitted..._
```

The `semanage` command output indicates that all the files and subdirectories in the `/var/www/` directory have the `httpd_sys_content_t` context by default. Running `restorecon` command on the wildcarded directory restores the default context on all files and subdirectories.
```bash
[root@host ~] restorecon -Rv /var/www/
`Relabeled` /var/www/html/`file1` from unconfined_u:object_r:`user_tmp_t`:s0 to unconfined_u:object_r:`httpd_sys_content_t`:s0
[root@host ~] ls -Z /var/www/html/file*
unconfined_u:object_r:`httpd_sys_content_t`:s0 /var/www/html/`file1`
unconfined_u:object_r:`httpd_sys_content_t`:s0 /var/www/html/`file2`
```

The following example uses the `semanage` command to add a context policy for a new directory. First, create the `/virtual` directory with an `index.html` file inside it. View the SELinux context for the file and the directory.
```bash
[root@host ~] mkdir /virtual
[root@host ~] touch /virtual/index.html
[root@host ~] ls -Zd /virtual/
unconfined_u:object_r:`default_t`:s0 /virtual
[root@host ~] ls -Z /virtual/
unconfined_u:object_r:`default_t`:s0 index.html
```

Next, use the `semanage fcontext` command to add an SELinux file context policy for the directory.
```bash
[root@host ~] semanage fcontext -a -t httpd_sys_content_t '/virtual(/.*)?'
```

Use the `restorecon` command on the wildcarded directory to set the default context on the directory and all files within it.
```bash
[root@host ~] restorecon -RFvv /virtual
Relabeled /virtual from unconfined_u:object_r:default_t:s0 to system_u:object_r:httpd_sys_content_t:s0
Relabeled /virtual/index.html from unconfined_u:object_r:default_t:s0 to system_u:object_r:httpd_sys_content_t:s0
[root@host ~] ls -Zd /virtual/
drwxr-xr-x. root root system_u:object_r:`httpd_sys_content_t`:s0 /virtual/
[root@host ~] ls -Z /virtual/
-rw-r--r--. root root system_u:object_r:`httpd_sys_content_t`:s0 index.html
```

Use the `semanage fcontext -l -C` command to view any local customizations to the default policy.
```bash
[root@host ~] semanage fcontext -l -C
SELinux fcontext     type         Context

/virtual(/.*)?       all files    system_u:object_r:httpd_sys_content_t:s0
```

# Troubleshoot SELinux Issues
When applications unexpectedly fail to work due to SELinux access denials, methods and tools are available to resolve these issues. It is helpful to start by understanding some fundamental concepts and behaviors when SELinux is enabled.

- SELinux consists of targeted policies that explicitly define allowable actions.
- A policy entry defines a labeled process and a labeled resource that interact.
- The policy states the process type, and the file or port context, by using labels.
- The policy entry defines one process type, one resource label, and the explicit action to allow.
- An action can be a system call, a kernel function, or another specific programming routine.
- If no entry is created for a specific process-resource-action relationship, then the action is denied.
- When an action is denied, the attempt is logged with useful context information.

Red Hat Enterprise Linux provides a stable targeted SELinux policy for almost every service in the distribution. Therefore, it is unusual to have SELinux access problems with common RHEL services when they are configured correctly. SELinux access problems occur when services are implemented incorrectly, or when new applications have incomplete policies. Consider these troubleshooting concepts before making broad SELinux configuration changes.

- Most access denials indicate that SELinux is working correctly by blocking improper actions.
- Evaluating denied actions requires some familiarity with normal, expected service actions.
- The most common SELinux issue is an incorrect context on new, copied, or moved files.
- File contexts can be fixed when an existing policy references their location.
- Optional Boolean policy features are documented in the `_selinux` man pages.
- Implementing Boolean features generally requires setting additional non-SELinux configuration.
- SELinux policies do not replace or circumvent file permissions or access control list restrictions.

When a common application or service fails, and the service is known to have a working SELinux policy, first see the service's `_selinux` man page to verify the correct context type label. View the affected process and file attributes to verify that the correct labels are set.

### Monitor SELinux Violations
The SELinux troubleshooting service, from the `setroubleshoot-server` package, provides tools to diagnose SELinux issues. When SELinux denies an action, an Access Vector Cache (AVC) message is logged to the `/var/log/audit/audit.log` security log file. The SELinux troubleshooting service monitors for AVC events and sends an event summary to the `/var/log/messages` file.

The AVC summary includes an event unique identifier (UUID). Use the ``sealert -l _`UUID`_`` command to view comprehensive report details for the specific event. Use the `sealert -a /﻿var/log/audit/audit.log` command to view all existing events.

Consider the following example sequence of commands on a standard Apache web server. You create `/root/mypage` and move it to the default Apache content directory (`/var/www/html`). Then, after starting the Apache service, you try to retrieve the file content.
```bash
[root@host ~] touch /root/mypage
[root@host ~] mv /root/mypage /var/www/html
[root@host ~] systemctl start httpd
[root@host ~] curl http://localhost/mypage
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
</body></html>
```

The web server does not display the content, and returns a `permission denied` error. An AVC event is logged to the `/var/log/audit/audit.log` and `/var/log/messages` files. Note the suggested `sealert` command and UUID in the `/var/log/messages` event message.
```bash
[root@host ~] tail /var/log/audit/audit.log
_...output omitted..._
type=AVC msg=audit(1649249057.067:212): avc:  denied  { getattr } for  pid=2332 comm="httpd" path="/var/www/html/mypage" dev="vda4" ino=9322502 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file permissive=0
...output omitted
[root@host ~] tail /var/log/messages
_...output omitted..._
Apr  6 08:44:19 host setroubleshoot[2547]: SELinux is preventing /usr/sbin/httpd from getattr access on the file /var/www/html/mypage. For complete SELinux messages run: sealert -l 95f41f98-6b56-45bc-95da-ce67ec9a9ab7
_...output omitted..._
```
The `sealert` output describes the event, and includes the affected process, the accessed file, and the attempted and denied action. The output includes advice for correcting the file's label, if appropriate. Additional advice describes how to generate a new policy to allow the denied action. Use the given advice only when it is appropriate for your scenario.

#### Important
The `sealert` output includes a confidence rating, which indicates the level of confidence that the given advice will mitigate the denial. However, that advice might not be appropriate for your scenario.

For example, if the AVC denial is because the denied file is in the wrong location, then advice that states either to adjust the file's context label, or to create a policy for this location and action, although technically accurate, is not the correct solution for your scenario. If the root cause is a wrong location or file name, then moving or renaming the file and then restoring a correct file context is the correct solution instead.
```bash
[root@host ~] sealert -l 95f41f98-6b56-45bc-95da-ce67ec9a9ab7
`SELinux is preventing /usr/sbin/httpd from getattr access on the file /var/www/html/mypage.`

*****  `Plugin restorecon (99.5 confidence) suggests`   ************************
```

If you want to fix the label.
`/var/www/html/mypage default label should be httpd_sys_content_t.`
Then you can run restorecon. The access attempt may have been stopped due to insufficient permissions to access a parent directory in which case try to change the following command accordingly.
```bash
/sbin/restorecon -v /var/www/html/mypage

*****  `Plugin catchall (1.49 confidence) suggests`   **************************

```

## Summary
- Use the `getenforce` and `setenforce` commands to manage the SELinux mode of a system.
- The `semanage` command manages SELinux policy rules. The `restorecon` command applies the context that the policy defines.
- Booleans are switches that change the behavior of the SELinux policy. You can enable or disable them to tune the policy.
- The `sealert` command displays useful information to help with SELinux troubleshooting.
