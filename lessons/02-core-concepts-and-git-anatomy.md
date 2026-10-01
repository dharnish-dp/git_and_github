# Lesson 02 — Core Concepts: The Three Trees & Git's Anatomy

## Goal
Learn the mental model that makes every Git command make sense: the
**three trees** (Working Directory, Staging Area, Repository) and how
files move between them. This is the single highest-leverage concept in
all of Git.

A quick word on "tree": here it just means "a place where Git keeps a
version of your files". It does not mean anything scary. Think "zone".

## Prerequisites
[Lesson 01 — Why Version Control?](01-why-version-control.md)

## After This Lesson You Will Be Able To
- Explain what `git add` and `git commit` actually do to your files
- Read `git status` output and know exactly what it means
- Explain why Git has a "staging area" when other tools don't
- Explain what a commit, a branch, and `HEAD` are, in plain words

## Words You Will Meet (quick glossary)

Read this once now. Each word is explained again where it first matters.

| Word | Plain meaning |
|------|---------------|
| **Repository** ("repo") | A project folder that Git is watching, plus Git's saved history of it |
| **Commit** | A saved snapshot of your whole project at one moment, with a message |
| **Stage** (verb) | Mark a changed file as "include this in my next commit" |
| **Tracked file** | A file Git already knows about |
| **Untracked file** | A file that exists on disk but Git has never been told about |
| **SHA / hash** | A long fingerprint (like `a1b2c3d...`) that uniquely names a commit |
| **Branch** | A named label pointing at a commit |
| **HEAD** | A "you are here" marker |

---

## Step 1: The Big Picture — Three Zones

### Why do we need this?
Beginners expect Git to work like "Save": edit a file, press save, done.
Git has two steps instead (`add`, then `commit`). Until you see why, every
command feels random. The three zones are the explanation.

### Analogy: packing a parcel
Imagine shipping a parcel.

1. Your **desk** is covered in stuff. You are working on it. (messy, changing)
2. You pick some items and put them in a **box on the table**. (chosen, not sent yet)
3. You seal the box, write a label, and put it in the **archive room**.
   (permanent record)

Git's three zones are exactly those three places.

```
Diagram legend (used in every diagram in this course)
  A, B, C ...   = commits (snapshots), each one a letter or short id
  time flows    = LEFT to RIGHT (oldest on the left, newest on the right)
  main, feature = branch names (labels)
  ^             = "points at" (the label underneath belongs to the commit above)
  HEAD          = where you are now
  D*            = a NEW commit created by the command just run
```

How to read this: each column is a zone; an arrow shows which command moves
your changes from one zone to the next.

```
  YOUR DESK                THE BOX ON THE TABLE      THE ARCHIVE ROOM
  WORKING DIRECTORY        STAGING AREA (the index)  REPOSITORY (.git)
  (your files, editable)   (chosen for next commit)  (saved history)
         |                          |                        |
         |---- git add ------------>|                        |
         |                          |---- git commit ------->|
         |<--- git restore --staged-|                        |
         |                          |                        |
```

What it shows: `git add` moves changes right into staging, `git commit`
moves them right into history, and `git restore --staged` takes them back
out of staging (your edits stay safe on disk).

Plain explanation of each zone:

1. **Working Directory** = the normal files and folders you see in Finder
   or VS Code, and edit. Nothing special. Git does not store these; they
   are just files on your disk.
2. **Staging Area** (also called **the index**; both names mean the same
   thing) = a waiting area. You put changes here to say "these specific
   changes go into my *next* commit." It is stored as one file,
   `.git/index`.
3. **Repository** = Git's permanent history, stored in the hidden `.git`
   folder. Once something is committed here, it is very hard to lose,
   even if you later delete the file from disk.

### Python / automation analogy
You know pytest. Think of it like this:

```python
# WORKING DIRECTORY = test code you are editing right now (unfinished, messy)

# STAGING AREA = the list of tests you have decided to run in this CI job
selected = ["test_login.py"]          # you chose exactly what is included

# REPOSITORY = the archived test report, saved permanently with a label
archive.append({"files": selected, "message": "Fix login test"})
```

You choose (stage) first, then you record (commit).

### Common confusion: "Is the staging area a copy of my files?"
Doubt: "Does `git add` copy my file somewhere? Will my file disappear from
my folder?"

Answer: Your file stays exactly where it is. `git add` records the file's
*current contents* into Git's internal staging file (`.git/index`). Your
original file on disk is untouched and still editable.

---

## Step 2: Why Does Staging Even Exist?

### Why do we need this?
Because it feels like extra work. It exists for one reason: so you can
choose *which* changes go into a commit.

### Analogy
You cleaned your desk AND finished a report. You do not want to file both in
one folder labeled "stuff". You want "Report" in one folder and "Desk
cleanup" in another. Staging lets you pick what goes in which folder.

### Concrete scenario
You fixed a bug in `login.py`. While there, you also tidied unrelated
formatting in `utils.py`. You want two clean commits, not one mixed one.

```bash
git add login.py
git commit -m "Fix null pointer on login timeout"

git add utils.py
git commit -m "Clean up formatting in utils.py"
```

Line by line:
- `git add login.py` : stage only `login.py`. `utils.py` is still only in
  the Working Directory.
- `git commit -m "..."` : save a snapshot of what is staged. `-m` means
  "here is the message, inline".
- Then the same two steps for `utils.py`.

Example output of the first commit:

```
[main 3f2a9c1] Fix null pointer on login timeout
 1 file changed, 2 insertions(+), 1 deletion(-)
```

What that output means:
- `main` : the branch you committed on.
- `3f2a9c1` : the first 7 characters of the new commit's hash (its ID).
- `Fix null pointer...` : your message.
- `1 file changed, 2 insertions(+), 1 deletion(-)` : 2 lines added and 1
  line removed, in one file.

Without a staging area you would be forced to commit everything at once.
Staging is how professionals get a readable history instead of a wall of
"fix stuff" commits.

You can even stage *part* of one file's changes with `git add -p` (patch
mode). That is covered in Lesson 12.

### Common confusion: "Why not `git commit` directly?"
Doubt: "If I only want everything saved, is staging useless?"

Answer: You can skip the two-step feel with `git commit -a` (commits all
changes to already-tracked files) in some cases, but the staging area is
still working behind that shortcut. Learn the two-step way first; the
shortcut makes sense afterwards.

---

## Step 3: The Three File States in `git status`

### Why do we need this?
`git status` is the command you will run the most. If you can read it, you
always know where you are.

### Plain definitions first
- **Untracked** = Git has never been told about this file. It is brand new.
- **Modified (not staged)** = Git knows the file, you changed it on disk,
  but you have not run `git add` for that change yet.
- **Staged** = you ran `git add`; this exact version is queued for the next
  commit.

(There is a fourth state, **committed/unmodified**: the file matches the
last commit, so `git status` does not mention it at all.)

### Example
Assume `login.py` is staged, `utils.py` is modified but not staged, and
`new_script.py` is brand new:

```
$ git status
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   login.py

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   utils.py

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        new_script.py
```

Line by line:
- `On branch main` : you are on the branch called `main`.
- `Changes to be committed:` : the STAGED section. The next `git commit`
  will include `login.py`.
- `(use "git restore --staged <file>..." to unstage)` : Git tells you the
  command to undo staging.
- `Changes not staged for commit:` : the MODIFIED section. `utils.py`
  changed but is not in the next commit yet.
- `Untracked files:` : Git has never seen `new_script.py`.

### Diagram: where each file is

How to read this: each row is a file; each column is a zone. Compare the
versions across a row to see how far the file has travelled.

```
File           Working Dir      Staging Area     Repository (last commit)
login.py       new version      new version      OLD version
utils.py       new version      old version      old version
new_script.py  exists          -                -
```

Read it this way: `login.py` has moved to the Staging Area. `utils.py` has
not. `new_script.py` is unknown to Git.

### Common confusion: "Modified AND staged at the same time?"
Doubt: "I staged a file, then edited it again. What does status show?"

Answer: The file appears in *both* sections. Staging saved the version from
the moment you ran `git add`. Your later edit is a new, unstaged change.
Commit now and you get the earlier version. Run `git add` again to include
the later edit.

```
Changes to be committed:
        modified:   login.py     <- the version you staged earlier
Changes not staged for commit:
        modified:   login.py     <- your newer edits after staging
```

### Common confusion: "Untracked vs tracked-but-unchanged"
Untracked: Git does not watch the file at all. Tracked and unchanged: Git
knows the file and it matches the last commit, so `git status` stays quiet
about it. Quiet does not mean "unknown".

Run `git status` constantly. Professionals run it after almost every
command as a sanity check.

---

## Step 4: Anatomy of a `.git` Folder

### Why do we need this?
Git feels magical until you see it is only files in a folder. Once you do,
it stops being scary.

### Analogy
A repository is like a filing cabinet. The cabinet is the `.git/` folder.
Your project files are the papers on your desk. Throw the cabinet away and
you keep your desk papers, but lose the whole filed history.

### What `git init` does
When you run `git init` (you will do it in Lesson 04), Git creates a hidden
folder called `.git/` inside your project. Hidden means its name starts
with a dot. On macOS, `ls -a` shows hidden items.

**That folder IS the repository.** Delete it, and your files remain, but
all history is gone forever.

How to read this: indentation means "inside". Everything below `.git/` is
a file or folder; the text after `<-` says what it is for.

```
.git/
|-- HEAD        <- which branch you are on right now
|-- config      <- settings for this repo (remotes, name/email overrides)
|-- objects/    <- the database: every commit and file snapshot (Lesson 03)
|-- refs/
|   |-- heads/     <- one file per local branch, holding its latest commit ID
|   `-- remotes/   <- last-known branch positions on remote servers
|-- index       <- the staging area, as an actual file
`-- logs/       <- reflog: record of where HEAD has pointed (Lesson 06)
```

Word check:
- A **remote** = a copy of the repo on another computer (for example on
  GitHub). Lesson 07 covers it.
- The **reflog** = a diary of where you have been. It can rescue "lost"
  work. Lesson 06.

Here is what you would actually see (output varies slightly):

```
$ ls -a .git
HEAD  config  description  hooks  info  objects  refs
```

(`index` and `logs` appear after your first `git add` and first commit.)

You will almost never edit these by hand. But knowing they exist removes
the mystery: **a Git repo is just files in a folder.** There is no hidden
server and no external database. Everything is in `.git/`.

### Common confusion: "Can I delete .git safely?"
Doubt: "What if I want to stop using Git in a folder?"

Answer: Deleting `.git/` makes the folder a normal folder again. Your
current files survive. The history does not. Do this only on purpose.

---

## Step 5: What Actually IS a Commit?

### Why do we need this?
Everything else in Git (branches, merging, undoing) is built on commits.

### Analogy
A commit is a photograph of your whole project, with a sticky note on the
back saying who took it, when, and why. Git does not store "what changed".
It stores "what everything looked like".

### Plain explanation
A commit is a **snapshot**, not a diff. (A **diff** is a list of
differences between two versions. Git can *show* you a diff by comparing
two snapshots, but the thing it stores is the snapshot.)

Each commit records:

- **The snapshot** : the state of every file at that moment (stored
  efficiently; Lesson 03 explains how).
- **Parent commit** : a pointer to the commit before it. This links commits
  into a chain, the chain you see in `git log`.
- **Author** : name and email of who wrote the change.
- **Committer** : name and email of who recorded the commit. Usually the
  same as the author, but different when, for instance, someone applies
  your patch for you.
- **Timestamp** : when.
- **Message** : your description.
- **SHA hash** : a unique fingerprint computed from everything above. If
  anything in the commit changed, the hash would be totally different.

### What a commit looks like
```
$ git log -1
commit 3f2a9c1e8d7b6a5f4e3d2c1b0a9f8e7d6c5b4a39
Author: Dharnish <you@example.com>
Date:   Thu Oct 1 10:15:00 2026 +0530

    Fix null pointer on login timeout
```

Line by line:
- `commit 3f2a9c1e...` : the full 40-character hash. People usually type or
  say only the first 7 characters.
- `Author:` : who wrote it.
- `Date:` : when.
- Indented text : the commit message.

### The chain of commits
Because each commit points to its parent, history is a chain.

Note on drawing: internally Git stores each commit's link to its PARENT
(the commit before it), never to its children. In this course we draw time
flowing left to right instead, so the oldest commit is on the left.

How to read this: A is the first commit ever; D is the newest. Each commit
was made on top of the one to its left.

```
  A---B---C---D
```

What it shows: D's parent is C, C's parent is B, B's parent is A, and A has
no parent (it was the first commit).

(Technically this structure is called a *directed acyclic graph*: links
have a direction, and you can never loop back to yourself. Merges create a
commit with two parents, which turns the chain into a branching shape,
like this:

```
        E---F           feature
       /     \
  A---B---C---G         main
```

G has two parents, F and C. Lesson 05 covers it. For now, "chain" is
right.)

### Python analogy
A commit is like an immutable (unchangeable) object with a reference to its
previous one, like a singly linked list where each node stores `prev`:

```python
class Commit:
    def __init__(self, snapshot, parent, author, message):
        ...
```

### Common confusion: "Snapshot vs diff"
Doubt: "If a commit stores a full snapshot, won't the repo get huge?"

Answer: No. Git reuses any file that did not change instead of storing it
again. Lesson 03 shows how. For now, trust that it is efficient.

---

## Step 6: What Is `HEAD`?

### Why do we need this?
Git needs to know "where am I right now?" so it knows what the next commit
should attach to. `HEAD` is that answer.

### Analogy
`HEAD` is the "You are here" red dot on a mall map.

### Plain explanation
`HEAD` is a pointer to the commit you currently have checked out.
(**Checked out** = the files in your Working Directory match that commit.)

Almost always, `HEAD` points to a **branch**, and the branch points to a
commit. A **branch** is simply a name attached to one commit:

How to read this: the label `main` sits under D, so `main` points at D.
`HEAD -> main` means you are "on" the branch `main`.

```
  A---B---C---D
              ^
            main
        HEAD -> main
```

When you make a new commit, Git creates it with the current commit as its
parent, then moves the branch label forward. `HEAD` follows automatically
because it points at the branch.

How to read this: BEFORE and AFTER of making one commit while on `main`.

```
BEFORE
  A---B---C
          ^
        main
    HEAD -> main

  $ git commit -m "add D"

AFTER
  A---B---C---D*
              ^
            main
        HEAD -> main
```

What changed: the new commit D* was added on the right, and the `main`
label moved from C to D*. `HEAD` still points at `main`, so it followed
along. A, B and C are unchanged.

You can see HEAD for real:

```
$ cat .git/HEAD
ref: refs/heads/main
```

Meaning: "HEAD is pointing at the branch called `main`."

```
$ cat .git/refs/heads/main
3f2a9c1e8d7b6a5f4e3d2c1b0a9f8e7d6c5b4a39
```

Meaning: the branch `main` is just a text file holding one commit hash.

**Key idea to plant now: a branch is nothing but a movable pointer to a
commit.** That is why Git branches are instant and nearly free. In older
version control tools, branching meant copying the entire codebase. Here it
means writing 41 characters into a file. Lesson 05 covers this in full.

A preview, so "branches are just labels" feels real. How to read this:
BEFORE and AFTER of creating a second branch (no commits are copied).

```
BEFORE
  A---B---C
          ^
        main
    HEAD -> main

  $ git branch feature

AFTER
  A---B---C
          ^
        main, feature
    HEAD -> main
```

What changed: only a new label, `feature`, was added; it points at the
same commit C as `main`. No commits are new. `HEAD` still points at `main`.
Once you commit on `feature`, the two labels drift apart into a fork
(Lesson 05).

### Common confusion: "HEAD vs branch"
Doubt: "Aren't HEAD and `main` the same thing?"

Answer: No. `main` is a named label on a commit. `HEAD` is your current
position and normally points at that label. Switch branches and `HEAD`
points at a different label.

---

## Exercise

You do not have a repo yet (that is Lesson 04), so this is a conceptual
checkpoint. On paper, draw the three zones and answer. Expected answers are
given so you can check yourself.

1. You edit `app.py` but never run `git add`. Which zone(s) know about the
   change?
   - Expected: only the Working Directory. Staging and Repository still
     hold the old version.
2. You run `git add app.py` but never `git commit`, then your computer
   restarts. Is your change safe?
   - Expected: yes. It lives on disk in `.git/index`. But it is not in
     permanent history until you commit.
3. What is the difference between "Git doesn't track this file" and "Git
   tracks this file but it hasn't changed since the last commit"?
   - Expected: untracked files show under `Untracked files:` in `git
     status`. Tracked-and-unchanged files are not listed at all.
4. Read this status and say what the next `git commit` will include:
   ```
   Changes to be committed:
           modified:   a.py
   Changes not staged for commit:
           modified:   b.py
   Untracked files:
           c.py
   ```
   - Expected: only `a.py`. `b.py` was edited but not staged, and `c.py`
     is unknown to Git.
5. Draw a chain of 3 commits A, B, C with `main` and `HEAD`. Which commit
   does each one point at?
   - Expected: `main` points at C; `HEAD` points at `main`; C points at B;
     B points at A; A has no parent.

---

## Recap in 5 Lines
1. Git has three zones: Working Directory (your files), Staging Area
   (chosen changes for the next commit), Repository (saved history in `.git`).
2. `git add` moves changes into staging; `git commit` saves staged changes
   as a snapshot.
3. `git status` shows untracked, modified, and staged files; run it often.
4. A commit is a snapshot plus author, message, parent, and a unique hash.
5. A branch is a movable label on a commit; `HEAD` is "you are here",
   usually pointing at a branch.

Continue to [Lesson 03 — Under the Hood: Git Objects](03-under-the-hood-git-objects.md)
to see exactly how Git stores these snapshots so efficiently.
