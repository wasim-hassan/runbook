# Lab: Linux DNS Troubleshooting

Add Google public DNS nameservers on App Server 2 as a temporary fix for intermittent DNS resolution issues.

### Commands

```sh
sudo vi /etc/resolv.conf
```

Inside `vi`, add these lines at the end of the file:

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

### Flag Reference
- `nameserver` — specifies a DNS resolver IP address; the resolver queries them in order (top to bottom)
- `search` — domain suffixes appended to single-label hostnames before querying DNS
- `options ndots:5` — only append search domains if the query has fewer than 5 dots (default is 1)

### Verification

Check the config file, then verify resolution using available tools:

```sh
cat /etc/resolv.conf
curl -s google.com | head
getent hosts google.com
```

### Outputs

```
[steve@stapp02 ~]$ cat /etc/resolv.conf
search rrrby2yptuvb62ij.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
nameserver 8.8.8.8
nameserver 8.8.4.4

[steve@stapp02 ~]$ curl -s google.com | head
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>

[steve@stapp02 ~]$ getent hosts google.com
2a00:1450:4026:801::200e google.com
```

> **Note:** This server doesn't have `ping`, `nslookup`, `dig`, or `host` installed. `curl` and `getent` are the easiest fallbacks to verify DNS is working.

### Also Useful

```sh
resolvectl status                           # check systemd-resolved status if in use
cat /etc/resolv.conf                        # view current DNS config
ping -c 2 google.com                        # verify DNS + network (needs iputils)
nslookup google.com                         # DNS lookup (needs bind-utils)
dig google.com                              # detailed DNS query (needs bind-utils)
host google.com                             # simple DNS lookup (needs bind-utils)
getent hosts google.com                     # libc resolver lookup (no extra packages)
curl -sI google.com                         # HTTP check without downloading body
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `-bash: nslookup: command not found` | `bind-utils` package is not installed | Use `curl` or `getent` instead, or `sudo yum install -y bind-utils` |
| `-bash: dig: command not found` | `bind-utils` package is not installed | Same as above |
| `-bash: ping: command not found` | `iputils` package is not installed or container has no `CAP_NET_RAW` | Use `curl -s google.com` or `getent hosts google.com` instead |
| `Sorry, try again (sudo)` | Typo'd the sudo password | KodeKloud default: password is the username (e.g., `steve`) |
| `/etc/resolv.conf` resets after reboot | File is managed by NetworkManager or systemd-resolved — it regenerates on restart | Use `nmcli` or `resolvectl` to make the change persistent instead of editing the file directly |

### Quick Context
**`/etc/resolv.conf` — the glue that isn't meant to be hand-edited**: This file tells the system's resolver library (`libc`) which DNS servers to query. On modern systems, NetworkManager or systemd-resolved regenerate it automatically — any manual edit gets blown away on reboot/restart. That's why the task calls this a "temporary fix."

**Nameserver ordering matters**: The resolver tries nameservers top to bottom and only moves to the next one if the first times out or returns an error. If `10.96.0.10` (internal K8s DNS) responds fast, Google DNS never gets queried. That's fine here — Google DNS is the fallback for when internal DNS is having "intermittent issues."

**`getent hosts` vs `dig`/`nslookup`**: `getent hosts` uses the system resolver (`/etc/resolv.conf`) and respects `/etc/nsswitch.conf`, so it reflects exactly what apps on the server will see. `dig` and `nslookup` bypass `/etc/resolv.conf` and query DNS directly — useful for debugging, but not representative of what applications experience.
