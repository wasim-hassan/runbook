# Lab: Firewall Configuration

Install and enable firewalld on App Server 2, then allow incoming traffic on port 3004/tcp in the public zone.

### Commands

```sh
# 1 — install firewalld
sudo dnf in -y firewalld

# 2 — start firewalld
sudo systemctl start firewalld

# 3 — allow port 3004/tcp in the public zone (permanent)
sudo firewall-cmd --zone=public --add-port=3004/tcp --permanent

# 4 — reload to apply permanent rules
sudo firewall-cmd --reload
```

### Flag Reference
- `--zone=public` — target the public zone (the default zone for external-facing interfaces)
- `--add-port=3004/tcp` — open TCP port 3004 for incoming traffic
- `--permanent` — persist the rule across reboots (without this, the rule is lost on reload/reboot)
- `--reload` — reload firewall rules from permanent config without restarting the service

### Verification

```sh
sudo firewall-cmd --zone=public --list-ports
sudo systemctl status firewalld
```

### Outputs

```
[steve@stapp02 ~]$ sudo firewall-cmd --zone=public --list-ports
3004/tcp
[steve@stapp02 ~]$ sudo systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-07-06 04:22:53 UTC; 37s ago
       Docs: man:firewalld(1)
   Main PID: 33666 (firewalld)
      Tasks: 2 (limit: 409892)
     Memory: 24.6M
        CPU: 578ms
     CGroup: /system.slice/firewalld.service
             └─33666 /usr/bin/python3 -s /usr/sbin/firewalld --nofork --nopid
```

> **Note:** `--permanent` alone doesn't apply the rule immediately — it only writes it to the config files. Use `--reload` to activate permanent changes, or use both `--add-port` (runtime, no `--permanent`) and `--add-port --permanent` to apply now + persist.

### Also Useful

```sh
# add a port at runtime only (lost on reload/reboot)
sudo firewall-cmd --zone=public --add-port=3004/tcp

# remove a port rule
sudo firewall-cmd --zone=public --remove-port=3004/tcp --permanent

# list all zones and their rules
sudo firewall-cmd --list-all-zones

# check which zone an interface belongs to
sudo firewall-cmd --get-zone-of-interface=eth0

# allow a service instead of raw port
sudo firewall-cmd --zone=public --add-service=http --permanent
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `sudo: firewall-cmd: command not found` | firewalld is not installed. `firewall-cmd` is the CLI tool that ships with the firewalld package. | Install it with `sudo dnf install -y firewalld` |
| `ERROR: ZONE_CONFLICT` | The port is already defined in another zone with different settings. firewalld doesn't allow the same port in multiple zones. | Check which zone already has it: `sudo firewall-cmd --list-all-zones \| grep 3004`, then remove from the conflicting zone |
| `FirewallD is not running` without `--permanent` | You tried to add a runtime rule but the service isn't started. Runtime changes require a running firewalld. | `sudo systemctl start firewalld` first, or use `--permanent` + `--reload` (which works even if firewalld is stopped) |
| `WARNING: COMMAND_FAILED` | The port 3004 is already open in the zone. This happens when re-running the same command. | Not an error — just a warning. The rule is already in place. |

### Quick Context

**Runtime vs permanent rules in firewalld**: firewalld has two rule sets — runtime (in memory, applies immediately) and permanent (in config files, applies after reload/reboot). `--add-port` without `--permanent` only adds to runtime. `--add-port --permanent` writes to config files. The workflow: add with `--permanent`, then `--reload` to apply both permanently and immediately. Or add once without `--permanent` for immediate effect, then again with `--permanent` to persist.

**Zones in firewalld**: Zones are named security levels that define trust for network connections. `public` is the default — it assumes the network is untrusted and only allows explicitly opened ports. Other common zones: `trusted` (open everything), `internal` (more trusting than public), `dmz` (limited access). Each network interface is assigned to exactly one zone.
