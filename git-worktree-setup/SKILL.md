---
name: git-worktree-setup
description: Set up a bare git repository with worktrees for parallel feature development. Use when the user wants to clone a repo as a bare repo with per-branch worktrees so multiple Claude sessions can develop features simultaneously.
---

Set up a bare git repository with a worktrees directory for parallel development. Follow these steps exactly:

## Steps

1. **Clone as bare repo** into `<cwd>/repo.git`:
   ```
   git clone --bare <github-url> <cwd>/repo.git
   ```

2. **Configure remote fetch refspec** so remote tracking branches work:
   ```
   git -C <cwd>/repo.git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"
   git -C <cwd>/repo.git fetch origin
   ```

3. **Create the worktrees directory**: `<cwd>/worktrees/`

4. **Create a worktree for the default branch** (check HEAD to determine name, usually `master` or `main`):
   ```
   git -C <cwd>/repo.git worktree add <cwd>/worktrees/<default-branch> <default-branch>
   ```

5. **Create a helper script** at `<cwd>/new-worktree.sh` with this exact content:
   ```bash
   #!/usr/bin/env bash
   # Usage: ./new-worktree.sh <branch-name> [base-branch]
   # Creates a new worktree at /worktrees/<branch-name>
   # If the branch doesn't exist, it's created from <base-branch> (default: master)

   set -e

   REPO_DIR="$(cd "$(dirname "$0")/repo.git" && pwd)"
   WORKTREES_DIR="$(cd "$(dirname "$0")" && pwd)/worktrees"

   BRANCH="$1"
   BASE="${2:-master}"

   if [ -z "$BRANCH" ]; then
     echo "Usage: $0 <branch-name> [base-branch]"
     exit 1
   fi

   WORKTREE_PATH="$WORKTREES_DIR/$BRANCH"

   if [ -d "$WORKTREE_PATH" ]; then
     echo "Worktree already exists at $WORKTREE_PATH"
     exit 1
   fi

   # Check if branch exists locally
   if git -C "$REPO_DIR" show-ref --verify --quiet "refs/heads/$BRANCH"; then
     echo "Checking out existing branch '$BRANCH'..."
     git -C "$REPO_DIR" worktree add "$WORKTREE_PATH" "$BRANCH"
   else
     echo "Creating new branch '$BRANCH' from '$BASE'..."
     git -C "$REPO_DIR" worktree add -b "$BRANCH" "$WORKTREE_PATH" "$BASE"
   fi

   echo "Worktree ready at: $WORKTREE_PATH"
   ```

   Note: update `BASE="${2:-master}"` to `main` if the default branch is `main`.

## Final Structure

```
<cwd>/
├── repo.git/               # bare repository
├── worktrees/
│   └── <default-branch>/  # worktree for default branch
└── new-worktree.sh         # helper script
```

## After Setup

Tell the user:
- Run `./new-worktree.sh <branch-name>` to create a new worktree for a feature branch
- Open a separate `claude` session inside each `worktrees/<branch>` directory for parallel development
