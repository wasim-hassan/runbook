# Lab: Process Limit Adjustment

Set soft and hard process limits for user `nfsuser` on App Server 1 to limit excessive processes.

### Commands

```sh
# 1 — check current process limit for nfsuser
sudo -u nfsuser ulimit -u

# 2 — set soft (warning) limit to 1024
echo "nfsuser soft nproc 1024" | sudo tee -a /etc/security/limits.conf

# 3 — set hard (absolute) limit to 2024
echo "nfsuser hard nproc 2024" | sudo tee -a /etc/security/limits.conf
```

### Flag Reference
- `ulimit -u` — show the maximum number of user processes for the current user
- `soft nproc 1024` — soft limit: a warning threshold the user can exceed until hitting the hard limit
- `hard nproc 2024` — hard limit: the absolute ceiling that cannot be exceeded by any user (even with `ulimit -p`)
- `sudo tee -a` — append to a root-owned file while echoing what was written

### Verification

```sh
grep nfsuser /etc/security/limits.conf
```

### Outputs

```
[tony@stapp01 ~]$ grep nfsuser /etc/security/limits.conf
nfsuser soft nproc 1024
nfsuser hard nproc 2024
```

> **Note:** Limits set in `/etc/security/limits.conf` take effect on **new login sessions** only — existing processes of the user keep their original limits. If you need immediate effect, the user must log out and back in.

### Also Useful

```sh
# check the limits from a running shell
ulimit -a                             # all current limits for your shell

# verify as nfsuser (su into the user)
sudo -u nfsuser sh -c 'ulimit -u'    # soft limit
sudo -u nfsuser sh -c 'ulimit -H -u' # hard limit

# set limits using a dedicated conf file under limits.d
echo "nfsuser soft nproc 1024" | sudo tee /etc/security/limits.d/99-nfsuser.conf
echo "nfsuser hard nproc 2024" | sudo tee -a /etc/security/limits.d/99-nfsuser.conf

# check PAM is configured to use limits.conf
grep pam_limits.so /etc/pam.d/*      # should show common-session or system-auth
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `sudo: -u nfsuser ulimit -u: command not found` | `ulimit` is a shell built-in, not a standalone command. `sudo -u` tries to run it as an external binary, which doesn't exist. | Use `sudo -u nfsuser sh -c 'ulimit -u'` to run it inside a shell |
| `sudo tee -a: Permission denied` | Forgot to pipe through `sudo` — `echo "..." >> /etc/security/limits.conf` runs the redirect as your user, not root. | Use `echo "..." \| sudo tee -a /etc/security/limits.conf` |
| `nproc: No such file or directory` / limits not applying on RHEL 8+ | RHEL 8+ moved process limits to systemd's user slice. `/etc/security/limits.conf` is handled by PAM, but systemd may override it for user sessions. | Also set via systemd: `sudo mkdir -p /etc/systemd/system/user-.slice.d/` and create a drop-in with `TasksMax=2024` |
| `ulimit -u` still shows old value after setting limits.conf | The user hasn't started a new session. Limits apply to new PAM-authenticated logins (SSH, su, login), not to existing shells. | Log out of the user and log back in, or `su - nfsuser` for a fresh login shell |

### Quick Context

**Soft vs hard limits in `/etc/security/limits.conf`**: The soft limit (`1024`) is the initial cap — if a process hits it, the kernel sends a warning but allows exceeding it up to the hard limit. The hard limit (`2024`) is the absolute maximum — the user can raise their soft limit up to this value using `ulimit -p`, but can't exceed the hard limit even with `sudo`. For process limits (`nproc`), hitting the hard limit means `fork()` calls fail with `EAGAIN`.

**PAM-based vs systemd-based limits**: On RHEL 7+, process limits can be set via two systems — PAM (`/etc/security/limits.conf` via `pam_limits.so`) and systemd (`UserTasksMax` in slice config). For SSH logins and interactive shells, PAM handles the limits. For systemd-managed services and user sessions, systemd's `TasksMax` may take precedence. In these labs, PAM-based limits via `limits.conf` are the expected configuration.
