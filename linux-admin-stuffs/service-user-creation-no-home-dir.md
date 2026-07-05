# Lab: Service User Creation without Home Directory

Create user `rose` on App Server 3 without a home directory.

### Commands

```sh
sudo useradd -M rose
```

### Flag Reference
- `-M` — skip home directory creation (service accounts)
- `-m` — force home directory creation (normal users)

### Verification

```sh
grep rose /etc/passwd
ls -ld /home/rose
```

### Outputs

```
[banner@stapp03 ~]$ sudo useradd -M rose
[banner@stapp03 ~]$ grep rose /etc/passwd
rose:x:1001:1001::/home/rose:/bin/bash
[banner@stapp03 ~]$ ls -ld /home/rose
ls: cannot access '/home/rose': No such file or directory
```

> **Note:** `rose` shows `/home/rose` in `/etc/passwd` but that's just metadata — verify the directory itself with `ls`.

### Also Useful

```sh
sudo useradd -m jane                    # force home dir creation
sudo useradd -m -d /var/www/app rose    # custom home path
sudo useradd -M -s /sbin/nologin rose   # locked service account
sudo userdel -r rose                    # delete user + files
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `ls: cannot access '/home/rose': No such file or directory` | `/etc/passwd` stores a home path field regardless of whether the directory exists. This output confirms no directory was physically created. | Nothing to fix. This is how you verify `-M` worked. |
| `/home/rose` exists despite `-M` | `-M` only prevents *creation* of a new home directory — it does not delete an existing one. The dir may already exist from a previous `useradd` run, or your distro's `/etc/login.defs` defaults to `CREATE_HOME yes`. | `sudo rm -rf /home/rose` to clean up. Check defaults: `grep -i create_home /etc/login.defs`. |
| `useradd: user 'rose' already exists` | `useradd` only creates new users. Running it twice on the same system fails. | Use `usermod` to modify the existing account, or `userdel rose` first to start fresh. |

### Quick Context
**passwd is not a directory listing**: `/home/rose` in `/etc/passwd` is a metadata field — like a phone contact storing a "home address" even if the house doesn't exist. Linux keeps user identity (`/etc/passwd`) and user data (`/home/`) as separate concerns. That's why `grep` shows the path but `ls` confirms nothing is there.

**Why service accounts skip home dirs**: Daemons like nginx or MySQL need a passwd entry to own processes but never log in interactively. A home directory is just filesystem clutter for them. Combine `-M` with `-s /sbin/nologin` to create an account that owns processes and files but has no login capability — a standard security pattern for reducing attack surface.

**`/etc/login.defs` exists because distros disagree**: Ubuntu defaults to `CREATE_HOME yes`. A hardened RHEL server might not. Rather than hardcoding this behavior, Linux puts the decision in a config file so admins set policy once instead of remembering flags. Check it when `-M` doesn't behave as expected.

**`/etc/skel` is the new-user template**: When `-m` creates a home directory, files from `/etc/skel/` (`.bashrc`, `.profile`) are automatically copied in. No home dir means no skel copy — another reason service accounts stay lean.
