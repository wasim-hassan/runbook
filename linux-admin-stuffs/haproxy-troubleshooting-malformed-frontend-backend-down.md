# Lab: HAProxy Troubleshooting — Malformed Frontend + Backends Down

HAProxy would not start on stlb01 and the site `curl http://stlb01:80/` was unreachable. Three injected faults in `/etc/haproxy/haproxy.cfg` — a malformed `frontend main *:80` line, a commented-out `#backend app` block, and a wrong backend port (`:5002`) that left all app servers DOWN.

## How It Works

**HAProxy config is section-based.** Each block (`global`, `defaults`, `frontend`, `backend`) must start with a correct header line followed by indented directives. A single broken header cascades: when `frontend` had `*:80` glued to it and `backend app` was commented out, the parser couldn't attach any following lines to a valid block — so every error trace pointed at the same two spots.

**One bad line = the whole service never starts.** HAProxy validates the whole config at startup and exits with code 1 on any fatal parse error. That's why `systemctl status` showed `failed (Result: exit-code)` — not a crash, a config that refused to load.

**503 is not the same as "down".** After fixing the parse errors, haproxy started but still returned `503 Service Unavailable` with `Server app/app1 is DOWN, reason: Layer4 connection error`. The frontend was healthy and answering — but every backend was unreachable because the config pointed at a closed port (`:5002`). The `Layer4` wording is the clue: a TCP connection to the backend failed, i.e. wrong port/address, not an app-level problem.

### Commands

```sh
# 1. Diagnose — the sequence that exposes every fault
systemctl status haproxy            # failed (Result: exit-code), nothing on port 80
ss -tlnp | grep 80                  # only *:3306 (MySQL) — nothing on 80
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # pinpoints exact line numbers
sudo cat -n /etc/haproxy/haproxy.cfg          # see the faults visually
sudo journalctl -u haproxy -n 50              # startup exit-code proof

# 2. Find the real backend port when backends are DOWN (recon before config edits)
timeout 2 bash -c 'echo > /dev/tcp/stapp01/8087' && echo "8087 OPEN" || echo "8087 closed"
timeout 2 bash -c 'echo > /dev/tcp/stapp01/8080' && echo "8080 OPEN" || echo "8080 closed"

# 3. Fix the config with vi
sudo vi /etc/haproxy/haproxy.cfg
#   - line:  frontend  main *:80   -->  frontend  main  +  bind *:80 (two lines)
#   - line:  #backend app          -->  backend app (remove the #)
#   - lines: server appX stappXX:5002 check  -->  :8080 (real app port)

# 4. Apply + verify
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
curl http://stlb01:80/
```

### Flag Reference
* `haproxy -c -f <file>` — check mode: parse the config and exit (no daemon start). Every validation step in this task starts here
* `systemctl start` vs `systemctl restart` — start runs stopped units; restart re-runs a running one after config edits
* `timeout 2 bash -c 'echo > /dev/tcp/HOST/PORT'` — quick TCP port probe using bash's `/dev/tcp`; timeout kills it after 2s if it hangs
* `ss -tlnp` — list TCP listeners: `-t` TCP, `-l` listening, `-n` numeric, `-p` process
* `bind *:80` — frontend listens on all interfaces, port 80

### Verification

Confirm the config parses, the service runs, and the site loads through the load balancer.

```sh
sudo haproxy -c -f /etc/haproxy/haproxy.cfg     # "Configuration file is valid"
systemctl status haproxy                        # active (running), enabled
curl http://stlb01:80/                          # "Welcome to xFusionCorp Industries!"
```

### Outputs

```
[loki@stlb01 ~]$ sudo haproxy -c -f /etc/haproxy/haproxy.cfg
[NOTICE]   (3672) : haproxy version is 2.8.14-c23fe91
[ALERT]    (3672) : config : parsing [/etc/haproxy/haproxy.cfg:39] : please use the 'bind' keyword for listening addresses.
[ALERT]    (3672) : config : parsing [/etc/haproxy/haproxy.cfg:40] : 'listen' or 'defaults' expected.
[ALERT]    (3672) : config : parsing [/etc/haproxy/haproxy.cfg:41] : 'listen' or 'defaults' expected.
[ALERT]    (3672) : config : parsing [/etc/haproxy/haproxy.cfg:60]: Missing LF on last line, file might have been truncated at position 36.
[ALERT]    (3672) : config : Fatal errors found in configuration.

[loki@stlb01 ~]$ timeout 2 bash -c 'echo > /dev/tcp/stapp01/8080' && echo "8080 OPEN" || echo "8080 closed"
8080 OPEN

[loki@stlb01 ~]$ sudo systemctl restart haproxy
[loki@stlb01 ~]$ curl http://stlb01:80/
Welcome to xFusionCorp Industries![loki@stlb01 ~]$
```

> **Note:** The `Server app/app1 is DOWN, reason: Layer4 connection error` lines in the logs after the first restart were the tell — a healthy frontend with unreachable backends means the backend address/port in the config is wrong.

### Also Useful

```sh
getent hosts stapp01 stapp02 stapp03           # resolve backend IPs (dynamic per env)
ss -tlnp | grep 80                              # confirm haproxy is LISTENing on :80
sudo journalctl -u haproxy -f                   # follow logs after restart
for p in 80 8080 8087 5002; do timeout 1 bash -c "echo > /dev/tcp/stapp01/$p" 2>/dev/null && echo "$p OPEN" || echo "$p closed"; done   # probe multiple ports at once
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `please use the 'bind' keyword for listening addresses` | `frontend main *:80` glued the bind address onto the section header — `bind` must be its own line | Split into `frontend main` + `bind *:80` |
| `'listen' or 'defaults' expected` (multiple) | `#backend app` was commented out, so the following `balance`/`server` lines had no valid block header | Remove the `#` so `backend app` is a real header |
| `Missing LF on last line, file might have been truncated` | File edited/saved without a trailing newline on the last line | Re-save with a final newline (vi `o` then `dd` adds one) |
| `systemctl status haproxy` → `failed (Result: exit-code)` | Config parse errors make haproxy exit code 1 at startup — it never reaches running | Fix the parse errors, then `haproxy -c` must return valid before starting |
| `curl http://stlb01:80/` → `503 Service Unavailable` + backends `DOWN` / `Layer4 connection error` | Frontend up, but backend servers point at a closed port (`:5002`); TCP connect to the app fails | Probe the real port (`8080` open here) and update every `server ... check` line |
| `ss -tlnp | grep 80` shows only `*:3306` | Nothing is listening on 80 — the MySQL match is just the digit "80" inside `3306` | Not a fault; wait until haproxy starts and re-check |
| `[1]+ Stopped` after `systemctl status` | Pressed Ctrl+Z inside the pager | Press `q` to exit a pager |

### Quick Context
**One malformed line kills the whole daemon**: haproxy parses `haproxy.cfg` in one pass at startup and aborts (exit 1) on the first fatal error. A glued `frontend main *:80` or a `#backend app` comment isn't "mostly working" — the service flat-out never starts. That's why `haproxy -c` is step one: it turns a startup mystery into exact line numbers in under a second.

**503 ≠ dead service**: once the frontend binds, haproxy answers on port 80 even with zero healthy backends — it just returns HTTP 503. The `Layer4 connection error` in the logs means the TCP handshake to a backend failed (bad port/address), which is why the `/dev/tcp` port probe is the fastest way to find the real app port before you touch the config.

**Section headers are load-bearing**: in HAProxy, `global`, `defaults`, `frontend`, and `backend` headers define what the indented lines below mean. Comment out a header and every line under it becomes an orphan the parser rejects. When you see repeated `'listen' or 'defaults' expected` errors, look for a block header that got commented or mangled.
