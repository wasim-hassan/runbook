# Lab: Install a Package

Install the `chrony` package on all 3 app servers in Stratos Datacenter.

### Commands

```sh
sudo dnf install -y chrony
```

### Flag Reference
- `dnf install -y` — install a package and automatically answer "yes" to prompts
- `-y` — skip confirmation prompts (non-interactive)
- `rpm -q <package>` — query the RPM database for a specific package; returns the version if installed, or "package is not installed" if absent

### Verification

* verify the package is installed

```sh
rpm -q chrony
```

### Outputs

```
[tony@stapp01 ~]$ rpm -q chrony
chrony-4.8-1.el9.x86_64
[steve@stapp02 ~]$ rpm -q chrony
chrony-4.8-1.el9.x86_64
[banner@stapp03 ~]$ rpm -q chrony
chrony-4.8-1.el9.x86_64
```

> **Note:** `rpm -q` queries the local RPM database — it doesn't contact a repository. If the package is installed, it prints the full name-version-release. If not, it says `package chrony is not installed`. Always verify with `rpm -q`, not just by checking if the binary exists in PATH.

### Also Useful

```sh
# check package details
rpm -qi chrony                          # detailed info (version, license, description)

# list all files the package installed
rpm -ql chrony                          # shows files like /usr/sbin/chronyd, /etc/chrony.conf

# check if chronyd service is running
systemctl status chronyd

# start and enable chronyd (if needed)
sudo systemctl start chronyd
sudo systemctl enable chronyd

# install a different version
sudo dnf install -y chrony-4.5
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `No match for argument: chrony` | The package name is misspelled or not available in the enabled repositories. Uncommon for `chrony` — it's in baseos. | Double-check spelling: `chrony` not `chrony` or `crony` |
| `Error: Unable to find a match` | Same root cause — typo or repository issue. | Run `dnf search chrony` to find the exact package name |
| `Package chrony is not installed` | You're querying with `rpm -q` before installing, or the install failed silently. | Re-run `sudo dnf install -y chrony` and check for errors |
| `Job for chronyd.service failed` | Starting `chronyd` failed — likely because there's no network connectivity or NTP servers are unreachable. This doesn't affect the install requirement. The task only requires the package to be installed, not the service to be running. | Check connectivity or configure `/etc/chrony.conf` with reachable NTP servers. For this task, just verify the package is installed |
| `chronyc tracking: 506 Cannot talk to daemon` | `chronyd` isn't running. This is expected if you only installed the package and didn't start the service. The task only requires installation. | No action needed — the package is installed. `chronyc` won't work until you `sudo systemctl start chronyd` |

### Quick Context

**`dnf install` vs `rpm -ivh` vs `yum`**: `dnf` is the modern package manager on RHEL 8+ (replaced `yum`). It handles dependencies, repository metadata, and downloads. `rpm` is the lower-level tool — it only installs local `.rpm` files and can't resolve remote dependencies. `dnf` calls `rpm` internally for the actual package installation. Use `dnf` for installations and `rpm -q` for quick verification queries. On RHEL 7, replace `dnf` with `yum` (same syntax).

**What `chrony` does**: `chrony` is an NTP (Network Time Protocol) implementation for keeping system time synchronized. It replaces the older `ntpd` on modern RHEL/CentOS. `chronyd` (the daemon) syncs time to NTP servers, and `chronyc` is the command-line interface for monitoring and control. Installing `chrony` without starting `chronyd` just places the files — no time synchronization happens until the service runs.
