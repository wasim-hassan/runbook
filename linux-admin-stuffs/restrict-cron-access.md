# Lab: Restrict Cron Access

Configure crontab access on App Server 3 — allow user `mark` and deny user `rod`.

### Commands

```sh
# 1 — check existing cron access control files
ls -l /etc/cron.allow /etc/cron.deny 2>/dev/null

# 2 — add mark to the allow list
echo "mark" | sudo tee -a /etc/cron.allow

# 3 — deny rod by adding to the deny list
echo "rod" | sudo tee -a /etc/cron.deny
```

### Flag Reference
- `tee -a` — write stdin to a file, appending (`-a`) instead of overwriting
- `2>/dev/null` — redirect stderr to /dev/null (suppresses "file not found" when checking files that may not exist)

### Verification

```sh
sudo cat /etc/cron.allow
sudo cat /etc/cron.deny
```

### Outputs

```
[banner@stapp03 ~]$ sudo cat /etc/cron.allow
mark
[banner@stapp03 ~]$ sudo cat /etc/cron.deny
rod
```

> **Note:** Only one of `cron.allow` or `cron.deny` is needed in most cases — `cron.allow` is a whitelist, `cron.deny` is a blacklist. If `cron.allow` exists, only listed users get access (and `cron.deny` is ignored for them). Having both is fine but the allow list is what actually gates access.

### Also Useful

```sh
# test as mark (should work)
sudo -u mark crontab -l

# test as rod (should fail)
sudo -u rod crontab -l

# remove a user from cron.allow
sudo sed -i '/^mark$/d' /etc/cron.allow

# check cron access rules for current user
crontab -l
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `You (rod) are not allowed to use this program (crontab)` | `rod` is in `cron.deny` — this is the expected denial behavior, not an error. The task requires this. | No fix needed — this is the correct outcome |
| `crontab: no crontab for mark` | `mark` is allowed but hasn't created any cron jobs yet. This is normal for a user with no crontab configured. | No fix needed — this confirms mark has access, just no jobs yet |
| `tee: /etc/cron.allow: Permission denied` | Forgot `sudo` — `cron.allow` and `cron.deny` are root-owned files in `/etc/`. | Re-run with `sudo tee` |
| `echo "mark" > /etc/cron.allow` overwrites instead of appending | Using `>` (redirect) overwrites the entire file. If `cron.allow` already had users, they'd be lost. | Use `>>` or `tee -a` to append instead |

### Quick Context

**`cron.allow` vs `cron.deny` — whitelist vs blacklist**: If `cron.allow` exists, **only** users listed there can use `crontab` — everyone else is implicitly denied, regardless of `cron.deny`. If only `cron.deny` exists, everyone **except** those listed gets access. If neither exists, access depends on the distro (RHEL usually allows all, Debian usually allows all). This means: to restrict access to a single user, create `cron.allow` with just that username — you don't even need `cron.deny`.

**Why `tee` instead of `echo > file`**: `echo "mark" | sudo tee /etc/cron.allow` solves the permission problem cleanly — the `echo` runs as your user (no sudo needed), and `tee` receives the sudo elevation to write the root-owned file. `tee -a` appends; `tee` without `-a` overwrites.
