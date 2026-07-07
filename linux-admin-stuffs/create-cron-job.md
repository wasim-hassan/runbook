# Lab: Create a Cron Job

Install cronie, start crond, and add a root cron job on all 3 app servers.

### Commands

On each server (stapp01, stapp02, stapp03):
```sh
# 1 — install cronie (if not already installed)
sudo dnf install -y cronie

# 2 — start crond service
sudo systemctl start crond

# 3 — add cron job for root
sudo crontab -e
```

Inside the crontab editor, paste this line:
```
*/5 * * * * echo hello > /tmp/cron_text
```

### Flag Reference
- `*/5` — run every 5 minutes (`*` = every unit, `/5` = step of 5)
- `* * * *` — every hour, every day of month, every month, every day of week
- `sudo crontab -e` — edit root's crontab (creates one if it doesn't exist)
- `sudo crontab -l` — list root's current cron entries

### Verification

```sh
sudo crontab -l
```

### Outputs

```
[tony@stapp01 ~]$ sudo crontab -l
*/5 * * * * echo hello > /tmp/cron_text
[steve@stapp02 ~]$ sudo crontab -l
*/5 * * * * echo hello > /tmp/cron_text
[banner@stapp03 ~]$ sudo crontab -l
*/5 * * * * echo hello > /tmp/cron_text
```

> **Note:** The cron syntax follows the order `minute hour day month weekday command`. `*/5` in the minute field means "every 5 minutes." The `echo hello > /tmp/cron_text` command appends "hello" to a file every 5 minutes — useful for testing cron is working.

### Also Useful

```sh
# check if cron job actually ran
cat /tmp/cron_text

# wait 5+ minutes and check again — each run appends "hello"
# if file doesn't exist yet, cron hasn't triggered on schedule yet

# view cron logs
sudo journalctl -u crond -n 20

# list cron jobs for other users
sudo crontab -u username -l

# start crond on boot (enable)
sudo systemctl enable crond
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `sudo: crontab: command not found` | `cronie` is not installed. `crontab` is part of the cronie package. | Install it with `sudo dnf install -y cronie` |
| `Failed to start crond.service: Unit crond.service not found` | Same root cause — `cronie` is not installed. The `crond.service` unit file ships with the cronie package. | `sudo dnf install -y cronie` (it creates the systemd unit and also enables it) |
| `$ sudo crontab -e: command not found` (after install) | The crontab binary may not be in the current shell's PATH. | Log out and back in, or use full path `sudo /usr/bin/crontab -e` |
| `crontab: no crontab for root` (but cron entry exists) | This appears when running `crontab -l` as a non-root user for a user with no crontab. If you used `sudo crontab -l`, it should show the entry. | Use `sudo crontab -l` to view root's crontab (not just `crontab -l`) |
| `Errors in crontab file, can't install.` | Syntax error — missed a field, wrong delimiter, or invalid value. Cron expects exactly 5 time fields followed by a command. | Double-check the line: `*/5 * * * * echo hello > /tmp/cron_text` (5 fields + command — verify no extra/missing spaces) |

### Quick Context

**Cron time fields — `minute hour day month weekday`**: The first field is `minute` (0-59), not the typical `* * * * *` mental shortcut people use. `*/5` in the minute field = every 5 minutes. A common mistake is putting `* * * * *` (every minute) when you meant `0 * * * *` (every hour at minute 0). The 5 fields are always: minute, hour, day of month, month, day of week.

**`crontab -e` vs `crontab -l`**: `-e` opens the user's crontab file in an editor (usually vi) — the system stores it in `/var/spool/cron/username`. `-l` prints its contents. For root, `sudo crontab -e` opens `/var/spool/cron/root`. Never edit these spool files directly with a text editor — always use `crontab -e`, which validates syntax before saving.
