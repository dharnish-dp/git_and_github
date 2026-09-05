# Lesson 02 — Core Concepts: The Three Trees & Git's Anatomy

## Goal
Learn the mental model that makes every Git command make sense: the
**three trees** (Working Directory, Staging Area, Repository) and how
files move between them. This is the single highest-leverage concept in
all of Git.

## Prerequisites
[Lesson 01 — Why Version Control?](01-why-version-control.md)

## After This Lesson You Will Be Able To
- Explain what `git add` and `git commit` actually do to your files
- Read `git status` output and know exactly what it means
- Explain why Git has a "staging area" when other tools don't

---

## The Three Trees

Every file in a Git-tracked project exists in up to three places at once:

```
┌─────────────────┐   git add    ┌─────────────┐   git commit   ┌────────────┐
│ WORKING DIRECTORY│ ───────────► │STAGING AREA │ ─────────────► │ REPOSITORY │
│  (your files on  │              │ (a.k.a. the │                │ (.git —    │
│   disk, editable)│ ◄─────────── │  "index")   │                │  history)  │
└─────────────────┘  git restore  └─────────────┘                └────────────┘
                      --staged
```

1. **Working Directory** — the actual files you see in Finder/VS Code and
   edit. This is the only tree that isn't "inside" Git's database — it's
   just your normal files on disk.
2. **Staging Area** (also called "the index") — a holding pen. You put
   files here to say "these specific changes are what I want in my *next*
   commit." This is the concept that trips up every beginner because no
   other common tool has it.
3. **Repository** — the permanent, immutable history stored in the hidden
   `.git` folder. Once a commit exists here, it (basically) never
   disappears, even if you delete the file from disk later.

**Python analogy:** think of staging as building up a function call before
you execute it.

```python
# Working directory = variables floating around in your script
draft_email = "Hey team, ..."

# git add = deciding what goes into this specific function call
send_email(subject="Update", body=draft_email)   # you chose exactly what to include

# git commit = actually executing/logging that call, permanently, with a message
# log.append({"action": "sent", "content": draft_email, "message": "weekly update"})
```

---

## Why Does Staging Even Exist?

This is the #1 "why is Git weird" question. The answer: **staging lets you
build a commit out of only *some* of your changes**, even if you've edited
ten files.

Concrete scenario: you fixed a bug in `login.py` AND, while you were in
there, you also cleaned up unrelated formatting in `utils.py`. You want two
separate, clean commits — not one messy commit mixing a bug fix with
unrelated formatting. Staging lets you do exactly that:

```bash
git add login.py           # stage ONLY the bug fix
git commit -m "Fix null pointer on login timeout"

git add utils.py           # now stage the formatting cleanup
git commit -m "Clean up formatting in utils.py"
```

Without a staging area, you'd be forced to commit everything at once, or
manually copy files elsewhere first. Staging is what lets professional
Git users produce a clean, readable history instead of a wall of "fix
stuff" commits.

You can even stage *part* of a file's changes with `git add -p` (patch
mode) — more on this in Lesson 12.

---

## Anatomy of a `.git` Folder

When you run `git init`, Git creates a hidden `.git/` folder. This folder
**is** the repository — deleting it deletes all Git history (the files
themselves remain, but their timeline is gone forever).

```
.git/
├── HEAD              ← a pointer to which branch you're currently on
├── config             ← repo-specific settings (remotes, your name/email overrides)
├── objects/            ← the actual database: every commit, file snapshot, and folder structure ever recorded (Lesson 03)
├── refs/
│   ├── heads/            ← one file per local branch, pointing to its latest commit
│   └── remotes/          ← last-known positions of remote branches
├── index              ← the staging area, as an actual file
└── logs/              ← reflog — a safety-net history of where HEAD has pointed (Lesson 06)
```

You will basically never touch these files by hand, but knowing they exist
demystifies Git completely. **A Git repo is just files in a folder** —
there's no hidden server-side magic, no external database. Everything is
right there in `.git/`.

---

## `git status` — Your Most-Used Command

`git status` tells you, in plain English, which of the three trees your
files currently live in relative to each other.

```
$ git status
On branch main
Changes to be committed:          ← STAGED (will be in next commit)
  (use "git restore --staged <file>..." to unstage)
        modified:   login.py

Changes not staged for commit:    ← MODIFIED but NOT staged
  (use "git add <file>..." to update what will be committed)
        modified:   utils.py

Untracked files:                  ← Git has NEVER seen this file before
  (use "git add <file>..." to include in what will be committed)
        new_script.py
```

Three distinct states, all in one output:
- **Untracked** — Git doesn't know this file exists yet. It's brand new.
- **Modified (unstaged)** — Git knows this file, it changed on disk, but
  you haven't run `git add` on the new changes yet.
- **Staged** — you ran `git add`, and this exact version of the file is
  queued for the next commit.

Run `git status` constantly — far more often than beginners think to.
Professionals run it after nearly every command as a sanity check.

---

## What Actually IS a Commit?

A commit is a **snapshot** — not a diff, though Git can *show* you a diff
by comparing two snapshots. Each commit records:

- A pointer to the complete state of every file at that moment (via a tree
  structure — Lesson 03)
- A pointer to its **parent commit(s)** (the commit before it — this is
  what creates the "chain" you see in `git log`)
- Author name + email
- Committer name + email (usually the same person, but not always — e.g.
  when someone applies your patch)
- Timestamp
- A commit message
- A unique **SHA hash** identifying it — a fingerprint of everything above

Because each commit points to its parent, the whole history forms a
**chain** (technically a *directed acyclic graph*, since merges create
commits with two parents — more in Lesson 05):

```
A ← B ← C ← D            (D is the latest commit, points back to C, etc.)
        ↑
     HEAD (you are here)
```

---

## What Is `HEAD`?

`HEAD` is simply **a pointer to whatever commit you currently have checked
out** — "you are here." Almost always, `HEAD` points to the tip of a
branch, and the branch name is really just a label attached to a specific
commit. We'll unpack this fully with diagrams in Lesson 05, but plant this
now: **a branch is nothing but a movable pointer to a commit.** That's the
entire trick behind why Git branches are instant and nearly free, unlike
in older VCS tools where branching meant copying the whole codebase.

---

## Exercise

You don't have a repo yet (that's Lesson 04), so this is a conceptual
checkpoint. Draw (on paper or in your head) the three trees, and answer:

1. If you edit `app.py` but never run `git add`, which tree(s) know about
   the change?
2. If you run `git add app.py` but never `git commit`, and then your
   computer restarts, is your change safe? *(Yes — staged changes persist
   on disk in `.git/index` until you commit or unstage them. But nothing is
   "permanent" history yet.)*
3. What's the difference between "Git doesn't track this file" and
   "Git tracks this file but it hasn't changed since the last commit"?

Continue to [Lesson 03 — Under the Hood: Git Objects](03-under-the-hood-git-objects.md)
to see exactly how Git stores these snapshots so efficiently.
