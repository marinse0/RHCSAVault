# Configure Password Aging
The following example demonstrates the `chage` command to change the password policy of the `sysadmin05` user. The command defines a minimum age (`-m`) of zero days, a maximum age (`-M`) of 90 days, a warning period (`-W`) of 7 days, and an inactivity period (`-I`) of 14 days.
[root@host ~]# **`chage -m 0 -M 90 -W 7 -I 14 sysadmin05`**

Assume that you manage the user password policies on a Red Hat server. The `cloudadmin10` user is new in the system, and you want to set a custom password aging policy. You want to set the account expiration 30 days from today, so you use the following commands:

[root@host ~]# **`date +%F`** ![1](https://rol.redhat.com/rol/static/roc/Common_Content/images/1.svg)
2022-03-10
[root@host ~]# **`date -d "+30 days" +%F`** ![2](https://rol.redhat.com/rol/static/roc/Common_Content/images/2.svg)
2022-04-09
[root@host ~]# **`chage -E $(date -d "+30 days" +%F) cloudadmin10`**
[root@host ~]# **`chage -l cloudadmin10 | grep "Account expires"`**
Account expires						: Apr 09, 2022

**Note**
The `date` command can calculate a future date. The `-u` option reports the time in UTC.
[user01@host ~]$ **`date -d "+45 days" -u`**
Thu May 23 17:01:20 UTC 2019