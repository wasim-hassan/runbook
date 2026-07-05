# Lab: Linux User Setup with Non-Interactive Shell

Create user `john` on App Server 2 with a non-interactive shell (`/sbin/nologin`).

### Commands

```sh
sudo useradd -s /sbin/nologin john
```

If the user already exists without `-s`, fix the shell on the existing user instead:

```sh
sudo usermod -s /sbin/nologin john
```

### Flag Reference
- `-s /sbin/nologin` — set shell to non-interactive (prints "This account is currently not available.")
- `-s /bin/false` — alternative non-interactive shell (silent exit, older style)

### Verification

```sh
grep john /etc/passwd
id john
```

### Outputs

```
[steve@stapp02 ~]$ sudo usermod -s /sbin/nologin john
[steve@stapp02 ~]$ grep john /etc/passwd
john:x:1001:1001::/home/john:/sbin/nologin
[steve@stapp02 ~]$ id john
uid=1001(john) gid=1001(john) groups=1001(john)
```

> **Note:** `/sbin/nologin` blocks SSH, console, and `su` — but `sudo -u john whoami` still works. It blocks interactive login sessions, not all process execution.

### Also Useful

```sh
grep john /etc/passwd | awk -F: '{print $NF}'   # show current shell
grep nologin /etc/passwd                         # list all non-login users
sudo -u john whoami                              # run command as john (still works)
sudo usermod -s /bin/bash john                   # restore login shell later
sudo userdel -r john                             # delete completely
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `useradd: user 'john' already exists` | User was already created. `useradd` can't create duplicates. | Use `usermod -s /sbin/nologin john` instead — no need to delete and recreate |
| `grep john /etc/passwd` shows `/bin/bash` | Forgot `-s` at creation time. Default shell is `/bin/bash`. | `sudo usermod -s /sbin/nologin john` — one command, no data loss |

### Quick Context
**`/sbin/nologin` vs `/bin/false`**: Both prevent interactive login. `/sbin/nologin` prints a message and exits with code 1. `/bin/false` exits silently with code 1. Prefer `/sbin/nologin` — when someone tries to `su` to the account, they get feedback instead of a confusing silent failure.

**What `-s` actually changes**: The shell field is the last column in `/etc/passwd`. It tells the system which program to run on login. Setting it to `/sbin/nologin` means "when someone tries to log in, run this program instead of bash — which immediately exits." The account itself still works for owning processes and files.

**The service account pattern**: A production service account needs three things: no home dir (`-M` or default), no interactive shell (`-s /sbin/nologin`), and a locked password (default with `useradd`). This creates an account that can own daemon processes and files but has zero login surface.

**`useradd` vs `usermod`**: `useradd` creates new users (fails if they exist). `usermod` modifies existing users. Don't delete and recreate a user to fix one field — `usermod -s` changes only the shell, leaving UID, groups, home dir, and files intact.
