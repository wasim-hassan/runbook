# Lab: Secure Root SSH Access

Disable direct SSH root login on all 3 app servers in Stratos Datacenter.

### Commands

On each server (stapp01, stapp02, stapp03):

```sh
# 1 — find the line number of PermitRootLogin
sudo cat -n /etc/ssh/sshd_config | grep PermitRootLogin

# 2 — set PermitRootLogin to no
sudo vi /etc/ssh/sshd_config
# or use sed for a non-interactive approach:
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# 3 — restart SSH daemon to apply
sudo systemctl restart sshd
```

### Flag Reference

- `-i` (sed) — edit file in-place
- `s/pattern/replacement/` (sed) — substitute first match on each line
- `^#\?` (regex) — optionally match a leading `#` (handles both commented and uncommented lines)
- `-T` (sshd) — extended test mode; prints the effective runtime config instead of starting the daemon

### Verification

```sh
sudo cat -n /etc/ssh/sshd_config | grep PermitRootLogin
sudo sshd -T | grep permitrootlogin
```

### Outputs

Same outputs on each servers
```
[tony@stapp01 ~]$ sudo sshd -T | grep permitrootlogin
permitrootlogin no
[tony@stapp01 ~]$ sudo cat -n /etc/ssh/sshd_config | grep PermitRootLogin
    40  #PermitRootLogin prohibit-password
    90  # the setting of "PermitRootLogin without-password".
   131  PermitRootLogin no
```

> **Note:** `sudo sshd -T` reads the actual config as the daemon sees it — more reliable than grepping the config file, because it accounts for `Include` directives and default values.

### Also Useful

```sh
# check current SSH port
sudo sshd -T | grep port

# test root login attempt without leaving a shell session
ssh root@localhost

# find which users match the DenyUsers / AllowUsers rules
sudo sshd -T | grep -i "allowusers\|denyusers\|allowgroups\|denygroups"

# dry-run config validation (parses config without applying)
sudo sshd -t
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `cat: /etc/ssh/sshd_config: Permission denied` | Non-root users can't read `/etc/ssh/sshd_config` — it's owned by root with restrictive permissions. | Prepend `sudo` to the command |
| Edited `/etc/ssh/ssh_config` instead of `/etc/ssh/sshd_config` | The two files are easy to confuse — `ssh_config` is the *client* config, `sshd_config` is the *daemon* (server) config. The daemon ignores `ssh_config`. | Edit `/etc/ssh/sshd_config` instead, then restart `sshd` |
| `sudo: /etc/ssh/ssh_config: command not found` | Accidentally tried to pipe a file path into `| wc` as if it were a command. Files are read with `cat`, `wc <`, or redirects — `\|` pipes command output, not files. | Use `sudo wc -l < /etc/ssh/ssh_config` or `sudo cat /etc/ssh/ssh_config \| wc -l` |

### Quick Context

**`ssh_config` vs `sshd_config`**: The single-letter difference (`d` for daemon) is easy to miss. `ssh_config` controls client-side settings (your outgoing SSH connections). `sshd_config` controls server-side settings (incoming SSH connections). If you edit the wrong one, the server won't care. Always double-check you're editing `sshd_config` when securing root login.

**`sudo sshd -T` verifies runtime config, not just file content**: Grepping the file shows you the raw text, but `Include /etc/ssh/sshd_config.d/*.conf` can pull in overrides from other files. `sshd -T` reads all included files and shows you the effective configuration the daemon actually uses — the only authoritative check.

**Disabling root SSH vs other auth methods**: `PermitRootLogin no` blocks *all* root SSH login methods — password, public key, keyboard-interactive. For a more nuanced approach, `prohibit-password` (the RHEL default) allows root login with keys but not passwords. In production, always use key-based auth for non-root users and `sudo` elevation — root should never SSH directly.
