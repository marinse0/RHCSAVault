# Bash
## Process Items from the Command Line - "for" loop
In Bash, the `for` loop construct uses the following syntax:
```bash
for `VARIABLE` in `LIST`; do
`COMMAND` `VARIABLE`
done

# sample
[user@host ~]$ for PACKAGE in $(rpm -qa | grep kernel); \
do echo "$PACKAGE was installed on \
$(date -d @$(rpm -q --qf "%{INSTALLTIME}\n" $PACKAGE))"; done
```

## Bash Script Exit Codes
After a script interprets and processes all of its content, the script process exits and passes back control to the parent process that called it. However, a script can be exited before it finishes, such as when the script encounters an error condition. Use the `exit` command to immediately leave the script, and skip processing the remainder of the script.

Use the `exit` command with an optional integer argument between `0` and `255`, which represents an _exit code_. An exit code is returned to a parent process to indicate the status at exit. An exit code value of `0` represents a successful script completion with no errors. All other nonzero values indicate an error exit code. The script programmer defines these codes. Use unique values to represent the different error conditions that are encountered. Retrieve the exit code of the last completed command from the built-in `$?` variable.
## Test Logic for Strings and Directories, and to Compare Values

To ensure that unexpected conditions do not disrupt scripts, it is recommended to verify command input such as command-line arguments, user input, command substitutions, variable expansions, and file name expansions. You can verify integrity in your scripts by using the Bash `test` command.

All commands produce an exit code on completion.

To see the exit status, view the `$?` variable immediately after executing the test command. An exit status of 0 indicates a successful exit with nothing to report. Nonzero values indicate some condition or failure. Use various operators to test whether a number is greater than (`gt`), greater than or equal to (`ge`), less than (`lt`), less than or equal to (`le`), or equal (`eq`) to another number.

Use operators to test whether a string of text is the same (`=` or `==`) or not the same (`!=`) as another string of text, or whether the string has zero length (`z`) or has a non-zero length (`n`). You can also test whether a regular file (`-f`) or directory (`-d`) exists, and has some special attributes, such as if the file is a symbolic link (`-L`), or if the user has read permissions (`-r`).

The following examples demonstrate the `test` command with Bash numeric comparison operators:
```bash
[user@host ~]$ test 1 -gt 0 ; echo $?
0
[user@host ~]$ test 0 -gt 1 ; echo $?
1
```

Test by using the Bash test command syntax, `[ <TESTEXPRESSION> ]` or the newer extended test command syntax, `[[ <TESTEXPRESSION> ]]`, which provides features such as file name globbing and regex pattern matching. In most cases, use the `[[ <TESTEXPRESSION> ]]` syntax.

The following examples demonstrate the Bash test command syntax and numeric comparison operators:
```bash
[user@host ~]$ [[ 1 -eq 1 ]]; echo $?
0
[user@host ~]$ [[ 1 -ne 1 ]]; echo $?
1
[user@host ~]$ [[ 8 -gt 2 ]]; echo $?
0
[user@host ~]$ [[ 2 -ge 2 ]]; echo $?
0
[user@host ~]$ [[ 2 -lt 2 ]]; echo $?
1
[user@host ~]$ [[ 1 -lt 2 ]]; echo $?
0
```

The following examples demonstrate the Bash string comparison operators:
```bash
[user@host ~]$ [[ abc = abc ]]; echo $?
0
[user@host ~]$ [[ abc == def ]]; echo $?
1
[user@host ~]$ [[ abc != def ]]; echo $?
0
```

The following examples demonstrate Bash string unary (one argument) operators:
```bash
[user@host ~]$ STRING=''; [[ -z "$STRING" ]]; echo $?
0
[user@host ~]$ STRING='abc'; [[ -n "$STRING" ]]; echo $?
0
```

## Conditional Structures
Simple shell scripts represent a collection of commands that are executed from beginning to end. Programmers incorporate decision-making into shell scripts by using conditional structures. A script can execute specific routines when stated conditions are met.

### Use the If/Then Construct
The simplest conditional structure is the _if/then_ construct, with the following syntax:

With this construct, if the script meets the given condition, then it executes the code in the statement block. It does not act if the given condition is not met. Common test conditions in the `if/then` statements include the previously discussed numeric, string, and file tests. The `fi` statement at the end closes the `if/then` construct. The following code section demonstrates an `if/then` construct to start the `psacct` service if it is not active:

```bash
[user@host ~]$ systemctl is-active psacct > /dev/null 2>&1
[user@host ~]$ if  [[ $? -ne 0 ]]; then sudo systemctl start psacct; fi
```

### Use the If/Then/Else Construct
You can further expand the `if/then` construct to take different sets of actions depending on whether a condition is met. Use the `if/then/else` construct to accomplish this behavior, as in this example:

The following code section demonstrates an `if/then/else` statement to start the `psacct` service if it is not active, and to stop it if it is active:

```bash
[user@host ~]$ systemctl is-active psacct > /dev/null 2>&1
[user@host ~]$ if  [[ $? -ne 0 ]]; then \ sudo systemctl start psacct; \ else \ sudo systemctl stop psacct; \ fi
```

### Use the If/Then/Elif/Then/Else Construct

Expand an `if/then/else` construct to test more than one condition and to execute a different set of actions when it meets a specific condition. The next example shows the construct for an added condition:

In this conditional structure, Bash tests the conditions as they are ordered in the script. When a condition is true, Bash executes the actions that are associated with the condition and then skips the remainder of the conditional structure. If none of the conditions are true, then Bash executes the actions in the `else` clause.

The following example demonstrates an `if/then/elif/then/else` statement to run the `mysql` client if the `mariadb` service is active, or to run the `psql` client if the `postgresql` service is active, or to run the `sqlite3` client if both the `mariadb` and the `postgresql` service are inactive:

```bash
[user@host ~]$ systemctl is-active mariadb > /dev/null 2>&1
[user@host ~]$ MARIADB_ACTIVE=$?
[user@host ~]$ sudo systemctl is-active postgresql > /dev/null 2>&1
[user@host ~]$ POSTGRESQL_ACTIVE=$?
[user@host ~]$ if  [[ "$MARIADB_ACTIVE" -eq 0 ]]; then \ mysql; \ elif  [[ "$POSTGRESQL_ACTIVE" -eq 0 ]]; then \ psql; \ else \ sqlite3; \ fi
```