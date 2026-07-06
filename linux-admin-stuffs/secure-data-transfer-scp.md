# Lab: Secure Data Transfer

Copy `/tmp/nautilus.txt.gpg` from the Jump Host to `/home/nfsdata` on App Server 3 using `scp`.

### Commands

```sh
scp /tmp/nautilus.txt.gpg banner@stapp03:/home/nfsdata
```

### Flag Reference
- `scp <source> <user>@<host>:<dest>` — securely copy a file over SSH to a remote host
- No flags used — this is the basic `scp` syntax

### Verification

Confirm the file landed on the destination server with correct size:

```sh
ls -l /home/nfsdata/
```

### Outputs

```
[banner@stapp03 ~]$ ls -l /home/nfsdata/
total 4
-rw-r--r-- 1 banner banner 105 Jul  6 03:07 nautilus.txt.gpg
```

> **Note:** `scp` output shows `100%` and a transfer speed — if you see that, the copy succeeded. Always verify with `ls -l` on the remote end to confirm size matches the source.

### Also Useful

```sh
# copy a file from remote to local
scp banner@stapp03:/home/nfsdata/nautilus.txt.gpg ./

# copy entire directory recursively
scp -r /local/dir banner@stapp03:/remote/dir

# use a specific SSH port (non-default)
scp -P 2222 file.txt banner@stapp03:/home/nfsdata/

# copy with compression for large files
scp -C largefile.tar.gz banner@stapp03:/home/nfsdata/
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `scp: /home/nfsdata: No such file or directory` | The destination directory doesn't exist on the remote server. `scp` doesn't create intermediate directories. | Create it first with `ssh banner@stapp03 sudo mkdir -p /home/nfsdata`, then run `scp` |
| `Permission denied (publickey,password)` | Wrong password or the remote user doesn't have SSH access. | Double-check credentials from `lab-infra-details.md` and re-enter the password carefully |
| `scp: /home/nfsdata/nautilus.txt.gpg: Permission denied` | The remote user (`banner`) doesn't have write permission on `/home/nfsdata`. | On the remote server, `sudo chown banner:banner /home/nfsdata` or `sudo chmod 777 /home/nfsdata` |

### Quick Context

**`scp` copies over SSH — same auth as `ssh`**: When you run `scp`, it uses the same SSH connection and authentication as `ssh` — same host keys, same password prompt. The transfer is encrypted end-to-end. The syntax is always `scp <from> <to>`, where either can be local or `user@host:/path`. Think of it as `cp` with SSH transport.

**Password prompt vs key-based auth**: In these labs, you always enter a password. In production, use SSH keys — generate a key pair with `ssh-keygen`, copy the public key to the server with `ssh-copy-id user@host`, then `scp` without password prompts. `scp` automatically falls back to key auth when available.
