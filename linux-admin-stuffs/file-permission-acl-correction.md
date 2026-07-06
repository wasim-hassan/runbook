# Lab: File Permission Correction

Set owner/group to root, others read-only, user anita should have no access, and garrett should have read-only on `/etc/hostname` on App Server 3.

### Commands

```sh
# 1 — check existing ACLs
getfacl /etc/hostname

# 2 — set anita to no permissions
sudo setfacl -m u:anita:--- /etc/hostname

# 3 — set garrett to read-only
sudo setfacl -m u:garrett:r-- /etc/hostname
```

### Flag Reference
- `-m` — modify the ACL on the file
- `u:username:perms` — user-specific ACL entry; `perms` can be `---`, `r--`, `rw-`, `rwx`

### Verification

Verify final ACLs
```sh
getfacl /etc/hostname
```

### Outputs

```
[banner@stapp03 ~]$ getfacl /etc/hostname
getfacl: Removing leading '/' from absolute path names
# file: etc/hostname
# owner: root
# group: root
user::rw-
user:anita:---
user:garrett:r--
group::r--
mask::r--
other::r--
```

> **Note:** The `mask::r--` entry automatically narrowed to `r--` because garrett's read-only ACL restricted the maximum effective permissions for named users and groups. This is normal ACL behavior.

### Also Useful

```sh
# remove a specific user's ACL entry
sudo setfacl -x u:anita /etc/hostname

# remove all ACL entries (back to basic Unix perms)
sudo setfacl -b /etc/hostname

# set default ACL for new files in a directory
sudo setfacl -m d:u:garrett:r-x /some/dir

# recursive ACL apply on a directory
sudo setfacl -R -m u:garrett:r-x /some/dir
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `setfacl: /etc/hostname: Operation not permitted` | Forgot `sudo` — only root can modify ACLs on files owned by root. | Re-run with `sudo setfacl` |
| `setfacl: Option -m: Invalid argument near character 3` | Typo in permission string — `r--` is valid, `-r` or `r` alone isn't. ACL permissions must be exactly 3 characters (e.g., `r--`, `rw-`, `---`). | Use exact 3-char permission strings |
| `getfacl: Removing leading '/' from absolute path names` | Not an error — `getfacl` strips `/` by default to show relative paths, preventing accidental root path confusion. | Normal behavior, no action needed |

### Quick Context

**ACLs extend Unix permissions — they don't replace them**: Standard Unix permissions (owner/group/others) still apply and show as the first entries in `getfacl` output (`user::`, `group::`, `other::`). ACL entries like `user:anita:---` are additional rules layered on top. The `mask` entry controls the maximum effective permissions for all named users and groups — it automatically adjusts when you add new ACL entries.

**`setfacl -m u:anita:---` vs removing the user entirely**: Setting `---` explicitly denies anita access — the entry stays in the ACL as a denial rule. If you used `setfacl -x u:anita`, the entry would be removed entirely, and anita would fall back to the `other::r--` permission (gaining read access). For explicit denial + no fallback, set `---` instead of removing.
