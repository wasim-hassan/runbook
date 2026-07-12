# Lab: Configure Local Yum Repos

Create a local yum repo named `epel_local` from packages in `/packages/downloaded_rpms/` on the Backup Server and install `samba` from it.

### Commands

```sh
# On stbkp01 as user clint:

# 1 — check if the standard repo directory exists (it didn't on this server)
ls -ld /etc/yum.repos.d/

# 2 — create it if missing
sudo mkdir -p /etc/yum.repos.d

# 3 — create the repo config file
sudo vi /etc/yum.repos.d/epel_local.repo
```

Inside vi:
1. Press `i` to enter insert mode
2. Paste the following content:
```
[epel_local]
name=epel_local
baseurl=file:///packages/downloaded_rpms/
enabled=1
gpgcheck=0
```
3. Press `Esc` to return to normal mode
4. Press `Shift+ZZ` to save and exit

```sh
# 4 — confirm dnf recognizes the new repo
sudo dnf repolist | grep epel_local

# 5 — install samba from the local repo only
sudo dnf install --disablerepo=* --enablerepo=epel_local -y samba
```

### Flag Reference
- `[epel_local]` — Repository ID (must match exactly what's in the `[brackets]`)
- `baseurl=file:///packages/downloaded_rpms/` — local path prefixed with `file://` (for HTTP repos use `http://` or `https://`)
- `enabled=1` — activate the repo (set to `0` to disable)
- `gpgcheck=0` — skip GPG signature verification (needed for unsigned local packages)
- `--disablerepo=*` — disable all repos (the `*` wildcard matches everything)
- `--enablerepo=epel_local` — enable only the specified repo

### Verification

```sh
sudo dnf repolist | grep epel_local
rpm -q samba
```

### Outputs

```
[clint@stbkp01 ~]$ sudo dnf repolist | grep epel_local
epel_local                              epel_local
[clint@stbkp01 ~]$ rpm -q samba
samba-4.24.3-1.el9.x86_64
```

> **Note:** If `/etc/yum.repos.d/` doesn't exist, DNF/Yum can still work with repos in `/etc/yum/` (RHEL 9) or via `/etc/dnf/dnf.conf`. The standard location is always `/etc/yum.repos.d/` — create it if missing.

### Also Useful

```sh
# list all enabled repos
sudo dnf repolist

# list all repos including disabled
sudo dnf repolist --all

# view repo config file
cat /etc/yum.repos.d/epel_local.repo

# install from a specific repo without disabling others
sudo dnf install --enablerepo=epel_local samba

# add GPG check (if packages are signed)
gpgcheck=1
gpgkey=file:///path/to/RPM-GPG-KEY
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `E212: Can't open file for writing` in `vi` | The parent directory `/etc/yum.repos.d/` doesn't exist. `vi` can't create files in a non-existent directory, even with `sudo vi`. | Create the directory first: `sudo mkdir -p /etc/yum.repos.d` |
| `ls: cannot access '/etc/yum.repos.d/': No such file or directory` | On some RHEL/CentOS versions, `/etc/yum.repos.d/` is not created by default (especially on minimal installs or certain versions). | Create it with `sudo mkdir -p /etc/yum.repos.d` |
| `Repository 'epel_local' is missing name in configuration, using id` | Missing `name=` field in the `.repo` file. The repo still works but shows a warning. | Add a `name=epel_local` line to the config (not strictly required but good practice) |
| `No match for argument: samba` | The package doesn't exist in the local repo, or the `baseurl` path is wrong. The RPM files in the directory may not include `samba`. | Verify the files exist: `ls /packages/downloaded_rpms/ \| grep samba`. Check `baseurl` points to the correct directory |
| `All mirrors were already tried but none are accessible` | The `baseurl` path is wrong or DNF can't read the directory. Common issues: missing `file://` prefix, trailing slash missing, or wrong path. | Verify with `ls /packages/downloaded_rpms/` and ensure the `.repo` file has `baseurl=file:///packages/downloaded_rpms/` (note the third `/` — `file:///` is correct for absolute paths) |

### Quick Context

**`file://` vs `http://` in repo baseurl**: The `baseurl` in a `.repo` file tells DNF where to find the packages. For local directories, use `file:///path` — three slashes because `file://` is the protocol prefix and the third `/` starts the absolute path (`file:///packages/...`). For remote repos, use `http://` or `https://`. The trailing `/` on the path is important too — DNF appends `repodata/` to it to find metadata.

**Why the E212 error happened**: The `/etc/yum.repos.d/` directory didn't exist on this server. RHEL 9 sometimes doesn't create it unless a package that ships repo files gets installed. Running `sudo vi /etc/yum.repos.d/epel_local.repo` fails because `vi` (even with sudo) can't create files in a non-existent directory — it's a kernel/filesystem limitation, not a permission issue. The fix is always `sudo mkdir -p /etc/yum.repos.d` first, then create the file.
