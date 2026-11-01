# Git Guide: Commit, Push, Merge

A short, practical guide for the team. Follow it and everyone's history stays clean.

---

## 1. The Big Picture

Git has four places where your work lives:

```
Working directory  --git add-->  Staging area  --git commit-->  Local repo  --git push-->  Remote (origin)
```

| Place | What it is |
|---|---|
| Working directory | Files you are editing right now |
| Staging area | Changes you picked for the next commit |
| Local repo | Your commits, on your machine |
| Remote (`origin`) | The shared repo (GitLab / GitHub) |

---

## 2. One-time Setup

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@company.com"
git config --global init.defaultBranch main
git config --global pull.rebase false      # pull = fetch + merge
git config --global core.editor "nano"     # or vim / code --wait
```

Check your settings:

```bash
git config --list
```

---

## 3. Daily Workflow

1. Update your local `main`.
2. Create a branch for your task.
3. Edit, commit small and often.
4. Push your branch.
5. Open a merge request (MR) / pull request (PR).
6. After review, merge into `main`.

```bash
git switch main
git pull
git switch -c feature/radar-filter
# ... edit files ...
git add .
git commit -m "Add Kalman filter for track smoothing"
git push -u origin feature/radar-filter
```

---

## 4. Commit

### 4.1 Check what changed

```bash
git status          # which files changed
git diff            # unstaged changes
git diff --staged   # staged changes
```

### 4.2 Stage files

```bash
git add file.cpp          # one file
git add src/              # one folder
git add -p                # choose pieces of a file (recommended)
git add .                 # everything (check git status first!)
```

### 4.3 Commit

```bash
git commit -m "Short clear message"
```

Skip staging for files git already tracks:

```bash
git commit -am "Fix null pointer in parser"
```

### 4.4 Fix the last commit (not yet pushed)

```bash
git commit --amend -m "Better message"     # change message
git add forgotten.cpp
git commit --amend --no-edit               # add a forgotten file
```

> Never amend a commit that is already pushed and shared.

### 4.5 Good commit messages

- One commit = one idea.
- First line: 50 characters or less, imperative mood ("Add", "Fix", "Remove").
- Explain *why*, not just *what*, in the body if needed.

```
Fix crash when radar track list is empty

The tracker dereferenced the first element without checking size.
Added an early return and a unit test.
```

Good: `Add CSV export to report page`
Bad: `update`, `fix stuff`, `wip`

### 4.6 Undo things

| Goal | Command |
|---|---|
| Unstage a file | `git restore --staged file.cpp` |
| Discard edits in a file | `git restore file.cpp` |
| Undo last commit, keep changes | `git reset --soft HEAD~1` |
| Undo last commit, discard changes | `git reset --hard HEAD~1` |
| Undo a pushed commit safely | `git revert <commit-hash>` |

> `reset --hard` deletes work. Use with care.

---

## 5. Push

### 5.1 Basic push

```bash
git push                          # branch already linked to remote
git push -u origin my-branch      # first push of a new branch
```

`-u` links your local branch to the remote one, so later `git push` and `git pull` just work.

### 5.2 Push rejected?

Message: `rejected ... fetch first` means someone else pushed before you.

```bash
git pull          # get their changes and merge
git push
```

### 5.3 Force push

```bash
git push --force-with-lease
```

- Only on **your own** branch, for example after a rebase or amend.
- Never on `main`.
- Prefer `--force-with-lease` over `--force`: it refuses if someone else pushed in the meantime.

### 5.4 Pull and fetch

```bash
git fetch          # download changes, do not touch your files
git pull           # fetch + merge into your current branch
```

---

## 6. Branches

```bash
git branch                    # list local branches
git branch -a                 # include remote branches
git switch -c new-branch      # create and switch
git switch other-branch       # switch
git branch -d old-branch      # delete (after merge)
git push origin --delete old-branch   # delete on remote
```

Naming suggestion:

| Prefix | Use |
|---|---|
| `feature/` | New functionality |
| `fix/` | Bug fix |
| `hotfix/` | Urgent production fix |
| `docs/` | Documentation only |

---

## 7. Merge

### 7.1 Merge a branch into main

```bash
git switch main
git pull
git merge feature/radar-filter
git push
```

### 7.2 Two kinds of merge

**Fast-forward**: `main` has no new commits, so git just moves the pointer. No merge commit.

**Merge commit**: both branches have new commits, so git creates a commit that joins them.

Force a merge commit even when fast-forward is possible:

```bash
git merge --no-ff feature/radar-filter
```

### 7.3 Keep your branch up to date

Bring the latest `main` into your feature branch regularly:

```bash
git switch feature/radar-filter
git fetch
git merge origin/main
```

This keeps conflicts small and easy.

### 7.4 Resolve a merge conflict

When git says `CONFLICT`, do this:

1. Run `git status` to see conflicted files.
2. Open each file. Find the markers:

```
<<<<<<< HEAD
your version
=======
their version
>>>>>>> origin/main
```

3. Edit the file to the final result. Delete all three marker lines.
4. Stage and finish:

```bash
git add file.cpp
git commit            # completes the merge
```

Cancel the merge if you are lost:

```bash
git merge --abort
```

Use a visual tool if you prefer: `git mergetool`.

### 7.5 Merge vs rebase (quick view)

| | Merge | Rebase |
|---|---|---|
| History | Keeps real history, adds merge commit | Straight line, rewrites commits |
| Safe on shared branches | Yes | No |
| Best for | Joining finished work into `main` | Cleaning your own local branch |

Simple rule: **rebase your own private branch, merge shared branches.**

```bash
git switch feature/radar-filter
git rebase main
```

---

## 8. Other Useful Commands

```bash
git log --oneline --graph --decorate -20   # compact history
git show <hash>                            # see one commit
git blame file.cpp                         # who changed each line
git stash                                  # save unfinished work
git stash pop                              # bring it back
git tag v1.0.0 && git push --tags          # mark a release
git cherry-pick <hash>                     # copy one commit to current branch
git clean -fd                              # delete untracked files (careful)
```

---

## 9. Shortcut Commands

Typing less means fewer mistakes. There are two ways: **git aliases** (work as `git gc`) and **shell aliases** (work as just `gc`).

### 9.1 Option A: Shell aliases (shortest)

Add to `~/.bashrc` (or `~/.zshrc`), then run `source ~/.bashrc`.

```bash
# Status and diff
alias gs='git status -sb'
alias gd='git diff'
alias gds='git diff --staged'

# Stage and commit
alias ga='git add'
alias gaa='git add .'
alias gap='git add -p'
alias gc='git commit -m'
alias gca='git commit -am'
alias gcm='git commit --amend --no-edit'

# Push and pull
alias gp='git push'
alias gpu='git push -u origin HEAD'
alias gpf='git push --force-with-lease'
alias gl='git pull'
alias gf='git fetch --all --prune'

# Branch and switch
alias gb='git branch'
alias gsw='git switch'
alias gswc='git switch -c'

# Merge and rebase
alias gm='git merge'
alias gma='git merge --abort'
alias grb='git rebase'

# History
alias glg='git log --oneline --graph --decorate -20'

# Stash
alias gst='git stash'
alias gstp='git stash pop'
```

Usage examples:

```bash
gaa
gc "Fix null pointer in parser"
gp
```

> `gc` here expects the message right after it: `gc "message"`.
> On some systems `gc` is also a different command (Ghostscript is `gs`, which conflicts with our `gs`). If you use Ghostscript, rename ours to `gss`. Check with `type gs gc`.

### 9.2 Option B: Git aliases (portable, live in git config)

Add with commands:

```bash
git config --global alias.st   "status -sb"
git config --global alias.co   "checkout"
git config --global alias.sw   "switch"
git config --global alias.br   "branch"
git config --global alias.ci   "commit"
git config --global alias.cm   "commit -m"
git config --global alias.unstage "restore --staged"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.lg   "log --oneline --graph --decorate -20"
git config --global alias.pf   "push --force-with-lease"
```

Or edit `~/.gitconfig` directly:

```ini
[alias]
    st = status -sb
    co = checkout
    sw = switch
    br = branch
    ci = commit
    cm = commit -m
    unstage = restore --staged
    last = log -1 HEAD --stat
    lg = log --oneline --graph --decorate -20
    pf = push --force-with-lease
```

Usage: `git st`, `git cm "message"`, `git lg`.

### 9.3 Quick reference table

| Shortcut | Full command | Meaning |
|---|---|---|
| `gs` | `git status -sb` | Short status |
| `gd` | `git diff` | See edits |
| `ga` | `git add` | Stage files |
| `gaa` | `git add .` | Stage everything |
| `gc "msg"` | `git commit -m "msg"` | Commit |
| `gca "msg"` | `git commit -am "msg"` | Stage tracked + commit |
| `gcm` | `git commit --amend --no-edit` | Add to last commit |
| `gp` | `git push` | Push |
| `gpu` | `git push -u origin HEAD` | Push new branch |
| `gpf` | `git push --force-with-lease` | Safe force push |
| `gl` | `git pull` | Pull |
| `gf` | `git fetch --all --prune` | Fetch all |
| `gb` | `git branch` | List branches |
| `gsw` | `git switch` | Switch branch |
| `gswc` | `git switch -c` | New branch |
| `gm` | `git merge` | Merge |
| `gma` | `git merge --abort` | Cancel merge |
| `glg` | `git log --oneline --graph ...` | Pretty history |
| `gst` / `gstp` | `git stash` / `git stash pop` | Save / restore work |

### 9.4 A small function: commit and push in one go

```bash
gcp() {
    git add -A && git commit -m "$1" && git push
}
```

Usage: `gcp "Add login page"`

> Convenient, but run `gs` first so you know exactly what gets committed.

---

## 10. Rules for the Team

1. Never commit directly to `main`. Use a branch and a merge request.
2. Pull before you start work and before you push.
3. Commit small and often, with clear messages.
4. Never force push to `main` or any shared branch.
5. Do not commit build output, secrets, or large binaries. Use `.gitignore`.
6. Resolve conflicts carefully, then build and test before you commit the merge.
7. Delete branches after they are merged.

Example `.gitignore` for a C++/Qt project:

```
build/
*.o
*.so
*.user
CMakeLists.txt.user
.vscode/
.idea/
```

---

## 11. Cheat Sheet

```bash
# Start
git switch main && git pull
git switch -c feature/my-task

# Work
git status
git add -p
git commit -m "Clear message"

# Share
git push -u origin feature/my-task

# Update from main
git fetch && git merge origin/main

# Finish
git switch main && git pull
git merge feature/my-task
git push
git branch -d feature/my-task
```

## 12. Getting Out of Trouble

| Problem | Fix |
|---|---|
| Committed to the wrong branch | `git reset --soft HEAD~1`, switch branch, commit again |
| Typo in last commit message | `git commit --amend -m "new message"` |
| Want to go back before a bad merge | `git reflog`, then `git reset --hard <hash>` |
| Merge went wrong | `git merge --abort` |
| Pushed a bad commit | `git revert <hash>` then push |
| Lost a commit | `git reflog` shows it, then `git switch -c rescue <hash>` |
