# Lab: Fix "Updates were rejected" Push — Remote Ahead of Local

Push failed because the remote has commits that don't exist locally (e.g. you committed from another machine/session), and `pull --rebase` refuses to run while you have uncommitted changes.

### Commands

```sh
# Check how far apart local and remote are
git status
git log --oneline origin/master..master

# Clean tree: rebase local commits on top of remote
git pull --rebase origin master

# Dirty tree: stash your work, rebase, then restore it
git stash
git pull --rebase origin master
git stash pop

# Push now that local is ahead cleanly
git push
```

### Flag Reference
- `--rebase` — replays your local commits on top of the remote's new ones (linear history) instead of creating a merge commit
- `stash` — shelves uncommitted changes to a stack; `stash pop` restores them and drops the entry

### Verification

```sh
git status
git log --oneline
```

`git status` should show no `!` or `⇡` markers — just `Your branch is up to date with 'origin/master'`. `git log --oneline` should show a linear history: the remote's commits followed by yours.

### Outputs

```
$ git push
! [rejected] master -> master (fetch first)
error: failed to push some refs to 'https://github.com/wasim-hassan/kvm-vm-provisioner.git'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally.

$ git pull --rebase origin master
error: cannot pull with rebase: You have unstaged changes.
error: Please commit or stash them.

$ git stash
$ git pull --rebase origin master   # success, no conflicts
$ git stash pop
Dropped refs/stash@{0} (6a8167f0741c432480b28af6bcbae53a01ec16fe)

$ git push
To https://github.com/wasim-hassan/kvm-vm-provisioner.git
   6f3d073..1612038  master -> master
```

> **Note:** If you keep committing from two different machines/sessions to the same repo, `git pull --rebase` before every push becomes a habit — it's a normal solo-dev workflow, not a mistake.

### Also Useful

```sh
git fetch          # update remote refs without touching your work, then inspect
git log --oneline --graph --all   # see both sides before deciding what to do
git stash list     # all stashed entries; git stash drop <name> to discard
git push --force   # NOT needed here; only after a history rewrite on a solo repo
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| `! [rejected] master -> master (fetch first)` | Remote has commits you don't have. Git blocks non-fast-forward pushes to avoid losing others' work. | `git pull --rebase` (or merge) to integrate, then push |
| `error: cannot pull with rebase: You have unstaged changes.` | Rebase rewrites your local commits and refuses to do so with a dirty working tree — uncommitted edits could be lost or mixed in. | `git stash` → `git pull --rebase` → `git stash pop` |
| Stash pop restores a file that also changed on remote | The rebased code now differs from what you stashed, so the two versions collide. | Fix the conflicts by hand, then `git stash drop` |

### Quick Context
**Why git rejects the push**: Git only accepts fast-forward pushes (the remote ref is a direct ancestor of your new one). When the remote has commits you don't have, a plain push would silently overwrite work — so git refuses. Rebasing is the safe way to fold the remote's commits in under yours.

**Why rebase over merge**: rebase replays your commits on top of the remote's history, keeping the timeline linear and readable. A merge would add a merge commit and produce a forked history. For a solo repo, rebase keeps it clean.

**Why stash exists**: sometimes you need to leave your current state — but rebasing a dirty tree is dangerous. `stash` shelves your edits safely and `pop` restores them, so you can integrate remote work without losing anything.
