# Lab: Fix Orphan Commits with Mismatched Git Author Email

Pushed commits to GitHub but they don't show up on the contributions graph — commits exist in the repo but the author email doesn't match any verified email on the GitHub account.

### Commands

```sh
# Check the author email on your commits
git log --format='%an <%ae>' --all

# Verify the commit is orphan (no author attribution in GitHub)
# Go to repo -> commit -> check if author avatar shows as gray silhouette

# Rewrite all commits with wrong email to the correct one
git filter-branch --env-filter '
if [ "$GIT_AUTHOR_EMAIL" = "wrong@email.com" ]
then
    GIT_AUTHOR_NAME="your-github-username"
    GIT_AUTHOR_EMAIL="your-email@users.noreply.github.com"
    GIT_COMMITTER_NAME="your-github-username"
    GIT_COMMITTER_EMAIL="your-email@users.noreply.github.com"
fi' -- --all

# Force push the rewritten history
git push --force
```

Set the correct email for future commits:

```sh
git config --global user.email "your-email@users.noreply.github.com"
```

### Flag Reference
- `--env-filter` — runs a shell script to rewrite environment variables (author name, email, committer name, email) per commit
- `-- --all` — applies the filter to every branch and tag (not just current branch)
- `--force` — overwrites remote history with rewritten local history

### Verification

```sh
git log --format='%an <%ae>' --all
```

After the rewrite, all commits should show the correct email. Reload your GitHub profile — the contributions graph should update within minutes.

### Outputs

```
$ git log --format='%an <%ae>' --all
solvex <solvex@runbook.com>          # wrong — no match on GitHub
wasim-hassan <wasim-hassan@users.noreply.github.com>   # correct — shows on graph
```

### Also Useful

```sh
# Check git's configured email globally and locally
git config --global user.email
git config user.email

# See which emails are verified on your GitHub account
gh auth status

# Rewrite only the most recent N commits (instead of all)
git rebase -i HEAD~3
# then use `git commit --amend --reset-author` per commit
# then `git push --force`

# Alternative: change just the last commit's author
git commit --amend --author="wasim-hassan <wasim-hassan@users.noreply.github.com>" --no-edit
```

### Errors
| Error | Why it happened | Fix |
| --- | --- | --- |
| Commits pushed but contributions graph shows nothing | Git author email doesn't match any verified email on your GitHub account. GitHub uses the commit's author field, not your login, to attribute contributions. | Add the old email to GitHub settings and verify it, or rewrite history with the correct email |
| Contributions graph shows a gray silhouette avatar on commits | Same root cause — orphan commit. The commit is in the repo but not linked to any GitHub account. | Same fix as above |
| `git push --force` rejected | Branch protection rules on the remote repo prevent force pushes. | Use `git push --force-with-lease` instead, or disable branch protection temporarily |
| `filter-branch` warning about gotchas | Git warns that `filter-branch` can mangle history if used incorrectly (tags, signed commits, etc.). | For a simple email rewrite on a solo repo, it's safe. For team repos, use `git filter-repo` instead |

### Quick Context
**Why GitHub doesn't attribute orphan commits**: GitHub matches commits to accounts by comparing the commit author email against verified emails in the user's account settings. If the email doesn't match, the commit is displayed but not credited. This is by design — it prevents impersonation and ensures contribution credit is accurate.

**`filter-branch` vs `filter-repo`**: `filter-branch` is the built-in git tool for rewriting history. It works for simple changes. `filter-repo` is a faster, more reliable third-party tool recommended for complex rewrites. For a solo repo with 6 commits, `filter-branch` is fine.

**Why you need force push**: Git history is immutable once published. A history rewrite changes commit SHAs (because the author metadata is part of the commit hash input). The remote still points to the old SHAs. `--force` tells the remote to accept the new SHAs as the canonical history. This is only safe on solo repos — on team repos it breaks everyone else's clones.

**The GitHub noreply email**: GitHub provides `username@users.noreply.github.com` as a privacy-preserved email that's always verified on your account. Commits with this email are always attributed correctly. It's the safest choice for git config on personal projects.
