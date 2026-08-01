# Lab: Postfix & Dovecot Mail Server (Stork DC)

Install postfix + dovecot on stmail01, create the mail account `siva@stratos.xfusioncorp.com` with Maildir at `/home/siva/Maildir`, and wire both services to deliver and serve that mailbox.

## How It Works

**Email is a two-step job — like a post office + your home mailbox:**

1. **Postfix = the post office (MTA).** Accepts mail from other servers and apps on port 25, then drops each message into the recipient's mailbox on the local disk.
2. **Dovecot = the mailbox (IMAP/POP3).** Users open a mail client (Outlook, Thunderbird, `mailx`) and connect to dovecot on ports 143 (IMAP) / 110 (POP3) to *read* the messages postfix delivered.

The only thing that links them is the **shared Maildir path**. Postfix writes to it, dovecot reads from it — they must point at the same place or mail vanishes into a folder nobody reads:

- Postfix side: `home_mailbox = Maildir/` → "deliver into the user's `~/Maildir` folder"
- Dovecot side: `mail_location = maildir:~/Maildir` → "serve mail from the user's `~/Maildir` folder"

**Why Maildir and not a single file (mbox):** old systems stored all messages in one big growing file. Maildir is a folder with `cur/ new/ tmp/` — every message is its own file. No file-locking, safe to read while being written. `new/` = unread mail. That's the folder we saw fill up after the test.

**Why the test used `sendmail`:** `sendmail` is postfix's compatible command-line front door. `echo "msg" | sendmail siva@...` is like dropping a letter into the post office slot — it tells postfix to deliver a message.

### Commands

1. Get on the right server (Mail Server = stmail01, NOT jump-host)
```sh
ssh groot@stmail01
```

2. Install both packages at once
```sh
sudo yum install postfix dovecot -y
```

3. Create the mail user (this becomes the email local-part)
```sh
sudo useradd siva
sudo passwd siva                  # password: Rc5C9EyvbU
```

4. Configure postfix via postconf -e (safe, syntax-checked edits)
```sh
sudo postconf -e 'mydomain=stratos.xfusioncorp.com'
sudo postconf -e 'myorigin=$mydomain'
sudo postconf -e 'inet_interfaces=all'
sudo postconf -e 'mydestination=$myhostname, localhost.$mydomain, localhost, $mydomain'
sudo postconf -e 'home_mailbox=Maildir/'
```

5. Configure dovecot (append after the "!include conf.d/*.conf" line so it overrides)
```sh
sudo vi /etc/dovecot/dovecot.conf
```

* add these three lines at the end:
```
mail_location = maildir:~/Maildir
disable_plaintext_auth = no
auth_mechanisms = plain login
```

6. Apply, start on boot, and confirm
```sh
sudo systemctl restart postfix dovecot
sudo systemctl enable postfix dovecot
sudo systemctl status postfix dovecot
```

### Flag Reference
* `postconf -e 'key=value'` — edit `main.cf` programmatically instead of opening the file with `vi`; no risk of a typo breaking the whole config
* `home_mailbox = Maildir/` — deliver mail to Maildir format inside each user's home dir. **The trailing `/` is required** — it's what marks "Maildir" (vs `mbox`, a plain file)
* `inet_interfaces = all` — listen for mail on every network interface, not just `localhost` (default is `localhost` only)
* `mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain` — the domains this server is the *final stop* for; mail addressed to them is accepted instead of bounced
* `mail_location = maildir:~/Maildir` — doveconf: where each user's mailbox lives
* `disable_plaintext_auth = no` — allow passwords sent in plain text (fine in a lab, never on the public internet)
* `auth_mechanisms = plain login` — enable the PLAIN and LOGIN login methods against the system's real users
* `-R` (on `ls`) — recursive; list subdirectory contents too

### Verification

First confirm the running config matches what we set, then prove delivery end-to-end.

```sh
# postfix: confirm our 5 settings are live
sudo postconf -n | grep -E 'mydomain|myorigin|inet_interfaces|mydestination|home_mailbox'

# dovecot: confirm the 3 settings are live
sudo doveconf -n | grep -E 'mail_location|disable_plaintext_auth|auth_mechanisms'

# ports: 25 SMTP, 110 POP3, 143 IMAP should all be LISTENing
ss -tlnp | grep -E ':25|:110|:143'

# end-to-end: inject a real mail, then check it lands in siva's Maildir
echo "Test mail body" | sendmail siva@stratos.xfusioncorp.com
sleep 2
sudo ls -R /home/siva/Maildir
```

`Maildir` must contain `cur/ new/ tmp/` and the test message should appear in `new/`.

### Outputs

```
[groot@stmail01 ~]$ sudo postconf -n | grep -E 'mydomain|myorigin|inet_interfaces|mydestination|home_mailbox'
home_mailbox = Maildir/
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mydomain = stratos.xfusioncorp.com
myorigin = $mydomain

[groot@stmail01 ~]$ sudo doveconf -n | grep -E 'mail_location|disable_plaintext_auth|auth_mechanisms'
auth_mechanisms = plain login
disable_plaintext_auth = no
mail_location = maildir:~/Maildir

[groot@stmail01 ~]$ ss -tlnp | grep -E ':25|:110|:143'
LISTEN 0      100          0.0.0.0:143       0.0.0.0:*
LISTEN 0      100          0.0.0.0:110       0.0.0.0:*
LISTEN 0      100          0.0.0.0:25        0.0.0.0:*
LISTEN 0      100             [::]:143          [::]:*
LISTEN 0      100             [::]:110          [::]:*
LISTEN 0      100             [::]:25           [::]:*

[groot@stmail01 ~]$ echo "Test mail body" | sendmail siva@stratos.xfusioncorp.com
sleep 2
sudo ls -R /home/siva/Maildir
/home/siva/Maildir:
cur  new  tmp

/home/siva/Maildir/cur:

/home/siva/Maildir/new:
1785603760.V4000c4I1830e0M193853.stmail01

/home/siva/Maildir/tmp:
```

> **Note:** The message filename pattern `1785603760.V...stmail01` = timestamp . unique-id . hostname. It proving delivery worked — one file per message in `new/`.

### Also Useful

```sh
sudo postfix check                                     # syntax-check main.cf and report problems
sudo doveconf -n                                       # dump ONLY non-default settings (clean overview)
mailx -s "Subject" siva@stratos.xfusioncorp.com        # alternative test sender if mailx is installed
sudo journalctl -u postfix -u dovecot -f               # follow both mail services' logs live
sudo tail -f /var/log/maillog                          # postfix's traditional log file
doveadm mailbox status -u siva all                     # inspect the mailbox from dovecot's point of view
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `Permission denied, please try again` (SSH as groot) | Typo in the SSH password — it's case-sensitive (`Gr00T123`, not `groot`/`gr00t123`) | Type the exact password from `lab-infra-details.md` |
| `-bash: [200~sudo: command not found` | Bracketed-paste artifact — your terminal pasted the text with a `[200~` prefix, so bash read the whole thing as one broken command | Retype the command cleanly instead of pasting |
| `[1]+ Stopped` after `systemctl status` | Pressed Ctrl+Z (suspends the job) while inside the pager instead of quitting it | Press `q` to exit a pager (less, more, systemctl status output) |
| Postfix listening on `localhost` only / mail rejected from other hosts | `inet_interfaces` was left at its default `localhost` | `sudo postconf -e 'inet_interfaces=all'` |
| Mail lands in `/var/spool/mail/siva` instead of `~/Maildir` | `home_mailbox = Maildir/` missing, or the trailing `/` was dropped | Set `home_mailbox=Maildir/` — the `/` is what switches to Maildir format |
| Dovecot shows empty inbox even though mail was delivered | `mail_location` and `home_mailbox` point to different paths | Make both use the same path: `maildir:~/Maildir` ⇄ `Maildir/` |
| Dovecot rejects client logins / `Auth failed` | `disable_plaintext_auth` defaults to `yes` — with no TLS, logins are refused | Set `disable_plaintext_auth = no` (lab only) |
| Postfix bounces mail for `@stratos.xfusioncorp.com` | `mydestination` doesn't include `$mydomain`, so postfix treats it as a foreign domain | Add `, $mydomain` to `mydestination` |
| Postfix/dovecot won't start after editing | Syntax error in the config file | Run `sudo postfix check` and `sudo journalctl -u postfix -u dovecot` to see the offending line |

### Quick Context
**Two programs, one mailbox**: postfix (MTA) receives and delivers mail; dovecot (IMAP/POP3) serves mail to clients. They never talk to each other — they only agree through the shared Maildir path. If `home_mailbox` (postfix) and `mail_location` (dovecot) don't match, mail is delivered to a folder dovecot never reads. Getting this link right is the whole task.

**Why Maildir**: one file per message in `cur/ new/ tmp/` instead of mbox's single growing file. No locking, safe under concurrent reads/writes. `new/` = unread. That's why the check is `ls -R` on `~/Maildir`.

**Plaintext auth is lab-only**: `disable_plaintext_auth = no` means passwords travel in cleartext. Fine in a sandbox to keep config simple; on the public internet you'd enable STARTTLS/SSL instead.

**`postconf -e` vs `vi`**: postconf edits `main.cf` programmatically — syntax-checked and typo-proof. Same result as hand-editing, but much safer on a live server.

**`sendmail` is postfix in disguise**: on RHEL the `sendmail` binary is actually provided by postfix as a compatible interface. That's why `echo ... | sendmail` works to inject test mail.
