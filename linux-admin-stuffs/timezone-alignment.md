# Lab: Timezone Alignment

Synchronize the timezone to `America/Lower_Princes` on all 3 app servers in Stratos Datacenter.

### Commands

On each server (stapp01, stapp02, stapp03):
```sh
# 1 — check current timezone
timedatectl status | grep "Time zone"

# 2 — verify the target timezone exists
timedatectl list-timezones | grep America/Lower_Princes

# 3 — set the timezone
sudo timedatectl set-timezone America/Lower_Princes
```

### Flag Reference
- `timedatectl status` — show current time and date settings including timezone, UTC offset, and NTP status
- `timedatectl list-timezones` — print all available timezone names (can be grepped to verify a timezone exists)
- `timedatectl set-timezone <zone>` — change the system timezone by updating `/etc/localtime` symlink

### Verification

```sh
timedatectl status | grep "Time zone"
```

### Outputs

```
[tony@stapp01 ~]$ timedatectl status | grep "Time zone"
                Time zone: America/Lower_Princes (AST, -0400)
[steve@stapp02 ~]$ timedatectl status | grep "Time zone"
                Time zone: America/Lower_Princes (AST, -0400)
[banner@stapp03 ~]$ timedatectl status | grep "Time zone"
                Time zone: America/Lower_Princes (AST, -0400)
```

> **Note:** You only need to `list-timezones | grep` once to confirm the zone name is valid — after that, `set-timezone` will either work or fail cleanly with an error if the name is wrong.

### Also Useful

```sh
# view full timedatectl output
timedatectl

# set timezone using the older /usr/share/zoneinfo symlink method
sudo ln -sf /usr/share/zoneinfo/America/Lower_Princes /etc/localtime

# check current time in a specific timezone without changing system time
TZ='America/Lower_Princes' date

# list all timezones in a region
timedatectl list-timezones | grep America/
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `Failed to set time zone: Invalid argument` | The timezone name is misspelled or doesn't exist in the zoneinfo database. Timezone names are case-sensitive and use underscore for spaces (e.g., `Lower_Princes`). | Verify with `timedatectl list-timezones \| grep -i lower` |
| `Failed to set time zone: Access denied` | `timedatectl set-timezone` requires root privileges because it modifies `/etc/localtime`. | Re-run with `sudo` |
| `Timezone America/Lower_Princes is not a valid timezone on this system` | The `tzdata` package may be outdated or the timezone was added recently. This is rare in KodeKloud labs. | Update tzdata with `sudo yum install -y tzdata` or use an alternative zone |
| `timedatectl: command not found` | The system uses an older init system (SysV) or `timedatectl` isn't installed. RHEL 6 and older don't have it. | Check OS version with `cat /etc/os-release`. Use the symlink method: `sudo ln -sf /usr/share/zoneinfo/America/Lower_Princes /etc/localtime` |

### Quick Context

**`timedatectl set-timezone` replaces the old symlink method**: Under the hood, `timedatectl set-timezone America/Lower_Princes` does exactly what `ln -sf /usr/share/zoneinfo/America/Lower_Princes /etc/localtime` does — it updates the `/etc/localtime` symlink to point to the correct zoneinfo file. The `timedatectl` command is just a friendlier interface on systemd systems. The zoneinfo files in `/usr/share/zoneinfo/` are the actual timezone database compiled from the IANA time zone database.

**UTC offset changes with DST**: `America/Lower_Princes` is `AST (Atlantic Standard Time, UTC-4)` but during Daylight Saving Time it shifts to `ADT (UTC-3)`. The system tracks this automatically based on the zoneinfo rules — you don't need to manually adjust for DST changes. `timedatectl status` shows both the timezone name and the current offset.
