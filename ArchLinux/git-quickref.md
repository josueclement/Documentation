# Git Quick Reference

Daily workflow, branching, remotes, undoing changes, stashing, tags, and advanced operations like cherry-pick, worktrees, bisect, interactive rebase, and submodules.

## Initial Setup

These one-time settings configure your identity and default behavior. `--global` applies to all repositories for your user.

```bash
# Set identity (used in every commit)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Set default branch name for new repos
git config --global init.defaultBranch main

# Set default editor (for commit messages, rebase, etc.)
git config --global core.editor vim

# Enable color output
git config --global color.ui auto

# Rebase on pull instead of merge (cleaner history)
git config --global pull.rebase true

# Useful aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --decorate -20"

# Show all config values and where they're set
git config --list
git config --list --show-origin
```

## Daily Workflow

The core loop: check status, stage changes, commit, repeat.

### Checking Status

```bash
# Full status (staged, unstaged, untracked)
git status

# Short format (letters indicate status)
git status -s
```

### Staging Changes

Staging selects which changes go into the next commit. You can stage entire files or individual hunks.

```bash
# Stage a specific file
git add file.txt

# Stage all files in a directory
git add src/

# Interactive staging — choose which hunks to stage (y/n per change)
git add -p

# Stage only modified and deleted files (not new untracked files)
git add -u
```

### Committing

```bash
# Commit with message
git commit -m "message"

# Open editor for commit message
git commit

# Stage all tracked changes and commit in one step
git commit -am "message"
```

### Viewing Changes

`git diff` shows what changed. Without flags it shows unstaged changes; `--staged` shows what's about to be committed.

```bash
# Unstaged changes (working tree vs index)
git diff

# Staged changes (index vs last commit)
git diff --staged

# All changes since last commit
git diff HEAD

# Changes between two branches
git diff main..feature

# Summary only (files changed and lines added/removed)
git diff --stat
```

### Viewing History

```bash
# Full log
git log

# Compact one-line log
git log --oneline

# Graph view with branch labels
git log --oneline --graph --all

# Last 20 commits
git log --oneline -20

# Filter by author
git log --author="jo"

# Filter by date
git log --since="2025-01-01"

# Search commit messages
git log --grep="fix"

# Show changes to a specific file (including through renames)
git log -p --follow file.txt
```

### Inspecting Commits

```bash
# Show a specific commit (diff + message)
git show abc1234

# Show commit from N commits ago
git show HEAD~3

# Blame — show who last modified each line
git blame file.txt
git blame -L 10,20 file.txt   # lines 10-20 only
```

## Branching

Branches let you develop features in isolation. They're cheap in Git — just a pointer to a commit.

### Listing Branches

```bash
git branch             # local branches
git branch -r          # remote branches
git branch -a          # all branches (local + remote)
git branch -v          # with last commit info
```

### Creating and Switching

```bash
# Create a branch
git branch feature-x

# Create and switch in one step
git checkout -b feature-x
git switch -c feature-x            # modern syntax

# Switch to existing branch
git checkout main
git switch main
```

### Renaming and Deleting

```bash
# Rename a branch
git branch -m old-name new-name
git branch -m new-name            # rename current branch

# Delete a branch (safe — only if merged)
git branch -d feature-x

# Force delete (even if unmerged)
git branch -D feature-x
```

### Merging

Merging integrates changes from one branch into another. `--no-ff` forces a merge commit even for fast-forward merges (preserves branch history).

```bash
# Merge feature into current branch (usually main)
git checkout main
git merge feature-x

# Always create a merge commit
git merge --no-ff feature-x

# Abort a merge in progress (if conflicts are too messy)
git merge --abort
```

### Rebasing

Rebasing replays your commits on top of another branch, creating a linear history. Use for feature branches before merging.

```bash
# Rebase current branch onto main
git checkout feature-x
git rebase main

# Abort a rebase in progress
git rebase --abort

# Continue rebase after resolving conflicts
git rebase --continue
```

## Remotes

Remotes are references to other copies of the repository (GitHub, GitLab, etc.).

### Managing Remotes

```bash
# List remotes
git remote -v

# Add remote
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git

# Remove remote
git remote remove origin

# Rename remote
git remote rename origin old-origin
```

### Fetching, Pulling, Pushing

`fetch` downloads changes without merging. `pull` fetches and merges (or rebases). `push` uploads your commits.

```bash
# Fetch updates (download, don't merge)
git fetch origin
git fetch --all

# Pull (fetch + merge/rebase)
git pull
git pull origin main
git pull --rebase

# Push
git push
git push origin main

# Push and set upstream tracking (first push of a new branch)
git push -u origin feature-x

# Delete remote branch
git push origin --delete feature-x

# Push all branches
git push --all origin

# Push tags
git push --tags
git push origin v1.0
```

### Syncing and Tracking

```bash
# Track a remote branch (creates local branch linked to remote)
git checkout --track origin/feature-x

# Set upstream for existing branch
git branch --set-upstream-to=origin/main main

# Sync fork with upstream
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# Prune local references to deleted remote branches
git fetch --prune
git remote prune origin
```

## Undoing Things

Git provides multiple levels of "undo" depending on how far back you need to go.

### Unstaging and Discarding Changes

```bash
# Unstage a file (keep changes in working tree)
git restore --staged file.txt

# Discard local changes to a file (revert to last commit)
git restore file.txt

# Discard all local changes
git restore .
```

### Amending the Last Commit

Amend replaces the last commit — useful for fixing typos in the message or adding forgotten files.

```bash
# Change the commit message
git commit --amend -m "new message"

# Add forgotten file to last commit
git add forgotten-file.txt && git commit --amend --no-edit
```

### Reverting a Commit

`revert` creates a **new** commit that undoes the changes of a previous commit. Safe for shared branches because it doesn't rewrite history.

```bash
git revert abc1234
```

### Resetting

`reset` moves the branch pointer backward. The three modes control what happens to the changes.

```bash
# Soft — undo commit, keep changes staged
git reset --soft HEAD~1

# Mixed — undo commit, keep changes unstaged (default)
git reset --mixed HEAD~1

# Hard — undo commit, discard all changes (destructive!)
git reset --hard HEAD~1
```

### Recovering Lost Commits

The reflog records every position HEAD has been at. Even after `reset --hard`, you can usually recover.

```bash
# Show reflog (history of HEAD positions)
git reflog

# Inspect a lost commit
git checkout abc1234

# Create a branch to save it
git branch recovery abc1234
```

### Removing Files

```bash
# Remove from git tracking but keep on disk
git rm --cached file.txt

# Remove from both git and disk
git rm file.txt
```

## Stashing

Stash temporarily shelves changes so you can switch branches or pull without committing incomplete work.

### Saving

```bash
# Stash current changes (staged + unstaged)
git stash
git stash push -m "description"

# Stash including untracked files
git stash -u

# Stash including ignored files
git stash -a
```

### Restoring

```bash
# Apply last stash (keep in stash list)
git stash apply

# Apply and remove from stash list
git stash pop

# Apply a specific stash
git stash apply stash@{2}
```

### Managing

```bash
# List all stashes
git stash list

# Show stash contents (diff)
git stash show -p stash@{0}

# Drop a specific stash
git stash drop stash@{1}

# Clear all stashes
git stash clear

# Create a branch from a stash (apply + delete stash)
git stash branch new-branch stash@{0}
```

## Tags

Tags mark specific commits — typically used for releases. Annotated tags (with `-a`) store tagger info and a message; lightweight tags are just a name.

```bash
# List tags
git tag
git tag -l "v1.*"

# Create annotated tag (recommended for releases)
git tag -a v1.0 -m "Release version 1.0"

# Create lightweight tag
git tag v1.0

# Tag a specific commit
git tag -a v0.9 abc1234 -m "Retroactive tag"

# Show tag info
git show v1.0

# Delete local tag
git tag -d v1.0

# Delete remote tag
git push origin --delete v1.0

# Push tags
git push origin v1.0
git push --tags
```

## Advanced

### Cherry-pick

Cherry-pick copies a specific commit onto the current branch — useful for applying a fix from one branch to another without merging everything.

```bash
# Apply a specific commit to current branch
git cherry-pick abc1234

# Cherry-pick without committing (stage changes only)
git cherry-pick --no-commit abc1234

# Cherry-pick a range of commits
git cherry-pick abc1234..def5678

# Abort cherry-pick in progress
git cherry-pick --abort
```

### Worktrees

Worktrees let you have multiple branches checked out simultaneously in separate directories. Useful for reviewing PRs while working on something else.

```bash
# Check out a branch in a separate directory
git worktree add ../feature-x feature-x

# Create new branch in a worktree
git worktree add -b hotfix ../hotfix main

# List worktrees
git worktree list

# Remove a worktree
git worktree remove ../feature-x

# Prune stale worktree entries
git worktree prune
```

### Bisect (Find Bug-Introducing Commit)

Bisect performs a binary search through commit history to find which commit introduced a bug. It tests O(log n) commits instead of all of them.

```bash
# Start bisect
git bisect start

# Mark current commit as bad (has the bug)
git bisect bad

# Mark a known good commit (before the bug existed)
git bisect good v1.0

# Git checks out a middle commit — test it, then mark:
git bisect good    # or
git bisect bad

# Repeat until found, then reset:
git bisect reset

# Automated bisect with a test script (exit 0 = good, non-zero = bad)
git bisect start HEAD v1.0
git bisect run ./test-script.sh
```

### Interactive Rebase

Interactive rebase lets you edit, reorder, squash, and drop commits. Use it to clean up a branch before merging.

```bash
# Rebase last 5 commits interactively
git rebase -i HEAD~5

# Commands available in the editor:
# pick   — keep commit as-is
# reword — change commit message
# edit   — stop to amend the commit
# squash — merge into previous commit (keep both messages)
# fixup  — merge into previous commit (discard this message)
# drop   — remove commit entirely

# Example: squash last 3 commits
git rebase -i HEAD~3
# Change "pick" to "squash" (or "s") for the 2nd and 3rd commits
```

### Submodules

Submodules embed one Git repository inside another. The parent repo tracks a specific commit of the submodule.

```bash
# Add submodule
git submodule add https://github.com/user/repo.git path/to/submodule

# Clone repo with submodules
git clone --recurse-submodules https://github.com/user/repo.git

# Initialize submodules after clone (if forgot --recurse-submodules)
git submodule update --init --recursive

# Update submodules to latest commit on their tracked branch
git submodule update --remote

# Remove submodule (3-step process)
git submodule deinit path/to/submodule
git rm path/to/submodule
rm -rf .git/modules/path/to/submodule
```

### Useful Patterns

```bash
# See what changed in a file over time (full diff history)
git log -p --follow file.txt

# Find when a specific string was added or removed
git log -S "searchString" --oneline

# Find when a regex pattern appeared
git log -G "regex" --oneline

# List all files tracked by git
git ls-files

# Show a file at a specific revision
git show HEAD~3:path/to/file.txt

# Create a tar archive of the current HEAD
git archive --format=tar.gz --prefix=project/ HEAD > project.tar.gz

# Clean untracked files
git clean -nd              # dry run (show what would be deleted)
git clean -fd              # force delete

# Sparse checkout (only check out specific paths)
git sparse-checkout init
git sparse-checkout set src/ docs/

# Shallow clone (only latest commit — faster for large repos)
git clone --depth 1 https://github.com/user/repo.git
```

## .gitignore

`.gitignore` tells Git which files to exclude from tracking. Patterns are matched against the repository root.

```gitignore
# Compiled output
*.o
*.pyc
__pycache__/
build/
dist/

# Dependencies
node_modules/
vendor/

# Environment and secrets
.env
.env.local
*.pem

# IDE files
.vscode/
.idea/
*.swp

# OS artifacts
.DS_Store
Thumbs.db

# Logs
*.log

# Negate a pattern (un-ignore a specific file)
!important.log
```

### Debugging .gitignore

```bash
# Check if a file is ignored (and which rule matches)
git check-ignore -v file.txt

# Global gitignore (applies to all repos for your user)
git config --global core.excludesFile ~/.config/git/ignore
```
