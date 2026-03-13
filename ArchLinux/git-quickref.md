# Git Quick Reference

## Initial Setup

```bash
# Set identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Set default branch name
git config --global init.defaultBranch main

# Set default editor
git config --global core.editor vim

# Enable color
git config --global color.ui auto

# Set merge strategy
git config --global pull.rebase true

# Useful aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --decorate -20"

# Show all config
git config --list
git config --list --show-origin
```

## Daily Workflow

```bash
# Check status
git status
git status -s          # short format

# Stage changes
git add file.txt
git add src/           # all files in directory
git add -p             # interactive: stage hunks selectively
git add -u             # stage modified and deleted (not new)

# Commit
git commit -m "message"
git commit             # opens editor for message
git commit -am "message"    # stage tracked + commit

# View changes
git diff               # unstaged changes
git diff --staged      # staged changes
git diff HEAD          # all changes since last commit
git diff main..feature # between branches
git diff --stat        # summary only

# View log
git log
git log --oneline
git log --oneline --graph --all
git log --oneline -20          # last 20 commits
git log --author="jo"
git log --since="2025-01-01"
git log --grep="fix"           # search commit messages
git log -p file.txt            # changes to a specific file
git log --follow file.txt      # follow renames

# Show a specific commit
git show abc1234
git show HEAD~3                # 3 commits ago

# Blame (who changed each line)
git blame file.txt
git blame -L 10,20 file.txt   # lines 10-20
```

## Branching

```bash
# List branches
git branch             # local
git branch -r          # remote
git branch -a          # all
git branch -v          # with last commit

# Create branch
git branch feature-x
git checkout -b feature-x          # create and switch
git switch -c feature-x            # create and switch (modern)

# Switch branches
git checkout main
git switch main

# Rename branch
git branch -m old-name new-name
git branch -m new-name            # rename current

# Delete branch
git branch -d feature-x           # safe delete (must be merged)
git branch -D feature-x           # force delete

# Merge
git checkout main
git merge feature-x
git merge --no-ff feature-x       # always create merge commit

# Rebase
git checkout feature-x
git rebase main

# Abort merge/rebase
git merge --abort
git rebase --abort

# Continue rebase after resolving conflicts
git rebase --continue
```

## Remotes

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
git push -u origin feature-x      # set upstream tracking
git push origin --delete feature-x # delete remote branch

# Push all branches
git push --all origin

# Push tags
git push --tags
git push origin v1.0

# Track a remote branch
git checkout --track origin/feature-x
git branch --set-upstream-to=origin/main main

# Sync fork with upstream
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# Prune deleted remote branches
git fetch --prune
git remote prune origin
```

## Undoing Things

```bash
# Unstage a file
git restore --staged file.txt
git reset HEAD file.txt            # older syntax

# Discard local changes to a file
git restore file.txt
git checkout -- file.txt           # older syntax

# Discard all local changes
git restore .

# Amend last commit (message or content)
git commit --amend -m "new message"
git add forgotten-file.txt && git commit --amend --no-edit

# Revert a commit (create a new undo commit)
git revert abc1234

# Reset to a previous commit
git reset --soft HEAD~1   # undo commit, keep changes staged
git reset --mixed HEAD~1  # undo commit, keep changes unstaged (default)
git reset --hard HEAD~1   # undo commit, discard all changes

# Recover lost commits
git reflog
git checkout abc1234              # inspect
git branch recovery abc1234       # create branch from it

# Remove file from git but keep on disk
git rm --cached file.txt

# Remove file completely
git rm file.txt
```

## Stashing

```bash
# Stash current changes
git stash
git stash push -m "description"

# Stash including untracked files
git stash -u

# Stash including ignored files
git stash -a

# List stashes
git stash list

# Apply last stash (keep in stash list)
git stash apply

# Apply and remove from stash list
git stash pop

# Apply a specific stash
git stash apply stash@{2}

# Show stash contents
git stash show -p stash@{0}

# Drop a specific stash
git stash drop stash@{1}

# Clear all stashes
git stash clear

# Create branch from stash
git stash branch new-branch stash@{0}
```

## Tags

```bash
# List tags
git tag
git tag -l "v1.*"

# Create lightweight tag
git tag v1.0

# Create annotated tag (recommended)
git tag -a v1.0 -m "Release version 1.0"

# Tag a specific commit
git tag -a v0.9 abc1234 -m "Retroactive tag"

# Show tag info
git show v1.0

# Delete tag
git tag -d v1.0

# Delete remote tag
git push origin --delete v1.0

# Push tags
git push origin v1.0
git push --tags
```

## Advanced

### Cherry-pick

```bash
# Apply a specific commit to current branch
git cherry-pick abc1234

# Cherry-pick without committing
git cherry-pick --no-commit abc1234

# Cherry-pick a range
git cherry-pick abc1234..def5678

# Abort cherry-pick
git cherry-pick --abort
```

### Worktrees

```bash
# Add a worktree (checkout branch in separate directory)
git worktree add ../feature-x feature-x

# Add worktree with new branch
git worktree add -b hotfix ../hotfix main

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../feature-x

# Prune stale worktree entries
git worktree prune
```

### Bisect (Find Bug-Introducing Commit)

```bash
# Start bisect
git bisect start

# Mark current as bad
git bisect bad

# Mark a known good commit
git bisect good v1.0

# Git checks out a middle commit — test it, then:
git bisect good    # or
git bisect bad

# Repeat until found, then:
git bisect reset

# Automated bisect with a test script
git bisect start HEAD v1.0
git bisect run ./test-script.sh
```

### Interactive Rebase

```bash
# Rebase last 5 commits interactively
git rebase -i HEAD~5

# Commands in the editor:
# pick   — keep commit as-is
# reword — change commit message
# edit   — stop to amend
# squash — merge into previous commit (keep message)
# fixup  — merge into previous commit (discard message)
# drop   — remove commit

# Squash last 3 commits
git rebase -i HEAD~3
# Change "pick" to "squash" (or "s") for the 2nd and 3rd commits
```

### Submodules

```bash
# Add submodule
git submodule add https://github.com/user/repo.git path/to/submodule

# Clone repo with submodules
git clone --recurse-submodules https://github.com/user/repo.git

# Initialize submodules after clone
git submodule update --init --recursive

# Update submodules to latest
git submodule update --remote

# Remove submodule
git submodule deinit path/to/submodule
git rm path/to/submodule
rm -rf .git/modules/path/to/submodule
```

### Useful Patterns

```bash
# See what changed in a file over time
git log -p --follow file.txt

# Find when a line was added
git log -S "searchString" --oneline

# Find when a regex appeared
git log -G "regex" --oneline

# List all files tracked by git
git ls-files

# Show file at a specific revision
git show HEAD~3:path/to/file.txt

# Archive a branch
git archive --format=tar.gz --prefix=project/ HEAD > project.tar.gz

# Clean untracked files (dry run first)
git clean -nd              # dry run
git clean -fd              # force delete

# Sparse checkout (only certain paths)
git sparse-checkout init
git sparse-checkout set src/ docs/

# Shallow clone
git clone --depth 1 https://github.com/user/repo.git
```

## .gitignore

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

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log

# Negate (un-ignore)
!important.log
```

```bash
# Check if a file is ignored (and why)
git check-ignore -v file.txt

# Global gitignore
git config --global core.excludesFile ~/.config/git/ignore
```
