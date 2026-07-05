# Lab: Create User with Custom UID and Home Directory

Create user `kareem` on App Server 1 with UID 1076 and home `/var/www/kareem`.

### Commands

```sh
sudo useradd -u 1076 -d /var/www/kareem -m kareem
```

### Flag Reference
- `-u 1076` — assign specific UID 1076
- `-d /var/www/kareem` — set custom home directory path
- `-m` — create the home directory if it doesn't exist

### Verification

```sh
grep kareem /etc/passwd
id kareem
```

### Outputs

```
[tony@stapp01 ~]$ grep kareem /etc/passwd
kareem:x:1076:1076::/var/www/kareem:/bin/bash
[tony@stapp01 ~]$ id kareem
uid=1076(kareem) gid=1076(kareem) groups=1076(kareem)
```

> **Note:** `-d` writes the path into `/etc/passwd` as metadata — without `-m`, the directory won't physically exist.

### Also Useful

```sh
sudo passwd kareem                    # set password for interactive login
sudo passwd -l kareem                 # lock password (service accounts)
sudo usermod -aG wheel kareem         # add to sudo group
ls -ld /var/www/kareem                # verify ownership and permissions
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `useradd: user 'kareem' already exists` | `useradd` only creates new users. Running it twice on the same system fails. | Verify with `grep kareem /etc/passwd` or `id kareem` |
| `Sorry, try again (sudo)` | Sudo password is wrong. KodeKloud defaults password to the username (e.g., `tony`). | Try the username as the password |
| `Permission denied (not in sudoers)` | The user you're logged in as isn't in the `wheel` group. | Switch to a privileged user or use `su -` |
| `useradd: UID 1076 is not unique` | Another user already has UID 1076. UIDs must be unique across the system. | `awk -F: '{print $3}' /etc/passwd \| sort -n` to find a free UID |
| `useradd: cannot open /etc/passwd` | Forgot `sudo` — only root can modify `/etc/passwd`. | Re-run with `sudo` |

### Quick Context
**`-d` sets metadata, `-m` creates the directory**: `-d` only writes the home path string into `/etc/passwd`. Without `-m`, no directory is created. Linux keeps user identity (`/etc/passwd`) and user files (`/home/`) as separate concerns — understanding this split explains a lot of Linux behavior.

**Why explicit UIDs matter**: By default, `useradd` picks the next available UID (1001+). Explicit UIDs are critical when files are shared over NFS — if UIDs don't match across servers, the wrong user gets access to files. This is why centralized auth (LDAP, AD) exists.

**The `/etc/passwd` database**: Format is `username:x:UID:GID:comment:home:shell`. The `x` means the real password is in `/etc/shadow` (root-only readable). UID 0 = root. System accounts (daemons) use UIDs below 1000. Regular users start at 1000.

**`useradd` vs `adduser`**: `useradd` is the low-level binary — consistent across all distros, predictable in scripts. `adduser` is a distro-specific wrapper (interactive on Debian/Ubuntu, may not exist on RHEL). Use `useradd` in automation.

