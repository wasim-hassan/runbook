# Lab: Group Creation and User Assignment

Create group `nautilus_sftp_users` across all App servers and add user `kano` to it.

### Commands

```sh
sudo groupadd --force nautilus_sftp_users
sudo useradd kano
sudo usermod -aG nautilus_sftp_users kano
```

Same commands repeated for the remaining two servers.

### Flag Reference
- `--force` — `groupadd` exits successfully if group already exists (idempotent)
- `-aG` — `usermod` appends user to supplementary group without removing existing groups
- `-a` — must be paired with `-G`; without it, `-G` **replaces** all supplementary groups

### Verification

```sh
groups kano
getent group nautilus_sftp_users
```

### Outputs

```
[tony@stapp01 ~]$ groups kano
kano : kano nautilus_sftp_users

[tony@stapp01 ~]$ getent group nautilus_sftp_users
nautilus_sftp_users:x:1001:kano
```

Same output from the remaining two servers.

> **Note:** `-aG` is the critical pattern — without `-a`, `-G` replaces all supplementary groups, which can silently lock a user out of `sudo` or `docker`.

### Also Useful

```sh
id kano &>/dev/null || sudo useradd kano   # idempotent user creation
sudo gpasswd -d kano nautilus_sftp_users   # remove user from group
sudo groupdel nautilus_sftp_users          # delete the group
sudo groupadd nautilus_sftp_users          # create without --force (fails if exists)
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `groups kano` shows only `kano : kano` | `usermod -aG` was never run on that particular server. Each server is independent. | `sudo usermod -aG nautilus_sftp_users kano` on the missing server |
| `getent group` shows `x:1001:` (no kano) | User exists but wasn't added to the group. Same root cause — `usermod` was skipped. | Same fix: `usermod -aG` |
| `groupadd: group already exists` | Forgot `--force`. The group exists from a previous run. | Re-run with `sudo groupadd --force nautilus_sftp_users` |
| `usermod: user 'kano' does not exist` | `useradd kano` was skipped. | `sudo useradd kano` first, then `usermod -aG` |
| `usermod: group does not exist` | `groupadd` was skipped. | `sudo groupadd nautilus_sftp_users` first |
| User lost `wheel`/`docker` access after `usermod` | Used `-G` without `-a`, which replaced all supplementary groups with just the new one. | `sudo usermod -aG wheel,docker,nautilus_sftp_users kano` to restore |

### Quick Context
**`-aG` vs `-G` — the dangerous difference**: `usermod -G group user` **replaces** all supplementary groups with just `group`. This can silently lock a user out of `sudo` (wheel) or `docker`. Always use `-aG` to append. This is one of the most common Linux admin mistakes.

**Why `--force` makes `groupadd` idempotent**: Without `--force`, `groupadd` exits with code 9 if the group exists, which stops scripts. `--force` makes it safe to run repeatedly in automation — the group is created once, and subsequent runs succeed silently.

**Primary vs supplementary groups**: Every user has exactly one primary GID from `/etc/passwd` (field 4). Supplementary groups come from `/etc/group`. `usermod -aG` only modifies supplementary groups — it never touches the primary GID.

**`getent` vs `grep`**: `getent group` queries all name sources configured in `/etc/nsswitch.conf` (local files, LDAP, SSSD, AD). `grep` only searches `/etc/group`. In environments with centralized auth, `grep` will miss remote groups. Prefer `getent` in scripts.
