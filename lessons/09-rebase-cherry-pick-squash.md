# Lesson 09 — Rebase, Cherry-Pick & Squashing History

## Goal
Master history rewriting: rebase vs merge (and when each is right),
interactive rebase for cleaning up commits, cherry-picking individual
commits, and squashing. This is what separates intermediate from advanced
Git users.

## Prerequisites
[Lesson 08 — Collaboration: Pull Requests](08-collaboration-pull-requests.md).
This lesson assumes you are comfortable with Lesson 06 (undoing things).
You will be rewriting history, so you must know your safety net (the
reflog, explained again below).

## After This Lesson You Will Be Able To
- Explain rebase vs merge and choose correctly between them
- Clean up a messy commit history with interactive rebase
- Cherry-pick a single commit onto another branch
- Know exactly why you should never rebase shared/public history

---

## Quick Vocabulary Refresher

You have seen these words before. Here they are in one place, because
this lesson uses all of them.

| Term | Plain meaning |
|---|---|
| **Commit** | A saved snapshot of your project, with a message and a parent (the commit before it). |
| **Hash** | The long ID of a commit (like `a1b2c3d`). It is calculated from the commit's content AND its parent. Change either one and you get a different hash. |
| **Branch** | A movable label pointing at one commit. It is a sticky note, not a copy of your files. |
| **HEAD** | "You are here." The commit you currently have checked out. |
| **Parent** | The commit directly before a given commit. |
| **`HEAD~4`** | "Four commits before HEAD." |
| **Reflog** | A private diary of every place `HEAD` has been. Your undo button for mistakes. |
| **Push** | Upload your commits to GitHub. |
| **Force-push** | Push even if it overwrites what is on GitHub. Dangerous if misused. |

Python analogy: a commit hash is like a `hashlib.sha1()` of a file's
contents plus the parent's hash. Change any input and the output is a
completely different string. Remember this; the whole lesson depends on it.

---

## Part 1 — Rebase vs Merge

### Why do we need this?

You create a branch `feature` and work on it. Meanwhile, your teammate
adds new commits to `main`. Now `main` and `feature` have gone in
different directions. You need to combine them. Git gives you two ways.

### Analogy: two ways to update your essay draft

You wrote a draft of an essay (your branch). Your editor has since
updated the master copy (`main`).

- **Merge** = staple your draft and the master copy together with a
  cover sheet that says "these two were combined here." Nothing is
  rewritten. The real timeline is preserved.
- **Rebase** = retype your draft from scratch on top of the newest
  master copy, so it looks like you started from the latest version all
  along. Cleaner to read, but the pages are new pages.

### Merge, drawn

Merge (Lesson 05) keeps both histories as they happened and adds one
extra commit (a **merge commit**) that joins them. A merge commit has two
parents, one from each side.

**Diagram legend** (used in every diagram in this lesson):
```
+-- Diagram legend ------------------------------------------------+
| A---B---C       commits: oldest on the LEFT, newest on the RIGHT
| main            a branch label (a sticky note on one commit)
| HEAD -> main    HEAD = the branch/commit you have checked out
| C'              "C prime": SAME change as C, but a NEW commit id
| D*              a commit just created by the command shown
| (E)             orphaned: no branch reaches it (still in reflog)
| /  and  \       a branch forking off, or merging back in
+------------------------------------------------------------------+
```

A prime (`'`) always means the same thing: **same change, new commit id**.
Git internally stores each commit's parent link (pointing to the older
commit), but we draw time flowing left to right.

How to read this: `main` moved on (E, F) while `feature` (C, D) forked
off at B. The merge joins them with a new commit `M*`. (Extra dashes in
`C-------D` are only spacing so the merge lines up.)

```
BEFORE   (HEAD -> feature)

            C---D              feature
           /
      A---B---E---F            main

$ git switch main
$ git merge feature

AFTER    (HEAD -> main)

            C-------D          feature
           /         \
      A---B---E---F---M*       main
                      ^
                HEAD -> main
```

What changed: the `main` label moved from F to the new merge commit
`M*`, which has two parents (F and D). `feature` did not move. A, B, C, D,
E, F are all unchanged.

### Rebase, drawn

Rebase takes YOUR commits (`C` and `D`) and replays them, one at a time,
on top of the current tip of `main`. "Replay" means: Git looks at what
`C` changed, applies that same change on top of `F`, and makes a new
commit. Then it does the same for `D`.

How to read this: same start as the merge. Rebase copies C and D onto
the tip of `main`. The copies are C' and D'. The old C and D are left
behind, reachable from no branch.

```
BEFORE   (HEAD -> feature)

            C---D                   feature
           /
      A---B---E---F                 main

$ git switch feature
$ git rebase main

AFTER    (HEAD -> feature)

                    C'*--D'*        feature
                   /                HEAD -> feature
      A---B---E---F                 main
           \
            (C)---(D)               orphaned, still in reflog
```

What changed: the `feature` label moved from D to the new commit `D'*`.
`C'*` and `D'*` are new commits (new ids, parent is now F). `main`, A, B,
E, F are unchanged. The old (C) and (D) are orphaned, still in reflog.

`C'` is read "C prime". It is a NEW commit with the same change and the
same message as `C`, but a different parent (`F` instead of `B`).

### Why are the hashes different?

A hash is calculated from content + parent. `C'` has a different parent
than `C`, so it gets a different hash. The old `C` and `D` are not edited.
They are simply no longer pointed to by any branch. They stay in Git's
database for a while and you can still find them with the reflog (Lesson 06).

### Common confusion: "Did rebase move my commits?"

No. Commits can never be moved or edited in Git. A commit is frozen.
Rebase makes **copies** on a new base and then moves your branch label to
the copies. The originals are abandoned.

Python analogy: strings are immutable. `s.upper()` does not change `s`,
it returns a new string. Rebase is the same: it returns new commits.

### Merge vs rebase on one picture

How to read this: one starting point, two different ending shapes.

```
START (identical for both)

            C---D              feature
           /
      A---B---E---F            main

RESULT 1: git merge   (keeps the fork, adds 1 commit M*)

            C-------D          feature
           /         \
      A---B---E---F---M*       main

RESULT 2: git rebase main   (straight line, C and D copied)

      A---B---E---F---C'*--D'*
                  ^         ^
                main     feature
```

What changed: merge adds 1 new commit and rewrites nothing. Rebase adds 2
new commits (C'*, D'*) and orphans the originals (C), (D) still in reflog.

### Side-by-side comparison

| | Merge | Rebase |
|---|---|---|
| History | Preserves exactly what happened, including the merge point | Rewrites to look like a straight line |
| Commit hashes | Original commits untouched | New hashes for every rebased commit |
| Result graph | Branching, with merge commits | One straight line |
| Safe on shared branches? | Always | **Only if nobody else has the old commits** |

### Common confusion: "Which one is better?"

Neither is "correct". They answer different needs.

- Want an honest record of what happened? Merge.
- Want a tidy, easy-to-read line of history? Rebase (on your own branch).

Many teams do: rebase locally to stay current, then merge (or squash
merge) through a PR on GitHub.

---

## Part 2 — The Golden Rule of Rebase

### Why do we need this?

Because rebase creates new commits, it can cause real trouble when other
people are involved. This is the most important safety rule in the lesson.

> **Never rebase commits that have been pushed and that someone else
> might have already pulled.**

### Analogy: the shared Google Doc

Imagine you and a colleague both have a printed copy of page 5. You then
secretly replace page 5 online with a retyped version. Your colleague
keeps writing notes on the old page 5. When you try to combine, the two
pages do not match, and you both waste time sorting it out.

### What actually goes wrong

1. You push commits `C` and `D`. Your teammate pulls them and builds on
   top (`E`).
2. You rebase. `C` and `D` become `C'` and `D'` (new hashes). You
   force-push.
3. Your teammate still has `C` and `D`. GitHub now says those commits do
   not exist on the branch. Git sees two different histories that both
   contain "the same" change, so you get confusing conflicts or duplicate
   commits.

How to read this: three rows = three copies of the same branch (GitHub,
you, your teammate). Watch what happens to C and D.

```
STEP 1: you pushed C and D; teammate pulled them and added E*

  GitHub   (origin/feature):  A---B---C---D
  You      (local feature):   A---B---C---D
  Teammate (local feature):   A---B---C---D---E*

STEP 2: you rebase onto main (X), then force-push

  $ git rebase main
  $ git push --force-with-lease

  GitHub   (origin/feature):  A---B---X---C'*--D'*
  You      (local feature):   A---B---X---C'*--D'*
  Teammate (local feature):   A---B---C---D---E
    (C and D no longer exist on GitHub, but the teammate still has them)

STEP 3: teammate runs git pull (Git merges the two histories)

          C-----D-----E         old C, D, E (teammate's)
         /             \
    A---B---X---C'--D'--M*      new C', D' (yours) + merge
```

What changed: the same two changes now exist twice (C/C' and D/D'), so the
pull can produce conflicts and a tangled history. Fix for the teammate:
`git fetch` then `git rebase --onto origin/feature <old-D> feature`
(see `--onto` in Part 3), which replays only E onto D'.

### When rebase IS safe (and a great habit)

- Your **local, not-yet-pushed** commits.
- A **personal feature branch** that nobody else uses.

Rebasing those to keep history clean before you open a PR is a good
habit.

### Doubt? "How do I know if someone else has my commits?"

Ask yourself two questions:
1. Have I pushed this branch? If no, you are safe.
2. If yes, is anyone else working on this branch? If it is only you, you
   are safe (but you will need a force-push, see below).

If unsure, do not rebase. Use merge.

---

## Part 3 — Using Rebase to Update Your Branch (The Common Case)

### Why do we need this?

While you work for days on a feature, `main` keeps moving. You want to
pick up those changes so you find conflicts early, without adding a merge
commit every time.

### The commands

```bash
git switch feature/my-branch
git fetch origin
git rebase origin/main
```

What each line does:

- `git switch feature/my-branch`: move to your branch.
- `git fetch origin`: download the newest commits from GitHub into your
  computer **without** changing your own branch. After this your local
  copy of GitHub's `main` (called `origin/main`) is up to date.
- `git rebase origin/main`: replay your branch's commits on top of that
  up-to-date `origin/main`.

Why `origin/main` and not `main`? `main` is your local copy, which may be
old. `origin/main` is the freshly fetched copy of what GitHub has.

### Expected output (no conflicts)

```
$ git rebase origin/main
Successfully rebased and updated refs/heads/feature/my-branch.
```

Meaning: all your commits were replayed cleanly and the branch label
`feature/my-branch` now points at the new copies.

### Expected output (with a conflict)

A **conflict** means two commits changed the same lines, and Git cannot
decide which to keep. Git stops and asks you:

```
$ git rebase origin/main
Auto-merging config.py
CONFLICT (content): Merge conflict in config.py
error: could not apply 3f2a1b9... Add retry setting
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply 3f2a1b9... Add retry setting
```

Line by line:

- `CONFLICT (content): Merge conflict in config.py`: the problem file.
- `could not apply 3f2a1b9... Add retry setting`: Git is replaying your
  commits one at a time, and THIS one is the one that clashes.
- The `hint:` lines tell you the exact next commands.

Fix it:

```bash
# 1. open config.py, fix the <<<<<<< ======= >>>>>>> markers (same as Lesson 05)
git add config.py            # tell Git "this file is now resolved"
git rebase --continue        # replay the next commit
```

Or bail out completely:

```bash
git rebase --abort     # returns you to exactly where you were before rebasing
```

### Common confusion: "Who is 'ours' and who is 'theirs' in a rebase?"

It feels backwards. During a rebase:

- "ours" / HEAD side = the branch you are rebasing ONTO (`main`'s code).
- "theirs" = YOUR commit being replayed.

Reason: Git is rebuilding your work on top of `main`, so `main` is the
"current" state and your commit is the "incoming" change.

### Pushing after a rebase

If you had already pushed this branch before rebasing, GitHub still has
the OLD commits. Your local branch now has NEW ones, so a normal push is
refused:

```
$ git push
 ! [rejected]        feature/my-branch -> feature/my-branch (non-fast-forward)
error: failed to push some refs to 'github.com:you/repo.git'
hint: Updates were rejected because the tip of your current branch is behind
```

(`non-fast-forward` means: your branch is not a simple continuation of
what GitHub has, because history was rewritten.)

How to read this: after a rebase, your local branch and GitHub's copy
disagree about how the branch began. Neither tip contains the other, so a
normal push cannot just "add" commits.

```
local feature    A---B---X---C'--D'
origin/feature   A---B---C---D

$ git push                      ---> rejected (non-fast-forward)
$ git push --force-with-lease   ---> origin/feature now = C'--D'
```

You must force-push, always with `--force-with-lease` (Lesson 07):

```bash
git push --force-with-lease
```

```
+ 3f2a1b9...8c7d6e5 feature/my-branch -> feature/my-branch (forced update)
```

Why `--force-with-lease` and not `--force`? It means "overwrite GitHub's
copy ONLY if it is still what I last saw." If a teammate pushed something
in the meantime, Git refuses, instead of silently destroying their work.
Plain `--force` has no such check.

### Moving a slice of commits: `git rebase --onto`

`git rebase main` moves EVERYTHING on your branch since it forked. Sometimes
you want to move only PART of a branch. Classic case: `feature2` was
started on top of `feature1`, then `feature1` was squash-merged into
`main` (as S). You want `feature2`'s own commits (E, F) on `main`, without
the already-merged C and D.

```bash
git rebase --onto main feature1 feature2
```

Read it as: "take the commits after `feature1` up to `feature2`, and replay
them onto `main`."

How to read this: `feature1` is a label on D, `feature2` is a label on F.

```
BEFORE

            C---D---E---F       feature2  (feature1 is on D)
           /
      A---B---S                 main

$ git rebase --onto main feature1 feature2

AFTER   (HEAD -> feature2)

            C---D---(E)-(F)     feature1 on D; (E),(F) orphaned
           /                    HEAD -> feature2
      A---B---S---E'*--F'*      feature2
```

What changed: `feature2` moved from F to `F'*`. E and F were copied to
E'* and F'* on top of S. C and D were NOT copied. `feature1` did not
move. Old (E), (F) are orphaned, still in reflog.

---

## Part 4 — Interactive Rebase: Cleaning Up Your Own History

### Why do we need this?

While working you make commits like "wip", "fix typo", "actually fix it".
That is normal while you work, but reviewers do not want to read it.
Interactive rebase lets you tidy those into one or two meaningful commits
before you share them.

Python analogy: this is like refactoring your code before code review.
The logic is the same, but it is organised so others can follow it.

### Setup: "squash" explained

**Squash** = combine several commits into one. The final files are
identical. Only the number of commits (and their messages) changes.

### The command

```bash
git rebase -i HEAD~4     # interactively rewrite the last 4 commits
```

- `-i` = interactive. Git opens a text file in your editor and lets YOU
  decide what happens to each commit.
- `HEAD~4` = "the last 4 commits".

Before running this, `git log --oneline` shows:

```
k1l2m3n actually fix the typo for real
h7i8j9k fix typo
e4f5g6h wip
a1b2c3d Add search endpoint
9z8y7x6 Initial commit
```

Each line is: short hash, then message. Newest at the top. The last 4
(`a1b2c3d` to `k1l2m3n`) are the ones being rewritten.

### What opens in the editor

```
pick a1b2c3d Add search endpoint
pick e4f5g6h wip
pick h7i8j9k fix typo
pick k1l2m3n actually fix the typo for real

# Commands:
# p, pick   = keep commit as-is
# r, reword = keep content, change commit message
# e, edit   = pause here to amend the commit (add files, split it, etc.)
# s, squash = combine into PREVIOUS commit, keep both messages (merged)
# f, fixup  = combine into PREVIOUS commit, DISCARD this commit's message
# d, drop   = delete this commit entirely
```

### Common confusion: "Why is the OLDEST commit at the top here?"

In `git log`, newest is at the top. In the rebase editor, OLDEST is at the
top. This is because the editor lists the order Git will **replay** them:
first to last. Expect this flip every time.

### What each word means

- `pick` = use the commit as it is.
- `reword` = keep the changes, let me rewrite the message.
- `edit` = stop here so I can change this commit.
- `squash` = glue this commit onto the one ABOVE it, and let me combine
  both messages.
- `fixup` = glue this commit onto the one ABOVE it and throw away its
  message.
- `drop` = delete this commit and its changes.

Note "above", not "below": a `squash` or `fixup` always merges into the
commit directly above it in this list. That is why the first line must
stay `pick`. (There would be nothing above it to merge into.)

How to read this: the editor is a to-do list; the word on the left decides
the fate of each commit. Here letters stand for six commits on top of A
(B is the oldest, so it is listed first).

```
EDITOR LIST (top = oldest)      RESULT

  pick    B  Add login       --->  B'  kept (new id, C folds in)
  fixup   C  typo            --->  folded into B', message discarded
  reword  D  wip             --->  D'  same change, new message
  pick    E  Add tests       --->  E'  kept
  squash  F  more tests      --->  folded into E', keep both messages
  drop    G  debug print     --->  deleted: its change is gone
```

```
BEFORE

  A---B---C---D---E---F---G       feature

$ git rebase -i HEAD~6   (editor list above)

AFTER

  A---B'*--D'*--E'*               feature
   \
    (B)-(C)-(D)-(E)-(F)-(G)       orphaned, still in reflog
```

What changed: `feature` moved from G to `E'*`. Six commits became three new
ones (B'*, D'*, E'*). C and F were absorbed, G was deleted, and all six
originals are orphaned.

### Squash the messy commits into the first one

Change the file to:

```
pick   a1b2c3d Add search endpoint
fixup  e4f5g6h wip
fixup  h7i8j9k fix typo
fixup  k1l2m3n actually fix the typo for real
```

Save and close the editor.

### Expected output

```
Successfully rebased and updated refs/heads/feature.
```

Now:

```
$ git log --oneline
7q8r9s0 Add search endpoint
9z8y7x6 Initial commit
```

Notice: the hash changed (`a1b2c3d` became `7q8r9s0`). It is a new commit
containing the combined changes of all four. Four entries are now one.

How to read this: the four messy commits become ONE new commit. `7q8r9s0`
is `a1b2c3d` with the other three folded in.

```
BEFORE

  9z8y7x6---a1b2c3d---e4f5g6h---h7i8j9k---k1l2m3n  feature

$ git rebase -i HEAD~4    (pick, fixup, fixup, fixup)

AFTER

  9z8y7x6---7q8r9s0*                               feature
         \
          (a1b2c3d)-(e4f5g6h)-(h7i8j9k)-(k1l2m3n)  orphaned, still in reflog
```

### Common confusion: "What if I want to keep the messages?"

Use `squash` instead of `fixup`. Git opens a second editor showing all the
messages joined together, and you edit the result.

### Common confusion: "I messed up in the editor!"

- To cancel before starting: delete all lines in the file (or make it
  empty) and save. Git aborts.
- To cancel during the rebase: `git rebase --abort`.
- After it finished: use the reflog (see the safety-net box below).

### Safety net: undo a finished rebase

```bash
git reflog
```

```
7q8r9s0 HEAD@{0}: rebase (finish): returning to refs/heads/feature
7q8r9s0 HEAD@{1}: rebase (squash): Add search endpoint
k1l2m3n HEAD@{5}: commit: actually fix the typo for real
```

Find the last line BEFORE the rebase began (here `k1l2m3n HEAD@{5}`) and
go back:

```bash
git reset --hard k1l2m3n
```

`--hard` moves your branch there and makes your files match. Your old
commits are back. (Lesson 06 explains this.)

**Rule:** only rewrite commits **you haven't pushed yet**, or on a branch
you are certain nobody else has pulled.

### Even quicker: `ORIG_HEAD`

Before a rebase (or merge, or reset) moves your branch, Git saves where
you were in `ORIG_HEAD`. Right after a bad rebase you can jump straight
back without searching the reflog:

```bash
git reset --hard ORIG_HEAD
```

How to read this: the rebase gave you C' and D' but you want your old C and
D back. Resetting to `ORIG_HEAD` moves the `feature` label back to D.

```
BEFORE   (rebase just finished, you regret it)

            (C)---(D)               orphaned, still in reflog (ORIG_HEAD = D)
           /
      A---B---X---C'--D'            feature
                       ^
                 HEAD -> feature

$ git reset --hard ORIG_HEAD

AFTER

            C---D                   feature
           /                        HEAD -> feature
      A---B---X---(C')-(D')         orphaned, still in reflog
```

What changed: the `feature` label moved from D' back to D. C and D are
reachable again; C' and D' are now the orphans. `ORIG_HEAD` is overwritten
by the next rebase/merge/reset, so use it immediately; otherwise use
`git reflog` as above.

---

## Part 5 — `--fixup` and `--autosquash`: The Pro Habit

### Why do we need this?

Manually moving `fixup` lines around the editor works for 2 or 3 commits.
With many stray fixes it becomes error-prone. Better: tell Git WHEN you
make the fix which earlier commit it belongs to, and let Git arrange the
editor for you.

Analogy: putting a sticky note on an earlier page saying "the correction
for this page is on page 12", rather than tidying all your notes at the
end.

### Step 1: make the fix commit

```bash
git log --oneline
```

```
k1l2m3n Update docs
h7i8j9k Add search endpoint        <- the one we actually meant to fix
```

```bash
git add fixed_file.py
git commit --fixup h7i8j9k
```

Expected output:

```
[feature 4d5e6f7] fixup! Add search endpoint
 1 file changed, 1 insertion(+), 1 deletion(-)
```

Line by line:
- `[feature 4d5e6f7]`: the branch and the new commit's hash.
- `fixup! Add search endpoint`: Git automatically named it "fixup!" plus
  the target's message. This special prefix is how Git recognises it
  later.

### Step 2: run the rebase with `--autosquash`

```bash
git rebase -i --autosquash HEAD~4
```

(You can enable it permanently so you never type the flag:
`git config --global rebase.autosquash true`.)

The editor opens with the `fixup!` commit **already moved directly below
its target and already marked `fixup`**:

```
pick   h7i8j9k Add search endpoint
fixup  4d5e6f7 fixup! Add search endpoint
pick   k1l2m3n Update docs
```

Just save and close. Result: the fix is folded into `Add search endpoint`
as if you had never made the mistake.

### `--squash` variant

`git commit --squash <hash>` works the same, except the new commit's own
message is kept too (combined with the original for you to edit). Use it
when the fix deserves a note in the final message.

### Doubt? "fixup vs squash vs --fixup vs --squash?"

Two different things with similar names:

- `fixup` / `squash` (no dashes): words you type in the rebase editor.
- `--fixup` / `--squash` (dashes): flags on `git commit` that create a
  specially named commit so the rebase can sort it for you.

---

## Part 6 — Splitting a Commit With `rebase -i edit`

### Why do we need this?

The opposite of squashing. Sometimes one commit did TOO much (for
example, a feature plus an unrelated typo fix). Reviewers prefer small,
focused commits, so you split it.

How to read this: one fat commit H becomes two commits H1 and H2. K (the
commit after it) is replayed on top, so it becomes K'.

```
BEFORE   (HEAD -> feature)

  A---B---H---K                 feature

$ git rebase -i HEAD~3       (mark H as 'edit')

PAUSED at H, then: git reset HEAD~1

  A---B---H---K                 feature  (not moved yet)
      ^
  HEAD -> B   (H's changes now sit UNSTAGED in your files)

$ git add ... ; git commit   (twice)
$ git rebase --continue

AFTER    (HEAD -> feature)

  A---B---H1*--H2*--K'*         feature
       \
        (H)---(K)               orphaned, still in reflog
```

What changed: `feature` moved from K to `K'*`. H1* and H2* are new commits
that together contain H's changes. K' has a new id because its parent
changed. (H) and (K) are orphaned.

### Step 1: start the rebase and mark the commit

```bash
git rebase -i HEAD~3
```

Change `pick` to `edit` for the offending commit:

```
edit   h7i8j9k Add search endpoint (and accidentally fix an unrelated typo)
pick   k1l2m3n Update docs
```

Save and close. Git replays up to that commit and then STOPS:

```
Stopped at h7i8j9k...  Add search endpoint (and accidentally fix an unrelated typo)
You can amend the commit now, with

  git commit --amend

Once you are satisfied with your changes, run

  git rebase --continue
```

Meaning: you are now paused with that commit as the latest one. Your
changes are already committed. You get to modify history at this point.

### Step 2: undo the commit but keep the changes

```bash
git reset HEAD~1
```

This is a **mixed** reset (the default). It:
- removes the commit (moves the branch back one step),
- keeps your file changes on disk,
- but un-stages them (they are back to "modified, not yet staged").

(Lesson 06 covers `--soft`, `--mixed`, `--hard`. Plain `git reset` is
`--mixed`, not `--soft`.)

Check:

```
$ git status --short
 M search_endpoint.py
 M unrelated_file.py
```

The space before `M` means "modified but not staged". Both files are waiting.

### Step 3: re-commit in separate pieces

```bash
git add search_endpoint.py
git commit -m "Add search endpoint"

git add unrelated_file.py
git commit -m "Fix unrelated typo in unrelated_file.py"

git rebase --continue      # replays the remaining commits (k1l2m3n, ...) on top
```

Expected final output:

```
Successfully rebased and updated refs/heads/feature.
```

Now `git log --oneline` shows two focused commits where there used to be
one. Commits after it were replayed too, so they have new hashes
(Lesson 03).

### Doubt? "What if the two changes are in the SAME file?"

Use `git add -p` (patch mode) instead of `git add <file>`. Git shows you
each changed chunk and asks `y` (stage it) or `n` (skip it). You then
commit only the chosen chunks.

---

## Part 7 — Cherry-Pick: Taking One Specific Commit

### Why do we need this?

Merge and rebase bring in a WHOLE branch. Sometimes you want only ONE
commit. Classic case: while working on a feature you fixed a critical bug
in one commit. That fix must reach `main` now, but the rest of the feature
is not ready.

Analogy: picking one cherry from a tree instead of taking the whole
branch.

### Step 1: find the commit's hash

```bash
git log --oneline feature
```

```
c9d8e7f Work in progress on UI
a1b2c3d Fix crash on empty input          <- the hotfix we want
b3c4d5e Start new UI
```

### Step 2: go to the destination branch and pick

```bash
git switch main
git cherry-pick a1b2c3d       # applies JUST that commit's changes onto main, as a new commit
```

Expected output:

```
[main 5f6a7b8] Fix crash on empty input
 Date: Mon Sep 29 10:12:33 2026 +0530
 1 file changed, 2 insertions(+)
```

Line by line:
- `[main 5f6a7b8]`: a new commit was created on `main` with hash `5f6a7b8`.
  Note it is NOT `a1b2c3d`.
- The message is copied from the original.
- The last line summarises the change.

How to read this: `feature` has three commits; E (`a1b2c3d`) is the hotfix.
Cherry-pick copies ONLY E onto `main`.

```
BEFORE   (HEAD -> main)

            D---E---F          feature  (E = a1b2c3d)
           /
      A---B---C                main

$ git cherry-pick a1b2c3d

AFTER    (HEAD -> main)

            D---E---F          feature  (unchanged)
           /
      A---B---C---E'*          main
                   ^
              HEAD -> main
```

What changed: `main` moved from C to the new commit `E'*` (same change as
E, new id, parent is C instead of B). `feature` and E are untouched.

### Common confusion: "Is it the same commit now on both branches?"

No. It is two separate commits with the same change and message but
different hashes, because their parents differ. That is why cherry-pick
is often called "copying" a commit.

### Common confusion: "If I later merge feature into main, will E apply twice?"

Usually Git notices that the change is already present and handles it
without harm. Sometimes it can produce a small conflict. It is not
dangerous, just occasionally noisy.

### If it conflicts

```bash
# resolve conflicts in the files, then:
git add <file>
git cherry-pick --continue
# or bail out:
git cherry-pick --abort
```

### Multiple commits

```bash
git cherry-pick a1b2c3d e4f5g6h
```

Applied in the order listed.

### Tip: add `-x`

`git cherry-pick -x a1b2c3d` adds "(cherry picked from commit a1b2c3d)"
to the new message. Helpful for tracing where it came from.

---

## Part 8 — Squash Merging (Command-Line Version)

### Why do we need this?

In Lesson 08 you saw GitHub's "Squash and merge" button. It takes all
commits of a PR and turns them into ONE commit on `main`. If you are
not using GitHub's button, you can do the same locally.

```bash
git switch main
git merge --squash feature/my-branch    # applies all changes, but does NOT commit yet
git commit -m "Add search endpoint (squashed from feature branch)"
```

Expected output of the merge step:

```
Updating 9z8y7x6..c9d8e7f
Fast-forward
Squash commit -- not updating HEAD
 search_endpoint.py | 20 ++++++++++++++++++++
 1 file changed, 20 insertions(+)
```

Line by line:
- "Squash commit -- not updating HEAD": Git staged all the changes but did
  not commit. You decide the final message.
- The file list shows what will be in your commit.

How to read this: `feature` has three commits; a squash merge turns all of
them into ONE new commit on `main`, with a single parent.

```
BEFORE   (HEAD -> main)

            C---D---E          feature
           /
      A---B---F                main

$ git merge --squash feature
$ git commit -m "Add search endpoint"

AFTER    (HEAD -> main)

            C---D---E          feature  (NOT marked as merged)
           /
      A---B---F---S*           main
```

What changed: `main` moved from F to `S*`, one commit holding the changes of
C, D and E together. S has one parent (F), not two, so Git has no record
that `feature` was merged. C, D, E stay unchanged on `feature`.

### Common confusion: "Why does Git not remember I merged the branch?"

A normal merge commit has two parents, so Git knows the branch was merged.
A squash commit has only ONE parent (the previous `main` commit). It is
just a brand-new commit that happens to contain the same changes.

Consequence: Git thinks `feature/my-branch` is still unmerged. Do not try
to merge the same branch into `main` again, and after squashing, delete the
branch (`git branch -D feature/my-branch`; capital `-D` because Git will
not consider it merged).

---

## Part 9 — `git rerere`: Never Resolve the Same Conflict Twice

### Why do we need this?

If you rebase a long-lived feature branch onto `main` repeatedly, the same
lines can clash every time, and you fix the same conflict again and again.

`rerere` stands for "**re**use **re**corded **re**solution". Think of it
as a cache: Python's `functools.lru_cache` for conflict fixes.

### Turn it on (one time)

```bash
git config --global rerere.enabled true
```

This sets a global option so it applies to all your repos.

### How it works

1. First time you resolve a conflict, Git silently records
   "this clash -> this fix".
2. Next time the exact same clash appears (in a rebase, merge or
   cherry-pick), Git applies your earlier fix automatically.

You will see:

```
Resolved 'config.py' using previous resolution.
```

Still check the result and run `git add` for it, since Git applies the fix
but may still pause the rebase for your confirmation.

### Helper commands

```bash
git rerere status     # files with a recorded resolution available
git rerere diff       # preview the resolution rerere applied or will apply
```

**Where it pays off:** a feature branch rebased daily for weeks that
keeps hitting the same conflicts. It is not useful for a one-off conflict.

---

## Part 10 — Choosing the Right Tool: Quick Decision Guide

```
Want to combine two branches, keeping full history + a merge point?
  -> git merge

Want your branch's commits to look like they were just written on top
of the latest main, with NO merge commit, before opening/updating a PR?
  -> git rebase (only if not yet pushed/shared)

Want to clean up your OWN messy commits before anyone sees them?
  -> git rebase -i

Want just ONE commit from another branch, not the whole thing?
  -> git cherry-pick

Want a whole PR's worth of commits squashed into one commit on merge?
  -> GitHub's "Squash and merge" button, or git merge --squash
```

---

## Exercise

Work in a throwaway folder. Nothing here touches real projects.

```bash
mkdir rebase-practice && cd rebase-practice && git init
echo "v1" > f.txt && git add . && git commit -m "Initial"
git switch -c feature
```

Expected: `Initialized empty Git repository ...`, a commit line like
`[main (root-commit) 1a2b3c4] Initial`, and `Switched to a new branch 'feature'`.

1. **Make messy commits.** On `feature`, make 4 small commits editing
   `f.txt` (for example):
   ```bash
   echo "wip" >> f.txt && git commit -am "wip"
   echo "fix" >> f.txt && git commit -am "fix"
   echo "actually fix" >> f.txt && git commit -am "actually fix"
   echo "final fix" >> f.txt && git commit -am "final fix"
   ```
   Check: `git log --oneline` shows 5 commits (4 messy + Initial).

2. **Squash.** Run `git rebase -i HEAD~4`. Keep the first line as `pick`
   and change the other three to `fixup`. Save.
   Expected: `Successfully rebased and updated refs/heads/feature.`
   Check: `git log --oneline` shows 2 commits (your squashed "wip" commit
   and "Initial"). Use `reword` on the first line if you want a nicer
   message. `cat f.txt` still contains all the lines.

3. **Diverge.** Note the squashed commit hash. Then:
   ```bash
   git switch main
   echo "teammate" > teammate.txt && git add . && git commit -m "Teammate change"
   ```
   Expected: `main` has 2 commits; `feature` still has its own.

4. **Rebase.**
   ```bash
   git switch feature
   git rebase main
   git log --oneline
   ```
   Expected: `Successfully rebased ...`. The log shows
   `Initial`, `Teammate change`, then your squashed commit on top. Compare
   its hash with step 3: it is DIFFERENT. That proves rebase made a new
   commit.

5. **Cherry-pick.**
   ```bash
   git switch main
   git switch -c hotfix
   echo "hot" > hot.txt && git add . && git commit -m "Hotfix"
   git log --oneline -1          # copy this hash
   git switch feature
   git cherry-pick <that-hash>
   ```
   Expected: `[feature ...] Hotfix`. `git log --oneline` on `feature` now
   shows the Hotfix commit with a hash different from the one on
   `hotfix`. `ls` shows `hot.txt`, but the `hotfix` branch was never merged.

6. **Bonus: undo with the reflog.** Run `git reflog`, find the entry
   from before step 2, and `git reset --hard <that-hash>` to see your four
   messy commits return. Then get back with `git reflog` again.

---

## Recap in 5 Lines

1. Merge joins histories with a merge commit; rebase replays your commits as NEW commits on top of another branch.
2. Rebase creates new hashes, so never rebase commits others may already have.
3. `git rebase -i` lets you pick, reword, squash/fixup, edit or drop your own commits before sharing.
4. `git cherry-pick <hash>` copies one commit onto your current branch as a new commit.
5. If you rewrite pushed history, use `--force-with-lease`; if you slip, `git reflog` can get your old commits back.

Continue to [Lesson 10 — Stash, Tags & Bisect](10-stash-tags-bisect.md).
