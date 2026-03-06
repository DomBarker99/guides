# Git Commands Guide

A comprehensive guide to the git commands used in this project. This covers what each command does, how to use it, and how they fit together in a typical development workflow.

---

## Table of Contents

1. [Initial Setup](#initial-setup)
2. [Initializing a New Repository](#initializing-a-new-repository)
3. [Core Commands](#core-commands)
4. [Common Workflows](#common-workflows)
5. [Pull Request Workflow](#pull-request-workflow)
6. [Branch Protection and Repository Access](#branch-protection-and-repository-access)

---

## Initial Setup

### `git config`

Configures your git identity and preferences. This must be done before making commits so your contributions are properly attributed.

```bash
# Set your name (used in commit history)
git config --global user.name "Your Name"

# Set your email (should match your GitHub email)
git config --global user.email "youremail@example.com"
```

### `git config --global --list`

Displays all of your global git configuration settings. Use this to verify your setup is correct.

```bash
git config --global --list
```

Example output:

```
user.name=Your Name
user.email=youremail@example.com
core.editor=vim
```

---

## Initializing a New Repository

If you have an existing project on your computer that isn't tracked by Git yet and you want to push it to GitHub, here's how to set it up from scratch.

### Step 1: Initialize Git

Navigate to your project folder and initialize an empty Git repository:

```bash
cd your-project-folder
git init
```

This creates a hidden `.git` directory that Git uses to track changes. Your files are not tracked yet — you still need to add and commit them.

### Step 2: Stage and Commit Your Files

Add the files you want to track and make your first commit:

```bash
# Stage all files in the project
git add .

# Or stage specific files
git add index.html style.css

# Create the initial commit
git commit -m "Initial commit"
```

### Step 3: Create a Repository on GitHub

1. Go to [github.com](https://github.com) and click **"New repository"**
2. Give it a name (e.g., `my-project`)
3. **Do not** check "Add a README" or "Add .gitignore" — your project already has files, and initializing with these will cause conflicts when you push
4. Click **"Create repository"**

GitHub will show you a set of commands. You'll use the ones for pushing an existing repository.

### Step 4: Connect Your Local Repo to GitHub

Link your local repository to the remote on GitHub:

```bash
git remote add origin https://github.com/your-username/my-project.git
```

This tells Git where to push your code. `origin` is the conventional name for your primary remote.

You can verify the remote was added:

```bash
git remote -v
```

### Step 5: Rename the Default Branch to `main`

Older versions of Git create a default branch called `master`. Most projects now use `main` as the default. Rename it to stay consistent:

```bash
git branch -M main
```

The `-M` flag forces the rename even if a `main` branch already exists.

### Step 6: Push to GitHub

Push your code to the remote and set `origin main` as the default upstream branch:

```bash
git push -u origin main
```

The `-u` flag sets up tracking so that future `git push` and `git pull` commands will default to `origin main` without you having to specify it each time.

After this, your code is live on GitHub. Refresh the repository page to see your files.

---

## Core Commands

### `git status`

Shows the current state of your working directory and staging area. It tells you which files have been modified, which are staged for commit, and which are untracked.

```bash
git status
```

Example output:

```
On branch feature-login
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
        modified:   app.js

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        README.md
```

**When to use:** Run this frequently to understand what state your working directory is in before staging, committing, or switching branches.

---

### `git fetch`

Downloads new commits, branches, and tags from the remote repository **without** modifying your local files. It updates your remote-tracking branches (e.g., `origin/main`) so you can see what others have pushed.

```bash
# Fetch all updates from the remote
git fetch

# Fetch from a specific remote
git fetch origin
```

**Key point:** `git fetch` is safe — it never changes your working directory or current branch. It only updates your knowledge of what the remote looks like.

---

### `git pull`

Fetches changes from the remote **and** merges them into your current branch. This is essentially `git fetch` + `git merge` in one step.

```bash
# Pull updates for your current branch
git pull

# Pull from a specific remote and branch
git pull origin main
```

**When to use:** Before starting new work, pull the latest changes to make sure your branch is up to date.

**Warning:** If you have uncommitted local changes that conflict with the incoming changes, git will stop and ask you to resolve the conflicts.

---

### `git add`

Stages files for the next commit. Staging is the step between modifying a file and committing it — it lets you choose exactly which changes to include in your commit.

```bash
# Stage a specific file
git add app.js

# Stage multiple specific files
git add app.js README.md

# Stage all changed and new files in the current directory
git add .

# Stage all changed and new files in the entire repository
git add -A
```

**Key point:** Modifying a file does not automatically include it in your next commit. You must explicitly stage it with `git add`.

---

### `git restore`

Unstages files or discards uncommitted changes. This command has both safe and destructive usages.

**Safe — Unstage a file (keeps your changes, just removes it from staging):**
```bash
git restore --staged app.js
```

**DESTRUCTIVE — Discard local changes to a file:**
```bash
git restore app.js
```
> **DANGER: This permanently deletes your uncommitted changes to that file. There is no undo. If you haven't committed or stashed your work, it is gone forever.**

**DESTRUCTIVE — Discard ALL local changes:**
```bash
git restore .
```
> **DANGER: This permanently deletes ALL uncommitted changes across every file in the current directory. There is no confirmation prompt. Make absolutely sure your work is committed or stashed before running this.**

---

### `git rm`

Removes a file from git tracking. The removal will be recorded in the next commit.

**Safe — Stop tracking a file but keep it on disk:**
```bash
git rm --cached config.bak
```

**DESTRUCTIVE — Remove a file from the repo AND delete it from disk:**
```bash
git rm config.bak
```
> **DANGER: This deletes the file from your computer. If the file has uncommitted changes, they will be lost.**

**DESTRUCTIVE — Remove an entire directory from the repo AND delete it from disk:**
```bash
git rm -r legacy-folder/
```
> **DANGER: This recursively deletes the directory and all its contents from your computer. Double-check the path before running this.**

**When to use:** When you need to delete a file and have that deletion tracked by git. Simply deleting a file with `rm` will show it as "deleted" in `git status`, but you still need to stage that deletion with `git add` or use `git rm` instead.

---

### `git commit`

Records the staged changes as a new commit in the repository history. Every commit should have a clear, descriptive message explaining **what** changed and **why**.

```bash
# Commit with an inline message
git commit -m "Add user authentication endpoint"

# Commit with a multi-line message (opens your editor)
git commit

# Stage all tracked modified files and commit in one step
git commit -am "Fix validation bug in login form"
```

**Commit message best practices:**
- Use the imperative mood: "Add feature" not "Added feature"
- Keep the first line under 72 characters
- Reference the issue number if applicable: "Fix scoring bug (#12)"

---

### `git push`

Uploads your local commits to the remote repository so others can see your work.

```bash
# Push the current branch to the remote
git push

# Push a specific branch
git push origin feature-login

# Push and set the upstream tracking branch (first push of a new branch)
git push -u origin feature-login
```

**Key point:** You must commit your changes locally before you can push them. `git push` sends commits, not individual file changes.

---

### `git checkout`

Switches between branches or restores files. This is how you move between different lines of development.

**Safe — Switch to an existing branch:**
```bash
git checkout main
```

**Safe — Create a new branch and switch to it:**
```bash
git checkout -b new-feature-branch
```

**DESTRUCTIVE — Restore a specific file to its last committed state:**
```bash
git checkout -- app.js
```
> **DANGER: This permanently deletes your uncommitted changes to that file — identical to `git restore app.js`. There is no undo. Make sure your work is committed or stashed first.**

**When to use:**
- Switching to `main` before pulling the latest changes
- Creating a new feature branch for your work
- Switching between branches when working on multiple features

---

### `git branch`

Lists, creates, or deletes branches. Branches let you work on features independently without affecting `main`.

```bash
**Safe — List and create branches:**
```bash
# List all local branches (current branch marked with *)
git branch

# List all branches including remote branches
git branch -a

# Create a new branch (does not switch to it)
git branch new-feature
```

**Safe — Delete a branch that has already been merged:**
```bash
git branch -d old-branch
```

**DESTRUCTIVE — Force delete a branch (even if it has unmerged work):**
```bash
git branch -D old-branch
```
> **DANGER: This deletes the branch even if it contains commits that haven't been merged anywhere. If those commits exist only on that branch, they are effectively lost.
```

---

### `git log`

Shows the commit history for the current branch. Useful for reviewing what changes have been made and by whom.

```bash
# Show full commit log
git log

# Show a condensed one-line-per-commit log
git log --oneline

# Show the last 5 commits
git log -5

# Show a graphical branch history
git log --oneline --graph --all
```

Example output for `git log --oneline`:

```
139b0c5 Added 4th Print Line
e5fedaf Added third print line
f6d683b Added second print line
dd0452d Delete cs-151-rock-paper-scissors.iml
```

---

### `git reflog`

Shows a log of every recent action that moved `HEAD` — commits, pulls, merges, resets, checkouts, and more. Each entry includes a commit hash you can use to go back to that state. Think of it as your personal undo history.

```bash
# Show the reflog
git reflog

# Show only the last 10 entries
git reflog -10
```

Example output:

```
c2323a9 HEAD@{0}: pull origin hao-van-vo: Fast-forward
b6fd99f HEAD@{1}: reset: moving to origin/main
b6fd99f HEAD@{2}: pull origin main: Fast-forward
70284ce HEAD@{3}: commit: Add .idea/ and *.iml to gitignore
```

**When to use:** When you need to find a previous state of your branch — especially after an accidental pull, merge, or reset. The reflog is your safety net for recovering from mistakes.

**Key point:** The reflog is local only. It is not shared with the remote and entries expire after 90 days by default.

---

### `git reset`

Moves your branch pointer to a different commit. This can be used to undo commits, unstage files, or revert your branch to a previous state.

**Safe — Unstage all files (keeps your changes):**
```bash
git reset
```

**Safe — Undo the last commit but keep the changes in your working directory:**
```bash
git reset --soft HEAD~1
```

**DESTRUCTIVE — Undo the last commit and discard all changes:**
```bash
git reset --hard HEAD~1
```
> **DANGER: This permanently deletes the commit and all associated changes. There is no undo.**

**DESTRUCTIVE — Reset your branch to a specific commit (found via `git reflog`):**
```bash
git reset --hard <commit-hash>
```
> **DANGER: All commits after the target are discarded along with any uncommitted changes. Double-check the commit hash with `git reflog` before running this.**

**When to use:** To undo an accidental pull or merge, or to move your branch back to a known good state. Always use `git reflog` first to find the right commit hash.

---

## Common Workflows

### Switching Branches

Switch to an existing branch before starting work:

```bash
# Switch to a feature branch
git checkout feature-login
```

### Updating Your Local Branch

Before starting work, make sure your branch has the latest changes from `main`:

```bash
# 1. Switch to your branch (if not already on it)
git checkout feature-login

# 2. Pull the latest changes from main into your branch
git pull origin main
```

This ensures you are working with the most up-to-date version of the codebase.

### Staging and Committing Changes

After making changes to your code:

```bash
# 1. Check what has changed
git status

# 2. Review your changes (optional but recommended)
git log --oneline

# 3. Stage the files you want to commit
git add app.js

# OR stage all changed files at once
git add .
```

**Note on `git add .`:** This stages every changed and new file in the current directory. You might worry about accidentally staging files that shouldn't be committed (like compiled binaries or IDE configuration folders). This is where the `.gitignore` file comes in — any files or directories listed in `.gitignore` are automatically excluded from staging. As long as your `.gitignore` is set up correctly, `git add .` will only stage the files you actually want.

```bash
# 4. Verify what is staged
git status

# 5. Commit with a descriptive message referencing the issue
git commit -m "Add input validation for login form (#5)"
```

### Pushing Your Work

After committing locally, push your branch to the remote:

```bash
git push origin feature-login
```

### Undoing Mistakes

**Safe — Unstage a file you accidentally staged (keeps your changes):**
```bash
git restore --staged app.js
```

**Safe — See what you're about to commit:**
```bash
git status
```

**DESTRUCTIVE — Discard local changes to a file:**
```bash
git restore app.js
```
> **DANGER: This permanently deletes your uncommitted changes to that file. There is no undo.**

**DESTRUCTIVE — Discard ALL local changes and match the remote branch exactly:**
```bash
git fetch origin
git reset --hard origin/feature-login
```
> **DANGER: This is the most destructive command in this guide. It permanently deletes ALL uncommitted and staged changes across every file and resets your branch to exactly match the remote. There is no undo. Only use this as an absolute last resort when you want to completely start over from the remote state.**

**DESTRUCTIVE — Undo a pull/merge and revert your branch to a previous state:**

If you accidentally pulled the wrong branch or merged something unintended, you can use `git reflog` and `git reset --hard` to go back to where you were before.

```bash
# 1. View recent history of where HEAD has been
git reflog

# Example output:
# c2323a9 HEAD@{0}: pull origin hao-van-vo: Fast-forward
# b6fd99f HEAD@{1}: reset: moving to origin/main

# 2. Find the entry BEFORE the unwanted pull (HEAD@{1} in this example)
#    and reset to that commit
git reset --hard b6fd99f
```

> **DANGER: `git reset --hard` permanently discards all commits after the target and any uncommitted changes. Make sure you are resetting to the correct commit. Double-check with `git reflog` before running the reset.**

**Tips:**
- `git reflog` shows every recent action (commits, pulls, resets, checkouts) with a corresponding commit hash — it is your safety net for finding previous states
- The reflog entry just **before** the unwanted action is typically the one you want to reset to
- If you have already pushed the unwanted commits to the remote, you will need `git push --force` after the reset (coordinate with your team before force-pushing)

---

## Pull Request Workflow

This is the typical process for working on a feature or bug fix and getting your changes merged into `main`.

### Step 1: Create and Update Your Branch

Create a feature branch from the latest `main`:

```bash
git checkout main
git pull origin main
git checkout -b feature-login
```

Or if your branch already exists, update it:

```bash
git checkout feature-login
git pull origin main
```

### Step 2: Make Your Changes

Make the necessary code changes on your branch, then stage and commit:

```bash
# Check your changes
git status

# Stage your changes
git add app.js

# Commit with a message referencing the issue number
git commit -m "Add login validation (closes #12)"
```

You can make multiple commits as you work. Each commit should represent a logical unit of change.

### Step 3: Push Your Branch

```bash
git push origin feature-login
```

### Step 4: Create the Pull Request on GitHub

1. Go to the repository on GitHub
2. Click **"Compare & pull request"** (or go to the Pull Requests tab and click **"New pull request"**)
3. Set the **base** branch to `main` and the **compare** branch to your feature branch (e.g., `feature-login`)
4. Fill in the PR details:
   - **Title:** A clear summary of what the PR does (e.g., "Add login validation")
   - **Description:** Explain what you changed, why, and reference the issue using `Closes #12` so it automatically closes when merged
5. Click **"Create pull request"**

### Step 5: Wait for Review

A reviewer will look over your PR. There are two possible outcomes:

- **Approved and merged** — your changes are now in `main`. Pull the latest to stay in sync:
  ```bash
  git checkout main
  git pull origin main
  ```
- **Changes requested** — the reviewer will leave comments explaining what needs to be fixed. Make the fixes on your branch, commit, and push again:
  ```bash
  # Fix the issues locally, then:
  git add app.js
  git commit -m "Address review feedback: fix validation edge case"
  git push origin feature-login
  ```
  The PR updates automatically with your new commits. No need to create a new PR.

---

## Branch Protection and Repository Access

Git commands control what happens locally, but GitHub adds layers of access control and branch protection on top. Understanding these layers is important for keeping your `main` branch safe — especially when working with collaborators.

### The Three Layers of Access Control

Access on GitHub is controlled in three layers, each building on the last:

| Layer | What It Controls |
|-------|-----------------|
| **Visibility** (public/private) | Who can **see** the repo |
| **Collaborators** (permissions) | Who can **write** to the repo |
| **Branch protection** | What **rules** writers must follow |

### Repository Visibility

- **Public repo** — anyone can see the code, clone the repo, and open pull requests. Only collaborators can push directly.
- **Private repo** — no one can see or access the repo unless you explicitly add them as a collaborator.

### Collaborator Roles

On personal GitHub repos, there are only two roles:

| Role | What They Can Do |
|------|-----------------|
| **Owner** (you) | Everything — push, merge, delete branches, change settings, manage collaborators |
| **Collaborator** | Push to branches, merge PRs, create/delete branches — but **cannot** change repo settings or manage collaborators |

There is no "read-only collaborator" on personal repos. If you add someone, they get write access.

**Who can do what by default:**

| Who | Public Repo | Private Repo |
|-----|------------|-------------|
| **You** (owner) | Full control | Full control |
| **Collaborators** | Read + Write | Read + Write |
| **Everyone else** | Read only (can fork + open PRs) | No access at all |

**Adding a collaborator:** Go to your repo's **Settings > Collaborators > Add people** and search by their GitHub username.

> **Note:** Organization repos (e.g., for a company or class) have more granular roles: **Read**, **Triage**, **Write**, **Maintain**, and **Admin**. This is how an organization owner can give members write access to their own branches while controlling who can merge to `main`. For personal repos, the simple model above is all you need.

### Why Branch Protection Matters

Without branch protection, any collaborator can:
- Push directly to `main`
- Merge their own PRs without anyone reviewing them
- Force-push and rewrite commit history

Even on a solo project, having no protection means **you** can accidentally push broken code directly to `main`, force-push and destroy history, or accidentally delete branches.

Branch protection fills that gap by enforcing rules that **all** writers must follow:

| Risk | Branch Protection Rule |
|------|----------------------|
| Collaborator pushes directly to `main` | **Require a pull request before merging** |
| Collaborator merges their own PR | **Require approvals** (they can't approve their own) |
| Collaborator force-pushes | **Do not allow force pushes** |

### Recommended Branch Protection Settings

To set up branch protection, go to your repo's **Settings > Branches > Add branch protection rule** and set the branch name pattern to `main`.

**For a solo or small public repo:**

| Setting | Enable? | Why |
|---------|---------|-----|
| **Do not allow force pushes** | Yes | Protects your commit history from accidental overwrites |
| **Do not allow deletions** | Yes | Prevents accidentally deleting `main` |
| **Require a pull request before merging** | Optional | Turn it on if you want the discipline of always working on branches. Skip it if it feels like overhead for a solo project. |
| **Require approvals** | No | There's no one else to approve — this would block you |

**For a collaborative repo (multiple contributors):**

| Setting | Enable? | Why |
|---------|---------|-----|
| **Do not allow force pushes** | Yes | Prevents anyone from rewriting shared history |
| **Do not allow deletions** | Yes | Prevents accidental deletion of `main` |
| **Require a pull request before merging** | Yes | Forces all changes to go through code review |
| **Require approvals** | Yes | Ensures at least one other person reviews before merging |

### The Typical Protected Repo Workflow

Once branch protection is enabled on `main`, the workflow looks like this:

1. Create a feature branch from `main`
2. Push your changes to that branch
3. Open a PR from your feature branch to `main`
4. Get a notification, review the changes, and either approve or request changes
5. Merge it (squash or merge commit)
6. Delete the feature branch (optional cleanup)

No one — including you — can push directly to `main` or merge without meeting the protection rules. This keeps the `main` branch stable and forces all changes through a reviewable process.

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git status` | See what has changed |
| `git fetch` | Download remote updates (no merge) |
| `git pull` | Download and merge remote updates |
| `git add <file>` | Stage a file for commit |
| `git restore <file>` | Discard local changes |
| `git restore --staged <file>` | Unstage a file |
| `git rm <file>` | Remove a file from the repo |
| `git commit -m "msg"` | Commit staged changes |
| `git push` | Upload commits to remote |
| `git checkout <branch>` | Switch branches |
| `git checkout -b <branch>` | Create and switch to new branch |
| `git branch` | List branches |
| `git log --oneline` | View commit history |
| `git reflog` | View history of all HEAD movements |
| `git reset --hard <hash>` | Reset branch to a specific commit |
| `git config --global --list` | View git configuration |
