# Lesson 10 — Stash, Tags & Bisect

## Goal
Three underused tools that solve very specific, very common problems:
context-switching mid-task (`stash`), marking releases (`tags`), and
finding exactly which commit introduced a bug (`bisect`).

## Prerequisites
[Lesson 09 — Rebase, Cherry-Pick & Squashing History](09-rebase-cherry-pick-squash.md)

## After This Lesson You Will Be Able To
- Switch branches instantly without committing half-finished work
- Tag releases properly (and push tags to GitHub)
- Binary-search your entire commit history to find the exact commit that
  broke something

---

## `git stash` — Shelving Work in Progress

Scenario: you're mid-edit on `feature-a`, nothing is commit-worthy yet,
and suddenly you need to switch to `main` to fix an urgent bug. Git
normally won't let you switch branches if it would overwrite
uncommitted changes that conflict with the target branch.

```bash
git stash                       # shelves ALL uncommitted changes (staged + unstaged), cleans working dir
git switch main
# ... fix the urgent thing, commit, push ...
git switch feature-a
git stash pop                   # reapplies the shelved changes AND removes them from the stash list
```

Think of the stash as a **clipboard for uncommitted changes** — a stack
you can push onto and pop from.

```bash
git stash                       # equivalent to: git stash push
git stash push -m "wip: half-done retry logic"    # name it, for clarity
git stash list                  # see everything currently stashed
# stash@{0}: On feature-a: wip: half-done retry logic
# stash@{1}: On main: quick experiment

git stash show stash@{0}        # preview what's in a specific stash entry (like a diff)
git stash apply stash@{0}       # reapply WITHOUT removing it from the list
git stash pop stash@{0}         # reapply AND remove it from the list
git stash drop stash@{0}        # delete a stash entry without applying it
git stash clear                 # delete ALL stash entries — careful, no undo (well — reflog might help, but don't rely on it)
```

Include untracked (new, never-`add`ed) files too:

```bash
git stash -u        # also stash new untracked files (normally stash skips these)
git stash -a        # stash EVERYTHING, including files git is told to ignore too
```

**Stash conflicts:** if `git stash pop` conflicts with your current
state, Git leaves the changes as a merge conflict (Lesson 05's markers)
AND keeps the stash entry (doesn't drop it) until you resolve and
manually `git stash drop` — so nothing is lost if it goes sideways.

---

## `git tag` — Marking Specific Points (Releases)

A tag is a **permanent, named pointer to a specific commit** — unlike a
branch, it doesn't move as you make new commits. Perfect for marking
releases: `v1.0.0`, `v2.3.1`, etc.

```bash
git tag v1.0.0                        # lightweight tag on current commit
git tag v1.0.0 a1b2c3d                # lightweight tag on a SPECIFIC commit
git tag -a v1.0.0 -m "First stable release"   # ANNOTATED tag — has its own message, author, date
```

Two kinds of tags:
- **Lightweight** — just a name pointing at a commit. Quick, no metadata.
- **Annotated** — a full object in the database (like a mini-commit) with
  its own message, tagger name, and date. **Use annotated tags for actual
  releases** — they show up properly in `git describe` and carry
  metadata worth having.

```bash
git tag                        # list all tags
git tag -l "v1.*"              # filter by pattern
git show v1.0.0                # see what commit + message a tag points to
git checkout v1.0.0            # inspect the code AS OF that release (detached HEAD — see below)
```

**Tags don't push automatically** — you must push them explicitly:

```bash
git push origin v1.0.0          # push a single tag
git push origin --tags          # push ALL tags at once
```

Deleting a tag:

```bash
git tag -d v1.0.0                    # delete locally
git push origin --delete v1.0.0      # delete on the remote too
```

### GitHub Releases

On GitHub, "Releases" are built on top of tags — go to your repo →
Releases → "Create a new release," pick (or create) a tag, add release
notes, and optionally attach built artifacts (binaries, zip files). This
is the standard way software projects publish versioned downloads.

---

## Detached HEAD — What It Means and Why It's Not Scary

When you `git checkout v1.0.0` (or any specific commit hash instead of a
branch name), Git puts you in **detached HEAD** state:

```
A ← B ← C ← D          ← main
        ↑
      HEAD              (HEAD points directly at commit C, not at a branch)
```

You can look around, run code, even make commits — but those new commits
won't belong to any branch. If you switch away without creating a branch
first, they become effectively orphaned (recoverable via reflog for a
while, per Lesson 06, but easy to lose track of).

**If you want to make real changes from a detached HEAD state, create a
branch immediately:**

```bash
git checkout v1.0.0
git switch -c hotfix-from-v1     # NOW your commits have a permanent home
```

---

## `git bisect` — Binary Search Through History to Find a Bug

Scenario: something broke. You know it worked in commit from 3 weeks ago
(a "good" commit) and it's broken now (a "bad" commit) — but there are
200 commits in between and you have no idea which one introduced the bug.
Checking all 200 by hand would take forever. `bisect` does a **binary
search**, so 200 commits takes about 8 checks (log₂ 200 ≈ 7.6).

```bash
git bisect start
git bisect bad                       # current commit is broken
git bisect good v1.0.0               # this known-good commit/tag worked fine

# Git checks out a commit exactly halfway between good and bad, and says:
# "Bisecting: 100 revisions left to test after this (roughly 7 steps)"

# YOU test the code right now (run the app, run a specific test, whatever proves it):
git bisect good      # if THIS commit works fine
# — or —
git bisect bad       # if THIS commit is also broken

# Git narrows the range and checks out the next candidate automatically.
# Repeat "good" or "bad" until Git announces:
# "a1b2c3d is the first bad commit"

git bisect reset      # IMPORTANT — returns you to your original branch/HEAD when done
```

### Automating It

If you have a script or test command that exits `0` for good and
non-zero for bad, skip the manual back-and-forth entirely:

```bash
git bisect start HEAD v1.0.0
git bisect run pytest tests/test_login.py
# Git runs the test at each step automatically and reports the exact
# culprit commit with zero manual intervention.
```

This is one of the highest-leverage tools in advanced Git for exactly one
situation: "this used to work, now it doesn't, and I don't know why" —
which is one of the most common real debugging scenarios there is.

---

## `git blame` — Who Wrote This Line, and Why

Scenario: you're staring at one confusing line of code and need to know
who wrote it, when, and — ideally — *why*. `git blame` annotates every
line of a file with the commit that last touched it.

```bash
git blame app.py
# a1b2c3d4 (Jane Doe  2025-11-02 14:20:00 -0500  42) if retries > MAX_RETRIES:
```

Useful flags:

```bash
git blame -L 10,20 app.py     # only annotate lines 10–20, not the whole file
git blame -w app.py             # ignore whitespace-only changes (so a reformat/reindent
                                 #   doesn't hide the last commit that actually changed the LOGIC)
git blame -C app.py             # detect lines that were moved or copied from elsewhere,
                                 #   so blame follows the code's real origin instead of just
                                 #   showing "this line arrived here" on the move/copy commit
```

**Blame only gives you "who and when," not "why."** The essential
follow-up habit is to take the commit hash blame gives you and actually
read that commit's message:

```bash
git show a1b2c3d4             # full diff + commit message for that commit
git log -1 a1b2c3d4            # just the message/metadata, no diff
```

This is exactly why "write good commit messages" (Lesson 04) isn't just
etiquette — a future teammate (often you, in six months) will find this
exact commit via blame and need the message to explain a decision the
code itself doesn't. In daily use, most people never type `git blame` by
hand — VS Code's GitLens extension and PyCharm's built-in "Annotate"
gutter show this inline as you read code. Know the raw command anyway;
it works everywhere GitLens doesn't (a plain SSH session on a server, a
teammate's unfamiliar editor, a CI log).

---

## Pickaxe Search — Finding *When* a String Was Added or Removed

Scenario: you need to know when a specific constant, config key, or
function call was introduced (or removed) — but it could be in any of
hundreds of commits, and the file it lives in may have been renamed or
refactored since. `git log <file>` alone won't survive a rename;
**pickaxe search** scans every commit's actual diff content, not just
file paths.

```bash
git log -S"MAX_RETRIES"          # commits where the COUNT of "MAX_RETRIES" occurrences changed
                                  #   (i.e., it was added, removed, or duplicated — not just
                                  #   a line near it being touched)
git log -G"MAX_RETRIES\s*="       # same idea, but a regex — matches any commit whose diff
                                  #   added/removed a line matching the PATTERN, which is more
                                  #   forgiving when the surrounding code's shape has changed
```

Add `-p` to see the actual diff for each matching commit (rather than
just the commit list), and `--all` to search every branch, not just the
one you're currently on:

```bash
git log -S"MAX_RETRIES" -p --all
```

This is the tool for "when did we start doing X" questions that span
refactors and renames — `-S` answers "did the number of times this exact
string appears change here," which is a surprisingly precise way to find
the one commit that actually introduced or deleted something, out of a
history that might otherwise be hundreds of commits deep.

---

## `git worktree` — Multiple Branches Checked Out at Once

Normally, one repo folder = one checked-out branch at a time; switching
branches means stashing or committing whatever's in progress first
(Lesson 10, above). `git worktree` removes that constraint: it lets you
check out a *second* (or third) branch into its own separate folder,
while sharing the exact same `.git` object database underneath — no
duplicate clone, no duplicate history storage (recall Lesson 03: objects
are content-addressed, so there's nothing to duplicate).

```bash
git worktree add ../repo-hotfix hotfix-branch
# creates ../repo-hotfix as a fully independent working directory,
# with hotfix-branch checked out there — your original folder is untouched

git worktree list
# /path/to/repo          abcd123 [main]
# /path/to/repo-hotfix   ef56789 [hotfix-branch]

git worktree remove ../repo-hotfix     # done with it — deletes the folder, keeps the branch/history
```

For an SDET/automation-engineer workflow specifically, this is genuinely
useful, not just a novelty: run the full test suite against `main` in one
worktree while you keep editing a feature branch, mid-change, in another
— no stash/pop dance, no risk of a half-finished edit leaking into the
suite you're validating. It's equally handy for keeping a long-running
manual QA checkout of a release branch open in its own folder while your
primary worktree keeps moving on normal day-to-day development.

---

## Exercise

```bash
mkdir stash-tag-bisect-practice && cd stash-tag-bisect-practice && git init
```

**Stash:**
1. Make a commit, then start editing a file without committing. Stash it
   with a message, switch to a new branch, make an unrelated commit,
   switch back, and `git stash pop` to recover your original edit.

**Tags:**
2. Make 3 commits. Tag the second one as `v1.0.0` (annotated, with a
   message). Use `git show v1.0.0` to confirm what it points to.

**Bisect (the important one):**
3. Write a tiny script `check.py` that returns exit code `0` (success).
   Commit it as your "good" baseline, tag it `good-start`.
4. Make 6 more commits. In one of them **in the middle** (pick any), edit
   `check.py` so it returns a non-zero exit code (simulating a bug being
   introduced) — but keep committing normally after that, as if you
   didn't notice.
5. From the latest commit, run:
   `git bisect start HEAD good-start`
   `git bisect run python3 check.py`
6. Confirm Git correctly identifies the exact commit where you broke it.
   Run `git bisect reset` when done.

Continue to [Lesson 11 — GitHub Pro Features](11-github-pro-features.md).
