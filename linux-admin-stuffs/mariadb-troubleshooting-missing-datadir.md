# Lab: MariaDB Troubleshooting — Missing Datadir (Service Down)

the Nautilus application could not connect to the database because `mariadb` was down on the database server `stdb01`; the `/var/lib/mysql` data directory had never been initialized. Fixed by running `mariadb-install-db` to create the datadir and starting + enabling the service.

## How It Works

**A database server needs three things to start: the package, the service user, and a data directory (`datadir`).** The `datadir` holds the system tables, sockets, and all databases. When `/var/lib/mysql` is missing, `mariadbd` cannot even get to the "open tables" stage — it aborts before writing anything to the error log. That's why `journalctl -u mariadb` showed `-- No entries --`: the service started and died *silently*.

**`mariadb-install-db` is the initializer, not a repair tool.** It creates `/var/lib/mysql`, fills it with the system tables, and is what the systemd pre-start script (`mariadb-prepare-db-dir`) normally runs on first boot. If the dir is gone, you run it manually — with `--user=mysql` so every file ends up owned by the `mysql` service account.

**Two different "can't connect" errors.** `ERROR 2002 ... (2)` means the socket file doesn't exist (server down). `ERROR 2002 ... (13)` means *permission denied* — the socket exists inside the `700 mysql:mysql` datadir, which a non-root user like `peter` cannot even traverse. The app bypasses all of this by connecting over TCP (`stdb01:3306`), so only the listener matters.

### Commands

```sh
# 1. Diagnose — confirm the fault
systemctl status mariadb             # inactive (dead), disabled
sudo tail -50 /var/log/mariadb/mariadb.log    # empty — daemon never wrote a log
sudo journalctl -u mariadb -n 50              # -- No entries --
sudo cat -n /etc/my.cnf.d/*.cnf               # stock config: datadir=/var/lib/mysql
sudo ls -ld /var/lib/mysql                    # ls: cannot access ... No such file or directory  <-- FAULT

# 2. Confirm prerequisites before initializing
rpm -q mariadb-server                # mariadb-server-10.5.29-4.el9.x86_64
id mysql                             # uid=27(mysql) gid=27(mysql)
sudo ls -la /var/lib/ | grep -iE 'mysql|maria'   # only 'mysqld' — datadir truly absent

# 3. Initialize the datadir + start the service
sudo mariadb-install-db --user=mysql --datadir=/var/lib/mysql
sudo chown -R mysql:mysql /var/lib/mysql
sudo systemctl start mariadb
sudo systemctl enable --now mariadb
```

### Flag Reference
* `--user=mysql` — make the `mysql` service account own everything in the new datadir
* `--datadir=/var/lib/mysql` — where to create/place the data directory (must match `datadir=` in my.cnf)
* `systemctl enable --now` — start now AND enable at boot in one command
* `ss -tlnp` — show listening TCP sockets: `-t` TCP, `-l` listening, `-n` numeric, `-p` process
* `timeout 2 bash -c 'echo > /dev/tcp/HOST/PORT'` — probe a TCP port using bash's `/dev/tcp`; fails fast (2s) if nothing listens

### Verification

Confirm the service is running, enabled, listening on 3306, and actually answering queries.

```sh
systemctl status mariadb                          # active (running), enabled
ss -tlnp | grep 3306                              # LISTEN on *:3306
sudo mysql -e "SELECT VERSION();"                 # 10.5.29-MariaDB
timeout 2 bash -c 'echo > /dev/tcp/127.0.0.1/3306' && echo "3306 OPEN"   # TCP path the app uses
```

### Outputs

```
[peter@stdb01 ~]$ systemctl status mariadb
○ mariadb.service - MariaDB 10.5 database server
     Active: inactive (dead)

[peter@stdb01 ~]$ sudo ls -ld /var/lib/mysql
ls: cannot access '/var/lib/mysql': No such file or directory

[peter@stdb01 ~]$ sudo mariadb-install-db --user=mysql --datadir=/var/lib/mysql
[peter@stdb01 ~]$ sudo systemctl start mariadb
[peter@stdb01 ~]$ sudo systemctl enable --now mariadb

[peter@stdb01 ~]$ systemctl status mariadb
● mariadb.service - MariaDB 10.5 database server
     Active: active (running) since Fri 2026-09-11 10:02:49 UTC; 5min
   Main PID: 43263 (mariadbd)
     Status: "Taking your SQL requests now..."
Sep 11 10:02:49 stdb01 mariadb-prepare-db-dir[43204]: Database MariaDB is probably initialized in /var/lib/mysql

[peter@stdb01 ~]$ sudo mysql -e "SELECT VERSION();"
+-----------------+
| VERSION()       |
+-----------------+
| 10.5.29-MariaDB |
+-----------------+

[peter@stdb01 ~]$ timeout 2 bash -c 'echo > /dev/tcp/127.0.0.1/3306' && echo "3306 OPEN"
3306 OPEN
```

> **Note:** `mysql -u root` as `peter` fails with `ERROR 2002 (13) Permission denied` — that is normal for a non-root user reaching into the `700 mysql:mysql` datadir. Use `sudo mysql` (root via unix_socket auth) or connect over TCP.

### Also Useful

```sh
sudo mysql -e "SHOW DATABASES;"                        # list databases, prove the server works
sudo mysql < /dev/null && echo "ok"                     # cheap connection sanity check
mysql -h 127.0.0.1 -P 3306 -u <user> -p                 # test the TCP path the app uses
sudo mysqladmin status                                   # "Uptime:" from the admin client
sudo systemctl is-enabled mariadb                       # prints 'enabled' / 'disabled'
rpm -ql mariadb-server | grep install-db                 # locate mariadb-install-db binary
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `ssh natasha@ststor01` → on wrong server | MariaDB lives on the **database** server `stdb01`, not the storage server `ststor01` | `ssh peter@stdb01` (password `Sp!dy`); confirm with `hostname` |
| `sudo journalctl -u mariadb -n 50` → `-- No entries --` | The daemon aborted before writing any log — there was no datadir to even reach the logging stage | Not a fault; it pointed to the missing `/var/lib/mysql` |
| `ls: cannot access '/var/lib/mysql': No such file or directory` | The data directory was never initialized (the actual injected fault) | `sudo mariadb-install-db --user=mysql --datadir=/var/lib/mysql` |
| `mysql -u root` → `ERROR 2002 (HY000) ... socket ... (13)` | EACCES: socket sits inside a `700 mysql:mysql` datadir that `peter` can't traverse; server is fine | Use `sudo mysql` (root via unix_socket) or `mysql -h 127.0.0.1 -P 3306` over TCP |
| `imeout 2 bash -c ...` → `-bash: imeout: command not found` | Typo — dropped the `t` from `timeout` | Re-type `timeout` |
| `Active: inactive (dead)` after fix attempt | Service was stopped/not enabled, or datadir still missing | Confirm datadir exists, then `systemctl start` + `enable --now` |
| `[1]+ Stopped` after `systemctl status` | Pressed Ctrl+Z inside the pager | Press `q` to exit a pager |

### Quick Context
**A missing datadir kills MariaDB before it can even complain**: the whole bootstrap depends on `/var/lib/mysql` existing, so the daemon exits in milliseconds with no log line at all. Empty error log + `inactive (dead)` is the signature of "datadir gone", not "server crashed".

**`mariadb-install-db` (10.5) replaced the old `mysql_install_db`**: the name changed with the mariadb/mysql split, but the job is the same — build the system tables in a datadir you then point the service at. Run it as `--user=mysql` so ownership matches what `mariadbd` expects when systemd launches it.

**`ERROR 2002` has two flavors and they mean opposite things**: `(2)` no socket = server down; `(13)` permission denied = server up but the client user can't reach the socket. Learning to read the error number first saves you from "fixing" a healthy server.
