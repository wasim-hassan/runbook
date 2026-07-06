# Lab: SELinux Installation and Configuration

Install SELinux packages and permanently disable SELinux on App Server 1 (no reboot).

### Commands

```sh
# 1 — check if SELinux policy package is installed
rpm -q selinux-policy-targeted

# 2 — install if not present
sudo dnf install -y selinux-policy-targeted

# 3 — find the line number of SELINUX= to edit
cat -n /etc/selinux/config | grep -i SELINUX

# 4 — edit SELinux config to disable permanently
sudo vi /etc/selinux/config
```

Inside vi:
1. Press `22G` to jump to line 22 (`SELINUX=enforcing`)
2. Press `W` to move cursor to the start of `enforcing`
3. Press `cW` to delete the word and enter insert mode
4. Type `disabled`
5. Press `Esc` to return to normal mode
6. Type `:wq` and press `Enter` to save and quit

### Flag Reference
- `22G` (vi) — jump to line 22
- `W` (vi) — move forward one WORD
- `cW` (vi) — change (delete + insert) from cursor to end of WORD
- `:wq` (vi) — write changes and quit

### Verification

```sh
cat -n /etc/selinux/config | grep -i SELINUX
grep ^SELINUX= /etc/selinux/config
```

### Outputs

```
[tony@stapp01 ~]$ grep ^SELINUX= /etc/selinux/config
SELINUX=disabled
```

> **Note:** The task says to disregard the current runtime status (`getenforce` / `sestatus`). Only the config file change matters — SELinux will be disabled after the next boot.

### Also Useful

```sh
# non-interactive method using sed (avoids vi)
sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config

# disable SELinux immediately (runtime, without reboot)
sudo setenforce 0                         # switches to permissive mode

# check current SELinux runtime status
getenforce                                # shows Enforcing / Permissive / Disabled
sestatus                                  # detailed SELinux status

# view SELinux config with line numbers
cat -n /etc/selinux/config | grep -i selinux
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `No match for argument: SELinux` | `SELinux` is not a package name. The correct package is `selinux-policy-targeted`. | Use `sudo dnf install -y selinux-policy-targeted` |
| `sed: -e expression #1, char 14: unterminated 's' command` | Missing closing delimiter or the regex `/` wasn't properly escaped. The pattern must be `s/old/new/`. | Verify the sed command syntax: `sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config` |
| `ssh: WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` | The server's host key changed (e.g., after a lab reset) and the old key is cached in `known_hosts`. The test runner fails because SSH strict checking rejects the connection. | Run `ssh-keygen -R stapp01` on the jump host to remove the stale key, then reconnect |
| `Transaction check error: file /usr/share/licenses/selinux-policy from install... conflicts` | `selinux-policy-targeted` is already installed, or there's a conflict with another SELinux package. | Check current packages with `rpm -qa \| grep selinux-policy` |

### Quick Context

**`SELINUX=disabled` vs `setenforce 0`**: `setenforce 0` switches to permissive mode immediately (runtime only) — it logs policy violations but doesn't enforce them. This is useful for debugging. `SELINUX=disabled` in the config file requires a reboot to take full effect — it stops SELinux from loading at all, including the SELinux filesystem and kernel security hooks. The task requires `disabled` (permanent, post-reboot) and explicitly says to disregard runtime status.

**Why the package is `selinux-policy-targeted` and not `selinux-policy`**: The SELinux policy comes in two flavors — `targeted` (default, protects specific daemons) and `mls` (Multi-Level Security, protects everything with strict classifications). `targeted` is the standard for most servers. The base `selinux-policy` package is pulled in as a dependency; `selinux-policy-targeted` is what provides the actual policy rules.
