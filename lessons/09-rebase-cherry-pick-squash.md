# Lesson 09 — Rebase, Cherry-Pick & Squashing History

## Goal
Master history rewriting: rebase vs merge (and when each is right),
interactive rebase for cleaning up commits, cherry-picking individual
commits, and squashing. This is what separates intermediate from advanced
Git users.

## Prerequisites
[Lesson 08 — Collaboration: Pull Requests](08-collaboration-pull-requests.md).
This lesson assumes total comfort with Lesson 06 (undoing things) — you'll
be rewriting history, so know your safety net.

## After This Lesson You Will Be Able To
- Explain rebase vs merge and choose correctly between them
- Clean up a messy commit history with interactive rebase
- Cherry-pick a single commit onto another branch
- Know exactly why you should never rebase shared/public history

---

## Rebase vs Merge — Same Goal, Different History Shape

Both solve the same problem: "I want my branch's changes combined with
`main`'s changes." They produce **different history shapes**.

**Merge** (Lesson 05) keeps both histories exactly as they happened and
adds a merge commit joining them:

```
Merge:
        E ← F                  ← main
       /     \
A ← B ┤       M                ← main, HEAD (merge commit, 2 parents)
       \     /
        C ← D                  ← feature (unchanged, still exists)
```

**Rebase** rewrites your branch's commits so they look like they were
made *starting from* the current tip of `main` — as if you'd started your
feature branch just now, rather than back when you actually branched.

```
Before rebase:
A ← B ← E ← F                  ← main
     \
      C ← D                    ← feature

git switch feature
git rebase main

After rebase:
A ← B ← E ← F                  ← main
             \
              C' ← D'          ← feature, HEAD
                    (C' and D' are NEW commits — same changes, different
                     parent, therefore DIFFERENT HASHES than C and D)
```

**Critical insight:** rebase doesn't move the *original* commits — it
creates **brand new commits** with the same content/message but a
different parent (and therefore a different hash, per Lesson 03). The
old `C` and `D` become unreachable (but recoverable via reflog if needed,
Lesson 06).

| | Merge | Rebase |
|---|---|---|
| History | Preserves exactly what happened, including the merge point | Rewrites to look linear, as if developed straight off latest `main` |
| Commit hashes | Original commits untouched | New hashes for every rebased commit |
| Result graph | Branching, with merge commits | One straight line |
| Safe on shared branches? | Always | **Only if nobody else has the old commits** |

---

## The Golden Rule of Rebase

> **Never rebase commits that have been pushed and that someone else
> might have already pulled.** Since rebase creates new commit hashes,
> anyone with the old ones now has a history that's diverged from yours —
> forcing confusing conflicts or duplicate commits when they eventually
> sync up.
>
> Rebasing your **own local, not-yet-pushed** commits, or a **personal
> feature branch nobody else is using**, is not just safe — it's a
> genuinely great habit for keeping history clean before opening a PR.

---

## Using Rebase to Update Your Branch (The Common Case)

This is the #1 real-world use of rebase: keeping your feature branch
current with `main` while you work, without cluttering history with merge
commits.

```bash
git switch feature/my-branch
git fetch origin
git rebase origin/main
```

If conflicts occur, Git pauses on the specific commit causing trouble:

```bash
# fix the conflicted file(s), same markers as Lesson 05, then:
git add <resolved-file>
git rebase --continue

# if it's going badly and you want to bail out entirely:
git rebase --abort     # returns you to exactly where you were before rebasing
```

Since this rewrites your branch's commits, if you'd already pushed this
branch before rebasing, you'll need to force-push afterward — always
with `--force-with-lease` (Lesson 07):

```bash
git push --force-with-lease
```

---

## Interactive Rebase — Cleaning Up Your Own History

`git rebase -i` lets you edit, reorder, combine, or delete commits before
sharing them. This is how professionals turn a messy series of "wip",
"fix typo", "actually fix it" commits into one or two clean, reviewable
commits before opening a PR.

```bash
git rebase -i HEAD~4     # interactively rewrite the last 4 commits
```

This opens your editor with something like:

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

To squash the messy typo-fixing commits into the first clean one:

```
pick   a1b2c3d Add search endpoint
fixup  e4f5g6h wip
fixup  h7i8j9k fix typo
fixup  k1l2m3n actually fix the typo for real
```

Save and close the editor. Git replays them in order, folding the
`fixup` commits silently into the one above them. Result: a single clean
commit, `Add search endpoint`, containing all four commits' combined
changes. `git log --oneline` now shows one entry instead of four.

**Rule:** only do this to commits **you haven't pushed yet**, or on a
branch you're certain nobody else has pulled.

---

## `--fixup` and `--autosquash` — The Pro Habit for Cleaning Up Commits

Manually re-ordering `fixup` lines in the editor above works, but it
doesn't scale once you're juggling more than a couple of stray commits.
The real-world habit is to tell Git *at commit time* which earlier
commit a fix belongs to, and let it sort the editor out for you.

```bash
git log --oneline
# k1l2m3n Fix typo in retry logic
# h7i8j9k Add search endpoint        ← this is the one we actually meant to fix

git add fixed_file.py
git commit --fixup h7i8j9k
# creates a new commit titled: "fixup! Add search endpoint"
```

Now run interactive rebase with `--autosquash` (or enable it permanently
with `git config --global rebase.autosquash true` so every future
`rebase -i` does this automatically):

```bash
git rebase -i --autosquash HEAD~4
```

The editor opens with the `fixup!` commit **already moved directly below
its target and already marked `fixup`** — no manual cut-and-paste
required:

```
pick   h7i8j9k Add search endpoint
fixup  k1l2m3n fixup! Add search endpoint
pick   e4f5g6h Update docs
```

`git commit --squash <hash>` works identically, except it keeps the new
commit's own message too (combined with the original, for you to edit) —
use it when the fix is worth a note in the final commit message, not just
silently folded in.

---

## Splitting a Commit With `rebase -i edit`

The interactive rebase table earlier in this lesson lists `edit` as an
option — here's what it's actually for: breaking one commit that did
**too much** into two or more smaller, focused commits.

```bash
git rebase -i HEAD~3
```

Mark the offending commit `edit` instead of `pick`, save and close. Git
stops with that commit checked out — its changes are already applied and
committed, exactly as they were:

```
edit   h7i8j9k Add search endpoint (and accidentally fix an unrelated typo)
pick   k1l2m3n Update docs
```

Now un-commit just that one commit, keeping its changes in your working
directory/staging area (the same `reset` mechanics from Lesson 06):

```bash
git reset HEAD~1          # soft-reset: commit undone, changes still staged/present on disk
```

Split it into two clean commits by staging selectively:

```bash
git add search_endpoint.py
git commit -m "Add search endpoint"

git add unrelated_file.py
git commit -m "Fix unrelated typo in unrelated_file.py"

git rebase --continue      # replays the rest of the original commits (k1l2m3n, ...) on top
```

The final history now has two focused commits where there used to be one
tangled one — with every commit *after* it automatically replayed on top,
unchanged in content but with new hashes (Lesson 03).

---

## Cherry-Pick — Taking One Specific Commit

Sometimes you want just *one* commit from another branch, not the whole
branch merged/rebased in. Classic case: a hotfix made on a feature branch
that also needs to go straight to `main` immediately, without waiting for
the whole feature to be ready.

```bash
git switch main
git cherry-pick a1b2c3d       # applies JUST that commit's changes onto main, as a new commit
```

```
Before:
A ← B ← C           ← main
     \
      D ← E(a1b2c3d) ← F      ← feature (E is the hotfix commit we want)

After: git cherry-pick a1b2c3d (while on main)
A ← B ← C ← E'       ← main, HEAD    (E' = new commit, same changes as E, new hash)
     \
      D ← E ← F                    ← feature (unchanged)
```

If it conflicts:

```bash
# resolve conflicts, then:
git add <file>
git cherry-pick --continue
# or bail out:
git cherry-pick --abort
```

Cherry-pick multiple commits: `git cherry-pick a1b2c3d e4f5g6h` (applies
in the order listed).

---

## Squash Merging (Recap From Lesson 08, Command-Line Version)

If your team doesn't use GitHub's "Squash and merge" button, you can do
it manually:

```bash
git switch main
git merge --squash feature/my-branch    # applies all changes, but does NOT commit yet
git commit -m "Add search endpoint (squashed from feature branch)"
```

Note: unlike a real merge, this does **not** record that `feature/my-branch`
was ever merged — `main` and the feature branch remain unrelated as far
as Git's ancestry tracking is concerned. This is fine for a one-off PR,
but means you shouldn't try to merge that same feature branch into `main`
again later expecting Git to recognize the overlap cleanly.

---

## `git rerere` — Never Resolve the Same Conflict Twice

If you keep rebasing a long-lived feature branch onto `main` (the common
case from earlier in this lesson), you can end up resolving the **exact
same conflict** over and over — the same lines keep colliding every time
you rebase again. `rerere` ("reuse recorded resolution") fixes this.

```bash
git config --global rerere.enabled true
```

Once enabled, the first time you resolve a conflict, Git silently records
*how* you resolved it. The next time the identical conflict shows up
(same conflicting text) — during a later rebase, merge, or cherry-pick —
Git automatically reapplies your previous resolution instead of stopping
to ask you again.

```bash
git rerere status     # see which currently-conflicted files have a recorded resolution available
git rerere diff        # preview the resolution rerere is about to (or did) apply
```

**Where this actually pays off:** a feature branch you rebase onto `main`
daily for weeks, hitting the same three stubborn conflicts in a
config file every single time. It's far less useful for a one-off
conflict you'll only ever resolve once.

---

## Choosing the Right Tool — Quick Decision Guide

```
Want to combine two branches, keeping full history + a merge point?
  → git merge

Want your branch's commits to look like they were just written on top
of the latest main, with NO merge commit, before opening/updating a PR?
  → git rebase (only if not yet pushed/shared)

Want to clean up your OWN messy commits before anyone sees them?
  → git rebase -i

Want just ONE commit from another branch, not the whole thing?
  → git cherry-pick

Want a whole PR's worth of commits squashed into one commit on merge?
  → GitHub's "Squash and merge" button, or git merge --squash
```

---

## Exercise

```bash
mkdir rebase-practice && cd rebase-practice && git init
echo "v1" > f.txt && git add . && git commit -m "Initial"
git switch -c feature
```

1. On `feature`, make 4 small, deliberately messy commits (e.g. "wip",
   "fix", "actually fix", "final fix") each editing `f.txt` slightly.
2. Run `git rebase -i HEAD~4` and squash all 4 into one clean commit with
   a proper message.
3. Switch to `main`, make one new commit there (simulating a teammate's
   change) so histories diverge.
4. Switch back to `feature` and run `git rebase main`. Observe the new
   commit hash for your squashed commit (`git log --oneline`) — proving
   rebase created a new commit object.
5. Create a second branch `hotfix` off `main`, make one commit there, then
   `git cherry-pick` that single commit onto `feature` without merging
   the whole `hotfix` branch.

Continue to [Lesson 10 — Stash, Tags & Bisect](10-stash-tags-bisect.md).
