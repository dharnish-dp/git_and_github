# Lesson 04 — Your First Repo: init, add, commit, status, diff, log

## Goal
Go end to end, for real, on your own machine: create a repo, make changes,
stage them, commit them, and review history. This is the daily-driver loop
you'll run hundreds of times.

## Prerequisites
[Lesson 03 — Under the Hood: Git Objects](03-under-the-hood-git-objects.md)

## After This Lesson You Will Be Able To
- Initialize a repository and make your first commits
- Write good commit messages
- Read diffs and history confidently
- Know exactly which command to reach for at each stage of editing

---

## `git init` — Starting a Repository

```bash
mkdir my-project && cd my-project
git init
```

This creates the `.git/` folder from Lesson 03. Your folder is now a Git
repository — but it's **empty of history**. Nothing is tracked yet.

> **Alternative:** if you're starting from a project someone already
> published (e.g. on GitHub), you'd use `git clone <url>` instead, which
> both creates the folder *and* downloads all existing history. We cover
> that fully in Lesson 07. `git init` is for starting brand new.

---

## The Daily Loop

```
 edit files  →  git status  →  git add  →  git status  →  git commit  →  git log
     ▲                                                                      │
     └──────────────────────────────────────────────────────────────────────┘
                              (repeat forever)
```

Let's run it for real:

```bash
echo "def greet():\n    print('hello')" > greet.py
git status
# Untracked files: greet.py

git add greet.py
git status
# Changes to be committed: new file: greet.py

git commit -m "Add greet function"
git log
```

`git log` output:

```
commit 3f2a91c8e4b7d6a5f0e9c8b7a6f5e4d3c2b1a0f9 (HEAD -> main)
Author: Your Name <you@example.com>
Date:   Fri Jul 31 09:14:22 2026 -0500

    Add greet function
```

Useful `git log` variants you'll use constantly:

```bash
git log --oneline                 # one line per commit — the one you'll use most
git log --oneline --graph --all   # ASCII graph of branches (essential once branching)
git log -p                        # show the actual diff for each commit
git log --stat                    # show which files changed + line counts per commit
git log -n 5                      # only the last 5 commits
git log --author="Jane"           # filter by author
git log --since="2 weeks ago"     # filter by date
```

---

## Staging More Precisely

```bash
git add file1.py file2.py     # stage specific files
git add .                     # stage everything in current dir and below
git add -A                    # stage everything in the whole repo (incl. deletions)
git add -p                    # interactively stage HUNKS (parts of a file) — huge for clean commits
```

`git add -p` is a pro-tier habit. It walks through each changed chunk of a
file and asks `y/n/s/q/...` whether to stage it — letting you split one
messy editing session into several focused, clean commits.

---

## Reading Diffs

```bash
git diff                 # unstaged changes: working dir vs staging area
git diff --staged        # staged changes: staging area vs last commit
git diff HEAD             # everything changed since last commit (staged + unstaged)
git diff <commit1> <commit2>   # compare any two commits directly
git diff main..feature-branch  # compare two branches (Lesson 05)
```

Diff output, explained:

```diff
diff --git a/greet.py b/greet.py
index ce01362..a1b2c3d 100644
--- a/greet.py
+++ b/greet.py
@@ -1,2 +1,3 @@
 def greet():
-    print('hello')
+    print('hello world')
+    return True
```

- `---`/`+++` — the "before" and "after" version of the file
- `@@ -1,2 +1,3 @@` — "starting at line 1, 2 lines shown from the old
  version; starting at line 1, 3 lines shown from the new version" — this
  is called a **hunk header**
- Lines starting with `-` were removed, `+` were added, unmarked lines are
  unchanged context

---

## Writing Good Commit Messages

This is a skill that separates junior from senior engineers immediately —
reviewers judge you on commit quality.

**The convention most professional teams use:**

```
Short summary in imperative mood, ≤50 chars     ← like a command: "Fix", "Add", "Remove"

Longer explanation if needed, wrapped at ~72 chars per line. Explain
WHY the change was made, not just what — the diff already shows what.
Mention any tradeoffs, or link a ticket number.

Fixes #142
```

Good vs bad:

| Bad | Good |
|---|---|
| `fix bug` | `Fix race condition in retry logic for flaky network calls` |
| `updates` | `Bump requests to 2.32 to patch CVE-2024-XXXX` |
| `asdf` | `WIP: refactor auth middleware (do not merge)` |
| `Fixed the thing where it was crashing sometimes` | `Guard against None response before parsing JSON in fetch_status()` |

Rule of thumb: **imperative mood** ("Add", "Fix", "Remove" — not "Added",
"Fixes", "Removing"). Why? Because a commit, when applied, *does* that
thing: `git log` reads naturally as "This commit will: Fix race
condition..."

**One logical change per commit.** If your commit message needs the word
"and" to describe unrelated things, split it into two commits.

---

## `git commit` Flags Worth Knowing Now

```bash
git commit -m "message"           # inline message
git commit                        # opens your default editor for a longer message
git commit -am "message"          # stage ALL tracked, modified files AND commit in one step
                                    # (does NOT pick up brand-new untracked files!)
git commit --amend                # fix the most recent commit (message and/or content)
```

`-am` is a convenient shortcut but has a sharp edge worth knowing now: it
skips staging new (untracked) files. If you create a new file and forget
this, it silently won't be in the commit. `git status` before committing
catches this every time — which is why we drilled that habit in Lesson 02.

---

## Removing and Renaming Files, Properly

```bash
git rm file.py            # deletes from disk AND stages the deletion
git mv old.py new.py      # renames on disk AND stages the rename
```

Using plain `rm` or `mv` (the shell commands) instead of `git rm`/`git mv`
still works — Git will detect the deletion/rename next time you run
`git add .` or `git status` — but the `git` versions do both steps at
once and are considered cleaner.

---

## Exercise

Build a tiny real project from scratch:

```bash
mkdir todo-cli && cd todo-cli
git init
git config user.name  # confirm it's set (or set locally with --local if you want a different identity than global)
```

1. Create `todo.py` with a function `add_task(name)` that just prints the
   name. `git add` + commit it with message `"Add add_task function"`.
2. Add a second function `remove_task(name)`. This time, stage it with
   `git add -p` and watch the hunk-by-hunk prompt. Commit with message
   `"Add remove_task function"`.
3. Introduce a deliberate bug (e.g. a typo), run `git diff` to see it,
   then fix it and run `git diff` again to confirm it's clean.
4. Run `git log --oneline` and `git log -p -1` (the `-1` shows only the
   most recent commit's full diff).
5. Rename `todo.py` to `main.py` using `git mv`, then commit.
6. Run `git log --follow --oneline main.py` — notice Git still shows the
   file's full history across the rename.

Continue to [Lesson 05 — Branching & Merging](05-branching-and-merging.md).
