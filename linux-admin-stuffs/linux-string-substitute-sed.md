# Lab: Linux String Substitute (sed)

Delete lines containing "software" (case-sensitive) and replace whole word "the" with "them" in `/home/BSD.txt` on App Server 3.

### Commands

```sh
# a — delete lines containing "software" and save to BSD_DELETE.txt
sudo sh -c "sed '/software/d' /home/BSD.txt > /home/BSD_DELETE.txt"

# b — replace whole word "the" with "them" and save to BSD_REPLACE.txt
sudo sh -c "sed 's/\\bthe\\b/them/g' /home/BSD.txt > /home/BSD_REPLACE.txt"
```

### Flag Reference
- `/software/d` — delete any line that matches the pattern "software" (case-sensitive)
- `s/\bthe\b/them/g` — substitute whole word "the" with "them", globally on each line
- `\b` — word boundary anchor (ensures "the" inside words like "other" or "contributor" is not matched)
- `sudo sh -c "..."` — run the entire quoted command (including redirect `>`) as root; without this, the redirect runs as the current user and may fail due to permissions

### Verification

```sh
# Part (a) — confirm no lowercase "software" lines remain
grep 'software' /home/BSD_DELETE.txt       # should show nothing

# Part (b) — confirm whole-word replacement worked
grep '\bthem\b' /home/BSD_REPLACE.txt       # should show replaced lines
head -5 /home/BSD_REPLACE.txt               # spot-check first lines
```

### Outputs

```
[banner@stapp03 ~]$ ls -l /home/BSD_DELETE.txt /home/BSD_REPLACE.txt
-rw-r--r-- 1 root root 8641 Jul  8 15:32 /home/BSD_DELETE.txt
-rw-r--r-- 1 root root 9991 Jul  8 15:33 /home/BSD_REPLACE.txt
```

> **Note:** The original file `/home/BSD.txt` is unchanged — both `sed` commands read from the original and write to new files. The `>` redirect creates/overwrites the output file. Use `>>` to append instead.

### Also Useful

```sh
# delete lines case-insensitively
sed '/software/Id' /home/BSD.txt > /home/BSD_DELETE.txt

# replace without word boundaries (matches inside words too)
sed 's/the/them/g' /home/BSD.txt > /home/BSD_REPLACE.txt

# delete lines AND replace in one pass
sed '/software/d; s/\bthe\b/them/g' /home/BSD.txt > output.txt

# preview changes without saving (removes the file redirect)
sed '/software/d' /home/BSD.txt | head -10

# in-place edit (modifies original file — use with caution)
sed -i.bak '/software/d' /home/BSD.txt
```

### Errors
| Error | Why it happened | Fix |
|---|---|---|
| `-bash: /home/BSD_DELETE.txt: Permission denied` | You ran `sed ... > /home/BSD_DELETE.txt` without elevating the redirect. `sudo sed ... > file` only elevates `sed`, but the redirect `>` is done by the shell as your user — and you can't write to `/home/`. | Use `sudo sh -c "sed ... > file"` or `sed ... \| sudo tee file` |
| `bash: !: event not found` | The `!` character in `sudo !!` was interpreted by bash history expansion before the shell could execute it. This usually happens when running without proper quoting. | Just type the full command instead of relying on `!!` in this context |
| `sed: -e expression #1, char 1: unknown command: `'\\b''` | Escaping issue with `\b` inside `sudo sh -c`. The backslash needs to be escaped as `\\b` because it's inside double quotes that go through the shell twice. | Use `sudo sh -c "sed 's/\\\\bthe\\\\b/them/g' ..."` or use single quotes around the outer command |
| Wrong output — "the" inside words like "other" was also replaced | Forgot word boundary `\b`. Without it, `s/the/them/g` replaces every occurrence of the letters "t-h-e" in any position, including inside words. | Add `\b` before and after the pattern: `s/\bthe\b/them/g` |

### Quick Context

**`\b` word boundaries prevent partial-word matches**: Without `\b`, `sed 's/the/them/g'` would change "other" to "othemr", "contributor" to "contribut them", and "therefore" to "themrefore". The `\b` anchor tells sed to match only when "the" is a complete word — preceded and followed by a non-word character (space, punctuation, start/end of line). Use word boundaries whenever replacing words that could appear as substrings of other words.

**`sudo sh -c` solves the redirect permission problem**: When you run `sudo command > file`, only `command` gets `sudo` — the shell processes `> file` before `sudo` even starts, using your user's permissions. `sudo sh -c "command > file"` runs the entire shell command (including the redirect) as root. The same applies to pipes: `sudo cmd1 | cmd2` only elevates `cmd1`. A safer alternative is `sed ... | sudo tee file > /dev/null`, which avoids shell quoting headaches.
