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

## The Golden Rule First

> **If a change was ever committed, it is almost never truly gone** — even
> if you deleted the branch, even if you force-reset. Git keeps every
> commit object around (Lesson 03) until garbage collection eventually
> cleans up genuinely unreachable ones, which by default doesn't happen
> for weeks. The `reflog` (bottom of this lesson) is your recovery map.
>
> **Uncommitted changes are the only things that can vanish easily.** This
> is the actual argument for committing often, even with messy
> "WIP" commits you'll clean up later (Lesson 09) — a bad commit is
> recoverable; an uncommitted deleted file often isn't.

---

## Undo Map — Which Tool for Which Situation

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

---

## `git restore` — Discarding Working Directory Changes

```bash
git restore file.py           # throws away UNSTAGED edits to file.py, reverting to last commit's version
git restore .                 # do that for every file in current dir and below
git restore --staged file.py  # unstage it — moves it from "staged" back to "modified," edits are kept
```

**This is destructive for the file's uncommitted edits** — there's no
undo for `git restore file.py` itself (the old content is just gone,
unless it happened to be staged or committed at some point, in which case
reflog-adjacent tricks *might* help, but don't count on it). Always
`git status` and `git diff` first to be sure what you're discarding.

---

## `git reset` — Moving Branch Pointers Backward

Recall: a branch is just a pointer to a commit (Lesson 05). `git reset`
**moves that pointer**, and optionally also changes your staging area
and/or working directory to match. This is the single most
misunderstood command in Git because it has three modes:

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

Visual: all three modes move the branch pointer identically (back one
commit) — they only differ in what happens to your staging area and
working files.

```
Before:  A ← B ← C          ← main, HEAD
                              (C has some file changes)

After any reset --X HEAD~1:
         A ← B               ← main, HEAD
                              (C's changes now live in: staging+workdir [soft],
                               workdir only [mixed/default], or nowhere [hard])
```

`HEAD~1` means "one commit before HEAD." `HEAD~3` means three before. You
can also reset to any specific commit hash: `git reset --hard a1b2c3d`.

**`git reset --hard` is the most dangerous common Git command** —
it silently discards uncommitted work with no confirmation prompt. Always
run `git status` first, and consider `git stash` (Lesson 10) instead if
you're not 100% sure you want to lose the changes.

---

## `git reset` vs `git revert` — The Question Every Beginner Gets Wrong

This is a critical distinction:

| | `git reset` | `git revert` |
|---|---|---|
| What it does | Moves the branch pointer **backward**, as if the commit never happened | Creates a **new commit** that undoes an old commit's changes, keeping full history |
| History | Rewrites it — old commits become unreachable | Never rewrites — adds to the timeline |
| Safe on shared/pushed branches? | **No** — anyone who already pulled the old commits now has a different history than you | **Yes** — always safe, this is the whole point |
| Use when | You made local mistakes nobody else has seen yet | The bad commit is already pushed/shared with others |

```
reset removes commit C from history:
A ← B ← C ← D    becomes    A ← B ← D-as-new-tip
                (main pointer just moved back to B, then D would need to be re-applied or is lost)

revert ADDS a new commit that undoes C, keeping the record:
A ← B ← C ← D  →  A ← B ← C ← D ← D'    (D' = "Revert: C's changes")
```

```bash
git revert <commit-hash>          # opens editor for the revert commit message, then commits
git revert --no-edit <commit-hash>  # use Git's default "Revert '...'" message without prompting
git revert HEAD                    # revert the most recent commit
```

**Rule of thumb:** if you've already run `git push` and a teammate might
have pulled that commit, use `revert`, never `reset` + force-push (we
cover force-push dangers in Lesson 07).

---

## `git commit --amend` — Fixing the Last Commit

```bash
git commit --amend -m "New corrected message"     # just change the message
# — or, stage additional/fixed files first, THEN: —
git add forgotten_file.py
git commit --amend --no-edit    # keep the same message, just add the staged changes to the commit
```

Important: `--amend` **replaces** the last commit with a brand new one
(new hash, same parent). It's not editing in place — under the hood
(Lesson 03), it's creating a new commit object. This is why amending a
commit you've already pushed and shared is the same category of danger
as `reset` — don't amend shared history.

---

## `git reflog` — Your Ultimate Safety Net

Every time `HEAD` moves — every commit, checkout, merge, reset, rebase —
Git logs it locally in the **reflog**. This is *not* part of the shared
commit history (it never gets pushed, it's purely local to your machine),
but it means you can almost always find your way back.

```bash
git reflog
```

```
a1b2c3d (HEAD -> main) HEAD@{0}: reset: moving to HEAD~1
9fceb02 HEAD@{1}: commit: Add retry logic
5c3a1e2 HEAD@{2}: commit: Fix typo
...
```

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

**The reflog typically retains entries for 90 days by default** (30 for
unreachable ones) — plenty of time to notice a mistake. It only lives on
*your* machine though; it's not something you can recover from a
teammate's copy or from GitHub if you never pushed the commit at all.

---

## Recovering a Deleted Branch

```bash
git branch -D oops-deleted-branch     # deleted it, panic
git reflog                            # find a commit that was the tip of that branch
git branch recovered-branch <hash-from-reflog>   # recreate it pointing at that commit
```

---

## Recovering When a Teammate Force-Pushed Over Your Branch

This is a different problem from everything above: instead of *you*
making a local mistake, someone else rewrote a **shared** branch's
history and pushed over it. Your reflog still remembers the commits you
had before — but the remote no longer agrees with you.

**Spotting it:**

```bash
git fetch origin
git status
# "Your branch and 'origin/main' have diverged, X and Y different commits each"

git log origin/main..main      # commits YOU have that origin no longer has
git log main..origin/main      # commits origin has that you don't (the new history)
```

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
2. **The force-push was intentional** (a deliberate history cleanup,
   e.g. scrubbing a leaked secret per Lesson 12) — in that case, your job
   is to reset your local branch to match the new remote, *not* to
   resurrect the old commits:
   ```bash
   git reset --hard origin/main
   ```

Either way, don't guess — ask. This scenario is exactly why Lesson 07
recommends turning on branch protection's "do not allow force pushes" for
any shared branch: it turns this whole recovery process from "necessary"
into "impossible to trigger by accident" in the first place.

---

## Exercise

```bash
mkdir undo-practice && cd undo-practice && git init
echo "v1" > f.txt && git add . && git commit -m "v1"
echo "v2" > f.txt && git add . && git commit -m "v2"
echo "v3" > f.txt && git add . && git commit -m "v3"
```

1. Run `git log --oneline`. Use `git reset --soft HEAD~1` and check
   `git status` — confirm v3's change is staged, ready to re-commit or
   modify.
2. Recommit it, then practice `git reset --mixed HEAD~1` — confirm the
   change is now unstaged but the file content is still `v3`.
3. Recommit again. Now practice the dangerous one deliberately:
   `git reset --hard HEAD~1`. Confirm `f.txt` reverted to `v2` on disk.
4. Run `git reflog`. Find the entry for the `v3` commit and use
   `git reset --hard <hash>` to bring it back. Confirm `f.txt` is `v3`
   again.
5. Now practice the *safe* alternative: with `f.txt` at `v3`, run
   `git revert HEAD~1` (reverting the "v2" commit specifically, not the
   most recent) and observe how Git creates a NEW commit and may report a
   conflict, since "v3" already overwrote "v2"'s change — resolve it and
   see the full history preserved with `git log --oneline`.

Continue to [Lesson 07 — Remotes & GitHub Basics](07-remotes-and-github-basics.md).
