# Lab: Linux Collaborative Directories

Create a collaborative directory `/dbadmin/data` on App Server 3 that is group-owned by `dbadmin` with SGID set, rwx for user and group, and no access for others.

### Commands

```sh
# 1 — verify the dbadmin group exists
getent group dbadmin

# 2 — create the directory (including parent)
sudo mkdir -p /dbadmin/data

# 3 — set group ownership
sudo chown :dbadmin /dbadmin/data

# 4 — set permissions with SGID bit
sudo chmod 2770 /dbadmin/data
```

### Flag Reference
- `mkdir -p` — create parent directories as needed (no error if they exist)
- `chown :dbadmin` — change only the group owner (colon before group name; omitting the user part leaves the user owner unchanged)
- `chmod 2770` — `2` = set SGID bit, `770` = rwx owner, rwx group, --- others
- `-p` (mkdir) — no error if the directory already exists

### Verification

```sh
ls -ld /dbadmin/data
```

### Outputs

```
[banner@stapp03 ~]$ ls -ld /dbadmin/data
drwxrws--- 2 root dbadmin 4096 Jul  8 15:22 /dbadmin/data
```

> **Note:** The `s` (instead of `x`) in the group execute position is the SGID bit. It means new files and directories created inside `/dbadmin/data` will automatically inherit the `dbadmin` group, regardless of the creating user's primary group.

### Also Useful

```sh
# set only SGID bit on an existing directory
sudo chmod g+s /dbadmin/data

# set SGID recursively for existing content
sudo find /dbadmin/data -type d -exec chmod g+s {} \;

# check the group inheritance in action
touch /dbadmin/data/test.txt && ls -l /dbadmin/data/test.txt

# remove SGID bit
sudo chmod g-s /dbadmin/data

# add a user to the dbadmin group
sudo usermod -aG dbadmin banner
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `chmod: invalid mode: '2770'` | Typo in the mode string — `2770` must be a 4-digit octal number with no spaces. Common mistake: `2770` vs `2 770` or `2,770`. | Use exact format: `sudo chmod 2770 /dbadmin/data` |
| `chown: invalid group: ':dbadmin'` | The group `dbadmin` doesn't exist on the system. A colon before a group name tells `chown` to change only the group — but the group must already exist. | Create it first: `sudo groupadd --system dbadmin` |
| `mkdir: cannot create directory '/dbadmin/data': Permission denied` | Forgot `sudo` — `/dbadmin/` is in the root filesystem and requires root to create directories there. | Re-run with `sudo mkdir -p /dbadmin/data` |
| New files inside the directory don't inherit the `dbadmin` group | The SGID bit wasn't set on the directory. Without `g+s` or the octal `2` prefix, new files get the creating user's primary group, not the directory's group. | Re-run with the SGID bit: `sudo chmod 2770 /dbadmin/data` or `sudo chmod g+s /dbadmin/data` |

### Quick Context

**SGID on directories — files inherit the group**: The SGID (Set Group ID) bit on a directory (`chmod g+s` or octal prefix `2`) means any file or subdirectory created inside it automatically gets the directory's group owner, not the creator's primary group. This is essential for collaborative directories where multiple users (all in `dbadmin`) need to share files. Without SGID, user A's files would be owned by user A's primary group, and user B might not have access.

**`2770` vs `770`**: A 4-digit octal mode like `2770` breaks down as: `2` (special bits: SGID), `7` (owner: rwx), `7` (group: rwx), `0` (others: ---). A 3-digit mode like `770` has no special bit prefix — SGID is not set. The leading digit can be `0` (no special bits), `1` (sticky), `2` (SGID), `4` (SUID), or combinations like `6` (SUID+SGID).
