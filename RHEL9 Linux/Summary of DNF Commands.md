# Summary of DNF Commands
Packages can be located, installed, updated, and removed by name or by package groups.

| Task:                                          | Command:                      |
| :--------------------------------------------- | :---------------------------- |
| List installed and available packages by name. | `dnf list [NAME-PATTERN]`     |
| List installed and available groups.           | `dnf group list`              |
| Search for a package by keyword.               | `dnf search KEYWORD`          |
| Show details of a package.                     | `dnf info PACKAGENAME`        |
| Install a package.                             | `dnf install PACKAGENAME`     |
| Install a package group.                       | `dnf group install GROUPNAME` |
| Update all packages.                           | `dnf update`                  |
| Remove a package.                              | `dnf remove PACKAGENAME`      |
| Display transaction history.                   | `dnf history`                 |
