# Lab: Linux Firewalld Setup

Install and configure firewalld on App Server 3 — allow Nginx (port 80), block Apache (port 8085), with permanent rules in the public zone.

### Commands

```sh
# Fix port conflict: Apache was on port 80, blocking Nginx
sudo vi /etc/httpd/conf/httpd.conf
# Change "Listen 80" to "Listen 8085"

# Restart Apache and start Nginx
sudo systemctl restart httpd
sudo systemctl start nginx
sudo systemctl enable httpd nginx

# Install, start, enable firewalld
sudo yum install -y firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld

# Apply firewall rules
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" port port="8085" protocol="tcp" drop' --permanent
sudo firewall-cmd --reload
```

### Flag Reference
- `--zone=public` — apply rule to the public zone (default, least-trusted)
- `--add-port=80/tcp` — allow incoming TCP traffic on port 80
- `--add-rich-rule=...` — add an advanced rule with action/family/port conditions
- `--permanent` — persist across reloads and reboots (without this, rules disappear on reload)
- `--reload` — apply runtime changes without restarting the firewalld service

### Verification

```sh
sudo firewall-cmd --zone=public --list-all
sudo systemctl status httpd nginx --no-pager -l
```

### Outputs

```
[banner@stapp03 ~]$ sudo vi /etc/httpd/conf/httpd.conf
[banner@stapp03 ~]$ sudo systemctl restart httpd && sudo systemctl start nginx
[banner@stapp03 ~]$ sudo systemctl enable httpd nginx
Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.

[banner@stapp03 ~]$ sudo firewall-cmd --zone=public --list-all
public
  target: default
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: cockpit dhcpv6-client ssh
  ports: 80/tcp
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules: 
        rule family="ipv4" port port="8085" protocol="tcp" drop

[banner@stapp03 ~]$ sudo systemctl status httpd nginx --no-pager -l
● httpd.service - The Apache HTTP Server
     Loaded: loaded; enabled
     Active: active (running)
       Docs: man:httpd.service(8)
   Main PID: 11458 (httpd)
     Status: "Server configured, listening on: port 8085"

● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded; enabled
     Active: active (running)
   Main PID: 11771 (nginx)
```

> **Note:** If you hit a port conflict, check it with `sudo lsof -i :<port>` before digging through config files — it's faster than grepping logs.

### Also Useful

```sh
sudo lsof -i :80                             # find what's listening on port 80
sudo ss -tlnp                                # list all listening TCP ports (needs iproute)
sudo firewall-cmd --list-all-zones           # view rules for all zones
sudo firewall-cmd --runtime-to-permanent     # save all runtime rules as permanent
sudo firewall-cmd --zone=public --remove-port=80/tcp --permanent   # remove a rule
sudo firewall-cmd --zone=public --add-service=http --permanent     # allow by service name
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `nginx: bind() to 0.0.0.0:80 failed (98: Address already in use)` | Apache (httpd) is already listening on port 80. Nginx can't bind to an occupied port. | Change Apache's `Listen 80` to `Listen 8085` in `/etc/httpd/conf/httpd.conf` and restart httpd |
| `Authorization failed. Make sure polkit agent is running or run the application as superuser.` | `firewall-cmd` was run without `sudo`. It needs root to talk to firewalld via D-Bus. | Prepend `sudo` — e.g., `sudo firewall-cmd --add-port=80/tcp --permanent` |
| `sudo: ss: command not found` | `iproute` package isn't installed on this minimal image | Use `sudo lsof -i :80` instead (needs `lsof`), or `cat /proc/net/tcp` |
| `Job for nginx.service failed because the control process exited with error code` | A process is already listening on the port Nginx needs, or the config has a syntax error | `sudo systemctl status nginx` to check, `sudo lsof -i :80` to find the port conflict, `sudo nginx -t` to test config |
| Firewall rule doesn't persist after reboot | `--permanent` was omitted, so the rule was runtime-only | Re-add with `--permanent` and `--reload`, or use `--runtime-to-permanent` to save all current runtime rules |

### Quick Context
**`firewall-cmd` without `sudo`**: `firewall-cmd` communicates with firewalld over D-Bus (a desktop IPC bus). On servers without a polkit agent, D-Bus auth fails and you get `Authorization failed`. Adding `sudo` makes the call as root, bypassing the polkit check entirely.

**Rich rules vs simple `--add-port`**: Simple `--add-port=8085/tcp` allows the port. To block (drop) it, you need a rich rule with `drop` action. Without it, port 8085 is implicitly blocked by the public zone's `default` target — but an explicit rich rule makes the intent clear and survives port additions elsewhere.

**Port conflicts are a config issue, not a process issue**: Apache and Nginx both default to port 80. The fix is changing the config file (`Listen` directive for Apache, `listen` in nginx.conf for Nginx) then restarting the service. Killing the process without changing config just brings it right back.
