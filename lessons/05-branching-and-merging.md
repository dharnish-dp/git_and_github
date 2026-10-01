# Lesson 05 — Branching & Merging

## Goal
Understand branches as what they really are (a tiny movable label, not a
folder copy), and master merging, including resolving conflicts without
fear.

## Prerequisites
[Lesson 04 — Your First Repo](04-your-first-repo.md)

## After This Lesson You Will Be Able To
- Create, switch, and delete branches confidently
- Explain fast-forward vs three-way merges, and why each happens
- Resolve a merge conflict from scratch, calmly
- Read `git log --graph` output for multi-branch history

---

## Words You Need First

You have met most of these already. Here is a one-line reminder of each,
so nothing in this lesson is a mystery.

| Term | Plain meaning |
|------|---------------|
| **Repository (repo)** | Your project folder plus Git's hidden `.git` folder, which stores all history. |
| **Commit** | A saved snapshot of your whole project, with a message, an author, and a link to the commit before it (its **parent**). |
| **Hash (SHA)** | The long ID of a commit, like `3f2a91c`. It is a fingerprint of the commit's content. |
| **HEAD** | Git's "you are here" marker. It tells Git which commit/branch you are working on right now. |
| **Branch** | A named label that points at one commit. (Explained fully below.) |
| **Merge** | Combining the work from one branch into another. |
| **Conflict** | Git cannot decide how to combine two edits, so it asks you. |

---

## What a Branch Actually Is

### Why do we need this?

Imagine you are testing a new feature in a framework, but your teammate
needs the working version of the framework to keep running tests. You need
two lines of work at the same time, without breaking each other. Branches
give you that.

### Analogy first

Think of a commit history as a trail of photographs, each one linked to the
photo before it. A **branch is a sticky note** you stick on one photo with
a name written on it, like "main". When you take a new photo, you move the
sticky note onto the new photo. The photos never change. Only the sticky
note moves.

In Python terms, a branch is like a variable that holds a reference to an
object. `main = commit_C`. Making a "new branch" is just creating a second
variable that points at the same object. No data is copied.

### The plain explanation

From Lesson 03: every commit points back to its parent. A branch is
nothing but **a small text file containing one commit hash**. You can
look at it yourself:

```bash
cat .git/refs/heads/main
```

Expected output:

```
3f2a91c8e4b7d6a5f0e9c8b7a6f5e4d3c2b1a0f9
```

How to read this:
- `.git/refs/heads/` is the folder where Git keeps one file per branch.
- `main` is the file for the branch named `main`.
- The one line inside is the hash of the commit that branch points at.

When you make a commit, Git does two things:
1. Creates the new commit (pointing back to the previous one).
2. Rewrites that one text file so it holds the new commit's hash.

That is why creating a branch is instant, even in a huge project. It
writes one tiny file. It does not copy your project.

**Diagram legend** (every diagram in this lesson follows it):

```
  A---B---C      commits. OLDEST on the LEFT, NEWEST on the RIGHT.
  D*             an asterisk marks a commit the command just CREATED.
  ^              points up at the commit that a label refers to.
  main           a branch label, written under the commit it points at.
  HEAD -> main   the HEAD label: "I am standing on branch main".
                 (HEAD -> abc123 would mean a detached HEAD.)
  /   \          history forks or joins. Each branch gets its own row,
                 with the branch name at the END of its row.
```

Note: internally, Git stores each commit's link to its *parent* (so the
link points backward in time). We still draw time flowing left to right,
because that is how humans read a timeline.

How to read this: the letters are commits, the label `main` sits under the
commit it points at, and `HEAD -> main` says you are standing on `main`.
First, what a commit does to the picture:

```
BEFORE  (main points at C)

      A---B---C
              ^
              main
              HEAD -> main

$ git commit -m "D"

AFTER   (main slid forward onto the new commit D)

      A---B---C---D*
                  ^
                  main
                  HEAD -> main
```

What changed: the `main` label moved from C to the new commit D*. A, B and
C are untouched. `HEAD` still says `-> main`, so it follows `main` automatically.

### HEAD points at a branch

HEAD usually does not point at a commit directly. It points at a
**branch**, and the branch points at a commit:

```
  HEAD   --->   main   --->   C
  (where you    (branch       (commit:
   are)          label)        a snapshot)
```

How to read this: follow the arrows left to right. HEAD names a branch, and
the branch names a commit.

### Why do we need this extra layer?

Because it separates two actions cleanly:
- **Switching branches** = changing what HEAD points at.
- **Committing** = moving the branch HEAD points at, to the new commit.

When you commit, Git follows HEAD to the branch, and moves that branch.
That is how the branch you are "on" automatically advances.

```
  Switching:   HEAD ---> (another branch)       no branch label moves
  Committing:  HEAD ---> main ---> (new commit)  the branch label moves,
                                                 HEAD stays on the branch
```

> **Common confusion: "Is a branch a copy of my files?"**
> No. A branch is a label on a commit. Your files on disk are just whatever
> the commit under your current branch contains. When you switch branches,
> Git rewrites the files in your folder to match the commit the new branch
> points at. The history of both branches lives in one shared `.git`
> folder.

---

## Creating and Switching Branches

### Why do we need this?

To start separate work, you need a way to create a new label and then move
HEAD onto it. These are two separate steps, though Git has shortcuts that
do both at once.

### Step 1: See where you are

```bash
git branch
```

Expected output:

```
* main
```

The `*` marks the branch HEAD is on right now. Only `main` exists so far.

### Step 2: Create a branch (without switching)

```bash
git branch feature-login
```

Expected output: nothing. In Git, silence usually means success.

Check it:

```bash
git branch
```

```
  feature-login
* main
```

- `feature-login` exists, and points at the same commit as `main`.
- The `*` is still on `main`. Creating a branch does **not** move you.

How to read this: two labels now sit under the same commit C. No commit was
created, so there is no asterisk.

```
BEFORE  (one label)

      A---B---C
              ^
              main
              HEAD -> main

$ git branch feature-login

AFTER   (a second label on the SAME commit)

      A---B---C
              ^
              main
              feature-login
              HEAD -> main
```

What changed: one new label, `feature-login`, pointing at C. `HEAD` did not
move (still `-> main`), and no commit is new. Both sticky notes are on
commit C. Nothing is different yet. Your working folder is unchanged too.

### Step 3: Switch to it

```bash
git switch feature-login
```

Expected output:

```
Switched to branch 'feature-login'
```

This moved HEAD from `main` to `feature-login`, and updated your files to
match that branch's commit. Right now they are identical, so no files
visibly change.

How to read this: only the HEAD line changes. The labels stay where they
are.

```
BEFORE  (standing on main)

      A---B---C
              ^
              main
              feature-login
              HEAD -> main

      Working folder: files of commit C

$ git switch feature-login

AFTER   (standing on feature-login)

      A---B---C
              ^
              main
              feature-login
              HEAD -> feature-login

      Working folder: files of commit C   (same commit, so no visible change)
```

What changed: HEAD now points at `feature-login` instead of `main`. No label
and no commit moved.

### Step 4: Commit on it

```bash
echo "login code" > login.py
git add login.py
git commit -m "Add login code"
```

Expected output:

```
[feature-login 7d3e1a9] Add login code
 1 file changed, 1 insertion(+)
 create mode 100644 login.py
```

Line by line:
- `[feature-login 7d3e1a9]` = the commit landed on branch `feature-login`,
  new hash starts with `7d3e1a9`.
- The other lines are the summary of what changed.

How to read this: the branch HEAD is on (`feature-login`) is the one that
moves. `main` stays behind on C.

```
BEFORE  (both labels on C, HEAD on feature-login)

      A---B---C
              ^
              main
              feature-login
              HEAD -> feature-login

$ git commit -m "Add login code"

AFTER   (only feature-login moved)

      A---B---C---D*
              ^   ^
              |   feature-login
              |   HEAD -> feature-login
              main

      Working folder: files of C, plus login.py (which D added)
```

What changed: `feature-login` moved from C to the new commit D*, and HEAD
went with it. `main` did not move. A, B, C are unchanged.

`main` did not move. Only `feature-login` moved forward. That is the whole
point: **branches let several lines of work exist side by side in one
repo, without disturbing each other.**

### Watching HEAD and your folder when you switch

Switching rewrites the files in your folder to match the commit the target
branch points at. Here is the same repo as above, switching back to `main`
(illustration: commit C holds only `app.py`, and D added `login.py`):

```
BEFORE  (standing on feature-login)

      A---B---C---D
              ^   ^
              |   feature-login
              |   HEAD -> feature-login
              main

      Working folder (matches D):   app.py   login.py

$ git switch main

AFTER   (standing on main)

      A---B---C---D
              ^   ^
              |   feature-login
              main
              HEAD -> main

      Working folder (matches C):   app.py
                                    (login.py left your disk, but it is
                                     safe inside commit D)
```

What changed: only HEAD moved (to `main`), and the folder was rewritten to
match C. Switch to `feature-login` again and `login.py` reappears. No commit
is new.

### The all-in-one shortcuts

```bash
git switch -c feature-login      # -c = "create": make the branch AND switch to it
git checkout -b feature-login    # older command that does the same thing
```

How to read this: one command does the two steps (new label, then move HEAD)
that you did separately above.

```
BEFORE  (only main exists)

      A---B---C
              ^
              main
              HEAD -> main

$ git switch -c feature-login

AFTER   (new label AND HEAD moved onto it)

      A---B---C
              ^
              main
              feature-login
              HEAD -> feature-login
```

What changed: a new label `feature-login` on C, and HEAD now points at it.
No commits are new and the working folder is unchanged.

> **Common confusion: `switch` vs `checkout`**
> `git checkout` is the older command, and it does *two unrelated jobs*:
> switching branches AND discarding file changes. That mix-up caused many
> accidents. Git 2.23 split it into `git switch` (for branches) and
> `git restore` (for files). Use `switch`/`restore` going forward. Still
> recognize `checkout`, because older tutorials and teammates use it.

### Listing and deleting

```bash
git branch                        # list local branches; * marks the current one
git branch -a                     # also show remote-tracking branches (Lesson 07)
git branch -d feature-login       # delete: SAFE, refuses if the branch has unmerged work
git branch -D feature-login       # force delete: dangerous, skips the safety check
```

What does "delete a branch" delete? Only the sticky note (the label). The
commits stay in the repo for a while. If the branch was already merged,
nothing is lost at all. If it was not merged, the commits become hard to
find, which is why `-d` refuses and `-D` is the "I'm sure" version.

How to read this: deleting removes a label only. The commits are not touched.
Case 1, the branch is already merged (safe):

```
BEFORE  (feature-login already merged into main)

      A---B---C---D
                  ^
                  main
                  feature-login
                  HEAD -> main

$ git branch -d feature-login

AFTER   (one label gone, every commit still there)

      A---B---C---D
                  ^
                  main
                  HEAD -> main
```

What changed: the `feature-login` label is gone. All commits (A to D) are
unchanged, and D is still reachable through `main`.

Case 2, the branch has work that is NOT merged:

```
BEFORE  (D and E exist only on feature-login)

                D---E        feature-login
               /
      A---B---C              main
              ^
              HEAD -> main

$ git branch -d feature-login     -> refused (error below), nothing changes
$ git branch -D feature-login     -> forced

AFTER -D  (label gone, D and E have no name pointing at them)

                D---E        (no label: hard to find later)
               /
      A---B---C              main
              ^
              HEAD -> main
```

What changed (with -D): only the label disappeared. D and E still exist for
a while, but nothing names them, so you would need the reflog (Lesson 06) to
find them.

Example of the safety check:

```
error: the branch 'feature-login' is not fully merged.
If you are sure you want to delete it, run 'git branch -D feature-login'.
```

> **Doubt?: "I deleted a branch after merging. Did I lose my work?"**
> No. Merging copied the branch's commits into `main`'s history. Deleting
> the label afterwards just removes a name you no longer need.

> **Doubt?: "I'm on a branch and I have uncommitted edits. What happens
> when I switch?"**
> If the edited files are the same in both branches, Git carries your edits
> along. If switching would overwrite your edits, Git refuses with an
> error and tells you to commit or stash first (stash is in Lesson 10).
> Git does not silently destroy uncommitted work here.

---

## Merging: Bringing Branches Back Together

### Why do we need this?

After finishing work on a branch, you want that work in `main`. Merging
does that.

### The two-line recipe

```bash
git switch main                 # step 1: go to the branch that should RECEIVE the work
git merge feature-login         # step 2: bring feature-login's commits into main
```

> **Common confusion: "Which branch do I merge from?"**
> You always stand on the branch that **receives** the work, then name the
> branch that **gives** it. `git merge X` means "bring X into where I am
> now." Only the branch you are standing on changes. `X` is untouched.

There are two different outcomes. Which one you get depends on one
question: **did `main` move since you branched off?**

### Outcome 1: Fast-Forward Merge (main did not move)

**Analogy:** You are on a road. Your friend walked ahead on the same road.
To catch up, you walk forward along the same road. There is no second road
to join.

If `main` has not moved since you created your branch, `feature-login` is
simply `main` plus extra commits on top. Git just slides the `main`
sticky note forward to the same commit. No new commit is created.

How to read this: `feature-login` is just `main` plus one more commit, all on
one straight line. The merge only slides the `main` label along that line.

```
BEFORE  (main has not moved; feature-login is ahead)

      A---B---C---D
              ^   ^
              |   feature-login
              main
              HEAD -> main

$ git merge feature-login

AFTER   (main slid forward onto D; no new commit)

      A---B---C---D
                  ^
                  main
                  feature-login
                  HEAD -> main

      Working folder: now matches D (login.py appears)
```

What changed: the `main` label moved from C to D. No commit is new, so there
is no asterisk. `feature-login` did not move.

Command and expected output:

```bash
git switch main
git merge feature-login
```

```
Updating 5a4b3c2..7d3e1a9
Fast-forward
 login.py | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 login.py
```

Line by line:
- `Updating 5a4b3c2..7d3e1a9` = `main` is moving from the old commit to the
  new one.
- `Fast-forward` = Git only moved the label. No merge commit was made.
- The remaining lines list the files that changed.

### Outcome 2: Three-Way Merge (main moved too)

**Analogy:** You and a colleague both copy the same shared document and
edit different paragraphs. To combine, you compare both edited copies
against the **original** they started from, and apply both sets of edits.
That is three documents involved: the original and the two edited copies.
That is why it is called "three-way."

If `main` got new commits while you worked on your branch (a teammate
pushed, or a hotfix landed), the history has **diverged**: two lines both
grew from the same starting point. Git cannot just slide a label. Instead
it creates a new commit, called a **merge commit**, which has **two
parents** (one from each line).

How to read this: starting at B, `main` grew by one commit (C) and
`feature-login` grew by two (D and E). Each branch has its own row.

```
BEFORE  (diverged: main and feature-login both moved past B)

            D---E        feature-login
           /
      A---B---C          main
              ^
              HEAD -> main

$ git merge feature-login

AFTER   (a NEW merge commit M* joins both rows)

            D---E        feature-login
           /     \
      A---B---C---M*     main
                  ^
                  HEAD -> main

      Working folder: contains the edits from C AND from D and E
```

What changed: a new merge commit M* was created, with two parents (C and E).
The `main` label moved from C to M*. `feature-login` did not move. A to E
are unchanged.

(The letters here are a fresh example, not the same commits as the
fast-forward example above.)

The important facts are:
- **B** is the **common ancestor** (also called the **merge-base**): the
  last commit both lines share.
- Git compares three snapshots: B (the original), C (tip of main), E (tip
  of feature).
- It applies both sets of changes. If they touch different lines, this
  happens automatically.
- The result is saved as the merge commit **M**, with two parents.

Here is the three-way comparison itself, step by step:

```
  B  (original) --- what main changed, B to C --------\
                                                       >--->  M*  (combined)
  B  (original) --- what feature changed, B to E -----/
```

How to read this: both sets of changes are measured against the same
original B, then applied together to make M.

Command and expected output:

```bash
git switch main
git merge feature-login
```

Git opens your editor with a pre-written message. Save and close it, then:

```
Merge made by the 'ort' strategy.
 login.py | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 login.py
```

- `Merge made by the 'ort' strategy` = a real merge commit was created.
  (`ort` is just the name of Git's default combining algorithm. More on
  this later.)

> **Common confusion: "How do I know in advance which one I'll get?"**
> Ask: "Is `main`'s tip an ancestor of my branch's tip?" If yes (main
> has nothing new), it is a fast-forward. If no (main has commits your
> branch lacks), it is a three-way merge. You can also just run the merge
> and read whether the output says `Fast-forward`.

```
Fast-forward: main's tip is ON the feature's line (one straight line)

      A---B---C---D          main's tip C is behind D, so just slide
              ^   ^
              |   feature
              main

Three-way: main's tip is on a DIFFERENT row (two lines)

            D---E            feature
           /
      A---B---C              main's tip C is not behind E: a real merge
```

---

## Merge Conflicts: What They Are and How to Resolve Them

### Why do we need this?

Git is good at combining edits to *different* places. But if both branches
changed the *same lines*, Git has no way to know which is right. It stops
and asks you. A conflict is not an error or a failure. It is Git saying
"I need a human decision here."

**Analogy:** Two people edit the same sentence in a shared doc. One
changes "30" to "45", another changes it to "60". The software cannot
guess which you want. In Python terms, it is like two people editing the
same line of `config.py` in different ways.

### Step 1: Trigger the merge

```bash
git switch main
git merge feature-login
```

Expected output:

```
Auto-merging config.py
CONFLICT (content): Merge conflict in config.py
Automatic merge failed; fix conflicts and then commit the result.
```

Line by line:
- `Auto-merging config.py` = Git tried to combine the file by itself.
- `CONFLICT (content)` = it hit a spot where both sides changed the same
  lines, so it could not decide.
- `Automatic merge failed; fix conflicts...` = your job now. The merge is
  **paused halfway**, not finished.

How to read this: the history looks exactly like a three-way merge that has
not happened yet. No merge commit exists, and the only change is in your
working folder.

```
BEFORE  (main and feature-login both changed TIMEOUT in config.py)

            D---E        feature-login     E sets  TIMEOUT = 60
           /
      A---B---C          main              C sets  TIMEOUT = 30
              ^                            B had   TIMEOUT = 10
              HEAD -> main

$ git merge feature-login

DURING  (merge PAUSED by a conflict; no M yet)

            D---E        feature-login
           /
      A---B---C          main      <- has NOT moved
              ^
              HEAD -> main         (merge in progress: incoming commit = E)

      Working folder: config.py now holds the conflict markers
```

What changed: no pointer moved and no commit was created. Only `config.py`
in your folder was rewritten (with markers), and Git remembers that a merge
is in progress.

### Step 2: See what Git thinks

```bash
git status
```

```
On branch main
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:   config.py
```

- `Unmerged paths` / `both modified` = this file was changed on both
  branches and needs your decision.
- Git also tells you the next steps, and how to escape (`--abort`).

### Step 3: Read the conflict markers

Git edited the file and inserted **conflict markers**, which are special
lines showing both versions:

```python
<<<<<<< HEAD
TIMEOUT = 30
=======
TIMEOUT = 60
>>>>>>> feature-login
```

How to read it:

| Part | Meaning |
|------|---------|
| `<<<<<<< HEAD` | Start of the conflict. What follows is **your side**, the branch you are on (here `main`). |
| `=======` | Divider between the two versions. |
| `>>>>>>> feature-login` | End of the conflict. What came just before is the **incoming side**, from the branch being merged. |

So here: `main` says `TIMEOUT = 30`, and `feature-login` says
`TIMEOUT = 60`.

How to read this: Git compares three versions and shows you both sides.

```
  B (original):            TIMEOUT = 10
  C (main, "ours"):        TIMEOUT = 30      ---> top half of the markers
  E (feature, "theirs"):   TIMEOUT = 60      ---> bottom half of the markers
```

> **Common confusion: "Is `HEAD` here the same HEAD as before?"**
> Yes. During a merge, `HEAD` means "the commit I'm standing on,"
> which is the tip of the receiving branch. That is why the top half is
> called "current" or "ours."

> **Warning:** The marker lines are literal text inside your file. If you
> forget to delete them, they end up in your code and Python will hit a
> `SyntaxError`.

### Step 4: Resolve it by hand

Edit the file so that it contains exactly the final code you want, and
**delete all three marker lines**. For example, if 60 is right:

```python
TIMEOUT = 60   # decided the higher timeout was right
```

You are free to keep one side, the other, or a mix of both.

### Step 5: Tell Git it is resolved

```bash
git add config.py
```

Why `git add`? Here, `add` means "I have finished fixing this file, mark it
as resolved." Git will not let you complete the merge while a file is still
marked unmerged.

Check progress:

```bash
git status
```

```
On branch main
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)

Changes to be committed:
	modified:   config.py
```

### Step 6: Finish the merge

```bash
git commit
```

Git opens your editor with a pre-filled message such as
`Merge branch 'feature-login'`. It is normally fine as-is. Save and close:

```
[main 9c8b7a6] Merge branch 'feature-login'
```

Done. The merge commit now exists, with two parents.

How to read this: the whole conflict flow, left to right, then the result.

```
  git merge ---> CONFLICT ---> edit file ---> git add ---> git commit
  (paused)       (markers)     (remove        (marks it    (creates M*)
                                markers)       resolved)

BEFORE git commit  (still paused, main still on C)

            D---E        feature-login
           /
      A---B---C          main
              ^
              HEAD -> main

$ git commit

AFTER   (merge commit M* created, main moved onto it)

            D---E        feature-login
           /     \
      A---B---C---M*     main
                  ^
                  HEAD -> main
```

What changed: a new merge commit M* with two parents (C and E), containing
your resolved `config.py`. `main` moved from C to M*. Nothing else moved.

### Escape hatch: abort

At any point before the final commit, you can cancel and go back to how
things were before you started the merge:

```bash
git merge --abort
```

This is safe. Nothing is lost. It is like closing a document without
saving.

```
DURING  (paused, markers in config.py)

            D---E        feature-login
           /
      A---B---C          main
              ^
              HEAD -> main

$ git merge --abort

AFTER   (exactly as before the merge)

            D---E        feature-login
           /
      A---B---C          main
              ^
              HEAD -> main

      Working folder: config.py is back to C's version, no markers
```

What changed: no pointer and no commit changed. Only the working folder was
restored.

> **Doubt?: "Did the conflict damage my files or history?"**
> No. A conflict only pauses the merge in your working folder. Both
> branches' commits are untouched. `git merge --abort` returns you to
> exactly where you were.

> **Doubt?: "There are several conflict blocks in one file. Is that normal?"**
> Yes. Each block is a separate spot to decide. Fix each one, delete its
> markers, then `git add` the file once.

---

## Conflict-Reading Tip: Use a Merge Tool

Raw markers are fine for small conflicts. For bigger ones, a **merge tool**
is a program that shows both versions side by side. Configure one once
(here, VS Code):

```bash
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
git mergetool
```

What each line does:
- Line 1: tells Git "when I ask for a merge tool, use the one named
  `vscode`."
- Line 2: defines what `vscode` means: run `code --wait`, giving it the
  file being merged (`$MERGED`). `--wait` makes Git pause until you close
  the editor.
- Line 3: opens the tool for each conflicted file.

Most IDEs (VS Code, PyCharm) also detect conflict markers automatically
and show clickable "Accept Current / Accept Incoming / Accept Both"
buttons. Use them. There is no shame in it. The result is the same text
edit you would have made by hand.

---

## Merge Strategies & Useful Merge Flags

This section is a toolbox. You do not need all of it on day one, but it is
good to know it exists.

### Finding the Common Ancestor Yourself

Git finds the common ancestor (commit B in the three-way diagram)
automatically. You can ask for it directly:

```bash
git merge-base main feature-login
```

Expected output:

```
b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1
```

That is the last commit both branches share.

```
            D---E        feature-login
           /
      A---B---C          main
          ^
          merge-base = B (the last commit on BOTH rows)
```

Why is this useful? It lets you
list only the commits that belong to the branch:

```bash
git log b2c3d4e..feature-login --oneline
```

```
3c4d5e6 Add password reset flow
2b3c4d5 Add login form validation
```

How to read `A..B`: "commits reachable from B but not from A." So you
see only the branch's own work, without noise from `main`.

```
           [D]-[E]       feature-login     <- listed by  B..feature-login
           /
      A---B---C          main              <- not listed (not in the range)
```

(Square brackets here mean "selected by the command". No commit is new.)

### Controlling Whether a Merge Commit Gets Created

```bash
git merge --ff-only feature-login
git merge --no-ff feature-login
```

- `--ff-only` = "merge only if a fast-forward is possible. Otherwise stop
  and change nothing." Useful in scripts and CI, where a surprise merge
  commit would be unwanted.
- `--no-ff` = "always create a merge commit, even if a fast-forward would
  have worked." Useful when your team wants history to visibly show "this
  group of commits was a feature branch."

Expected output of a refused `--ff-only`:

```
fatal: Not possible to fast-forward, aborting.
```

How to read this: the same starting history gives different results. First
`--no-ff` when a fast-forward WAS possible:

```
BEFORE  (main at C; feature-login is one commit ahead)

      A---B---C---D
              ^   ^
              |   feature-login
              main
              HEAD -> main

$ git merge --no-ff feature-login

AFTER   (a merge commit M* is forced, so the branch stays visible)

                D            feature-login
               / \
      A---B---C---M*         main
                  ^
                  HEAD -> main
```

What changed: a new merge commit M* with two parents (C and D). `main`
moved from C to M*. Without `--no-ff`, `main` would have slid to D and the
side row would not show.

Now `--ff-only` when the history has diverged (fast-forward impossible):

```
BEFORE  (diverged)                 $ git merge --ff-only feature-login

            D---E        feature-login
           /
      A---B---C          main
              ^
              HEAD -> main

AFTER   (REFUSED: "fatal: Not possible to fast-forward, aborting.")

            D---E        feature-login
           /
      A---B---C          main
              ^
              HEAD -> main
```

What changed: nothing. No pointer moved, no commit was created, and your
folder is untouched.

### Squash Merge: `git merge --squash`

A squash merge takes all the branch's changes and lands them as ONE new
commit on your branch, with no link back to the branch's commits.

```
BEFORE  (feature-login has two commits, D and E)

                D---E        feature-login
               /
      A---B---C              main
              ^
              HEAD -> main

$ git merge --squash feature-login

MIDDLE  (nothing committed yet; the changes are only STAGED)

                D---E        feature-login
               /
      A---B---C              main
              ^
              HEAD -> main

      Working folder + staging area: C's files plus the changes of D and E

$ git commit -m "Add login"

AFTER   (ONE new commit S* with ONE parent)

                D---E        feature-login   (unchanged, now unlinked)
               /
      A---B---C---S*         main
                  ^
                  HEAD -> main
```

What changed: `main` moved from C to the new commit S*. S* has a single
parent (C). D and E are untouched, but S* does not point to them, so Git
does not consider `feature-login` merged (`git branch -d` would refuse;
you would need `-D`).

### Conflict Shortcuts: `-X ours` / `-X theirs`

```bash
git merge -X ours feature-login     # on conflicting lines, silently keep YOUR side
git merge -X theirs feature-login   # on conflicting lines, silently keep THEIR side
```

Key points:
- These still do a **real merge**. Non-conflicting changes from both sides
  are combined normally. The flag only picks a winner *where there is a
  conflict*.
- Warning: you are silently discarding someone's code. For real source
  code, rarely what you want.
- A reasonable use: a machine-generated file (like a lockfile) that you
  will regenerate right after the merge anyway.

> **Common confusion: `-X ours` vs `-s ours`**
> They look alike but are very different. `-X ours` (capital X, an
> *option*) resolves only conflicting lines in your favor. `-s ours` (lower
> `s`, a *strategy*) throws away **everything** from the other branch while
> still recording that a merge happened. Almost always you want neither.

### Which Strategy Is Actually Running?

A **merge strategy** is the algorithm Git uses to combine the two sides.
Since Git 2.33, the default for a normal two-branch merge is **`ort`**. It
replaced the older **`recursive`** (still available with
`git merge -s recursive`). `ort` does the same job, faster and more
correctly. You do not need to think about it day to day.

One strategy you might pick on purpose is **octopus**, which merges *more
than two* branches into one merge commit (`git merge b1 b2 b3`). It is
rare, for example when assembling a release from several finished feature
branches. It refuses to run if any of them would need manual conflict
resolution.

How to read this: one merge commit with three parents (main plus two
branches). The same idea extends to more branches.

```
BEFORE  (two finished branches off C)

                D            feat-1
               /
      A---B---C              main
               \
                E            feat-2

$ git merge feat-1 feat-2

AFTER   (ONE merge commit M* with THREE parents: C, D, E)

                D            feat-1
               / \
      A---B---C---M*         main
               \ /
                E            feat-2
```

What changed: a single new commit M* joins all three lines, and `main`
moved from C to M*. `feat-1` and `feat-2` did not move.

---

## Visualizing Branch History

### Why do we need this?

Branching history is a shape, not a list. A picture makes it easy to
understand. Git can draw one in your terminal:

```bash
git log --oneline --graph --all --decorate
```

What each flag does:
- `--oneline` = one line per commit (short hash plus message).
- `--graph` = draw lines showing how branches split and join.
- `--all` = show every branch, not just the one you are on.
- `--decorate` = show branch names next to the commits they point at.

Expected output:

```
*   4d5e6f7 (HEAD -> main) Merge branch 'feature-login'
|\
| * 3c4d5e6 (feature-login) Add password reset flow
| * 2b3c4d5 Add login form validation
* | 1a2b3c4 Hotfix: patch XSS in comment field
|/
* 0f1e2d3 Initial commit
```

How to read it:
- Read **bottom to top**: oldest at the bottom, newest at the top.
- Each `*` is one commit.
- `0f1e2d3 Initial commit` is the shared starting point.
- Then history splits. The `|` on the left is the `main` line (the hotfix),
  and the `| *` column is `feature-login` (two commits).
- `|/` shows the two lines closing back together.
- At the top, `4d5e6f7` is the merge commit that joins the two.
- `(HEAD -> main)` = you are on `main`, and it points at that merge commit.

How to map this to our left-to-right diagrams: `git log --graph` prints
newest at the TOP, our diagrams put newest at the RIGHT. So turn the log
output a quarter turn. Each `*` is a commit, each column of `|` is one row
of our diagram, `|/` is the fork point, and `|\` right under a merge is
where the merge commit's two parents split.

```
git log --graph (newest at TOP)

*   4d5e6f7  Merge branch 'feature-login'      <- M
|\
| * 3c4d5e6  Add password reset flow           <- F2
| * 2b3c4d5  Add login form validation         <- F1
* | 1a2b3c4  Hotfix: patch XSS                 <- H
|/
* 0f1e2d3  Initial commit                      <- A
```

```
Same history, our style (oldest at LEFT)

            F1---F2        feature-login
           /       \
      A---H---------M      main
                    ^
                    HEAD -> main

      A = 0f1e2d3   H = 1a2b3c4   F1 = 2b3c4d5   F2 = 3c4d5e6   M = 4d5e6f7
```

Here the left column `*` in the log is the `main` row (A, H, M); the `| *`
column is the `feature-login` row (F1, F2).

You will run this command constantly to *see* the shape of history instead
of guessing.

---

## Naming and Branching Strategy (Preview)

Full team workflows come in Lesson 08. A habit to start now: give branches
descriptive names with a prefix that says what kind of work it is.

```bash
git switch -c feature/add-retry-logic     # feature/ = new capability
git switch -c fix/null-pointer-login      # fix/ or bugfix/ = bug fix
git switch -c chore/upgrade-deps          # chore/ = maintenance, no new feature
```

Never do serious work directly on `main`. Branch first. Branches cost
almost nothing (one tiny file), so there is no reason not to.

---

## Exercise

Setup:

```bash
mkdir branch-practice && cd branch-practice && git init
echo "v1" > file.txt && git add . && git commit -m "Initial commit"
```

Expected: a commit output like `[main (root-commit) abc1234] Initial commit`.
(If your default branch is called `master` instead of `main`, that is fine.
Use that name below.)

1. Create branch `feature-a` and switch to it (`git switch -c feature-a`).
   Edit `file.txt` to say `v1 + feature A`. Commit.
2. Switch back to `main`. Create and switch to branch `feature-b`. Edit
   `file.txt` to say `v1 + feature B` (the *same line*). Commit.
   Check: `git log --oneline --graph --all` shows two branches growing
   from the initial commit.
3. Switch to `main` and run `git merge feature-a`.
   Expected: output contains `Fast-forward`. It worked because `main` had
   not moved since you branched.
4. Now run `git merge feature-b`.
   Expected: `CONFLICT (content): Merge conflict in file.txt`. This is
   deliberate: `main` now contains feature A's version of that line, and
   `feature-b` has a different version.
   Open `file.txt`, delete the markers, and write one line combining both
   changes, such as `v1 + feature A + feature B`. Then
   `git add file.txt` and `git commit`.
5. Run `git log --oneline --graph --all`.
   Expected: a merge commit at the top with two lines joining into it. Its
   two parents are the tip of `feature-a`/`main` before the merge, and the
   tip of `feature-b`. (Verify with `git show --no-patch --format=%P HEAD`,
   which prints the two parent hashes.)
6. Bonus: practice the escape hatch. Make two more conflicting branches,
   start a merge, then run `git merge --abort`. Expected: `git status`
   says `nothing to commit, working tree clean`.

---

## Recap in 5 Lines

1. A branch is a tiny file holding one commit hash. It is a label, not a copy.
2. HEAD points at your current branch. Committing moves that branch forward.
3. `git merge X` brings X into the branch you are standing on.
4. Fast-forward = just move the label. Three-way = new merge commit with two parents.
5. A conflict means both sides edited the same lines. Edit, delete the markers, `git add`, `git commit` (or `git merge --abort` to cancel).

Continue to [Lesson 06 — Undoing Things](06-undoing-things.md), arguably
the most valuable lesson for building real confidence with Git.
