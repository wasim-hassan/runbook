# Lab: Install Ansible

Install Ansible version 4.10.0 globally on the Jump Host using `pip3`.

### Commands

```sh
# On jump-host as user thor:

# 1 — check if pip3 is available
pip3 --version

# 2 — install ansible 4.10.0 globally
sudo pip3 install ansible==4.10.0
```

### Flag Reference
- `==4.10.0` (pip) — pin the exact version to install; without this, pip installs the latest available version
- `sudo pip3 install` — install packages system-wide to `/usr/local/` (all users); without `sudo`, it installs to `~/.local/` (current user only)

### Verification

```sh
ansible --version
```

### Outputs

```
thor@jump-host ~$ ansible --version
ansible [core 2.11.12] 
  config file = None
  configured module search path = ['/home/thor/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/thor/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.19 (main, Jun 11 2024, 00:00:00) [GCC 11.4.1 20231218 (Red Hat 11.4.1-3)]
  jinja version = 3.1.6
  libyaml = True
```

> **Note:** Ansible 4.10.0 ships with `ansible-core` 2.11.12. If `ansible --version` shows `core 2.11.12`, you have the correct Ansible 4.10.0 installed. The `executable location` shows `/usr/local/bin/ansible` — confirming global availability.

### Also Useful

```sh
# install a different version
sudo pip3 install ansible==5.0.0

# upgrade to latest
sudo pip3 install ansible --upgrade

# uninstall ansible
sudo pip3 uninstall ansible

# check pip3 and python versions
pip3 --version
python3 --version

# list all installed python packages
pip3 list | grep -i ansible
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `bash: pip3: command not found` | `pip3` is not installed. It ships with the `python3-pip` package. | Install it: `sudo dnf install -y python3-pip` |
| `ERROR: Could not find a version that satisfies the requirement ansible==4.10.0` | Version 4.10.0 may not exist for your Python version or architecture. Ansible dropped Python 3.9 support after version 5.x. | Check available versions: `pip3 index versions ansible` or use a newer version like `ansible==5.0.0` if Python version permits |
| `WARNING: Running pip as the 'root' user can result in broken permissions` | `sudo pip3 install` runs pip as root. This is a warning, not an error — it's fine for this use case but not recommended for production Python development. | Ignore the warning, or use `pip3 install --user` (but that won't be global). For this task, `sudo pip3 install` is the expected approach |
| `ansible: command not found` after installation | The install path (`/usr/local/bin/`) may not be in `$PATH` for your shell session, or the install failed silently. | Check with `which ansible` or log out and back in. If still missing, re-run the install and check for errors |
| `ImportError: cannot import name 'AnsibleLoader' from 'ansible.parsing.yaml.loader'` | Version mismatch between `ansible` and `ansible-core`. Pinning `ansible==4.10.0` should pull the correct `ansible-core==2.11.12`. | Uninstall both and reinstall: `sudo pip3 uninstall ansible ansible-core -y && sudo pip3 install ansible==4.10.0` |

### Quick Context

**`sudo pip3 install` installs globally, `pip3 install --user` installs locally**: Without `sudo`, `pip3 install` puts packages in `/home/thor/.local/lib/python3.9/site-packages/` — only `thor` can run the installed tools. With `sudo`, it goes to `/usr/local/lib/python3.9/site-packages/` — every user on the system can access it. For a tool like Ansible that multiple admins might use, global installation is the right choice. The trade-off is that global pip installs can conflict with system-managed packages (that's what the warning is about).

**Ansible version numbering changed after 2.9**: Ansible 2.9 and earlier used a single version number for everything. Starting with Ansible 4.x, the project split into `ansible` (the curated collection package) and `ansible-core` (the engine). Version mapping: Ansible 4.x → ansible-core 2.11.x, Ansible 5.x → ansible-core 2.12.x, etc. So `ansible==4.10.0` pulls `ansible-core==2.11.12` as a dependency.
