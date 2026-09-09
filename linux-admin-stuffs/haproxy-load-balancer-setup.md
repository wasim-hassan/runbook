# Lab: Install & Configure HAProxy Load Balancer

Install HAProxy on the LBR server (stlb01), add all three app servers to its load balancer backend, serve on the default http port 80, and keep the stock `stats socket /var/lib/haproxy/stats` line intact. Apache runs on **port 8087** on the app servers.

## How It Works

**HAProxy is the front door.** All traffic hits the load balancer on port 80 (`frontend`), and HAProxy forwards each request to one of the healthy backend servers (`backend app`). With `balance roundrobin`, it cycles through stapp01 → 02 → 03 so no single app server gets overloaded. The `check` keyword makes HAProxy skip any backend that's down, so a crashed app server doesn't break the site.

**Default config is a starting point, not the answer:** the stock `haproxy.cfg` binds to `*:5000` and points at fake `127.0.0.1:5001-5004` servers. You must point the frontend at the real http port (80) and replace the placeholder backend servers with your real app servers + correct port.

**Two critical, easy-to-miss details in this task:**
1. **Apache's port is 8087**, not 80 — the backend `server` lines must use `:8087`. Using 80 means the load balancer forwards nowhere.
2. **Keep the `stats socket /var/lib/haproxy/stats` line** exactly as-is (task explicitly says not to remove it).

### Commands

```sh
# 1. Install via yum only
sudo yum install haproxy -y

# 2. Resolve app server IPs (dynamic per environment — always look them up)
getent hosts stapp01 stapp02 stapp03

# 3. Edit the config
sudo vi /etc/haproxy/haproxy.cfg
#   - frontend main:  bind *:5000  -->  bind *:80
#   - backend app:    replace placeholder servers with the 3 real app servers on :8087
#   - keep "stats socket /var/lib/haproxy/stats" untouched
#   - delete any leftover placeholder lines (e.g. app4)

# 4. Validate, start, enable
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

### Flag Reference
* `bind *:80` — frontend listens on all interfaces, port 80 (the default http port)
* `server stapp01 <IP>:8087 check` — backend server entry; `check` makes HAProxy health-check it and route around it if down
* `balance roundrobin` — distribute requests in rotation across the backend servers
* `getent hosts <names>` — resolve hostnames to IPs (respects `/etc/hosts` and DNS)
* `haproxy -c -f <file>` — validate the config file and exit, without starting the daemon

### Verification

Confirm the config validates, the service is running + enabled, the stats socket survives, and the site is reachable through the load balancer.

```sh
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # "Configuration file is valid"
sudo systemctl status haproxy                 # active (running), enabled
curl http://localhost:80                      # "Welcome to xFusionCorp Industries!"
ls -l /var/lib/haproxy/stats                  # stats socket still present
```

### Outputs

```
[loki@stlb01 ~]$ sudo haproxy -c -f /etc/haproxy/haproxy.cfg
Configuration file is valid

[loki@stlb01 ~]$ sudo systemctl enable haproxy
Created symlink /etc/systemd/system/multi-user.target.wants/haproxy.service → /usr/lib/systemd/system/haproxy.service.

[loki@stlb01 ~]$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; preset: disabled)
     Active: active (running) since Wed 2026-09-09 06:45:12 UTC; 27s ago
   Main PID: 54013 (haproxy)
     Status: "Ready."

[loki@stlb01 ~]$ curl http://localhost:80
Welcome to xFusionCorp Industries!

[loki@stlb01 ~]$ ls -l /var/lib/haproxy/stats
srwxr-xr-x 1 root root 0 Sep  9 06:45 /var/lib/haproxy/stats
```

> **Note:** The `[ALERT] backend 'static' has no server available` line in the logs is from the **stock** `backend static` section (it points at a fake `127.0.0.1:4331`). It's unrelated to the app backend you configured and doesn't fail the task.

### Also Useful

```sh
sudo systemctl restart haproxy                      # apply config changes
sudo journalctl -u haproxy -f                       # follow HAProxy logs live
sudo ss -tlnp | grep 80                             # confirm port 80 is LISTENing
echo "show servers state" | sudo nc -U /var/lib/haproxy/stats   # query backend health via stats socket
sudo killall haproxy                                # force stop (if normal stop hangs)
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `curl http://localhost:80` returns nothing / connection refused | HAProxy not running, not enabled, or bound to the wrong port (stock config binds `*:5000`) | Set `bind *:80`, then `sudo systemctl restart && enable haproxy` |
| Backend servers still named `app1/app2/app3` pointing to `127.0.0.1` | Only the port was changed — `127.0.0.1` is the LBR server itself, and the names stayed as placeholders | Replace with real hostnames + real app IPs: `server stapp01 <IP>:8087 check` |
| Leftover `server app4 127.0.0.1:5004 check` under backend app | Editing with vi left one of the four stock lines behind | Delete the stray line (`dd` on it in vi) so backend has only the 3 real servers |
| Site loads but all traffic seems to hit one server | Backend all pointing to the same IP (e.g. all `127.0.0.1`) | Give each `server` line a distinct app IP |
| `Network is unreachable` / servers never go UP | Wrong port in backend — app servers run Apache on **8087**, not 80 | Use `:8087` in every `server` line |
| `haproxy -c` reports a parsing/`unknown server` error | A corrupt or leftover line in the config | `sudo cat -n /etc/haproxy/haproxy.cfg` around `backend app` and remove/fix the bad line |
| `[1]+ Stopped` after `systemctl status` | Pressed Ctrl+Z while inside the pager | Press `q` to exit the pager instead |

### Quick Context
**Frontend vs backend**: the `frontend` block is where clients connect (the published address/port); the `backend` block is the list of real servers the traffic gets sent to. `default_backend app` wires the frontend to the backend. Change the binding to match how clients reach you, and the backend server list to match where your apps actually live.

**`127.0.0.1` means "this machine"**: in a load balancer's backend it's almost always wrong — it makes HAProxy forward to itself. Real app servers need their real IPs. That mistake (plus leftover placeholder lines) is the most common reason "it validates but nothing works."

**`check` is what makes balancing safe**: without it, HAProxy blindly sends traffic to dead servers. With it, a failed health check takes a server out of rotation automatically — that's what makes HAProxy an actual load *balancer* instead of a dumb forwarder.
