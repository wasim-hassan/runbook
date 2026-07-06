# Lab: Default GUI Boot Configuration

Change the default systemd target from CLI (`multi-user.target`) to GUI (`graphical.target`) on all 3 app servers without rebooting.

### Commands

On each server (stapp01, stapp02, stapp03):
```sh
# 1 — check current default target
sudo systemctl get-default

# 2 — set to GUI boot
sudo systemctl set-default graphical.target
```

### Flag Reference
- `systemctl get-default` — print the current default target (e.g., `multi-user.target`)
- `systemctl set-default <target>` — change the default boot target by updating the `/etc/systemd/system/default.target` symlink

### Verification

```sh
sudo systemctl get-default
```

### Outputs

```
[tony@stapp01 ~]$ sudo systemctl get-default
graphical.target
[steve@stapp02 ~]$ sudo systemctl get-default
graphical.target
[banner@stapp03 ~]$ sudo systemctl get-default
graphical.target
```

> **Note:** `systemctl set-default` only changes the symlink in `/etc/systemd/system/` — it doesn't reboot or change the current session. The new target takes effect on next boot only.

### Also Useful

```sh
# switch to GUI right now (without reboot)
sudo systemctl isolate graphical.target

# switch back to CLI mode
sudo systemctl set-default multi-user.target

# list all available targets
systemctl list-units --type=target --all

# check if graphical target has all dependencies available
systemctl list-dependencies graphical.target
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `Failed to create symlink /etc/systemd/system/default.target: File exists` | The default target symlink already exists. This could mean `set-default` failed or you ran it twice. Run `sudo systemctl get-default` to check the current setting | If already set to `graphical.target`, no action needed. Otherwise, remove the old target and re-run: `sudo systemctl set-default graphical.target` |
| `sudo: systemctl: command not found` | You're not on a systemd-based system (e.g., older SysV init). This is unlikely in KodeKloud labs. | Verify the OS with `cat /etc/os-release` — RHEL 7+ / CentOS 7+ use systemd |
| `Access denied` when running `systemctl` | Forgot `sudo` — only root can change system-wide default targets. | Re-run with `sudo` |
| `Failed to read PID: Inappropriate ioctl for device` | Usually appears in containers — not applicable to these VMs. | Ignore it — the command still works |

### Quick Context

**`multi-user.target` vs `graphical.target`**: These are systemd's equivalent of the old SysV runlevels. `multi-user.target` (runlevel 3) boots to a terminal/CLI — no GUI. `graphical.target` (runlevel 5) boots the same plus a display manager (like GDM, LightDM) for a desktop environment. In server environments, `multi-user.target` is standard — GUI components just waste resources. Changing to `graphical.target` is useful for desktops or when you need a browser/GUI tool on a server temporarily.

**Why no reboot is needed**: `set-default` only changes a symlink at `/etc/systemd/system/default.target` pointing to the chosen target unit file. This symlink is read by systemd **during boot** to decide what to start. Changing it now has zero effect on the current running session. To switch modes live, you'd use `systemctl isolate graphical.target` instead.
