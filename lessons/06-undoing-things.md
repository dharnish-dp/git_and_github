# Lesson 06 — Undoing Things: checkout, restore, reset, revert, reflog

## Goal
Build real confidence: learn every "undo" tool in Git, when to use each,
and the ultimate safety net (`reflog`) that means **you almost never
truly lose committed work**, even after a scary mistake.

## Prerequisites
[Lesson 05 — Branching & Merging](05-branching-and-merging.md)

## After This Lesson You Will Be Able To
- Discard uncommitted changes safely
- Unstage files without losing edits
- Choose correctly between `reset` and `revert`
- Recover a branch or commit you thought was deleted forever

---

## Words We Will Use (quick refresher)

You met these earlier. Here is a one-line reminder of each, because this
lesson leans on them heavily.

- **Commit** = a saved snapshot of your whole project, with a message, an
  author, and a unique ID (a long hash like `9fceb02...`).
- **Hash** = that unique ID of a commit. We usually show only the first 7
  characters, like `9fceb02`.
- **Working directory** = the real files on your disk that you edit.
- **Staging area** (also called the "index") = a waiting room. You put
  changes there with `git add` to say "these go into my next commit".
- **Branch** = a sticky note that points at one commit. `main` is a branch.
- **HEAD** = a second sticky note that says "you are here". Normally it
  points at your current branch, which points at a commit.
- **Tracked file** = a file Git already knows about (it was committed or
  added before). **Untracked file** = a brand-new file Git has never seen.
- **Pushed** = you sent your commits to a shared copy on GitHub so other
  people can get them.

Python analogy: think of the three places as
`disk file (working dir)` -> `pending list (staging)` -> `saved version (commit)`.

```
 working directory  --git add-->  staging area  --git commit-->  repository (commits)
 (files you edit)                 (waiting room)                 (permanent snapshots)
```

Every undo command in this lesson is just "move something backward along
this line". Keep this picture in your head.

```
Diagram legend (used in every diagram below)
+--------------------------------------------------------------------+
| A---B---C     commits; time flows LEFT (old) to RIGHT (new)        |
| ^             a label under a commit = a pointer sitting on it      |
| main          branch label.  "HEAD -> main" = you are on main       |
| D*            a NEW commit created by the command just run          |
| (C)           commit no branch reaches any more (still in reflog)   |
|   E---F       a fork: each branch on its own row, name at row end   |
|  /                                                                  |
| A---B---C     main                                                  |
+--------------------------------------------------------------------+
Note: internally Git stores each commit's PARENT link (C knows B), but we
draw time flowing left to right, so you never need to read arrows backward.
```

---

## The Golden Rule First

> **If a change was ever committed, it is almost never truly gone** — even
> if you deleted the branch, even if you force-reset. Git keeps every
> commit object around (Lesson 03) until garbage collection eventually
> cleans up genuinely unreachable ones, which by default doesn't happen
> for weeks. The `reflog` (bottom of this lesson) is your recovery map.
>
> **Uncommitted changes are the only things that can vanish easily.** This
> is the actual argument for committing often, even with messy
> "WIP" (work in progress) commits you'll clean up later (Lesson 09) — a
> bad commit is recoverable; an uncommitted deleted file often isn't.

**Why do we need this rule?** Because fear makes beginners avoid
experimenting. Knowing committed work is safe lets you try things.

Analogy: a committed change is like a file saved in Google Drive with
version history. An uncommitted change is like text typed in Notepad and
never saved. If Notepad closes, it is gone.

**Garbage collection (gc)** = Git's automatic cleanup that eventually
deletes commits nothing points to anymore. "Unreachable" means: no branch,
tag, or reflog entry leads to that commit.

---

## Undo Map — Which Tool for Which Situation

Do not memorize this table. Skim it now, then come back after each section.

| Situation | Command |
|---|---|
| Discard edits in a file, not yet staged | `git restore <file>` |
| Unstage a file (keep the edits) | `git restore --staged <file>` |
| Discard **everything** uncommitted (staged + unstaged) | `git reset --hard HEAD` |
| Change your last commit's message or add forgotten files | `git commit --amend` |
| Undo the last commit but keep the changes in your working dir | `git reset --soft HEAD~1` |
| Undo the last commit and discard its changes entirely | `git reset --hard HEAD~1` |
| Undo a commit that's **already been pushed/shared** | `git revert <commit>` |
| Recover a commit/branch you thought was gone | `git reflog` |

Beginner question: "Which places does each situation touch?"

```
Problem is in...            Tool
-----------------------     ---------------------------
working directory           git restore <file>
staging area                git restore --staged <file>
last commit (local only)    git commit --amend  /  git reset
commit already pushed       git revert
"I lost something"          git reflog
```

---

## Step 1: `git restore` — Discarding Working Directory Changes

**Why do we need this?** You edited a file, things went wrong, and you want
the file back the way it was at the last commit.

Analogy: in Python you edit `test_login.py`, break everything, and want
`Ctrl+Z` all the way back to the last saved-in-Git version. `git restore`
is that "throw away my changes" button.

```bash
git restore file.py           # throws away UNSTAGED edits to file.py, reverting to last commit's version
git restore .                 # do that for every file in current dir and below
git restore --staged file.py  # unstage it — moves it from "staged" back to "modified," edits are kept
```

Let us see it with real output. Say `file.py` was committed with
`print("hello")`, and you changed it to `print("broken")`.

```bash
git status
```
```
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   file.py

no changes added to commit (use "git add" and/or "git commit -a")
```

Line by line:
- `Changes not staged for commit` = the edit is in your working directory
  only. It is not in the waiting room.
- `(use "git restore <file>..." to discard changes...)` = Git itself tells
  you the command for this exact situation.
- `modified: file.py` = Git sees the file differs from the last commit.

Always look at what you are about to lose:

```bash
git diff file.py
```
```
diff --git a/file.py b/file.py
index 3b18e51..a9c2f04 100644
--- a/file.py
+++ b/file.py
@@ -1 +1 @@
-print("hello")
+print("broken")
```

- `-` line = what the last commit has. `+` line = what you have now.
- `git restore` will make the `+` line disappear and bring back the `-` line.

Now discard:

```bash
git restore file.py
git status
```
```
On branch main
nothing to commit, working tree clean
```

`git restore` prints nothing when it succeeds. That silence is normal.
"working tree clean" means your files match the last commit again.

**This is destructive for the file's uncommitted edits.** There is no undo
for `git restore file.py` itself. Those edits were never saved by Git, so
Git has nothing to bring back. Always `git status` and `git diff` first.

### Unstaging: `git restore --staged`

**Why do we need this?** You ran `git add` on a file by mistake and do not
want it in the next commit. But you also do not want to lose your edits.

```bash
git add file.py
git status
```
```
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   file.py
```

`Changes to be committed` = the file is in the waiting room.

```bash
git restore --staged file.py
git status
```
```
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   file.py
```

The file moved back from "staged" to "not staged". Your edits (`print("broken")`)
are still on disk. Only the waiting-room entry was removed.

> **Common confusion: `git restore file` vs `git restore --staged file`**
>
> | Command | Changes the waiting room? | Changes your file on disk? |
> |---|---|---|
> | `git restore file.py` | No | **Yes — edits lost** |
> | `git restore --staged file.py` | Yes (removes it) | No — edits kept |
>
> The word `--staged` means "operate on the waiting room, not on my files".
> No `--staged` means "operate on my files". When unsure, use `--staged`;
> it is the safe one.

> **Doubt? "I see old tutorials using `git checkout -- file`. Same thing?"**
> Yes. Before Git 2.23, `git checkout` did many unrelated jobs (switch
> branches AND discard edits), which confused everyone. `git restore`
> (for files) and `git switch` (for branches) were added to split those
> jobs. `git checkout -- file.py` still works and does the same as
> `git restore file.py`. Prefer `restore` because its name says what it does.

---

## Step 2: `git reset` — Moving Branch Pointers Backward

**Why do we need this?** `restore` fixes files. But sometimes the mistake
is already a *commit*. `reset` lets you step backward over commits.

Recall: a branch is just a pointer to a commit (Lesson 05). `git reset`
**moves that pointer**, and optionally also changes your staging area
and/or working directory to match. This is the single most
misunderstood command in Git because it has three modes.

First, learn the shorthand for "a commit before this one":

- `HEAD` = the commit you are on now.
- `HEAD~1` = one commit before HEAD (its parent).
- `HEAD~3` = three commits before HEAD.
- You can also use a hash: `git reset --hard a1b2c3d`.

(The `~` is "tilde", next to the Esc key on most keyboards.)

Analogy: history is a stack of saved game slots. `reset` says "load slot 2
and pretend slots 3+ don't exist on this branch". The three modes decide
what to do with the progress you made *after* slot 2.

How to read this: the row of letters is history; the labels under it show
what main/HEAD point at; the two lines beneath show the other two "trees".
Commit C is the one that changed `f.txt`.

```
BEFORE (all three modes start from exactly this)

  A---B---C
          ^
        main
        HEAD -> main

  Staging area  : same as C   (nothing waiting)
  Working folder: same as C   (f.txt has C's edits)
```

The three modes move the pointer the same way (main goes from C to B).
They only differ in what happens to C's changes. Each AFTER below starts
from the BEFORE above.

```
git reset --soft HEAD~1
AFTER --soft

  A---B---(C)
      ^
    main
    HEAD -> main

  Staging area  : C's changes are STAGED  (ready to commit again)
  Working folder: unchanged (f.txt still has C's edits)
```
What changed: main (and HEAD with it) moved C -> B. Only the pointer moved.
C is still in the object store, reachable through reflog.

```
git reset --mixed HEAD~1        (the default)
AFTER --mixed

  A---B---(C)
      ^
    main
    HEAD -> main

  Staging area  : EMPTY  (reset to match B)
  Working folder: unchanged (f.txt still has C's edits, now "modified")
```
What changed: main moved C -> B AND the staging area was reset to B.
Your files are untouched, so no edits are lost.

```
git reset --hard HEAD~1
AFTER --hard

  A---B---(C)
      ^
    main
    HEAD -> main

  Staging area  : EMPTY  (reset to match B)
  Working folder: RESET to B  -> C's edits are GONE from your files
```
What changed: main moved C -> B, staging reset, AND files overwritten to
match B. The commit (C) survives in reflog; C's file edits only survive
*because they were committed* (an uncommitted edit here would be lost).

Compare side by side:

| Mode | Branch pointer | Staging area | Your files on disk |
|---|---|---|---|
| `--soft` | moves back | **keeps** the undone changes staged | untouched |
| `--mixed` (default) | moves back | cleared | untouched (edits remain) |
| `--hard` | moves back | cleared | **reset to match — edits GONE** |

Memory trick: soft = gentle (changes stay staged). mixed = medium (changes
stay in files). hard = harsh (changes destroyed).

```bash
git reset --soft  HEAD~1     # move branch pointer back 1 commit.
                              #   Staging area: KEEPS old commit's changes staged.
                              #   Working dir:  untouched.
                              #   Use when: "I want to redo this commit's message/contents"

git reset --mixed HEAD~1     # (this is the DEFAULT if you omit the flag)
                              #   Staging area: cleared — changes become unstaged.
                              #   Working dir:  untouched (files still have the edits).
                              #   Use when: "Undo the commit AND unstage it, but keep my edits to redo"

git reset --hard  HEAD~1     # Staging area: cleared.
                              #   Working dir: RESET to match — all edits GONE.
                              #   Use when: "I want to completely erase this commit and its changes"
```

### Seeing it: `--soft`

Start with this history:

```bash
git log --oneline
```
```
9fceb02 (HEAD -> main) Add retry logic
5c3a1e2 Fix typo
a1b2c3d Initial test suite
```

Each line is one commit: hash, then message. `HEAD -> main` marks where
you are.

```bash
git reset --soft HEAD~1
git log --oneline
git status
```
```
5c3a1e2 (HEAD -> main) Fix typo
a1b2c3d Initial test suite
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   retry.py
```

- "Add retry logic" vanished from the log: the pointer moved back.
- `retry.py` is listed under "Changes to be committed": still staged. You
  can `git commit` again with a better message.

### Seeing it: `--mixed` (default)

```bash
git reset HEAD~1        # same as --mixed
git status
```
```
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   retry.py
```

Same pointer move, but now the change is "not staged". You would need
`git add` again before committing.

### Seeing it: `--hard`

```bash
git reset --hard HEAD~1
```
```
HEAD is now at 5c3a1e2 Fix typo
```

`HEAD is now at ...` tells you where the pointer landed. Your file
`retry.py` is now exactly as it was at "Fix typo". The edits are gone from
disk. (You *can* still get the commit back through reflog; see below.)

> **Common confusion: "If reset --hard removes a committed change, why
> did the Golden Rule say committed work is safe?"**
> The commit object still exists inside `.git`. Only the branch pointer
> moved away from it. The reflog remembers where it was. What `--hard`
> truly destroys is **uncommitted** edits in your working files, because
> those never had a commit at all.

`--hard` also works with no `~1`:

```bash
git reset --hard HEAD
```

`HEAD` means "the current commit", so no pointer movement happens. What it
does is force your files and waiting room to match the last commit. Result:
all uncommitted edits to tracked files vanish. (Untracked files, meaning
brand-new files Git never saw, are left alone.)

**`git reset --hard` is the most dangerous common Git command** —
it silently discards uncommitted work with no confirmation prompt. Always
run `git status` first, and consider `git stash` (Lesson 10) instead if
you're not 100% sure you want to lose the changes. (`git stash` = a
temporary shelf where Git puts your uncommitted changes aside.)

> **Doubt? "Can I reset to a hash instead of counting with ~?"**
> Yes. `git reset --hard 5c3a1e2` moves the pointer to exactly that
> commit. Use `~N` when the target is "N commits back"; use a hash when you
> copied it from `git log` or `git reflog`.

---

## Step 3: `git reset` vs `git revert` — The Question Every Beginner Gets Wrong

**Why do we need this?** Both "undo a commit", but they do it in opposite
ways, and picking the wrong one on a shared branch causes real trouble.

First, one new word. **Rewriting history** = changing commits that already
exist so the timeline looks different. Why is that risky? Your teammates
have copies of the old timeline. If yours differs, Git cannot line them up.

Analogy:
- `reset` = tearing a page out of a shared notebook. Anyone with a photocopy
  of that page now disagrees with you.
- `revert` = writing a new line below: "Ignore the entry above, it was
  wrong." The old page stays. Nobody is confused.

| | `git reset` | `git revert` |
|---|---|---|
| What it does | Moves the branch pointer **backward**, as if the commit never happened | Creates a **new commit** that undoes an old commit's changes, keeping full history |
| History | Rewrites it — old commits become unreachable | Never rewrites — adds to the timeline |
| Safe on shared/pushed branches? | **No** — anyone who already pulled the old commits now has a different history than you | **Yes** — always safe, this is the whole point |
| Use when | You made local mistakes nobody else has seen yet | The bad commit is already pushed/shared with others |

Pictures. Suppose the bad commit is C, and D came after it. Both start
from the same history.

How to read this: same history, two different fixes; compare the AFTERs.

```
BEFORE (same for both)

  A---B---C---D
              ^
            main
            HEAD -> main
```

```
git reset --hard B
AFTER reset

  A---B---(C)---(D)
      ^
    main
    HEAD -> main

  Staging + working folder: reset to match B
```
What changed: main moved D -> B. C and D are now unreachable (drawn in
parentheses); they are still in reflog. History was REWRITTEN.

```
git revert C
AFTER revert

  A---B---C---D---R*
                  ^
                main
                HEAD -> main

  R* = "Revert C": a NEW commit holding the opposite of C's changes
  Staging + working folder: clean, files now match R*
```
What changed: main moved D -> R* (a brand-new commit). A, B, C, D are all
unchanged and still reachable. Nothing was rewritten, only added.

Note: with revert, D stays. Only C's *effect* is cancelled.

```bash
git revert <commit-hash>          # opens editor for the revert commit message, then commits
git revert --no-edit <commit-hash>  # use Git's default "Revert '...'" message without prompting
git revert HEAD                    # revert the most recent commit
```

Real output of `git revert --no-edit 9fceb02`:

```
[main 3f9a1bc] Revert "Add retry logic"
 Date: Tue Oct 1 10:42:11 2026 +0530
 1 file changed, 4 deletions(-)
```

- `[main 3f9a1bc]` = a new commit with hash `3f9a1bc` was created on `main`.
- `Revert "Add retry logic"` = Git's auto message, naming what it undid.
- `4 deletions(-)` = the 4 lines that commit added were removed again.

```bash
git log --oneline
```
```
3f9a1bc (HEAD -> main) Revert "Add retry logic"
9fceb02 Add retry logic
5c3a1e2 Fix typo
```

The bad commit is still in history, and so is its cancellation. That is
honest, safe, and auditable.

**Rule of thumb:** if you've already run `git push` and a teammate might
have pulled that commit, use `revert`, never `reset` + force-push.
(**Force-push** = overwriting the shared copy with your version of history,
even when it disagrees. We cover its dangers in Lesson 07.)

> **Common confusion: "Does revert delete the old commit?"**
> No. It never deletes anything. It adds a new commit that does the
> opposite. Think of it as `git apply` of the reverse patch, committed.

> **Doubt? "Not pushed yet. Can I use revert anyway?"**
> Yes, it works anywhere. But for a purely local mistake, `reset` gives a
> cleaner history. Choose by audience: has anyone else seen this commit?
> Yes -> `revert`. No -> either, usually `reset`.

---

## Step 4: `git commit --amend` — Fixing the Last Commit

**Why do we need this?** Typos in commit messages and forgotten files are
very common. You want to fix the last commit without stacking a
"oops, fix" commit on top.

```bash
git commit --amend -m "New corrected message"     # just change the message
# — or, stage additional/fixed files first, THEN: —
git add forgotten_file.py
git commit --amend --no-edit    # keep the same message, just add the staged changes to the commit
```

Example output:

```bash
git commit --amend -m "Add retry logic to login test"
```
```
[main e7d41a9] Add retry logic to login test
 Date: Tue Oct 1 10:30:02 2026 +0530
 1 file changed, 4 insertions(+)
```

Notice the hash is `e7d41a9`, not the old `9fceb02`. That is the key fact.

Important: `--amend` **replaces** the last commit with a brand new one
(new hash, same parent). It's not editing in place. Commits are
immutable: a commit's hash is computed from its contents (Lesson 03), so
changing anything produces a different hash, which means a different
commit. Git builds a new one and moves the branch pointer to it; the old
one is left behind (and is still findable in the reflog).

How to read this: amend does not edit C; it builds a twin and moves main.

```
BEFORE

  A---B---C          C = 9fceb02
          ^
        main
        HEAD -> main
```

```
git commit --amend -m "Add retry logic to login test"
AFTER

        (C)          old 9fceb02, still in reflog
       /
  A---B---C*         main     C* = e7d41a9 (new hash, same parent B)
                     HEAD -> main

  Staging area  : emptied (its contents went into C*)
  Working folder: unchanged
```
What changed: main moved from C to C*. A and B are unchanged. (C) is no
longer on any branch but is still in reflog.

This is why amending a commit you've already pushed and shared is the same
category of danger as `reset` — it rewrites history. Don't amend shared
history.

> **Common confusion: `--amend` vs `--no-edit`**
> `--amend` = "replace the last commit". `--no-edit` = "and keep the old
> message, don't open the editor". Use `-m "..."` instead if you want a
> new message.

---

## Step 5: `git reflog` — Your Ultimate Safety Net

**Why do we need this?** After a reset, the commit no longer shows in
`git log`. You need a tool that remembers *where you have been*, not just
where the branch is now.

Analogy: `git log` is your browser's current tab history for this page.
`git reflog` is your full browser history: every page you visited, even if
you navigated away.

Every time `HEAD` moves — every commit, checkout, merge, reset, rebase —
Git logs it locally in the **reflog** (short for "reference log"). This is
*not* part of the shared commit history (it never gets pushed, it's purely
local to your machine), but it means you can almost always find your way back.

```bash
git reflog
```

```
5c3a1e2 (HEAD -> main) HEAD@{0}: reset: moving to HEAD~1
9fceb02 HEAD@{1}: commit: Add retry logic
5c3a1e2 HEAD@{2}: commit: Fix typo
a1b2c3d HEAD@{3}: commit (initial): Initial test suite
```

Reading it:
- Newest entry on top. `HEAD@{0}` = where HEAD is now, `HEAD@{1}` = where it
  was one move ago, and so on.
- First column = the commit hash HEAD pointed at after that move.
- Last part = what caused the move (`commit:`, `reset:`, `checkout:` ...).
- Line 1 says: "you did a reset back to HEAD~1". Line 2 is the commit that
  reset left behind. The hash you want is `9fceb02`.

Scenario: you ran `git reset --hard HEAD~1` and immediately realize the
commit you erased actually had important work.

```bash
git reflog
# find the line showing the commit BEFORE your mistake, e.g.:
# 9fceb02 HEAD@{1}: commit: Add retry logic

git reset --hard 9fceb02      # jump right back to it — fully recovered
# or, more surgically:
git checkout 9fceb02 -- important_file.py    # just recover one file from it
```

Output of the recovery:

```
HEAD is now at 9fceb02 Add retry logic
```

How to read this: the mistake first, then the reflog jump back. C = 9fceb02.

```
BEFORE (just after the mistaken  git reset --hard HEAD~1)

  A---B---(C)
      ^
    main
    HEAD -> main

  Staging + working folder: match B  (C's edits gone from your files)
  reflog:  HEAD@{1} = 9fceb02 (C)   <- the way back
```

```
git reset --hard 9fceb02
AFTER

  A---B---C
          ^
        main
        HEAD -> main

  Staging + working folder: match C again (files restored from C)
```
What changed: main moved B -> C, straight to the hash reflog showed. No
commits were created; C was never deleted, only unlabeled.

Your branch and your files are back. The second command
(`git checkout <hash> -- <file>`) means "copy that one file as it was in that
commit into my working directory", leaving everything else alone.

**How long does the reflog remember?** By default about 90 days for
entries whose commits are still reachable, and 30 days for unreachable
ones — plenty of time to notice a mistake. It only lives on
*your* machine though; it's not something you can recover from a
teammate's copy or from GitHub if you never pushed the commit at all.

> **Common confusion: `git log` vs `git reflog`**
> `git log` shows the history of the current branch (shared, part of the
> project). `git reflog` shows the journey of your HEAD (private, only on
> your machine). After a reset, `log` forgets the undone commits; `reflog`
> does not.

> **Doubt? "Reflog shows a commit, but I never committed it (just edited)."**
> Then reflog can't help. It only tracks things that had a commit (or a
> stash). This is the reason for the Golden Rule: commit early.

---

## Step 6: Recovering a Deleted Branch

**Why do we need this?** `git branch -D` deletes the pointer, not the
commits. So the commits are still in Git, just unlabeled.

(`-D` = force delete even if the branch is not merged. Lowercase `-d`
refuses to delete unmerged work, which is safer.)

```bash
git branch -D oops-deleted-branch     # deleted it, panic
git reflog                            # find a commit that was the tip of that branch
git branch recovered-branch <hash-from-reflog>   # recreate it pointing at that commit
```

Output:

```
Deleted branch oops-deleted-branch (was 7b2e5f1).
```

Git even tells you the hash in parentheses: `7b2e5f1`. You can use it
directly:

```bash
git branch recovered-branch 7b2e5f1
git log --oneline recovered-branch
```
```
7b2e5f1 (recovered-branch) Add flaky-test quarantine
...
```

Why does this work? A branch is just a label. Deleting the label leaves the
commit intact, and the hash tells Git where to stick a new label.

How to read this: a side branch is deleted, then relabeled. F = 7b2e5f1.

```
BEFORE (after  git branch -D oops-deleted-branch)

        E---(F)
       /
  A---B---C---D      main
                     HEAD -> main

  (E) and (F) have no label; still in reflog.
```

```
git branch recovered-branch 7b2e5f1
AFTER

        E---F        recovered-branch
       /
  A---B---C---D      main
                     HEAD -> main
```
What changed: one new label, recovered-branch, now points at F. No commits
were created or changed; main and HEAD did not move.

---

## Step 7: Recovering When a Teammate Force-Pushed Over Your Branch

This is a different problem from everything above: instead of *you*
making a local mistake, someone else rewrote a **shared** branch's
history and pushed over it. Your reflog still remembers the commits you
had before — but the remote no longer agrees with you.

(**Remote** = the shared copy of the repository on GitHub. `origin` is the
usual nickname for it. `origin/main` is your local record of what GitHub's
`main` looked like last time you checked. Lesson 07 covers this fully; here
just read it as "the GitHub version of main".)

How to read this: GitHub's history was replaced by X; yours still has C-D.

```
BEFORE (everyone agrees)

  A---B---C---D      main          (your local)
                     origin/main   (GitHub, as last fetched)
```

```
teammate runs:  git push --force   (their history: A---B---X)
AFTER  git fetch origin

        C---D        main          (yours; HEAD -> main)
       /
  A---B---X*         origin/main   (GitHub now)
```
What changed: origin/main moved D -> X*. Your main did NOT move, so C and D
sit on your side only (they are also in your reflog). "Diverged" = this fork.

```bash
git fetch origin
git status
# "Your branch and 'origin/main' have diverged, X and Y different commits each"

git log origin/main..main      # commits YOU have that origin no longer has
git log main..origin/main      # commits origin has that you don't (the new history)
```

- `git fetch origin` = download the latest from GitHub without changing your
  files or branch. Safe.
- "diverged" = your history and GitHub's history split at some point and each
  has commits the other lacks.
- `A..B` reads as "commits reachable from B that are not reachable from A".
  So `origin/main..main` = "what I have that GitHub doesn't".

If the first command shows commits you recognize as real, shared work —
not just your own unpushed drafts — that work got dropped by the
force-push.

**Recovering the dropped work:**

```bash
git reflog                     # your local history still has the OLD commits,
                                 # from before you fetched — reflog doesn't care
                                 # that the remote changed underneath you
```

Find the commit(s) that contained the missing work (they'll be sitting
right there, since your reflog only tracks *your* `HEAD` movements, not
the remote's). From there you have two honest paths:

1. **The force-push was a mistake** — cherry-pick the lost commit(s) onto
   the new `origin/main`, push them back, and tell your teammate what
   happened so they stop force-pushing shared branches.
   (**Cherry-pick** = copy one specific commit's changes onto your current
   branch as a new commit. Lesson 09.)
2. **The force-push was intentional** (a deliberate history cleanup,
   e.g. scrubbing a leaked secret per Lesson 13) — in that case, your job
   is to reset your local branch to match the new remote, *not* to
   resurrect the old commits:
   ```bash
   git reset --hard origin/main
   ```

How to read this: two recovery outcomes, starting from the fork above.

```
Path 1: mistake. git switch -c rescue origin/main ; git cherry-pick C D
AFTER

        (C)---(D)                  your old commits, in reflog
       /
  A---B---X---C'*---D'*                rescue  (HEAD -> rescue)
          ^
     origin/main
```
What changed: a new branch rescue was created at X and two NEW commits
(C', D') were added; push it back for review. Nothing was rewritten.

```
Path 2: intentional. git reset --hard origin/main
AFTER

  A---B---(C)---(D)
      \
       X             main, origin/main   (HEAD -> main)
```
What changed: your main moved D -> X and files match X. (C) and (D) are
unreachable from any branch but still in reflog.

Either way, don't guess — ask. This scenario is exactly why Lesson 07
recommends turning on branch protection's "do not allow force pushes" for
any shared branch: it turns this whole recovery process from "necessary"
into "impossible to trigger by accident" in the first place.
(**Branch protection** = a GitHub setting that blocks risky actions on
important branches.)

---

## Decision Flow: "I made a mistake. What do I run?"

```
Is the mistake committed?
 |
 +-- No (only edited files)
 |     +-- Already ran git add, want to unstage?  -> git restore --staged <file>
 |     +-- Want to throw the edits away?          -> git restore <file>
 |
 +-- Yes
       +-- Already pushed/shared?                 -> git revert <commit>
       +-- Only local:
             +-- Just the message/forgot a file?  -> git commit --amend
             +-- Want to redo it, keep edits?     -> git reset --soft/--mixed HEAD~1
             +-- Want it completely gone?         -> git reset --hard HEAD~1
 
Something vanished?                                -> git reflog
```

---

## Exercise

Expected results are given so you can check yourself.

```bash
mkdir undo-practice && cd undo-practice && git init
echo "v1" > f.txt && git add . && git commit -m "v1"
echo "v2" > f.txt && git add . && git commit -m "v2"
echo "v3" > f.txt && git add . && git commit -m "v3"
```

1. Run `git log --oneline`. Use `git reset --soft HEAD~1` and check
   `git status` — confirm v3's change is staged, ready to re-commit or
   modify.
   *Expected:* log shows three commits (v3, v2, v1). After the reset,
   `git status` shows `modified: f.txt` under "Changes to be committed",
   and `cat f.txt` prints `v3`.
2. Recommit it (`git commit -m "v3"`), then practice
   `git reset --mixed HEAD~1` — confirm the change is now unstaged but the
   file content is still `v3`.
   *Expected:* `git status` shows `modified: f.txt` under "Changes not
   staged for commit"; `cat f.txt` prints `v3`.
3. Recommit again (`git add . && git commit -m "v3"`). Now practice the
   dangerous one deliberately: `git reset --hard HEAD~1`. Confirm `f.txt`
   reverted to `v2` on disk.
   *Expected:* `HEAD is now at <hash> v2`; `cat f.txt` prints `v2`;
   `git log --oneline` shows only v2 and v1.
4. Run `git reflog`. Find the entry for the `v3` commit and use
   `git reset --hard <hash>` to bring it back. Confirm `f.txt` is `v3`
   again.
   *Expected:* `HEAD is now at <hash> v3`; `cat f.txt` prints `v3`;
   `git log --oneline` shows v3, v2, v1 again.
5. Now practice the *safe* alternative: with `f.txt` at `v3`, run
   `git revert HEAD~1` (reverting the "v2" commit specifically, not the
   most recent) and observe how Git creates a NEW commit and may report a
   conflict, since "v3" already overwrote "v2"'s change — resolve it and
   see the full history preserved with `git log --oneline`.
   *Expected:* Git prints `CONFLICT (content): Merge conflict in f.txt` and
   `error: could not revert ... v2`. Open `f.txt`: it contains conflict
   markers (`<<<<<<<`, `=======`, `>>>>>>>`) showing both versions. Edit the
   file to the content you want (for example `v3`), then run
   `git add f.txt` and `git revert --continue` (save the message). Final
   `git log --oneline` shows 4 commits: a `Revert "v2"` commit on top of
   v3, v2, v1. Nothing was deleted.

   Why a conflict? Reverting v2 means "change v2 back to v1". But the file
   is now v3, so Git cannot tell whether you want v1 or v3. It asks you.

Bonus: delete a branch with `git branch -D` and bring it back using the
hash Git printed.

---

## Recap in 5 Lines
1. Uncommitted edits are fragile: `git restore <file>` discards them for
   good; `git restore --staged <file>` only unstages.
2. `git reset` moves the branch pointer back; `--soft` keeps changes staged,
   `--mixed` keeps them as edits, `--hard` destroys them.
3. Local, unseen mistake -> `reset` or `--amend`. Already shared -> `revert`
   (adds an undoing commit, rewrites nothing).
4. Committed work is almost never lost: `git reflog` lists where HEAD has
   been, so `git reset --hard <hash>` or `git branch <name> <hash>` restores it.
5. Habit: run `git status` and `git diff` before any `--hard`, and commit
   often.

Continue to [Lesson 07 — Remotes & GitHub Basics](07-remotes-and-github-basics.md).
