# Control services as daemons
```bash
systemctl list-units --type=service --all
systemctl list-unit-files --type=service # list all service, including disabled
systemctl is-active sshd.service
systemctl is-enabled sshd.service
systemctl is-failed sshd.service
```
