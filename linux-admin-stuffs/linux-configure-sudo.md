# Lab: Linux Configure Sudo

Configure password-less sudo access for user `kirsty` on all three app servers.

### Commands

```sh
sudo usermod -aG wheel kirsty
sudo visudo -f /etc/sudoers.d/kirsty
```

Inside `visudo`, add this single line:

```
kirsty ALL=(ALL) NOPASSWD: ALL
```

Then `ZZ` to save and exit.

### Flag Reference
- `-aG wheel` — append user to the `wheel` group without removing from other groups
- `-f /etc/sudoers.d/kirsty` — edit a specific sudoers drop-in file (visudo validates syntax on save)
- `NOPASSWD: ALL` — allow all commands without re-entering password

### Verification

Verify kirsty's sudo privileges on each server:

```sh
sudo -l -U kirsty
```

### Outputs

```
[tony@stapp01 ~]$ sudo usermod -aG wheel kirsty
[tony@stapp01 ~]$ sudo visudo -f /etc/sudoers.d/kirsty
[tony@stapp01 ~]$ sudo -l -U kirsty
Matching Defaults entries for kirsty on stapp01:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset,
    env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR
    USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT
    LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User kirsty may run the following commands on stapp01:
    (ALL) ALL
    (ALL) NOPASSWD: ALL
```

Repeat same process on stapp02 and stapp03 — same output structure.

> **Note:** `visudo -f` locks the file and validates syntax on save. A bad sudoers file can lock you out of root — always use `visudo`, never `vi` directly.

### Also Useful

```sh
sudo tee /etc/sudoers.d/kirsty <<<'kirsty ALL=(ALL) NOPASSWD: ALL'   # one-liner, no vi needed
sudo cat /etc/sudoers.d/kirsty                                        # view drop-in
sudo visudo -c                                                        # check all sudoers files for syntax
sudo -l -U kirsty                                                     # list a specific user's sudo rules
grep wheel /etc/group                                                 # check wheel group members
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `User kirsty is not allowed to run sudo on stapp03` | kirsty is not in the `wheel` group and has no sudoers drop-in yet | `sudo usermod -aG wheel kirsty` then `sudo visudo -f /etc/sudoers.d/kirsty` |
| `sudo: unable to open /etc/sudoers.d/kirsty: Permission denied` | Forgot `sudo` — `/etc/sudoers.d/` is root-owned with 440 permissions | Re-run with `sudo` |
| `>>> /etc/sudoers.d/kirsty: syntax error` | Line has a typo (e.g., misspelled `NOPASSWD`, wrong syntax) | `visudo` catches this before save — re-edit with `sudo visudo -f /etc/sudoers.d/kirsty` and fix the line |
| `sudo: /etc/sudoers.d/kirsty is world writable` | File permissions are wrong (should be 440, root-owned) | `sudo chmod 440 /etc/sudoers.d/kirsty && sudo chown root:root /etc/sudoers.d/kirsty` |
| `kirsty is not in the sudoers file. This incident will be reported.` | User tried `sudo` without being in `wheel` or having a drop-in | `sudo usermod -aG wheel kirsty` — group membership is the proper way, not editing `/etc/sudoers` directly |

### Quick Context
**`visudo` vs `vi` for sudoers files**: `visudo` locks the file to prevent concurrent edits and validates syntax before writing. A broken sudoers file is the fastest way to lose root access — always use `visudo`. Direct `vi` skips both checks.

**`wheel` group + drop-in file**: The `wheel` group gives basic sudo access (`(ALL) ALL`), which still prompts for a password. The drop-in `/etc/sudoers.d/kirsty` with `NOPASSWD` overrides that for password-less access. Both lines showing in `sudo -l` is correct (the more specific rule wins).

**Drop-in files in `/etc/sudoers.d/`**: Files in this directory are automatically included by the main `/etc/sudoers` via the `#includedir` directive. File names cannot contain `.` or `~` — `visudo` enforces this on save. Using drop-ins for individual users keeps things modular and survives OS upgrades without conflict.
