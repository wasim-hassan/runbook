# Lab: Linux SSH Authentication

Set up password-less SSH access from user `thor` on jump host to all 3 app servers using SSH key pairs.

### Commands

* On jump-host as user thor:
```sh
# 1 — check if SSH keys already exist
ls -l ~/.ssh/id_rsa ~/.ssh/id_ed25519

# 2 — generate RSA key pair (only if no keys exist yet)
ssh-keygen -t rsa -b 2048 -N "" -f ~/.ssh/id_rsa

# 3 — copy public key to each app server
ssh-copy-id tony@stapp01
ssh-copy-id steve@stapp02
ssh-copy-id banner@stapp03
```

### Flag Reference
- `-t rsa` — key type: RSA (widely compatible)
- `-b 2048` — key size: 2048 bits (standard minimum)
- `-N ""` — empty passphrase for password-less login (no prompt)
- `-f ~/.ssh/id_rsa` — output file path for the key pair
- `ssh-copy-id user@host` — copy your public key to the remote server's `~/.ssh/authorized_keys` in one step

### Verification

```sh
ssh tony@stapp01 hostname
ssh steve@stapp02 hostname
ssh banner@stapp03 hostname
```

### Outputs

```
thor@jump-host ~$ ssh tony@stapp01 hostname
stapp01
thor@jump-host ~$ ssh steve@stapp02 hostname
stapp02
thor@jump-host ~$ ssh banner@stapp03 hostname
stapp03
```

> **Note:** `ssh-copy-id` does exactly what manual key setup does — appends your `id_rsa.pub` to `~/.ssh/authorized_keys` on the remote server and sets correct permissions. It's just a helper that saves you from manually creating the `.ssh` directory, editing the file, and fixing permissions.

### Also Useful

```sh
# view the public key you copied
cat ~/.ssh/id_rsa.pub

# check the authorized_keys file on a remote server
ssh tony@stapp01 "cat ~/.ssh/authorized_keys"

# disable password auth entirely (after keys are working)
# edit /etc/ssh/sshd_config → PasswordAuthentication no

# copy key manually (same as ssh-copy-id)
ssh tony@stapp01 "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
cat ~/.ssh/id_rsa.pub | ssh tony@stapp01 "tee -a ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# test with verbose output for debugging
ssh -v tony@stapp01 hostname
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password)` | Key copy failed or the key isn't being offered. This usually means the wrong key was copied or `authorized_keys` has wrong permissions. | Check remote `authorized_keys` permissions (must be `600`): `ssh user@host "chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh"` |
| `ssh-copy-id: ERROR: No identities found` | No SSH keys exist on the jump host. `ssh-copy-id` looks for default key files (`id_rsa`, `id_ed25519`, etc.) and finds nothing. | Generate a key pair first: `ssh-keygen -t rsa -b 2048` |
| `ssh: connect to host stapp01 port 22: Connection timed out` | Network issue — the server is unreachable. Could be down, wrong IP, or firewall blocking port 22. | Verify connectivity with `ping stapp01` and check the IP/hostname from `lab-infra-details.md` |
| Still prompted for password after `ssh-copy-id` | `StrictHostKeyChecking` may be rejecting the connection, or the key was copied to the wrong user's home directory. | Verify which user's `authorized_keys` the key went to — `ssh-copy-id` copies to the user you specify (e.g., `tony`). Make sure you're SSHing as the same user |

### Quick Context

**How SSH key authentication works**: When you SSH to a server, your client signs a challenge with your **private** key (`id_rsa`). The server checks this signature against the **public** keys in `~/.ssh/authorized_keys`. If a matching public key is found, you're authenticated — no password needed. The private key never leaves your machine. This is why copying `id_rsa.pub` (the public key) is safe but copying `id_rsa` (the private key) would compromise security.

**`ssh-copy-id` vs manual copy — what's different?**: `ssh-copy-id` automates the three-step manual process: (1) `ssh user@host mkdir -p ~/.ssh && chmod 700 ~/.ssh`, (2) append your public key to `~/.ssh/authorized_keys`, (3) `chmod 600 ~/.ssh/authorized_keys`. It also checks for duplicate keys so you don't add the same key twice. The end result is identical — your `.pub` key ends up in `~/.ssh/authorized_keys` on the remote server.
