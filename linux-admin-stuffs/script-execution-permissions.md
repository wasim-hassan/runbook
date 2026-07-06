# Lab: Script Execution Permissions

Grant executable permissions on `/tmp/xfusioncorp.sh` on App Server 3 so all users can execute it.

### Commands

```sh
# 1 — check current permissions
ls -l /tmp/xfusioncorp.sh

# 2 — add read + execute for all users
sudo chmod +rx /tmp/xfusioncorp.sh
```

### Flag Reference
- `+rx` — add read (`r`) and execute (`x`) permissions for all (owner, group, others)
- `sudo !!` — re-run the previous command with `sudo` prefix (handy when you forget `sudo` the first time)

### Verification

```sh
ls -l /tmp/xfusioncorp.sh
```

### Outputs

```
-r-xr-xr-x 1 root root 40 Jul  5 08:48 /tmp/xfusioncorp.sh
```

> **Note:** Shell scripts need **read permission** too — the interpreter reads the file content line by line. Execute-only (`--x`) only works for compiled binaries, not scripts.

### Also Useful

```sh
# set specific permission bits manually
sudo chmod 755 /tmp/xfusioncorp.sh        # -rwxr-xr-x

# test execution as a regular user
/tmp/xfusioncorp.sh

# check which user can run the file
namei -l /tmp/xfusioncorp.sh              # trace path permissions
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `chmod: changing permissions of '/tmp/xfusioncorp.sh': Operation not permitted` | The file is owned by `root`, and you ran `chmod` as `banner` (regular user). Only root can change permissions on root-owned files. | Re-run with `sudo chmod` |
| `'script' is not executable` (KodeKloud check fails with `---x--x--x`) | You used `chmod +x` which sets only execute bits. Shell scripts need read permission because the kernel reads the shebang line and the interpreter reads the script content. Execute-only (`--x`) works for compiled ELF binaries but not scripts. | Use `chmod +rx` to grant both read and execute |
| `sudo chmod +x` succeeded but task still failed | Same root cause as above — `+x` with `sudo` sets `---x--x--x`. The permissions look changed but a script needs `r` permission to be executed by the shell. | Run `sudo chmod +rx` to add read + execute |

### Quick Context

**Execute-only (`--x`) works for binaries, not scripts**: A compiled ELF binary can be executed with only the execute bit — the kernel loads it directly. But a script starting with `#!/bin/bash` requires the kernel to first **read** the script to find the interpreter, then the interpreter needs to **read** the script content. Without read permission, both steps fail. Always use `+rx` (or `755`/`555`) for shell scripts, not just `+x`.

**`sudo !!` repeats the last command as root**: When you run a command and forget `sudo`, typing `sudo !!` re-executes your previous command with `sudo` prepended. The `!!` is a shell history expansion for "last command." This is faster than arrow-up + home + typing `sudo ` + space.
