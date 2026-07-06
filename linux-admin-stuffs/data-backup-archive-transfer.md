# Lab: Data Backup for Developer

Create a compressed archive of `/data/mariyam` and transfer it to `/home/` on the Jump Host.

### Commands

```sh
# a — create a gzip-compressed tarball
tar -czf mariyam.tar.gz /data/mariyam

# b — move the archive to /home/
sudo mv mariyam.tar.gz /home/
```

### Flag Reference
- `-c` — create a new archive
- `-z` — compress through gzip (produces `.tar.gz`)
- `-f` — specify the output archive filename

### Verification

```sh
ls -l /home/mariyam.tar.gz
```

### Outputs

```
thor@jump-host ~$ ls -l /home/mariyam.tar.gz
-rw-r--r-- 1 thor thor 191 Jul  5 08:28 /home/mariyam.tar.gz
```

> **Note:** `tar` strips the leading `/` from paths inside the archive as a safety measure — extracting will restore to `data/mariyam/...` (relative) instead of `/data/mariyam/...` (absolute). This prevents accidental overwrites if you extract on a different system.

### Also Useful

```sh
# view archive contents without extracting
tar -tzf /home/mariyam.tar.gz

# extract the archive
tar -xzf /home/mariyam.tar.gz

# create a bzip2-compressed archive instead (.tar.bz2)
tar -cjf mariyam.tar.bz2 /data/mariyam

# copy via scp instead of mv (if transferring to a remote server)
scp mariyam.tar.gz user@remote:/home/
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `tar: Removing leading '/' from member names` | Not an error — `tar` automatically strips the leading `/` to store paths as relative. This is a safety feature to prevent accidental root overwrites. | No action needed; it's expected behavior |
| `ls: cannot access 'mariyam.tar.gz': No such file or directory` | You ran `ls` right after `mv` but checked the old location (current directory) instead of the destination `/home/`. The file was already moved. | Use `ls -l /home/mariyam.tar.gz` to check the new location |
| `Permission denied` when moving to `/home/` | Regular users don't have write permission to `/home/` on some systems. | Use `sudo mv ... /home/` |

### Quick Context

**`tar` strips leading `/` for safety**: When you create a tarball from an absolute path like `/data/mariyam`, `tar` converts it to `data/mariyam` inside the archive. This means extracting will create `./data/mariyam` relative to the current directory, not dump files at `/data/`. If you want absolute paths preserved, use `-P` (but you almost never want that).

**`-czf` is the "make a .tar.gz" shorthand**: The three flags combined — `c` (create), `z` (gzip compress), `f` (filename) — are the standard way to produce compressed archives in Linux. Without `-z`, you'd get an uncompressed `.tar` file. The compression type determines the extension (`.tar.gz`, `.tar.bz2`, `.tar.xz`), and each uses a different algorithm trading speed vs size.
