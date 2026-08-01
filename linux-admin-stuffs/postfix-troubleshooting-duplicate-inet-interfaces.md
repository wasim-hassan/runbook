# Lab: Postfix Troubleshooting — Duplicate inet_interfaces

Postfix on stmail01 silently stopped serving the monitoring app. Root cause: `main.cf` had **two `inet_interfaces` entries** — an earlier `inet_interfaces = all` and a leftover stock `inet_interfaces = localhost` at line 135. Postfix reads the file top-to-bottom and the **last entry wins**, so it bound to `localhost` only and became unreachable on the network.

## How It Works

**Why "the service seems to fail" when it's actually configured wrong:** postfix starts fine, but binds to the loopback interface only (`inet_interfaces = localhost`). The monitoring app connects over the network, gets nothing, and reports the mail server as down. Same symptom as a crash — very different root cause. This is why you diagnose before restarting: restarting a healthy-but-misconfigured postfix just gives you the same broken behavior again.

**Why the last `inet_interfaces` wins:** postfix reads `main.cf` line-by-line, applying directives in order. The last value for any parameter overwrites earlier ones — there's no merge, no error, just a warning during `postfix check`. So `all` at line ~130 followed by `localhost` at line 135 means "localhost". The warning (`overriding earlier entry`) is the breadcrumb that leads you to the duplicate.

### Commands

```sh
# 1. Get on the mail server (NOT jump-host)
ssh groot@stmail01

# 2. Diagnose — confirm it's dead and why
systemctl status postfix                    # inactive (dead), disabled
ss -tlnp | grep 25                          # nothing listening on SMTP
sudo postfix check                          # THE key command — flags the duplicate line
sudo postconf -n inet_interfaces            # shows effective value (localhost = the bug)

# 3. Fix — comment out the stock default line (line 135) that overrides "all"
sudo sed -i '135s/^/#/' /etc/postfix/main.cf

# 4. Start + enable on boot
sudo systemctl start postfix
sudo systemctl enable postfix
```

### Flag Reference
* `postfix check` — validate `main.cf`; prints warnings for every config problem, like duplicate parameters ("overriding earlier entry")
* `postconf -n` — show the **effective** (final) value of a parameter after all directives are applied — use `-n` to see the resolved value, not just the file
* `sed -i '135s/^/#/'` — edit file in place: on line 135, replace start-of-line (`^`) with `#`, turning the line into a comment. `-i` = in-place edit
* `ss -tlnp | grep 25` — list TCP listening sockets (`-t` TCP, `-l` listening, `-n` numeric, `-p` process) filtered to port 25
* `systemctl enable` — create the boot-time symlink so the service starts on reboot

### Verification

Confirm the config warning is gone, the effective value is right, the port is open, and the service stays up.

```sh
sudo postfix check                 # should print nothing (no warnings)
sudo postconf -n inet_interfaces   # should show: inet_interfaces = all
ss -tlnp | grep 25                 # postfix should be LISTENing on 0.0.0.0:25 and [::]:25
systemctl status postfix           # active (running), enabled
```

### Outputs

```
[groot@stmail01 ~]$ sudo sed -i '135s/^/#/' /etc/postfix/main.cf
[groot@stmail01 ~]$ sudo systemctl start postfix
[groot@stmail01 ~]$ sudo systemctl enable postfix
Created symlink /etc/systemd/system/multi-user.target.wants/postfix.service → /usr/lib/systemd/system/postfix.service.

[groot@stmail01 ~]$ sudo postfix check
[groot@stmail01 ~]$ sudo postconf -n inet_interfaces
inet_interfaces = all

[groot@stmail01 ~]$ ss -tlnp | grep 25
LISTEN 0      100          0.0.0.0:25        0.0.0.0:*
LISTEN 0      100             [::]:25           [::]:*

[groot@stmail01 ~]$ systemctl status postfix
● postfix.service - Postfix Mail Transport Agent
     Loaded: loaded (/usr/lib/systemd/system/postfix.service; enabled; preset: disabled)
     Active: active (running) since Sat 2026-08-01 17:28:54 UTC; 1min 32s ago
   Main PID: 29387 (master)
      Tasks: 3 (limit: 404712)
     Memory: 3.8M
        CPU: 288ms
     CGroup: /system.slice/postfix.service
             ├─29387 /usr/libexec/postfix/master -w
             ├─29388 pickup -l -t unix -u
             └─29389 qmgr -l -t unix -u
```

> **Note:** Before the fix, `postfix check` printed `warning: /etc/postfix/main.cf, line 135: overriding earlier entry: inet_interfaces=all` repeatedly (once per sub-command). After commenting the line, it's silent.

### Also Useful

```sh
grep -n inet_interfaces /etc/postfix/main.cf   # find ALL occurrences + line numbers (dup detection)
sudo postconf -n | grep -E 'inet_interfaces|mydomain|mydestination'  # sanity-check effective mail settings
sudo journalctl -u postfix -n 50               # service logs (empty here — service never started)
sudo systemctl restart postfix                 # restart after config edits
sudo systemctl is-enabled postfix              # confirms boot-time enablement
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `postfix check` → `line 135: overriding earlier entry: inet_interfaces=all` | `main.cf` contains two `inet_interfaces` directives; postfix applies them in order and the **last one wins** — the earlier `all` was silently overwritten by the leftover stock `localhost` | Comment out the duplicate line that should not win (`sudo sed -i '135s/^/#/'`) |
| `systemctl status postfix` → `inactive (dead)` | Service was stopped and not enabled; combined with the config bug it never served mail | `sudo systemctl start postfix && sudo systemctl enable postfix` |
| `ss -tlnp | grep 25` shows nothing | Two causes: service not running, and/or bound to `localhost` only so nothing shows on the external interface | Start the service AND fix `inet_interfaces` |
| `journalctl -u postfix` → `-- No entries --` | Service had never successfully started, so systemd had nothing to log | Not a separate problem — follows from the two issues above; logs appear after a clean start |
| `postfix check` still warns after editing | The wrong duplicate was commented out, or the line number shifted | `grep -n inet_interfaces /etc/postfix/main.cf` and comment the remaining unwanted entry |

### Quick Context
**Last entry wins**: postfix applies `main.cf` top-to-bottom and later directives override earlier ones — no merge, no error. That's why one stale `localhost` line near the end silently neutralized an `all` placed earlier. The warning postfix prints is a clue, not a crash.

**"Service seems to fail" ≠ service is down**: here the service could start fine, but loopback-only binding made it invisible to the network. Symptom-first troubleshooting (restart everything) wouldn't have fixed it — you have to read the config and the `postfix check` output to find the real cause.

**Diagnose before restart**: `postfix check` + `postconf -n` turn a config problem into two readable lines before you touch anything. On a config-driven daemon, "just restart it" treats the symptom.
