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

## Quick Vocabulary (read this first)

You met most of these in Lessons 02 and 03. Here they are again in one
place, because this lesson uses them in every command.

| Term | Plain meaning | Python / everyday analogy |
|---|---|---|
| **Repository (repo)** | A project folder that Git is watching, plus a hidden `.git/` folder where Git keeps the history. | A project folder that also has a built-in "undo history" database. |
| **Working directory** | The normal files and folders you see and edit. | Your desk with papers on it. |
| **Staging area (index)** | A waiting room where you place the changes you want in the *next* snapshot. | A shopping cart: items are in it, but you have not paid yet. |
| **Commit** | A saved snapshot of your whole project at one moment, with a message, an author, and a date. | Pressing "checkout" on the cart: a permanent receipt. |
| **Tracked file** | A file Git already knows about (it was in a past commit or is staged). | A file in your test suite that CI already runs. |
| **Untracked file** | A file in the folder that Git has never been told about. | A new file you created but never added to the project. |
| **HEAD** | A label meaning "the commit I am currently on". | The "you are here" pin on a map. |
| **Branch** | A named label pointing at a commit. The default one is called `main`. | A bookmark. (Full story in Lesson 05.) |
| **Hash (SHA)** | The long ID like `3f2a91c8...` that uniquely names a commit. | Like a UUID for a commit. |

**Diagram legend** (used in every lesson):

```
A---B---C      commits; time flows LEFT to right (oldest left, newest right)
        ^      a ^ under a commit: the label below points at that commit
      main     a branch label
HEAD -> main   HEAD = "you are here"; it points at a branch (or at a commit
               id such as "HEAD -> abc123" when detached)
D*             an asterisk marks a commit that is NEW after a command
--->           data or a change moving from one place to another
```

Note: Git internally stores, in each commit, a link back to its parent. We
draw time flowing left to right instead, because it is easier to read.

The three places a change can live:

How to read this: a change starts in the left column and moves right, one
step per command.

```
 WORKING DIRECTORY        STAGING AREA              REPOSITORY (history)
 (files you edit)         (next snapshot)           (saved commits)

 [ greet.py ] --git add--> [ greet.py ] --git commit--> [ commit 3f2a91c ]
```

Keep this picture in mind. Almost every command in this lesson either
moves a change one step to the right, or shows you which box a change is in.

---

## `git init` — Starting a Repository

**Why do we need this?** A normal folder has no history. Before Git can
save snapshots, it needs a place to store them. `git init` creates that
place.

**Analogy:** buying a new notebook and writing "Project history" on the
cover. The pages are blank. The notebook just exists now.

```bash
mkdir my-project && cd my-project
git init
```

- `mkdir my-project` makes a new empty folder.
- `cd my-project` moves your terminal into it.
- `git init` tells Git "start watching this folder".

Expected output:

```
Initialized empty Git repository in /Users/you/my-project/.git/
```

Line by line:
- `Initialized empty Git repository` means it worked.
- `empty` means there are zero commits. Nothing is saved yet.
- The path ending in `.git/` is the hidden folder Git created (the one you
  explored in Lesson 03). That folder *is* the repository's database.

Check what you have:

```bash
ls -a
```

```
.    ..    .git
```

(`-a` shows hidden files. `.git` is hidden because its name starts with a
dot.)

> **Common confusion: "Is my folder or the `.git` folder the repo?"**
> Both, in practice. Your folder is the *working directory* (your files).
> The `.git/` folder inside it holds the *history*. If you delete `.git/`,
> your files stay, but all history is gone and the folder is a normal
> folder again. Never delete `.git/` by accident.

> **Doubt: "When do I use `git init` vs `git clone`?"**
> - `git init` = start a brand-new project from nothing.
> - `git clone <url>` = copy a project that already exists somewhere (for
>   example on GitHub). It creates the folder *and* downloads its full
>   history. Covered in Lesson 07.
> You never run `init` after `clone`. A cloned folder is already a repo.

---

## The Daily Loop

**Why do we need this?** Git does not save your edits automatically. You
decide when a snapshot is taken. This loop is how you decide.

How to read this: follow the arrows left to right; the bottom line sends
you back to the start.

```
 edit files ---> git status ---> git add ---> git status ---> git commit
     ^                                                             |
     |                                                             v
     +------------------- git log <--------------------------------+
                      (repeat forever)
```

**Test-automation analogy:** edit code, run the tests (`status` is
"what's the state of things?"), then only commit when you are happy.

We will walk through this loop one command at a time.

### Step 0: Tell Git who you are (once per machine)

Every commit records an author. If you did this in Lesson 01, skip it.

```bash
git config user.name
git config user.email
```

Expected output (your values will differ):

```
Your Name
you@example.com
```

If either prints nothing, set them (see Lesson 01 for `--global`).

### Step 1: Create a file

```bash
printf "def greet():\n    print('hello')\n" > greet.py
cat greet.py
```

- `printf "..."` writes the text. `\n` means "new line".
- `> greet.py` sends that text into a new file called `greet.py`.
- `cat greet.py` prints the file so you can check it.

Expected output of `cat`:

```
def greet():
    print('hello')
```

> **Why `printf` and not `echo`?** `echo` treats `\n` differently in
> different shells (zsh turns it into a new line; bash prints a literal
> backslash-n). `printf` behaves the same everywhere. Or just create the
> file in your editor.

### Step 2: Ask Git what it sees — `git status`

**Why do we need this?** `git status` is Git's "where am I?" command. It
costs nothing, changes nothing, and tells you which box (working
directory, staging, history) each file is in. Run it all the time.

```bash
git status
```

Expected output:

```
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	greet.py

nothing added to commit but untracked files present (use "git add" to track)
```

Line by line:
- `On branch main` — you are on the branch named `main`.
- `No commits yet` — the history is empty (we have not saved anything).
- `Untracked files:` — files in the folder that Git has never been told to
  track.
- `greet.py` — that is our new file. Git sees it but will **not** put it in
  any snapshot until you ask.
- The last line is a hint: use `git add` to start tracking.

> **Common confusion: "Why does Git ignore my new file?"** It does not
> ignore it, it just waits for your permission. Git never saves anything
> you did not explicitly choose. That is a feature: you control exactly
> what goes into each snapshot.

### Step 3: Stage the file — `git add`

**Why do we need this?** You might have edited five files but want only
two in this snapshot. Staging lets you pick. It is the shopping cart: you
put chosen items in, then check out.

```bash
git add greet.py
git status
```

`git add` itself prints nothing. That is normal: in Git, no output usually
means success. The status now says:

```
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   greet.py
```

Line by line:
- `Changes to be committed:` — these are in the staging area, waiting for
  the next commit.
- `new file: greet.py` — Git will record this file as newly added.

The file moved from "Untracked" to "Changes to be committed". It is now in
the cart.

### Step 4: Commit — `git commit`

**Why do we need this?** Staging only prepares. A commit actually saves
the snapshot into history, permanently, with a message explaining why.

```bash
git commit -m "Add greet function"
```

- `git commit` — take everything in the staging area and save it.
- `-m "..."` — the message, written inline (m = message).

Expected output:

```
[main (root-commit) 3f2a91c] Add greet function
 1 file changed, 2 insertions(+)
 create mode 100644 greet.py
```

Line by line:
- `main` — which branch got the commit.
- `(root-commit)` — this is the very first commit in the repo (it has no
  parent). Later commits will not show this word.
- `3f2a91c` — the first 7 characters of the commit's hash. Short hashes
  are enough to refer to a commit.
- `Add greet function` — your message.
- `1 file changed, 2 insertions(+)` — the snapshot differs from the
  previous one by 1 file and 2 added lines.
- `create mode 100644 greet.py` — a new file was recorded. `100644` means
  "ordinary, non-executable file" (the file mode you saw in Lesson 03).

How to read this: the repo's history, before and after the first commit.
`main` is a label on a commit; `HEAD` says which label you are on.

BEFORE (just after `git init`: no commits, `main` points at nothing yet):

```
   (no commits)

   HEAD -> main
```

```
$ git commit -m "Add greet function"
```

AFTER:

```
   A*
   ^
  main
  HEAD -> main       (A = commit 3f2a91c)
```

What changed: a new commit A* exists, and the `main` label moved onto it.
`HEAD` still points at `main`, so you are now "on" A.

### Step 5: Look at history — `git log`

```bash
git log
```

Expected output:

```
commit 3f2a91c8e4b7d6a5f0e9c8b7a6f5e4d3c2b1a0f9 (HEAD -> main)
Author: Your Name <you@example.com>
Date:   Fri Jul 31 09:14:22 2026 -0500

    Add greet function
```

Line by line:
- `commit 3f2a91c8...` — the full 40-character hash, the commit's unique ID.
- `(HEAD -> main)` — HEAD (the "you are here" pin) points at `main`, and
  `main` points at this commit.
- `Author:` — who made it (from `git config`).
- `Date:` — when. `-0500` is the timezone offset from UTC.
- The indented text is your commit message.

Run `git status` once more:

```
On branch main
nothing to commit, working tree clean
```

`working tree clean` means: your files exactly match the last commit. No
pending changes anywhere. This is the state you start every task in.

> **Common confusion: "What is the difference between `git add` and
> `git commit`?"**
> - `git add` = choose what goes in the next snapshot. Nothing is saved
>   yet. Reversible.
> - `git commit` = save the chosen things as a snapshot in history.
> Think: `add` = put in cart, `commit` = pay. Two steps on purpose.

### Useful `git log` variants

Plain `git log` is long. These are the ones you will use constantly:

```bash
git log --oneline                 # one line per commit: the one you'll use most
git log --oneline --graph --all   # ASCII picture of branches (essential once branching)
git log -p                        # show the actual diff (patch) for each commit
git log --stat                    # which files changed + line counts per commit
git log -n 5                      # only the last 5 commits
git log --author="Jane"           # filter by author
git log --since="2 weeks ago"     # filter by date
```

Example of `--oneline` after a few commits:

```
a91c4de Add remove_task function
3f2a91c Add greet function
```

Each line is: short hash, then the message. Newest is on top.

How to read this: the same two commits as a history drawing. The log lists
newest first (top to bottom); the drawing goes oldest to newest (left to
right).

```
   3f2a91c---a91c4de
                 ^
               main
          HEAD -> main
```

Branches (preview of Lesson 05): once a second line of work exists,
`git log --oneline --graph --all` draws it with `*`, `|`, `/` and `\`.
Here the `feature` branch split off after commit B:

```
* e5f6a7b (feature) Add tests
* d4c3b2a Add login
| * c9d8e7f (HEAD -> main) Fix typo
|/
* b1a2c3d Add remove_task function
* a0b1c2d Add greet function
```

How to read this: the same history in our left-to-right style. Each
letter is one line of the graph above (A = a0b1c2d, B = b1a2c3d,
C = c9d8e7f, D = d4c3b2a, E = e5f6a7b).

```
            D---E      feature
           /
      A---B---C        main
              ^
         HEAD -> main
```

The graph's bottom (oldest) becomes the left; its top becomes the right.
Both branches share A and B, then each goes its own way. Do not worry about
the details yet; Lesson 05 explains branching step by step.

> **Doubt: "git log opened a screen and I cannot type or exit!"** Git
> pipes long output through a pager (a scrolling viewer). Press `q` to
> quit. Use arrow keys or space to scroll.

---

## Staging More Precisely

**Why do we need this?** Real work is messy: you fix a bug, tweak a
comment, and start a feature, all in one sitting. You want separate
commits for each. So you need finer control over what gets staged.

```bash
git add file1.py file2.py     # stage specific files
git add .                     # stage everything in the current folder and below
git add -A                    # stage everything in the whole repo (incl. deletions)
git add -p                    # interactively stage HUNKS (parts of a file)
```

> **Doubt: "`git add .` vs `git add -A`?"** In modern Git (2.x) both stage
> new, modified, and deleted files. The difference: `.` only looks at the
> current folder and below, `-A` looks at the entire repo. If you are
> standing in the repo's top folder, they do the same thing. Be careful
> with both: they can stage files you did not mean to (secrets, junk).
> Check with `git status` afterwards.

### `git add -p` (patch mode)

A **hunk** is one contiguous chunk of changed lines inside a file. If you
edited the top and bottom of one file, that is two hunks.

`git add -p` shows each hunk and asks whether to stage it. This lets you
split one messy editing session into several clean commits. It is a
pro-tier habit.

The prompt looks like this:

```
Stage this hunk [y,n,q,a,d,s,e,?]?
```

What the letters mean:
- `y` — yes, stage this hunk
- `n` — no, leave it unstaged
- `s` — split this hunk into smaller hunks
- `q` — quit (hunks already answered stay as you chose)
- `?` — show help

---

## Reading Diffs

**Why do we need this?** Before committing, you want to see exactly what
you changed. A **diff** is a list of differences between two versions,
like a "track changes" view in a document.

**Python analogy:** like `difflib.unified_diff(old, new)`, which Git
effectively does for you.

First, the key idea: there are three versions of a file floating around
(working copy, staged copy, last-committed copy). Each diff command
compares two of them.

How to read this: three places a file version can live, left to right. Each
command compares the two places it spans.

```
  LAST COMMIT                STAGING AREA               WORKING DIRECTORY
  (HEAD)                     (index)                    (your files)
     |                          |                          |
     |<-- git diff --staged --->|                          |
     |                          |<------ git diff -------->|
     |<------------------ git diff HEAD ------------------>|
```

```bash
git diff                 # unstaged changes: working directory vs staging area
git diff --staged        # staged changes: staging area vs last commit
git diff HEAD            # everything changed since last commit (staged + unstaged)
git diff <commit1> <commit2>   # compare any two commits directly
git diff main..feature-branch  # compare two branches (Lesson 05)
```

> **Common confusion: "I ran `git diff` and got nothing, but I changed
> files!"** Almost always you already ran `git add`. `git diff` (no flags)
> only shows changes that are NOT staged yet. Use `git diff --staged` to
> see staged ones, or `git diff HEAD` to see both.
> Another cause: the file is untracked. `git diff` ignores brand-new
> untracked files.

Suppose you edit `greet.py` to this:

```python
def greet():
    print('hello world')
    return True
```

Then run `git diff`. Output:

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

Line by line:
- `diff --git a/greet.py b/greet.py` — header: comparing the "a" (old)
  version and the "b" (new) version of `greet.py`. The `a/` and `b/` are
  just labels, not real folders.
- `index ce01362..a1b2c3d 100644` — the short hashes of the old and new
  file contents (the blobs from Lesson 03), and the file mode.
- `--- a/greet.py` and `+++ b/greet.py` — the "before" and "after" files.
- `@@ -1,2 +1,3 @@` — the **hunk header**. Read it as: "in the old file,
  this chunk starts at line 1 and spans 2 lines; in the new file, it
  starts at line 1 and spans 3 lines". It tells you where the change is.
- Line starting with `-` (red in your terminal) — removed.
- Lines starting with `+` (green) — added.
- Line with a leading space (`def greet():`) — unchanged context, shown so
  you can see where the change sits.

Note that Git sees `print('hello')` becoming `print('hello world')` as one
line removed and one line added. Git has no concept of "edited a line".

---

## Writing Good Commit Messages

**Why do we need this?** Six months from now you (or a teammate) will be
hunting for "when did this break?" The log is the only guide. Clear
messages turn history into documentation. Reviewers judge you on this.

**The convention most professional teams use:**

```
Short summary in imperative mood, 50 chars or fewer
                                  <- blank line (required)
Longer explanation if needed, wrapped at ~72 chars per line. Explain
WHY the change was made, not just what: the diff already shows what.
Mention any tradeoffs, or link a ticket number.

Fixes #142
```

- **Imperative mood** means phrased like an order: "Add", "Fix", "Remove".
- The blank line separates the summary (shown in `--oneline`) from the body.
- `Fixes #142` links a GitHub issue; GitHub can close it automatically
  later (Lesson 08).

Good vs bad:

| Bad | Good |
|---|---|
| `fix bug` | `Fix race condition in retry logic for flaky network calls` |
| `updates` | `Bump requests to 2.32 to patch CVE-2024-XXXX` |
| `asdf` | `WIP: refactor auth middleware (do not merge)` |
| `Fixed the thing where it was crashing sometimes` | `Guard against None response before parsing JSON in fetch_status()` |

Why imperative? A commit, when applied, *does* that thing. Test: complete
the sentence "If applied, this commit will ___". "...will Fix race
condition" reads naturally; "...will Fixed the thing" does not.

**One logical change per commit.** If your message needs the word "and"
to describe two unrelated things, make two commits. Why: if one change
turns out to be a bug, you can undo just that one without losing the other.

**Test-automation analogy:** a commit is like a test case. One clear
purpose, a clear name, easy to find when it fails.

---

## `git commit` Flags Worth Knowing Now

```bash
git commit -m "message"           # inline message
git commit                        # opens your default editor for a longer message
git commit -am "message"          # stage ALL tracked, modified files AND commit in one step
                                    # (does NOT pick up brand-new untracked files!)
git commit --amend                # fix the most recent commit (message and/or content)
```

- **`-m`**: fine for short messages. Use plain `git commit` when you want
  a body; your editor opens and the commit is made when you save and close.
  (If the editor is `vim` and you are stuck: press `Esc`, type `:wq`, then
  Enter to save and quit. Or `:q!` to abort.)
- **`-am`**: this is `-a` plus `-m`. The `-a` stages every file Git
  *already tracks* that you modified, then commits. It is a shortcut that
  skips a separate `git add`.
- **`--amend`**: redoes the latest commit (for example to fix a typo in the
  message or add a forgotten file). It replaces the last commit with a new
  one. Only amend commits you have not shared yet; sharing rewritten
  history causes trouble (Lesson 09).

`-am` has a sharp edge: it skips *untracked* (brand-new) files. If you
create a new file and use `-am`, the file silently will not be in the
commit. Running `git status` before committing catches this every time,
which is why we drilled that habit in Lesson 02.

> **Common confusion: "`git commit -am` vs `git add .` + `git commit`?"**
> `-am` only covers files Git already tracks. `git add .` also picks up
> new files. If you added a new file, use `git add` explicitly.

---

## Removing and Renaming Files, Properly

**Why do we need this?** Deleting or renaming a file is itself a change
that Git must record. These commands do the file operation *and* tell Git
in one go.

```bash
git rm file.py            # deletes from disk AND stages the deletion
git mv old.py new.py      # renames on disk AND stages the rename
```

Example output of `git mv old.py new.py` followed by `git status`:

```
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	renamed:    old.py -> new.py
```

Using the plain shell commands `rm` or `mv` still works. Git notices the
deletion or rename the next time you run `git status`, but you must then
run `git add -A` (or `git add .`) yourself to stage it. The `git` versions
do both steps at once, so there is less to forget.

> **Doubt: "Does Git actually store a 'rename'?"** No. Git stores
> snapshots, not operations. It *detects* a rename by noticing a file
> disappeared and another file with (nearly) identical content appeared.
> That is why `git log --follow` (used in the exercise) can trace history
> across a rename.

---

## Exercise

Build a tiny real project from scratch. Each step lists what you should see.

```bash
mkdir todo-cli && cd todo-cli
git init
git config user.name  # confirm it's set (or set locally with --local if you want a different identity than global)
```

Expected: `Initialized empty Git repository in .../todo-cli/.git/`, then your
name printed.

1. Create `todo.py` with a function `add_task(name)` that just prints the
   name. `git add` + commit it with message `"Add add_task function"`.
   Expected: `git status` says `working tree clean`; `git log --oneline`
   shows 1 commit.
2. Add a second function `remove_task(name)`. This time, stage it with
   `git add -p` and watch the hunk-by-hunk prompt (answer `y`). Commit with
   message `"Add remove_task function"`.
   Expected: `git log --oneline` shows 2 commits.
3. Introduce a deliberate bug (e.g. a typo), run `git diff` to see it,
   then fix it and run `git diff` again to confirm it's clean.
   Expected: first `git diff` shows your typo as a `+` line; second shows
   no output at all.
4. Run `git log --oneline` and `git log -p -1` (the `-1` shows only the
   most recent commit's full diff).
   Expected: the second command shows one commit with the
   `remove_task` diff.
5. Rename `todo.py` to `main.py` using `git mv`, then commit.
   Expected: `git status` before the commit shows `renamed: todo.py -> main.py`.
6. Run `git log --follow --oneline main.py` — notice Git still shows the
   file's full history across the rename.
   Expected: 3 commits listed (the rename plus the two earlier ones).
   Without `--follow` you would see only the rename commit.

---

## Recap in 5 lines

1. `git init` creates the hidden `.git/` folder and turns a folder into a repo.
2. Changes travel: working directory, then `git add` (staging), then `git commit` (history).
3. `git status` shows which of those boxes each file is in; run it constantly.
4. `git diff` = unstaged changes, `git diff --staged` = staged ones, `git diff HEAD` = both.
5. Write imperative, single-purpose commit messages; use `git rm` / `git mv` for deletes and renames.

Continue to [Lesson 05 — Branching & Merging](05-branching-and-merging.md).
