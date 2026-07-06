# Lab: String Replacement

Substitute all occurrences of `Random` with `Echo-Location` in `/root/nautilus.xml` on the Jump Host.

### Commands

```sh
sudo sed -i 's/Random/Echo-Location/g' /root/nautilus.xml
```

### Flag Reference
- `-i` (sed) — edit file in-place
- `s/Random/Echo-Location/g` — substitute all occurrences globally (`g` flag replaces every match on each line, not just the first)

### Verification

Verify that no `Random` remains and all are replaced with `Echo-Location`:

```sh
sudo grep -i 'Random\|Echo-Location' /root/nautilus.xml | head -5
```

### Outputs

```
thor@jump-host ~$ sudo grep -i 'Random\|Echo-Location' /root/nautilus.xml | head -5
      <type>Echo-Location</type>
      <type>Echo-Location</type>
      <type>Echo-Location</type>
      <type>Echo-Location</type>
      <type>Echo-Location</type>
```

> **Note:** Always test your pattern with `grep` before running `sed -i` so you can confirm the string exists and won't silently do nothing.

### Also Useful

```sh
# dry-run — preview changes without saving (removes -i and pipes to less)
sudo sed 's/Random/Echo-Location/g' /root/nautilus.xml | less

# case-insensitive replacement
sudo sed -i 's/random/Echo-Location/gi' /root/nautilus.xml

# count how many replacements were made
sudo sed -n 's/Random/Echo-Location/gp' /root/nautilus.xml | wc -l

# backup original before editing (-i with suffix creates .bak)
sudo sed -i.bak 's/Random/Echo-Location/g' /root/nautilus.xml
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `sed: -e expression #1, char 11: unterminated 's' command` | Missing closing delimiter — `s/Random/Echo-Location/g` has three `/` delimiters. If you forget the final `g` or the closing `/`, sed doesn't know where the expression ends. | Ensure the pattern is `s/find/replace/g` with all slashes in place |
| `Permission denied` on `/root/nautilus.xml` | File is in `/root/`, which is only accessible by root. Running `sed` without `sudo` fails because thor can't read or write root's files. | Prepend `sudo` to the sed command |
| `sed: couldn't open temporary file /root/sedXXXXXX: Permission denied` | `sed -i` creates a temp file in the same directory. The `/root/` directory is root-only, so sed needs `sudo` for both reading and writing. | Use `sudo sed -i` — both read and write happen as root |

### Quick Context

**`sed 's/old/new/g'` — the global replace pattern**: The `s` command stands for "substitute". Without the trailing `g` (global), sed replaces only the **first** occurrence on each line. Adding `g` tells it to replace **every** occurrence on the line. For XML files where elements often repeat on different lines, `g` is usually what you want — but if you only need the first match per line, drop the `g`.

**Always preview before `-i`**: The `-i` flag modifies the file permanently with no undo. The safe workflow is: `grep` to confirm the target exists → `sed 's///g'` (without `-i`) to preview changes → then `sed -i` to apply. Or use `-i.bak` to keep a backup.
