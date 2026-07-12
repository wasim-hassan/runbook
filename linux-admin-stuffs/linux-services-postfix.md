# Lab: Linux Services

Install the `postfix` package and enable it to start on boot on all 3 app servers.

### Commands

```sh
# 1 — install postfix
sudo dnf install -y postfix

# 2 — enable postfix to start on boot
sudo systemctl enable postfix
```

### Flag Reference
- `sudo dnf install -y` — install a package and auto-confirm prompts
- `rpm -q <package>` — query the RPM database; returns version if installed
- `sudo systemctl enable <service>` — create a symlink so the service starts automatically on boot (does not start it immediately)

### Verification

```sh
rpm -q postfix
systemctl is-enabled postfix
```

### Outputs

```
[tony@stapp01 ~]$ rpm -q postfix
postfix-3.5.25-3.el9.x86_64
[tony@stapp01 ~]$ sudo systemctl enable postfix
Created symlink /etc/systemd/system/multi-user.target.wants/postfix.service → /usr/lib/systemd/system/postfix.service.

[steve@stapp02 ~]$ rpm -q postfix
postfix-3.5.25-3.el9.x86_64
[steve@stapp02 ~]$ sudo systemctl enable postfix
Created symlink /etc/systemd/system/multi-user.target.wants/postfix.service → /usr/lib/systemd/system/postfix.service.

[banner@stapp03 ~]$ rpm -q postfix
postfix-3.5.25-3.el9.x86_64
[banner@stapp03 ~]$ sudo systemctl enable postfix
Created symlink /etc/systemd/system/multi-user.target.wants/postfix.service → /usr/lib/systemd/system/postfix.service.
```

> **Note:** `systemctl enable` does not start the service — it only creates the `multi-user.target.wants/` symlink so systemd launches it on boot. To start it immediately, use `sudo systemctl start postfix`.

### Also Useful

```sh
# start the service now (without waiting for reboot)
sudo systemctl start postfix

# check if the service is running
systemctl status postfix

# disable postfix (remove from boot)
sudo systemctl disable postfix

# check all enabled services at boot
systemctl list-unit-files --type=service | grep enabled

# view postfix logs
sudo journalctl -u postfix -n 20
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `dnf install -y` prompts for password | `sudo` isn't being used. Only root can install system packages. | Prepend `sudo`: `sudo dnf install -y postfix` |
| `rpm -q postfix: package postfix is not installed` | The install failed or you're querying before the install completes. | Check the install output for errors, then re-run `sudo dnf install -y postfix` |
| `systemctl enable: Unit postfix.service not found` | postfix is not installed. The unit file ships with the postfix package, so it won't exist until after install. | Install postfix first: `sudo dnf install -y postfix` |
| `Failed to enable unit: Unit file postfix.service is already enabled` | The service was already enabled (possibly by a dependency or earlier step). Not an error — the symlink already exists. | No action needed — verify with `systemctl is-enabled postfix` |

### Quick Context

**`systemctl enable` vs `systemctl start`**: `enable` creates a symlink in `/etc/systemd/system/multi-user.target.wants/` pointing to the service unit file in `/usr/lib/systemd/system/`. This tells systemd to launch the service automatically at boot time. `start` launches the service immediately — regardless of whether it's enabled or not. A service can be: enabled + started (runs now and on boot), enabled + stopped (starts on next boot but not now), disabled + started (runs now but won't survive a reboot), or disabled + stopped (off completely).

**The `multi-user.target.wants/` directory system**: systemd doesn't use init scripts or rc.d directories. Instead, it uses "wants" directories — when systemd boots into `multi-user.target`, it reads `/etc/systemd/system/multi-user.target.wants/` and starts every service whose symlink is there. `systemctl enable postfix` simply creates that symlink. `systemctl disable` removes it.
