# Log Events to the System

## About Syslog

Many programs use the syslog protocol to log events to the system. Each log message is categorized by facility (the subsystem that produces the message) and priority (the message's severity).

The following table lists the standard syslog facilities:

**Table 3.2. Overview of Syslog Facilities**

|Code|Facility|Facility description|
|:--|:--|:--|
|0|kern|Kernel messages|
|1|user|User-level messages|
|2|mail|Mail system messages|
|3|daemon|System daemon messages|
|4|auth|Authentication and security messages|
|5|syslog|Internal syslog messages|
|6|lpr|Printer messages|
|7|news|Network news messages|
|8|uucp|UUCP protocol messages|
|9|cron|Clock daemon messages|
|10|authpriv|Non-system authorization messages|
|11|ftp|FTP protocol messages|
|16-23|local0 to local7|Custom local messages|

The following table lists the standard syslog priorities in descending order:

**Table 3.3. Overview of Syslog Priorities**

|Code|Priority|Priority description|
|:--|:--|:--|
|0|emerg|System is unusable|
|1|alert|Action must be taken immediately|
|2|crit|Critical condition|
|3|err|Non-critical error condition|
|4|warning|Warning condition|
|5|notice|Normal but significant event|
|6|info|Informational event|
|7|debug|Debugging-level message|

The `rsyslog` service uses the facility and priority of log messages to determine how to handle them. Rules configure this facility and priority in the `/etc/rsyslog.conf` file and in any file in the `/etc/rsyslog.d` directory with the `.conf` extension. Software packages can add rules by installing an appropriate file in the `/etc/rsyslog.d` directory.

Each rule that controls how to sort syslog messages has a line in one of the configuration files. The left side of each line indicates the facility and priority of the syslog messages that the rule matches. The right side of each line indicates which file to save the log message in (or where else to deliver the message). An asterisk (`*`) is a wildcard that matches all values.

For example, the following line in the `/etc/rsyslog.d` file would record messages that are sent to the `authpriv` facility at any priority to the `/var/log/secure` file:
```
authpriv.*                  /var/log/secure
```

Sometimes, log messages match more than one rule in the `rsyslog.conf` file. In such cases, one message is stored in more than one log file. The `none` keyword in the priority field indicates that no messages for the indicated facility are stored in the given file, to limit stored messages.

Instead of being logged to a file, syslog messages can also be printed to the terminals of all logged-in users. The `rsyslog.conf` file has a setting to print all the syslog messages with the `emerg` priority to the terminals of all logged-in users.
## Send Syslog Messages Manually

The `logger` command sends messages to the `rsyslog` service. By default, the `logger` command sends the message to the user type with the `notice` priority (`user.notice`) unless specified otherwise with the `-p` option. It is helpful to test any change to the `rsyslog` service configuration.

To send a message to the `rsyslog` service to be recorded in the `/var/log/boot.log` log file, execute the following `logger` command:

```
[root@host ~]# logger -p local7.notice "Log entry created on host"
```