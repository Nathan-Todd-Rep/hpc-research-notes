# GitHub Notes

## What Is Source Control?

Source control is a system that acts as a vault for your code, stored in the cloud.

- If your project breaks, you can restore it from a previous snapshot.
- The "vault" is called a **repository**.
- A snapshot is called a **commit**.

---

## Forking

A **fork** is GitHub's built-in way of making your own copy of someone else's repository under your own account.

### Why Fork?

- Your fork is completely independent — you can break things, experiment, delete files, and push half-finished work without affecting the original at all.
- The connection between a fork and the original is one-directional: you can pull updates *in* from the original whenever you want, but nothing flows back automatically.
- To send work back to the original (called a **pull request**), you do it intentionally and the original owner must approve it.

### Fork vs New Independent Repo

| | Fork | New Repo |
|---|---|---|
| Connected to original | Yes — can pull updates | No |
| Affects original | Never | Never |
| Best for | Research/collaboration on an existing project | Starting something brand new |

---

## Branches

A **branch** is a parallel copy of your code inside the same repository. Your `main` branch stays clean and working while you experiment freely on another branch. If the experiment goes wrong, just delete the branch.

### Why Use Branches?

- Lets you experiment without risking your stable code
- If something goes wrong, delete the branch — the rest of your repo is untouched
- Standard practice: keep `main` clean, do all work on a separate branch

### Branch Structure (Inkly Example)

```
your local machine
├── main        ← stable, don't touch
├── dev         ← the project's development branch
└── nathan-dev  ← personal experiment branch
        │
        └── pushed to → github.com/Nathan-Todd-Rep/hpc-ink-setup
```

### Creating and Pushing a Branch

```bash
git checkout -b <branch-name>    # create a new branch and switch to it
git push origin <branch-name>    # push it up to GitHub
```

### Switching Between Branches

```bash
git checkout main         # switch to main
git checkout nathan-dev   # switch back to your branch
git branch                # list all local branches (* marks current)
```

### Checking Your Status

```bash
git status            # shows current branch + any modified files
git branch            # lists all local branches, * marks current one
git log --oneline -5  # shows your last 5 commits

---

## GitHub CLI (`gh`)

The GitHub CLI lets you manage GitHub (create repos, forks, pull requests, etc.) from the terminal instead of the browser.

### Installing on Windows

```powershell
winget install --id GitHub.cli -e --source winget
```

### Authenticating

```bash
gh auth login
```

Choices to make during login:
1. Account → `GitHub.com`
2. Protocol → `HTTPS`
3. Authenticate Git with GitHub credentials → `Yes`
4. How to authenticate → `Login with a web browser`

### Forking a Repo via CLI

```bash
gh repo fork <owner>/<repo> --clone=false
```

The `--clone=false` flag creates the fork on GitHub without downloading a second copy locally (useful when you already have a local clone).

---

## Remotes

A **remote** is a named pointer to a copy of the repository hosted somewhere (usually GitHub). Your local clone can have multiple remotes.

| Remote Name | Convention | Points To |
|---|---|---|
| `origin` | Your copy | Your fork on GitHub — where you push your work |
| `upstream` | Original | The original repo you forked from — where you pull updates |

### Useful Remote Commands

```bash
git remote -v                          # list all remotes and their URLs
git remote set-url origin <url>        # change where origin points
git remote add upstream <url>          # add a new remote called upstream
```

---

## The Basic Workflow

Pushing to GitHub is always three steps:

```bash
git add <filename>        # stage a file (mark it ready to commit)
git commit -m "message"   # save a snapshot with a description
git push origin <branch>  # send your commits up to your fork on GitHub
```

### Checking What's Changed Before You Stage

Always run `git status` first to see what git can see:

```bash
git status
```

Output breaks files into two groups:
- **Modified** — files that already existed in git and have been changed
- **Untracked** — brand new files git has never seen before

Both need to be `git add`-ed before they can be committed.

### Writing a Good Commit Message

A commit message should explain *what changed and why*, not just repeat the file names. For multi-line messages, use a short summary on the first line followed by detail:

```bash
git commit -m "Add Gaussian docs scraper module

- Add inkly/scraper/ with fetcher, extractor, and sources modules
- Update docs_gaussian plugin to load scraped data with static fallback
- Add tests covering all new scraper behaviour"
```

### Checking Your Commit History

```bash
git log --oneline -5      # show last 5 commits in a compact format
```

### The LF/CRLF Warning

On Windows you may see warnings like:

```
warning: in the working copy of 'file.py', LF will be replaced by CRLF
```

This is **not an error** — just git noting it will convert Unix line endings (LF) to Windows line endings (CRLF). It does not affect the code or how it runs. Safe to ignore.

### Pulling Updates from the Original Project

```bash
git fetch upstream         # download what's new in the original repo
git merge upstream/main    # bring those changes into your current branch
```

---

## Setting Up a Fork of an Existing Local Clone

If you already have a local clone pointing at the original, here's how to rewire it to your fork:

```bash
# 1. Create the fork on GitHub
gh repo fork <owner>/<repo> --clone=false

# 2. Point your local origin at your new fork
git remote set-url origin https://github.com/<your-username>/<repo>.git

# 3. Add the original as upstream so you can pull updates later
git remote add upstream https://github.com/<original-owner>/<repo>.git

# 4. Push your local code up to your fork
git push origin --all
```

---

## The Contribution Graph

GitHub's contribution graph (the green squares on your profile) only counts commits that meet both of these rules:

1. The commit is in a **standalone repository**, not a fork.
2. The commit is on the **default branch** (usually `main` or `master`).

If you are pushing to a branch on a fork, neither rule is met and nothing shows up.

**The fix:** Keep a separate personal repo that is not a fork. Commit your notes, logs, or any personal work there. Each commit counts.

For this project, that repo is `hpc-research-notes`.

---

## Related Notes

- [[Work Session 1]] — first session where git push, branches, and forking were used in practice
- [[Work Session 2]] — session where the contribution graph issue was discovered and fixed
- [[HPC Intro]] — the cluster environment this repo is built for
