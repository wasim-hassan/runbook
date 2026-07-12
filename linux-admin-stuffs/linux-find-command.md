# Lab: Linux Find Command

Find all `.php` files (not directories) under `/var/www/html/official` on App Server 1 and copy them preserving the directory structure to `/official`.

### Commands

```sh
# 1 — find all .php files (excluding directories)
sudo find /var/www/html/official -type f -name "*.php"

# 2 — copy them preserving parent directory structure to /official
sudo find /var/www/html/official -type f -name "*.php" -exec cp --parents {} /official \;
```

### Flag Reference
- `-type f` — match only regular files (not directories)
- `-name "*.php"` — match files ending in `.php` (case-sensitive)
- `-exec cp --parents {} /official \;` — copy each matched file with its full directory path preserved under `/official`
- `--parents` — preserve the source directory structure under the destination (e.g., `/var/www/html/official/index.php` becomes `/official/var/www/html/official/index.php`)

### Verification

```sh
sudo find /var/www/html/official -type f -name "*.php" | wc -l
sudo find /official -type f -name "*.php" | wc -l
```

### Outputs

```
[tony@stapp01 ~]$ sudo find /var/www/html/official -type f -name "*.php" | wc -l
1288
[tony@stapp01 ~]$ sudo find /official -type f -name "*.php" | wc -l
1288
```

> **Note:** `--parents` is the key to this task. Without it, `cp` would flatten all `.php` files into `/official/` with no subdirectories. With `--parents`, the full path from the find root is preserved, so you get subdirectory structure like `wp-admin/`, `wp-includes/`, etc.

### Also Useful

```sh
# spot-check the first few copied files
ls -l /official/var/www/html/official/*.php | head -10

# find and copy all .html files instead
sudo find /var/www/html/official -type f -name "*.html" -exec cp --parents {} /official \;

# copy with a different source root (changes parent path)
sudo find /var/www/html/official -type f -name "*.php" -exec cp --parents {} /tmp/ \;
# result: /tmp/var/www/html/official/...

# dry-run — see what would be copied without actually copying
sudo find /var/www/html/official -type f -name "*.php" | head -20
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `find: '/var/www/html/official': Permission denied` | Forgot `sudo` — the directory or some files inside are owned by root and not readable by regular users. | Prepend `sudo` to the find command |
| `cp: cannot create regular file '/official/...': No such file or directory` | The `/official` directory doesn't exist. Without `-p` (parents), `mkdir` or `cp --parents` creates intermediate directories if the destination exists, but `/official` itself must exist. | Create it first: `sudo mkdir -p /official` |
| Copied the entire `/var/www/html/official` directory instead of just `.php` files | Used `cp -r` or `cp -a` instead of `find -exec cp --parents`. Full directory copy moves everything — files, subdirectories, hidden files — regardless of type or extension. | Use `find` with `-type f -name "*.php"` to filter only PHP files |
| File count doesn't match between source and destination | Some files may have failed to copy due to permissions or path length limits. The `find` command may have encountered errors silently with some paths. | Re-run with `-v` to see each file: `sudo find ... -exec cp -v --parents {} /official \; 2>&1 \| tail -20` |

### Quick Context

**`cp --parents` preserves directory structure**: Without `--parents`, `cp /var/www/html/official/index.php /official/` puts `index.php` directly in `/official/`. With `--parents`, it recreates the full source path under the destination: `/official/var/www/html/official/index.php`. This is useful when you want to copy selected files (not entire directories) but maintain their relative hierarchy — exactly what `find -exec cp --parents {} dest \;` does in one pass.

**`find -type f` vs `-type d`**: `-type f` matches only regular files (no directories, symlinks, pipes, sockets). `-type d` matches only directories. A common mistake is omitting `-type f`, which means `find` also returns directory names. If you then `cp --parents` a directory, you also copy everything inside it — defeating the purpose of selective file copy.
