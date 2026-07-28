# Lab: Fix Phantom File Modifications After Copying a Git Repo

Copied a git repository from one machine to another and now git status shows every file as modified — but git diff shows zero content changes. The working tree is dirty even though nothing actually changed.

### Commands

```sh
# Discard all metadata-only changes and restore committed file modes
git checkout -- .
```

### Flag Reference
* `--` — separates paths from branches (prevents ambiguity if a branch name matches a file)
* `.` — targets all files in the current directory and subdirectories

### Verification

```sh
git status
```

Confirm the working tree is clean with nothing to commit.

### Outputs

```
wasim ~/Sync/Repos/runbook on  master
❯ git checkout -- .

wasim ~/Sync/Repos/runbook on  master
❯ git status
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
```

> **Note:** `git diff --stat` showing all zeros confirms it's a permission issue, not a content issue.

### Also Useful

```sh
# See exactly which file modes changed
git diff --stat                  # shows per-file line change counts (all 0 = permission-only)
git diff                         # full diff output — shows "old mode" / "new mode" lines

# Discard changes in a specific file only
git checkout -- path/to/file.md

# Discard changes but keep staged files
git reset HEAD .

# Nuclear option: reset everything to match remote (CAUTION — loses local uncommitted work)
git reset --hard origin/master
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `git status` shows all files modified but `git diff --stat` shows 0 changes | File permission bits (executable, read/write) changed during copy between machines — git tracks file modes on Linux | Run `git checkout -- .` to restore original file modes |
| `git checkout -- .` doesn't clear modified status | File content actually changed, not just permissions — `git diff` will show real differences | Review changes with `git diff`, then either commit or revert |
| `git checkout -- .` discards files you actually changed | `.` targets everything in the working tree — there's no undo | Use `git checkout -- path/to/specific/file` next time for surgical discard |
| Modified files on a fresh clone with no copy involved | Line ending differences (CRLF vs LF) between Windows and Linux — git's `core.autocrlf` setting differs per OS | Set `git config --global core.autocrlf input` on Linux before cloning |

### Quick Context
**Phantom modifications from file mode changes**: Git tracks file permission bits (e.g., 755 vs 644) as part of the repository state on Linux. When you copy files across systems — especially different filesystems, USB drives, or rsync/scp with different flags — the permission bits can shift. Git sees this as a modification even though the content is byte-identical. `git diff --stat` reports zero line changes because no lines changed, only metadata. The fix is `git checkout -- .`, which restores the file modes to what git has committed.

**Why `git diff --stat` is the diagnostic**: If every file shows modified but the line change counts are all zeros, it's almost always a permission/mode issue. If you see actual line changes, the files were genuinely modified and you need to review before discarding.
