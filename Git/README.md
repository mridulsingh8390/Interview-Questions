# Git — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [Git Fundamentals](#git-fundamentals)
- [Branching & Merging](#branching-merging)
- [Remote Operations](#remote-operations)
- [History & Rewriting](#history-rewriting)
- [Advanced Operations](#advanced-operations)
- [Git Hooks](#git-hooks)
- [Git LFS & Attributes](#git-lfs)
- [Monorepo Strategies](#monorepo)
- [Security: GPG Signing](#gpg-signing)
- [Git Workflows](#git-workflows)
- [Git Internals](#git-internals)
- [Master Cheatsheet](#master-cheatsheet)

---

## Git Fundamentals

### 🟢 Q1. What are the three states of Git and how do files move between them?

**Explanation:**
Git tracks files across three areas:
- **Working Directory** — files on disk, may be modified
- **Staging Area (Index)** — snapshot prepared for next commit
- **Repository (.git)** — committed history

```bash
git status                          # See state of all files
git add file.txt                    # Working → Staging
git add .                           # Stage all changes
git add -p                          # Interactive patch staging (hunk by hunk)
git commit -m "message"             # Staging → Repository
git commit -am "message"            # Stage tracked files + commit
git restore file.txt                # Discard working dir changes
git restore --staged file.txt       # Unstage (Staging → Working)
git diff                            # Working vs Staging
git diff --staged                   # Staging vs Last commit
git diff HEAD                       # Working vs Last commit
git diff HEAD~1 HEAD                # Compare two commits
git diff main..feature              # Compare branches
git diff main...feature             # Compare from common ancestor
```

---

### 🟢 Q2. How does git diff work and what are its options?

```bash
git diff                            # Unstaged changes
git diff --staged                   # Staged changes
git diff HEAD                       # All changes (staged + unstaged)
git diff <commit1> <commit2>        # Between two commits
git diff <branch1>..<branch2>       # Tip of branch1 vs tip of branch2
git diff <branch1>...<branch2>      # From merge-base (common ancestor)
git diff --name-only                # Only file names
git diff --name-status              # File names + status (M/A/D)
git diff --stat                     # Summary with stats
git diff -- path/to/file            # Diff specific file only
git diff --word-diff                # Show word-level diffs
git diff --ignore-whitespace        # Ignore whitespace changes
git diff --check                    # Check for whitespace errors
```

---

### 🟢 Q3. What is .gitignore and .gitattributes?

```bash
# .gitignore — exclude files from tracking
*.log
*.tmp
node_modules/
dist/
.env
.DS_Store
*.pyc
__pycache__/
*.class
target/
.terraform/
*.tfstate
*.tfstate.backup

# Negate (re-include)
!important.log

# Global gitignore (applies to all repos)
git config --global core.excludesFile ~/.gitignore_global
```

```ini
# .gitattributes — control how Git handles files

# Normalize line endings to LF in repo, checkout as OS native
* text=auto

# Force LF for shell scripts (always LF regardless of OS)
*.sh text eol=lf
Makefile text eol=lf

# Force CRLF for Windows batch files
*.bat text eol=crlf

# Mark binary files — skip diff/merge
*.png binary
*.jpg binary
*.gif binary
*.pdf binary
*.zip binary
*.gz binary
*.jar binary

# Custom diff driver for minified files
*.min.js diff=minified

# Custom merge driver for lock files
package-lock.json merge=union
yarn.lock merge=union

# Export ignore — excluded from git archive exports
.gitignore export-ignore
.gitattributes export-ignore
tests/ export-ignore

# Linguist overrides (GitHub language stats)
vendor/** linguist-vendored
docs/** linguist-documentation
*.generated.ts linguist-generated
```

---

## Branching & Merging

### 🟢 Q4. How do you create and manage branches?

```bash
git branch                          # List local branches
git branch -a                       # List all branches (including remote)
git branch -r                       # List remote branches
git branch feature/login            # Create branch
git checkout feature/login          # Switch to branch
git checkout -b feature/login       # Create + switch
git switch -c feature/login         # Modern: create + switch
git switch main                     # Modern: switch

git branch -d feature/login         # Delete (safe — only if merged)
git branch -D feature/login         # Force delete
git push origin --delete feature/login  # Delete remote branch

git branch -m old-name new-name     # Rename branch
git branch -M main                  # Force rename to main

# Track remote branch
git checkout --track origin/feature/login
git branch -u origin/feature/login  # Set upstream for existing branch

# Prune stale remote tracking refs
git remote prune origin
git fetch --prune
```

---

### 🟡 Q5. What is the difference between merge, rebase, and squash?

**Explanation:**
- **merge** — creates a merge commit, preserves full history, non-destructive
- **rebase** — rewrites commits onto another base, linear history, rewrites SHAs (don't rebase shared branches)
- **squash** — combines multiple commits into one before merging, clean history

```bash
# ===== MERGE =====
git checkout main
git merge feature/login             # Fast-forward if possible
git merge --no-ff feature/login     # Always create merge commit (preserves branch topology)
git merge --squash feature/login    # Squash all commits into staged changes
git merge --abort                   # Abort in-progress merge

# ===== REBASE =====
git checkout feature/login
git rebase main                     # Replay feature commits on top of main
git rebase --interactive HEAD~5     # Interactive rebase last 5 commits
git rebase --onto main feature/base feature/sub  # Rebase sub onto main, skipping base
git rebase --abort                  # Abort
git rebase --continue               # After resolving conflicts
git rebase --skip                   # Skip current conflicting commit

# Interactive rebase options (editor opens):
# pick   = use commit as-is
# reword = use commit but edit message
# edit   = use commit but pause for amending
# squash = meld into previous commit
# fixup  = meld into previous, discard message
# drop   = remove commit entirely

# ===== SQUASH MERGE =====
git checkout main
git merge --squash feature/login    # Stage all changes
git commit -m "feat: add login feature"  # Single clean commit

# ===== FAST-FORWARD CHECK =====
git merge --ff-only feature/login   # Fail if FF not possible
```

---

### 🟡 Q6. How do you resolve merge conflicts?

```bash
# When merge/rebase hits a conflict:
git status                          # Shows conflicted files

# Conflict markers in file:
# <<<<<<< HEAD
# your changes on current branch
# =======
# incoming changes from other branch
# >>>>>>> feature/login

# Option 1: Manual edit — remove markers, keep desired code
code conflicted-file.js

# Option 2: Use a merge tool
git mergetool                       # Opens configured tool
git config --global merge.tool vimdiff
git config --global merge.tool vscode
# VSCode setting: git config --global merge.tool code

# Option 3: Accept ours/theirs entirely
git checkout --ours file.txt        # Keep current branch version
git checkout --theirs file.txt      # Accept incoming version
git add file.txt

# After resolving
git add .                           # Stage resolved files
git commit                          # Complete merge
# Or for rebase:
git rebase --continue

# Abort if needed
git merge --abort
git rebase --abort

# Preview what will conflict before merging
git merge --no-commit --no-ff feature/login
git diff --cached                   # See what would be committed
git merge --abort                   # Back out
```

---

## Remote Operations

### 🟢 Q7. How do you work with remote repositories?

```bash
git remote -v                       # List remotes
git remote add origin https://github.com/org/repo.git
git remote add upstream https://github.com/upstream/repo.git
git remote set-url origin git@github.com:org/repo.git
git remote rename origin old-origin
git remote remove upstream

# Fetch vs Pull
git fetch                           # Download remote changes (no merge)
git fetch origin                    # Fetch specific remote
git fetch --all                     # Fetch all remotes
git pull                            # fetch + merge (or rebase)
git pull --rebase                   # fetch + rebase (cleaner history)
git pull --ff-only                  # Fail if can't fast-forward

# Push
git push origin main
git push -u origin feature/login    # Set upstream + push
git push --force-with-lease         # Safe force push (check upstream hasn't changed)
git push --force                    # Force push (DANGEROUS — use force-with-lease instead)
git push origin --tags              # Push all tags
git push origin v1.2.3              # Push specific tag
git push origin --delete feature/old  # Delete remote branch
```

---

### 🟡 Q8. What is cherry-pick and when do you use it?

```bash
# Apply specific commit(s) to current branch
git cherry-pick abc1234

# Cherry-pick a range
git cherry-pick abc1234..def5678    # Range (exclusive start)
git cherry-pick abc1234^..def5678   # Range (inclusive start)

# Cherry-pick from another branch
git cherry-pick origin/hotfix/security-patch

# Options
git cherry-pick --no-commit abc1234    # Stage but don't commit
git cherry-pick --edit abc1234         # Edit commit message
git cherry-pick --signoff abc1234      # Add Signed-off-by

# When conflicts occur:
git cherry-pick --continue
git cherry-pick --abort
git cherry-pick --skip              # Skip conflicting commit

# Use case: hotfix applied to main, need it in release branch too
git checkout release/v1.2
git cherry-pick $(git log main --oneline | grep "hotfix:" | head -1 | awk '{print $1}')
```

---

## History & Rewriting

### 🟡 Q9. How do you view and search Git history?

```bash
git log
git log --oneline                   # Compact
git log --oneline --graph --all     # Visual branch graph
git log -n 10                       # Last 10 commits
git log --author="John"             # Filter by author
git log --since="2024-01-01"        # Filter by date
git log --until="2024-12-31"
git log --grep="fix:"               # Search commit messages
git log -S "function login"         # Pickaxe — find commits adding/removing string
git log -G "pattern.*regex"         # Pickaxe with regex
git log --all --full-history -- deleted-file.txt  # Find deleted file
git log main..feature               # Commits in feature not in main
git log feature..main               # Commits in main not in feature
git log --merges                    # Only merge commits
git log --no-merges                 # Exclude merge commits
git log --format="%H %an %ae %s"    # Custom format
git log --stat                      # Files changed per commit
git log -p                          # Show diffs
git log -p -- path/to/file          # History + diffs for specific file

# Show a specific commit
git show abc1234
git show HEAD
git show HEAD~2                     # 2 commits ago
git show HEAD~2:path/to/file        # File content at that commit

# Blame
git blame file.txt
git blame -L 10,20 file.txt         # Lines 10-20 only
git blame --ignore-rev abc1234      # Ignore a refactor commit
git blame --ignore-revs-file .git-blame-ignore-revs
```

---

### 🔴 Q10. What is git reset vs git revert?

```bash
# ===== GIT RESET — rewrite history (LOCAL only) =====
# --soft: move HEAD, keep staging + working dir
git reset --soft HEAD~1             # Undo last commit, keep changes staged
git reset --soft abc1234

# --mixed (default): move HEAD, clear staging, keep working dir
git reset HEAD~1                    # Undo commit + unstage, keep files
git reset abc1234

# --hard: move HEAD, clear staging AND working dir (DESTRUCTIVE)
git reset --hard HEAD~1             # Undo commit + discard all changes
git reset --hard origin/main        # Reset to match remote exactly

# ===== GIT REVERT — safe, creates new commit =====
git revert HEAD                     # Create commit that undoes last commit
git revert HEAD~3..HEAD             # Revert a range (creates multiple commits)
git revert --no-commit HEAD~3..HEAD # Stage reversals without committing
git revert -m 1 <merge-commit>      # Revert a merge (1 = keep first parent)

# When to use which:
# reset: local feature branch — rewrite before pushing
# revert: already pushed to shared branch — safe, preserves history

# Fix last commit message
git commit --amend -m "new message"   # Only if not pushed!

# Add forgotten file to last commit
git add forgotten.txt
git commit --amend --no-edit          # Keep same message
```

---

### 🔴 Q11. What is git reflog and how does it save you?

```bash
# reflog records every HEAD movement (even reset --hard, rebase, etc.)
git reflog                          # Show reflog
git reflog show main                # Reflog for specific branch
git reflog --relative-date          # Show dates

# RECOVERY SCENARIOS:

# Scenario 1: Accidentally reset --hard
git reset --hard HEAD~5             # Oops, lost 5 commits
git reflog                          # Find the SHA before reset
# HEAD@{3}: commit: my important work
git reset --hard HEAD@{3}           # Restore!

# Scenario 2: Deleted a branch accidentally
git branch -D feature/important
git reflog                          # Find last commit of deleted branch
git checkout -b feature/important HEAD@{2}

# Scenario 3: Bad rebase
git rebase main                     # Went wrong
git reflog                          # Find pre-rebase state
git reset --hard HEAD@{5}           # Go back before rebase

# Scenario 4: Lost stash
git stash drop
git fsck --unreachable | grep commit  # Find dangling commits
git show <sha>                        # Preview
git stash apply <sha>                 # Recover

# Reflog expiry (default 90 days)
git config gc.reflogExpire 180.days.ago
```

---

## Advanced Operations

### 🟡 Q12. How does git stash work?

```bash
git stash                           # Stash working + staged changes
git stash push -m "WIP: login form" # With message
git stash push -u                   # Include untracked files
git stash push --all                # Include ignored files too
git stash push -- path/to/file      # Stash specific files

git stash list                      # List stashes
git stash show                      # Show latest stash diff summary
git stash show -p                   # Show full diff
git stash show stash@{2}            # Show specific stash

git stash pop                       # Apply latest + remove from stash
git stash apply                     # Apply latest, keep in stash
git stash apply stash@{2}           # Apply specific stash

git stash drop                      # Remove latest stash
git stash drop stash@{2}            # Remove specific stash
git stash clear                     # Remove all stashes

# Create branch from stash
git stash branch feature/recovery stash@{1}

# Partial stash — interactive
git stash push -p                   # Choose hunks to stash
```

---

### 🟡 Q13. How do you tag releases?

```bash
# Lightweight tag (just a pointer)
git tag v1.0.0

# Annotated tag (full object with message, tagger, date)
git tag -a v1.0.0 -m "Release v1.0.0 - initial release"
git tag -a v1.0.0 abc1234 -m "Tag older commit"

# List tags
git tag
git tag -l "v1.*"                   # Filter by pattern
git tag --sort=-version:refname     # Sort by version (newest first)

# Show tag details
git show v1.0.0

# Push tags
git push origin v1.0.0             # Push single tag
git push origin --tags             # Push all tags
git push --follow-tags             # Push commits + annotated tags

# Delete tags
git tag -d v1.0.0                  # Delete local
git push origin --delete v1.0.0    # Delete remote

# Sign a tag (GPG)
git tag -s v1.0.0 -m "Signed release"
git tag -v v1.0.0                  # Verify signature

# Check out a tag (detached HEAD)
git checkout v1.0.0
```

---

### 🔴 Q14. What is git bisect?

```bash
# Binary search through commits to find which one introduced a bug

# Start bisect
git bisect start
git bisect bad                      # Current commit is bad
git bisect good v1.0.0              # v1.0.0 was good

# Git checks out middle commit
# Test your application...
git bisect good                     # This commit is OK
git bisect bad                      # This commit has the bug

# Git halves the remaining commits each time
# Eventually: "abc1234 is the first bad commit"

# Automated bisect with test script
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./run-tests.sh       # Script returns 0=good, 1=bad

# End bisect (returns to original HEAD)
git bisect reset

# Bisect with terms (skip "good/bad" for non-bug searches)
git bisect start --term-old fast --term-new slow
git bisect fast v1.0.0
git bisect slow HEAD
git bisect run ./benchmark.sh
```

---

### 🔴 Q15. How do submodules and subtrees work?

```bash
# ===== SUBMODULES =====
# Add submodule
git submodule add https://github.com/org/lib.git libs/mylib
git submodule add -b main https://github.com/org/lib.git libs/mylib

# Clone repo with submodules
git clone --recurse-submodules https://github.com/org/repo.git
# Or after cloning:
git submodule init
git submodule update
git submodule update --init --recursive   # All nested submodules

# Update submodule to latest remote commit
git submodule update --remote libs/mylib
git submodule update --remote --merge     # Merge new commits

# Run command in all submodules
git submodule foreach git pull origin main
git submodule foreach --recursive git status

# Remove submodule
git submodule deinit libs/mylib
git rm libs/mylib
rm -rf .git/modules/libs/mylib

# ===== SUBTREES (simpler alternative to submodules) =====
# Add subtree
git subtree add --prefix=libs/mylib \
  https://github.com/org/lib.git main --squash

# Pull updates from subtree remote
git subtree pull --prefix=libs/mylib \
  https://github.com/org/lib.git main --squash

# Push changes back to subtree remote
git subtree push --prefix=libs/mylib \
  https://github.com/org/lib.git main

# Split subtree into its own branch
git subtree split --prefix=libs/mylib --branch mylib-split
```

---

### 🔴 Q16. How do git worktree and sparse-checkout work?

```bash
# ===== GIT WORKTREE =====
# Multiple working directories from same repo (no need to clone again)

# Add worktree for a branch
git worktree add ../hotfix-branch hotfix/v1.2.1
# Now you can work in ../hotfix-branch while still in main dir

# Add worktree with new branch
git worktree add -b feature/parallel ../feature-work main

# List worktrees
git worktree list

# Remove worktree
git worktree remove ../hotfix-branch
git worktree prune                  # Clean up stale worktrees

# Use case: work on hotfix while keeping main branch clean
# No stashing needed!

# ===== SPARSE CHECKOUT =====
# Only checkout a subset of files (huge monorepos)

# Enable sparse checkout
git sparse-checkout init
git sparse-checkout set services/api services/auth  # Only these paths
git sparse-checkout list

# Cone mode (faster, only directories)
git sparse-checkout init --cone
git sparse-checkout set services/api

# Add more paths
git sparse-checkout add services/billing

# Disable (get full checkout)
git sparse-checkout disable

# Clone with sparse checkout
git clone --filter=blob:none --sparse https://github.com/org/monorepo.git
cd monorepo
git sparse-checkout set services/api
```

---

## Git Hooks

### 🟡 Q17. What are Git hooks and how do you use them?

**Explanation:**
Git hooks are scripts that run automatically at key points in Git's workflow. They live in `.git/hooks/` (local, not shared) or can be managed with tools like **pre-commit** (shared hooks). Hooks are executable shell/Python/Node scripts.

```bash
# Hook files live in .git/hooks/
ls .git/hooks/
# applypatch-msg   post-commit    post-update  pre-commit
# commit-msg       post-merge     pre-applypatch  pre-push
# post-checkout    post-receive   pre-rebase   prepare-commit-msg

# Make a hook executable
chmod +x .git/hooks/pre-commit
```

```bash
#!/bin/bash
# .git/hooks/pre-commit — runs before every commit
# Exit non-zero to abort commit

set -e

echo "Running pre-commit checks..."

# 1. Run linter
echo "Running ESLint..."
npx eslint src/ --ext .ts,.tsx
if [ $? -ne 0 ]; then
  echo "❌ ESLint failed. Fix errors before committing."
  exit 1
fi

# 2. Run tests
echo "Running tests..."
npm test -- --passWithNoTests
if [ $? -ne 0 ]; then
  echo "❌ Tests failed."
  exit 1
fi

# 3. Check for debug statements
if git diff --cached | grep -E "console\.log|debugger|TODO:|FIXME:" ; then
  echo "⚠️  Found console.log/debugger/TODO. Are you sure?"
  # Don't exit — just warn
fi

echo "✅ Pre-commit checks passed!"
```

```bash
#!/bin/bash
# .git/hooks/commit-msg — validate commit message format
# $1 = path to temp file containing commit message

commit_msg=$(cat "$1")

# Enforce Conventional Commits format
pattern="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,72}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
  echo "❌ Commit message doesn't follow Conventional Commits format"
  echo "Expected: type(scope): description"
  echo "Types: feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert"
  echo "Example: feat(auth): add JWT token refresh"
  exit 1
fi

# Check length
first_line=$(echo "$commit_msg" | head -1)
if [ ${#first_line} -gt 72 ]; then
  echo "❌ First line too long (${#first_line} chars, max 72)"
  exit 1
fi

echo "✅ Commit message OK"
```

```bash
#!/bin/bash
# .git/hooks/pre-push — runs before push to remote

protected_branch="main"
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

if [ "$current_branch" = "$protected_branch" ]; then
  echo "❌ Direct push to '$protected_branch' is not allowed."
  echo "Please create a feature branch and submit a PR."
  exit 1
fi

# Run full test suite before pushing
echo "Running test suite before push..."
npm run test:ci
```

```yaml
# ===== USING pre-commit FRAMEWORK (shareable hooks) =====
# .pre-commit-config.yaml — committed to repo, shared with team
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key
      - id: no-commit-to-branch
        args: [--branch, main, --branch, master]

  - repo: https://github.com/psf/black
    rev: 23.12.0
    hooks:
      - id: black

  - repo: https://github.com/PyCQA/flake8
    rev: 7.0.0
    hooks:
      - id: flake8

  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.13.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

```bash
# Install pre-commit
pip install pre-commit
pre-commit install                  # Install hooks in .git/hooks/
pre-commit install --hook-type commit-msg

# Run manually
pre-commit run --all-files
pre-commit run black                # Run specific hook

# Bypass hooks (emergency)
git commit --no-verify -m "emergency hotfix"
git push --no-verify
```

---

## Git LFS & Attributes

### 🟡 Q18. What is Git LFS and how do you use it?

**Explanation:**
Git LFS (Large File Storage) replaces large binary files with text pointers in Git, storing the actual content on a remote LFS server. This keeps the main repo small and fast, essential for repos containing assets, models, datasets, or binaries.

```bash
# Install Git LFS
git lfs install                     # One-time setup per user
# (installs git lfs hooks into .git/hooks)

# Track file types
git lfs track "*.psd"
git lfs track "*.png"
git lfs track "*.mp4"
git lfs track "datasets/**"
git lfs track "models/*.bin"

# This creates/updates .gitattributes:
cat .gitattributes
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.png filter=lfs diff=lfs merge=lfs -text

# Commit .gitattributes
git add .gitattributes
git commit -m "chore: add LFS tracking"

# Normal git operations work transparently
git add large-model.bin
git commit -m "add model"
git push                            # LFS files upload automatically

# List LFS files
git lfs ls-files
git lfs ls-files --size             # Show sizes

# Check LFS status
git lfs status

# Pull LFS files
git lfs pull                        # Download all LFS files
git lfs pull --include "models/"    # Only specific paths

# Fetch without downloading LFS content (fast clone)
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/org/repo.git
git lfs pull --include "models/small-model.bin"  # Then pull selectively

# Migrate existing files to LFS
git lfs migrate import --include="*.zip,*.tar.gz" --everything
git lfs migrate import --include="*.psd" --include-ref=main
```

---

## Monorepo Strategies

### 🔴 Q19. How do you manage monorepos with Git?

**Explanation:**
A monorepo stores multiple services/packages in one repo. Tools like **Nx**, **Turborepo**, **Bazel**, and **Rush** layer on top of Git to manage builds efficiently. Key Git features for monorepos: sparse-checkout, partial clone, worktrees, and commit filtering.

```bash
# ===== PARTIAL CLONE — don't download all objects =====
git clone --filter=blob:none https://github.com/org/monorepo.git   # No blobs
git clone --filter=tree:0 https://github.com/org/monorepo.git      # No trees at checkout

# ===== SPARSE CHECKOUT — work on subset =====
git clone --filter=blob:none --no-checkout https://github.com/org/monorepo.git
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set services/api shared/utils
git checkout main

# ===== CODEOWNERS — assign owners per path =====
cat > .github/CODEOWNERS << 'EOF'
# .github/CODEOWNERS
*                        @org/platform-team     # Default owner

/services/api/           @org/api-team
/services/auth/          @org/security-team
/services/frontend/      @org/frontend-team
/shared/                 @org/platform-team
/infrastructure/         @org/devops-team
/docs/                   @org/all-teams
EOF

# ===== ONLY CHANGED PACKAGES in CI =====
# Detect which services changed
changed_services=$(git diff --name-only HEAD~1 HEAD \
  | grep "^services/" \
  | cut -d'/' -f2 \
  | sort -u)

for service in $changed_services; do
  echo "Building and testing: $service"
  cd services/$service
  make test
  make build
  cd ../..
done

# ===== SPLIT MONOREPO SUBTREE TO OWN REPO =====
git subtree split --prefix=services/api -b api-only
git remote add api-remote https://github.com/org/api-service.git
git push api-remote api-only:main

# ===== SQUASH HISTORY PER SERVICE =====
# When splitting from monorepo
git filter-branch --subdirectory-filter services/api main
# or use git-filter-repo (faster, recommended)
pip install git-filter-repo
git filter-repo --path services/api --path-rename services/api/:
```

---

## Security: GPG Signing

### 🟡 Q20. How do you sign commits with GPG?

```bash
# Generate GPG key
gpg --full-generate-key
# Choose: RSA and RSA, 4096 bits, no expiration

# List keys
gpg --list-secret-keys --keyid-format LONG

# Configure Git to use your key
# From output: sec   rsa4096/ABCDEF1234567890
git config --global user.signingkey ABCDEF1234567890
git config --global commit.gpgsign true      # Auto-sign all commits
git config --global tag.gpgsign true         # Auto-sign all tags

# Sign a commit manually
git commit -S -m "feat: signed commit"

# Sign a tag
git tag -s v1.0.0 -m "Signed release"

# Verify commit signature
git log --show-signature
git verify-commit HEAD
git verify-tag v1.0.0

# Export public key (add to GitHub/GitLab)
gpg --armor --export ABCDEF1234567890

# Use SSH key for signing (simpler — Git 2.34+)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

# Allowed signers file (for verification)
echo "user@example.com namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" >> ~/.config/git/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers
git verify-commit HEAD
```

---

## Git Workflows

### 🟡 Q21. What are the common Git branching strategies?

```
# ===== GITFLOW =====
main          ─────────────────────────────●──────────
                                           │ v1.0 tag
release/v1.0  ───────────────────────────●─┘
                                         │
develop       ────────────────────●──────┤
              ╱          ╲       ╱       │
feature/A  ──●──●──●      ●─────╱        │
feature/B        ●──●──●─╱               │
hotfix/fix                        ●──────┘ (merged to main + develop)

Branches:
  main       — production, tagged releases only
  develop    — integration branch
  feature/*  — from develop, merged back to develop
  release/*  — from develop, merged to main + develop
  hotfix/*   — from main, merged to main + develop

# ===== GITHUB FLOW (simpler) =====
main          ─────────────────────────────────●
                                               │
feature/login ────●──●──●──── PR ─── Review ──┘

Rules:
  - main is always deployable
  - All work in feature branches
  - PR → review → merge to main → deploy

# ===== TRUNK-BASED DEVELOPMENT (TBD) =====
main          ─────●──●──●──●──●──────────────
              ╱               ╲
short-lived  ●──●──── < 2 days ──→ merge
feature

Rules:
  - Integrate to main at least daily
  - Feature branches live max 1-2 days
  - Use feature flags for incomplete features
  - CI must pass before merge
```

---

## Git Internals

### 🔴 Q22. How does Git store data internally?

```bash
# Git object types: blob, tree, commit, tag
# All stored in .git/objects/ as zlib-compressed SHA1 content-addressed files

# Explore a commit
git cat-file -t HEAD              # Type: commit
git cat-file -p HEAD              # Print content
# tree abc123...
# parent def456...
# author John Doe <j@example.com> 1704067200 +0000
# committer John Doe <j@example.com> 1704067200 +0000
# feat: add login

# Explore a tree
git cat-file -p HEAD^{tree}
# 100644 blob a1b2c3... README.md
# 040000 tree d4e5f6... src
# 100755 blob g7h8i9... script.sh

# Explore a blob
git cat-file -p HEAD:README.md    # Print file content at HEAD

# File modes:
# 100644 = regular file
# 100755 = executable file
# 120000 = symbolic link
# 040000 = directory (tree)
# 160000 = gitlink (submodule)

# Find object by hash
find .git/objects -type f | head -5
# .git/objects/ab/cdef1234567890...

# Verify repository integrity
git fsck
git fsck --unreachable | grep blob  # Find dangling objects

# Garbage collection
git gc                             # Run GC
git gc --aggressive                # Thorough (slower)
git prune                          # Remove unreachable objects
git count-objects -v               # Show object database stats

# Packfiles
git verify-pack -v .git/objects/pack/pack-*.idx | sort -k3 -n | tail -10
# Shows largest objects

# The INDEX (staging area)
git ls-files --stage               # Show index content
```

---

## Master Cheatsheet

### Daily Commands
```bash
git status                         # What's going on
git add -p                         # Stage interactively
git commit -m "type: message"      # Commit
git push -u origin branch          # Push + set upstream
git pull --rebase                  # Pull with rebase
git log --oneline --graph --all    # Visual history
git diff HEAD                      # All changes vs last commit
git stash                          # Save work in progress
git stash pop                      # Restore WIP
```

### Branch Management
```bash
git checkout -b feature/name       # Create + switch
git switch -c feature/name         # Modern syntax
git branch -d feature/name         # Delete (safe)
git push origin --delete feature   # Delete remote
git merge --no-ff feature          # Merge with commit
git rebase main                    # Rebase onto main
git rebase -i HEAD~5               # Interactive rebase
```

### Recovery
```bash
git reflog                         # History of HEAD movement
git reset --hard HEAD@{N}         # Go back to state N
git reset --soft HEAD~1            # Undo commit, keep staged
git revert HEAD                    # Safe undo (new commit)
git checkout -- file               # Discard file changes
git restore --staged file          # Unstage file
```

### Remote
```bash
git remote -v                      # List remotes
git fetch --prune                  # Fetch + clean stale refs
git push --force-with-lease        # Safe force push
git push origin --tags             # Push all tags
```

### Searching
```bash
git log -S "code string"           # Find commits adding/removing string
git log --grep "fix:"              # Search commit messages
git grep "function login"          # Search in files
git bisect start                   # Binary search for bug
git blame file.txt                 # Who changed each line
```

### Config
```bash
git config --global user.name "Name"
git config --global user.email "email"
git config --global core.editor "code --wait"
git config --global pull.rebase true
git config --global push.default current
git config --global commit.gpgsign true
git config --list --global
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Three states, staging | 🟢 |
| Q2 | git diff options | 🟢 |
| Q3 | .gitignore & .gitattributes | 🟢 |
| Q4 | Branch management | 🟢 |
| Q5 | Merge vs rebase vs squash | 🟡 |
| Q6 | Conflict resolution | 🟡 |
| Q7 | Remote operations | 🟢 |
| Q8 | Cherry-pick | 🟡 |
| Q9 | Viewing and searching history | 🟡 |
| Q10 | reset vs revert | 🔴 |
| Q11 | reflog and recovery | 🔴 |
| Q12 | git stash | 🟡 |
| Q13 | Tags and releases | 🟡 |
| Q14 | git bisect | 🔴 |
| Q15 | Submodules and subtrees | 🔴 |
| Q16 | git worktree and sparse-checkout | 🔴 |
| Q17 | Git hooks and pre-commit | 🟡 |
| Q18 | Git LFS | 🟡 |
| Q19 | Monorepo strategies | 🔴 |
| Q20 | GPG / SSH commit signing | 🟡 |
| Q21 | Branching strategies | 🟡 |
| Q22 | Git internals | 🔴 |
