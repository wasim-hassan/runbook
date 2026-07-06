# Lab: Linux User Data Transfer

Find all files owned by user `ravi` under `/home/usersdata` and copy them preserving the directory structure to `/ecommerce`.

### Commands

```sh
# On stapp02 as user steve

# 1 — locate files owned by ravi
sudo find /home/usersdata -type f -user ravi

# 2 — copy them preserving the full directory path under /ecommerce
sudo find /home/usersdata -type f -user ravi -exec cp --parents {} /ecommerce \;
```

### Flag Reference
- `-type f` — match only regular files, not directories
- `-user ravi` — match files owned by user `ravi`
- `-exec ... {} \;` — run the `cp` command once per matched file; `{}` is replaced by the file path
- `--parents` — preserve the source directory structure under the destination (e.g., `/home/usersdata/wp-load.php` becomes `/ecommerce/home/usersdata/wp-load.php`)

### Verification

```sh
sudo find /home/usersdata -type f -user ravi | wc -l
sudo find /ecommerce/home/usersdata -type f | wc -l
```

### Outputs

```
[steve@stapp02 ~]$ sudo find /home/usersdata -type f -user ravi | wc -l
1880
[steve@stapp02 ~]$ sudo find /ecommerce/home/usersdata -type f | wc -l
1880
```

### Also Useful

```sh
# dry-run — just list what would be copied (no actual copy)
sudo find /home/usersdata -type f -user ravi > /tmp/ravi-files.txt

# copy using cpio (alternative method, faster for many files)
sudo find /home/usersdata -type f -user ravi | cpio -pdm /ecommerce

# find files NOT owned by ravi
sudo find /home/usersdata -type f ! -user ravi
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `ls: cannot open directory '.': Permission denied` | You `sudo su`'d to user `ravi` but stayed in `/home/steve` — ravi has no read permission on steve's home dir. | Stay as your sudo-user (steve) and use `sudo find` instead of switching users |
| `find: '/home/usersdata/...': Permission denied` | Ran `find` without `sudo` — steve doesn't have permission to traverse all directories under `/home/usersdata`. | Prepend `sudo` to the find command |
| `cp: cannot create regular file '/ecommerce/...': No such file or directory` | The `/ecommerce` directory doesn't exist yet. | `sudo mkdir -p /ecommerce` first, or run the find+cp with `sudo` so it can create intermediate dirs |

### Quick Context

**`cp --parents` keeps the full source path**: Without `--parents`, `cp` would flatten all files into a single directory. With `--parents`, it recreates the entire source path under the destination — so `/home/usersdata/wp-load.php` lands at `/ecommerce/home/usersdata/wp-load.php`. This is how you maintain directory structure during selective file transfers.

**`sudo su` vs `sudo command`**: `sudo su ravi` drops you into an interactive shell as ravi — but you inherit the environment (like working directory) from the calling user. `sudo find ...` runs one command as root and returns you to your normal shell. For one-off commands or scripts, `sudo <command>` is cleaner and avoids permission surprises.
