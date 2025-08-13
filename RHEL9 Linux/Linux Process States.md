# Linux Process States
Process state table

| Name     | Flag | Kernel-defined state name and description                                                                                                                                                                                                                                                                 |
| :------- | :--- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Running  | `R`  | TASK_RUNNING: The process is either executing on a CPU or waiting to run. The process can be executing user routines or kernel routines (system calls), or be queued and ready when in the _Running_ (or _Runnable_) state.                                                                               |
| Sleeping | `S`  | TASK_INTERRUPTIBLE: The process is waiting for some condition: a hardware request, system resource access, or a signal. When an event or signal satisfies the condition, the process returns to _Running_.                                                                                                |
|          | `D`  | TASK_UNINTERRUPTIBLE: This process is also sleeping, but unlike the `S` state, does not respond to signals. It is used only when process interruption might cause an unpredictable device state.                                                                                                          |
|          | `K`  | TASK_KILLABLE: Same as the uninterruptible `D` state, but modified to allow a waiting task to respond to the signal to kill it (exit completely). Utilities often display _Killable_ processes as the `D` state.                                                                                          |
|          | `I`  | TASK_REPORT_IDLE: A subset of state `D`. The kernel does not count these processes when calculating the load average. It is used for kernel threads. The TASK_UNINTERRUPTIBLE and TASK_NOLOAD flags are set. It is similar to TASK_KILLABLE, and is also a subset of state `D`. It accepts fatal signals. |
| Stopped  | `T`  | TASK_STOPPED: The process is stopped (suspended), usually by being signaled by a user or another process. The process can be continued (resumed) by another signal to return to running.                                                                                                                  |
|          | `T`  | TASK_TRACED: A process that is being debugged is also temporarily stopped and shares the `T` state flag.                                                                                                                                                                                                  |
| Zombie   | `Z`  | EXIT_ZOMBIE: A child process signals to its parent as it exits. All resources except for the process identity (PID) are released.                                                                                                                                                                         |
|          | `X`  | EXIT_DEAD: When the parent cleans up (reaps) the remaining child process structure, the process is now released completely. This state cannot be observed in process-listing utilities.                                                                                                                   |
## Control jobs
`jobs` -> list jobs, "+" - default job
```
[student@servera ~]$ **`jobs`**
[1]   Running                 control technical &
[2]-  Running                 control documents &
[3]+  Running                 control database &
```
`CTRL+Z` -> suspend job 
`bg` -> restart job
`fg` -> bring job to foreground

## Process Control with Signals
A _signal_ is a software interrupt that is delivered to a process. Signals report events to an executing program. Events that generate a signal can be an error, an external event (an I/O request or an expired timer), or by the explicit use of a signal-sending command or keyboard sequence.
**Fundamental process management signals**

|Signal|Name|Definition|
|:--|:--|:--|
|1|HUP|`Hangup` : Reports termination of the controlling process of a terminal. Also requests process re-initialization (configuration reload) without termination.|
|2|INT|`Keyboard interrupt` : Causes program termination. It can be blocked or handled. Sent by pressing the INTR (Interrupt) key sequence (**Ctrl**+**c**).|
|3|QUIT|`Keyboard quit` : Similar to SIGINT; adds a process dump at termination. Sent by pressing the QUIT key sequence (**Ctrl**+**\**).|
|9|KILL|`Kill, unblockable` : Causes abrupt program termination. It cannot be blocked, ignored, or handled; consistently fatal.|
|15 _default_|TERM|`Terminate` : Causes program termination. Unlike SIGKILL, it can be blocked, ignored, or handled. The "clean" way to ask a program to terminate; it allows the program to complete essential operations and self-cleanup before termination.|
|18|CONT|`Continue` : Sent to a process to resume if stopped. It cannot be blocked. Even if handled, it always resumes the process.|
|19|STOP|`Stop, unblockable` : Suspends the process. It cannot be blocked or handled.|
|20|TSTP|`Keyboard stop` : Unlike SIGSTOP, it can be blocked, ignored, or handled. Sent by pressing the suspend key sequence (**Ctrl**+**z**).|
__Note__: Signal numbers vary between Linux hardware platforms, but signal names and meanings are standard.  

You can use the `kill` command to send any signal, not just those signals for terminating programs. You can use the `kill` command `-l` option to list the names and numbers of all available signals.
```bash
[user@host ~]$ ps aux | grep job
5194 0.0 0.1 222448 2980 pts/1 S  16:39 0:00 /bin/bash /home/user/bin/control job1
5199 0.0 0.1 222448 3132 pts/1 S  16:39 0:00 /bin/bash /home/user/bin/control job2
5205 0.0 0.1 222448 3124 pts/1 S  16:39 0:00 /bin/bash /home/user/bin/control job3
5430 0.0 0.0 221860 1096 pts/1 S+ 16:41 0:00 grep --color=auto job

[user@host ~]$ kill 5194
[user@host ~]$ ps aux | grep job
user   5199  0.0  0.1 222448  3132 pts/1    S    16:39   0:00 /bin/bash /home/user/bin/control job2
user   5205  0.0  0.1 222448  3124 pts/1    S    16:39   0:00 /bin/bash /home/user/bin/control job3
user   5783  0.0  0.0 221860   964 pts/1    S+   16:43   0:00 grep --color=auto job
[1]   Terminated              control job1

[user@host ~]$ kill -9 5199
[user@host ~]$ ps aux | grep job
user   5205  0.0  0.1 222448  3124 pts/1    S    16:39   0:00 /bin/bash /home/user/bin/control job3
user   5930  0.0  0.0 221860  1048 pts/1    S+   16:44   0:00 grep --color=auto job
[2]-  Killed                  control job2

[user@host ~]$ kill -SIGTERM 5205
user   5986  0.0  0.0 221860  1048 pts/1  S+  16:45  0:00 grep --color=auto job
[3]+  Terminated              control job3
```

## Control Specific Processes
- Use the `pkill` command to signal one or more processes that match selection criteria. Selection criteria can be a command name, a process that a specific user owns, or all system-wide processes.  
- Use the `pgrep` command with the `-l` option to list the process names and IDs. Use either command with the `-u` option to specify the ID of the user who owns the processes.  
__Important__
Administrators commonly use SIGKILL.  
It is always fatal, because the SIGKILL signal cannot be handled or ignored. However, it forces termination without allowing the killed process to run self-cleanup routines.  
Red Hat recommends sending SIGTERM first, and then trying SIGINT; and only if both fail, trying again with SIGKILL
- Use the `pstree` command to view a process tree for the system or a single user
- The `killall` command can signal multiple processes, based on their command name.