# Manage Files from the Command Line

## Command-line Expansions
- _Brace expansion_, which can generate multiple strings of characters
- _Tilde expansion_, which expand to a path to a user home directory
- _Variable expansion_, which replaces text with the value that is stored in a shell variable
- _Command substitution_, which replaces text with the output of a command
- _Pathname expansion_, which helps to select one or more files by pattern matching

Pathname expansion, historically called _globbing_, is one of the most useful features of Bash. 

## Pathname Expansion and Pattern Matching
**Table 3.2. Table of Metacharacters and Matches**

|Pattern|Matches|
|:--|:--|
|*|Any string of zero or more characters|
|?|Any single character|
|[_abc…​_]|Any one character in the enclosed class (between the square brackets)|
|[!_abc…​_]|Any one character _not_ in the enclosed class|
|[^_abc…​_]|Any one character _not_ in the enclosed class|
|[[:alpha:]]|Any alphabetic character|
|[[:lower:]]|Any lowercase character|
|[[:upper:]]|Any uppercase character|
|[[:alnum:]]|Any alphabetic character or digit|
|[[:punct:]]|Any printable character that is not a space or alphanumeric|
|[[:digit:]]|Any single digit from 0 to 9|
|[[:space:]]|Any single white space character, which might include tabs, newlines, carriage returns, form feeds, or spaces|
## Brace Expansion
Brace expansion is used to generate discretionary strings of characters. Braces contain a comma-separated list of strings, or a sequence expression. The result includes the text that precedes or follows the brace definition. Brace expansions might be nested, one inside another. You can also use double-dot syntax (..), which expands to a sequence. For example, the `{m..p}` double-dot syntax inside braces expands to `m n o p`.
```
[user@host glob]$ echo {Sunday,Monday,Tuesday,Wednesday}.log
Sunday.log Monday.log Tuesday.log Wednesday.log
[user@host glob]$ echo file{1..3}.txt
file1.txt file2.txt file3.txt
[user@host glob]$ echo file{a..c}.txt
filea.txt fileb.txt filec.txt
[user@host glob]$ echo file{a,b}{1,2}.txt
filea1.txt filea2.txt fileb1.txt fileb2.txt
[user@host glob]$ echo file{a{1,2},b,c}.txt
filea1.txt filea2.txt fileb.txt filec.txt
```
A practical use of brace expansion is to create multiple files or directories.
```
[user@host glob]$ mkdir ../RHEL{7,8,9}
[user@host glob]$ ls ../RHEL*
RHEL7 RHEL8 RHEL9
```
## Tilde Expansion
The tilde character (~), matches the current user's home directory. 
```
[user@host glob]$ echo ~user
/home/user
[user@host glob]$ echo ~/glob
/home/user/glob
[user@host glob]$ `echo ~nonexistinguser
~nonexistinguser
```
## Variable Expansion
A variable acts like a named container that stores a value in memory. 

You can assign data as a value to a variable with the following syntax:
```
[user@host ~]$ VARIABLENAME=value
```

You can use variable expansion to convert the variable name to its value on the command line.
```
[user@host ~]$ USERNAME=operator
[user@host ~]$ echo $USERNAME
operator
```

o prevent mistakes due to other shell expansions, you can put the name of the variable in curly braces, for example `${VARIABLENAME}`.
```
[user@host ~]$ USERNAME=operator
[user@host ~]$ echo ${USERNAME}
operator
```
Variable names can contain only letters (uppercase and lowercase), numbers, and underscores. Variable names are case-sensitive and cannot start with a number.

## Command Substitution
- Command substitution enables the output of a command to replace the command itself on the command line. 
- Command substitution occurs when you enclose a command in parentheses and precede it by a dollar sign (`$`). The ``$(command)`` form can nest multiple command expansions inside each other.
```
[user@host glob]$ echo Today is $(date +%A).
Today is Wednesday.
[user@host glob]$ echo The time is $(date +%M) minutes past $(date +%l%p).
The time is 26 minutes past 11AM.
```
## Protecting Arguments from Expansion

The backslash (`\`) is an escape character in the Bash shell. It protects the following character from expansion.
```
[user@host glob]$ echo The value of $HOME is your home directory.
The value of /home/user is your home directory.

[user@host glob]$ echo The value of \$HOME is your home directory.
The value of $HOME is your home directory.
```
In the preceding example, with the dollar sign protected from expansion, Bash treats it as a regular character, without variable expansion on `$HOME`.

To protect longer character strings, you can use single quotation marks (`'`) or double quotation marks (`"`) to enclose strings. 
- Single quotation marks stop all shell expansion.
- Double quotation marks stop _most_ shell expansion.

Double quotation marks suppress special characters other than the dollar sign (`$`), backslash (`\`), backtick (\`) , and exclamation point (`!`) from operating inside the quoted text. Double quotation marks block pathname expansion, but still allow command substitution and variable expansion to occur.
```
[user@host glob]$ myhost=$(hostname -s); echo $myhost
host
[user@host glob]$ echo "***** hostname is ${myhost} *****"
***** hostname is host *****
```

Use single quotation marks to interpret _all_ text between the quotes literally.
```
[user@host glob]$ echo "Will variable $myhost evaluate to $(hostname -s)?"
Will variable host evaluate to host?

[user@host glob]$ echo 'Will variable $myhost evaluate to $(hostname -s)?'
Will variable $myhost evaluate to $(hostname -s)?
```
