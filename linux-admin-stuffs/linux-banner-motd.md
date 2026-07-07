# Lab: Linux Banner

Update the Message of the Day (MOTD) on all 3 app servers using the template from `/home/thor/nautilus_banner` on the jump host.

### Commands

```sh
# On jump-host as user thor:
cat nautilus_banner

# For each server (stapp01, stapp02, stapp03):

# 1 — copy the banner file to /tmp/ on the remote server
scp nautilus_banner tony@stapp01:/tmp/

# 2 — move it to /etc/motd using sudo with a TTY
ssh -t tony@stapp01 "sudo mv /tmp/nautilus_banner /etc/motd"
```

### Flag Reference
- `-t` (ssh) — force pseudo-TTY allocation; needed when running commands that prompt for a password (like `sudo`) inside a remote command
- `/etc/motd` — the "message of the day" file displayed to users after a successful SSH login

### Verification

```sh
ssh tony@stapp01 "cat /etc/motd | head -3"
ssh steve@stapp02 "cat /etc/motd | head -3"
ssh banner@stapp03 "cat /etc/motd | head -3"
```

### Outputs

```
tony@stapp01's password: 
################################################################################################
  .__   __.      ___      __    __  .___________. __   __       __    __       _______.        # 
       |  \ |  |     /   \    |  |  |  | |           ||  | |  |     |  |  |  |     /       |   #
```

> **Note:** The MOTD file is read by `sshd` and displayed after login. You don't need to restart any service — changes to `/etc/motd` take effect immediately for the next SSH login.

### Also Useful

```sh
# view the full MOTD banner
ssh tony@stapp01 cat /etc/motd

# update banner without leaving a temp file (single command)
cat nautilus_banner | ssh -t tony@stapp01 "sudo tee /etc/motd > /dev/null"

# check if MOTD display is enabled in sshd config
sudo sshd -T | grep printmotd

# temporarily disable MOTD for a session
ssh -q tony@stapp01    # -q suppresses MOTD
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `sudo: a terminal is required to read the password` | You ran `ssh host "sudo command"` without `-t`. `sudo` needs an interactive terminal (TTY) to prompt for a password, but SSH doesn't allocate one by default for remote commands. | Add `-t` flag: `ssh -t host "sudo command"` |
| `sudo: a terminal is required` when piping with `<` | Running `ssh host "sudo tee" < file` uses stdin for the file content, so `sudo` can't read the password from the same input. Pipes consume stdin, leaving nothing for the password prompt. | Use a separate `scp` + `ssh -t` step, or use `-S` flag with `sudo` to read password from a different source |
| `Permission denied, please try again.` | Wrong password for the remote user. KodeKloud passwords are case-sensitive and match the lab-infra-details. | Double-check the password (e.g., `Am3ric@` not `america`) |

### Quick Context

**`/etc/motd` vs `/etc/issue` vs `/etc/issue.net`**: The MOTD ("message of the day") at `/etc/motd` is displayed **after** successful SSH login, right before the shell prompt. `/etc/issue` shows a banner **before** login on the local console (Ctrl+Alt+F1/F2). `/etc/issue.net` is the SSH pre-login banner (configurable in `sshd_config` via `Banner` directive). Each serves a different purpose: MOTD for post-login notices, issue/issue.net for pre-login legal warnings.

**Why `ssh -t` is needed for sudo**: When you run `ssh host "sudo command"`, SSH doesn't allocate a terminal by default — the remote command's stdin is connected to the SSH channel, not a real TTY. `sudo` detects this and refuses to ask for a password ("no TTY"). `-t` forces SSH to allocate a pseudo-terminal on the remote side, making sudo think it has an interactive terminal it can use for the password prompt.
