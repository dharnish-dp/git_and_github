# Lesson 10 — Stash, Tags & Bisect

## Goal
Learn five small tools that each solve one very specific, very common problem:

| Tool | The problem it solves |
|---|---|
| `git stash` | "I am mid-edit and must switch tasks right now." |
| `git tag` | "I want a permanent name for this exact version (a release)." |
| `git bisect` | "It worked 3 weeks ago, it is broken now. Which commit broke it?" |
| `git blame` / pickaxe (`git log -S`) | "Who wrote this line, and when did this string appear?" |
| `git worktree` | "I want two branches open at the same time, in two folders." |

## Prerequisites
[Lesson 09 — Rebase, Cherry-Pick & Squashing History](09-rebase-cherry-pick-squash.md)

## After This Lesson You Will Be Able To
- Switch branches instantly without committing half-finished work
- Tag releases properly (and push tags to GitHub)
- Explain what "detached HEAD" is and get out of it safely
- Binary-search your entire commit history to find the exact commit that
  broke something
- Find who last changed a line, and when a string first appeared

## Words you must know first
You have met these before. Here they are again in one place.

- **Commit** = a saved snapshot of your whole project, with a message.
- **Working directory** = the actual files in your folder, as you see them now.
- **Staging area** = the "next commit" waiting room. `git add` puts changes there.
- **Uncommitted changes** = edits you made that are not yet saved in any commit.
- **Branch** = a movable name that points at the newest commit of a line of work.
- **HEAD** = a pointer meaning "the commit (or branch) I am standing on right now".
- **Remote** = a copy of the repo on another machine (usually GitHub). `origin` is the usual nickname for it.

### Diagram legend
Every diagram in this lesson uses the same few symbols. Read this once.

```
  A---B---C       commits. Time flows LEFT to RIGHT: oldest on the left,
                  newest on the right.
  D*              an asterisk marks a NEW commit made by the command shown
  ^               a label sits under the commit it points at
  main            a branch name (a movable label)
  HEAD -> main    HEAD is a separate label: "I am standing on main"
  HEAD -> abc123  HEAD pointing straight at a commit = "detached"
  /  \            a fork: each branch gets its own row, name at the END
  --->            something moves or flows in that direction
```

Note: internally Git stores, for each commit, a link back to its parent
(the commit before it). We do NOT draw those backward arrows. We always
draw time flowing left to right, because that is easier to read.

Every big diagram has "How to read this" before it and "What changed"
after it. When a command changes things you get a BEFORE and an AFTER.

---

## `git stash` — Shelving Work in Progress

### Why do we need this?
Picture your desk. You are halfway through a jigsaw puzzle. Your boss
walks in: "Drop everything, fix this urgent bug."

You do not want to throw the puzzle away. You do not want to glue it
half-finished into a "finished" box either. You slide the whole puzzle
onto a shelf, clear the desk, do the urgent job, and later take the
puzzle back off the shelf.

That is `git stash`. **Stash = a shelf where Git keeps your uncommitted
changes safely, while giving you a clean working directory.**

Python analogy: it is like pickling your unsaved work-in-progress object
to disk so you can run something else, then unpickling it later.

### Why can't I just switch branches?
Git refuses to switch branches when your uncommitted edits would be
overwritten by the other branch's version of the same file. Example of the
error:

```
error: Your local changes to the following files would be overwritten by checkout:
        app.py
Please commit your changes or stash them before you switch branches.
Aborting
```

Line by line:
- `Your local changes ... would be overwritten` = Git is protecting your unsaved edits.
- `commit your changes or stash them` = Git itself tells you the two ways out.

You could commit, but your work is not commit-worthy yet. So: stash.

### Step 1: shelve the work

```bash
git stash
```

Expected output:

```
Saved working directory and index state WIP on feature-a: 3f2a1bc Add retry helper
```

Line by line:
- `Saved working directory and index state` = Git saved both your edited files and what you had staged.
- `WIP on feature-a` = "work in progress" on the branch `feature-a`.
- `3f2a1bc Add retry helper` = the latest commit on that branch (your edits were based on it).

Check that your desk is clean:

```bash
git status
```

```
On branch feature-a
nothing to commit, working tree clean
```

"working tree clean" = no uncommitted changes. Your edits are now on the shelf.

**How to read this:** the left column is your folder, the right column is
the stash shelf. We watch where the edits to `app.py` live.

BEFORE (you are mid-edit):

```
  WORKING FOLDER (feature-a)      STASH STACK
  app.py   = EDITED               (empty)
```

Command: `git stash`

AFTER:

```
  WORKING FOLDER (feature-a)      STASH STACK
  app.py   = clean      --->      stash@{0}: app.py edits   <- top
```

**What changed:** your edits moved from the folder onto the shelf. The
folder is back to the last commit (`3f2a1bc`). No branch and no commit
changed.

### Step 2: do the urgent work elsewhere

```bash
git switch main
# ... fix the urgent thing, commit, push ...
git switch feature-a
```

### Step 3: take the work back off the shelf

```bash
git stash pop
```

Expected output:

```
On branch feature-a
Changes not staged for commit:
        modified:   app.py

Dropped refs/stash@{0} (9d8c7b6a5e4f3d2c1b0a9f8e7d6c5b4a3f2e1d0c)
```

Line by line:
- `Changes not staged for commit: modified: app.py` = your edits are back in the working directory.
- `Dropped refs/stash@{0} (...)` = the shelf entry was removed, because `pop` means "apply and remove".

**How to read this:** same two columns as before, now going the other way.

BEFORE (you are back on `feature-a`, folder is clean):

```
  WORKING FOLDER (feature-a)      STASH STACK
  app.py   = clean                stash@{0}: app.py edits   <- top
```

Command: `git stash pop`

AFTER:

```
  WORKING FOLDER (feature-a)      STASH STACK
  app.py   = EDITED               (empty)
```

**What changed:** the edits were copied back into the folder and the
entry was removed from the shelf. Commits and branches are untouched.

### Common confusion: stash is a stack
A **stack** is a pile where the last item you put on top is the first one
you take off (like a stack of plates). In Python terms: `list.append()`
and `list.pop()`.

So `git stash` = push onto the pile, `git stash pop` = take the top item.
Entries are numbered: `stash@{0}` is the newest (top), `stash@{1}` the one
below it, and so on.

```
stash@{0}  <- newest, top of pile
stash@{1}
stash@{2}  <- oldest
```

**How to read this:** each box is one shelf entry; the numbers in braces
are its position from the top. Suppose A is older than B.

BEFORE (two entries):

```
  STASH STACK
  stash@{0}   B   <- top
  stash@{1}   A
```

Command: `git stash push -m "C"` (after making new edits)

AFTER:

```
  STASH STACK
  stash@{0}   C*  <- top
  stash@{1}   B
  stash@{2}   A
```

**What changed:** C* is the new entry on top. B and A are unchanged but
their numbers each went up by one. That is why you should check
`git stash list` before using a number.

### Naming and managing stashes
If you have several, names help. Without a name you only get "WIP on ..."
and will forget what each one is.

```bash
git stash push -m "wip: half-done retry logic"    # same as "git stash", plus a label
git stash list                                     # show everything on the shelf
```

Expected output of `git stash list`:

```
stash@{0}: On feature-a: wip: half-done retry logic
stash@{1}: On main: quick experiment
```

Line by line:
- `stash@{0}` = the ID of that entry. You use it in later commands.
- `On feature-a` = the branch you were on when you stashed.
- the text after = your label.

Working with a specific entry:

```bash
git stash show stash@{0}        # preview which files changed in that entry
git stash show -p stash@{0}     # same, with the full line-by-line diff
git stash apply stash@{0}       # put the changes back, but KEEP the entry on the shelf
git stash pop stash@{0}         # put the changes back AND remove the entry
git stash drop stash@{0}        # throw the entry away without applying it
git stash clear                 # throw away ALL entries
```

Example output of `git stash show stash@{0}`:

```
 app.py | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
```

`app.py | 4 ++--` = file name, 4 lines changed, shown as 2 added (`+`) and 2 removed (`-`).

**Warning:** `git stash clear` and `git stash drop` have no normal "undo".
(Git does print the dropped entry's hash, and advanced recovery is
sometimes possible, but do not rely on it.)

### Common confusion: `apply` vs `pop`
| Command | Puts changes back? | Removes entry from shelf? |
|---|---|---|
| `git stash apply` | yes | no (safe copy stays) |
| `git stash pop` | yes | yes |

**How to read this:** compare only the right-hand column (the shelf)
after each command. The folder gets the edits back in both cases.

```
  AFTER git stash apply            AFTER git stash pop
  FOLDER      STASH STACK          FOLDER      STASH STACK
  app.py      stash@{0}: kept      app.py      (empty)
  = EDITED    app.py edits         = EDITED
```

**What changed:** only the shelf differs. `apply` leaves the entry
there; `pop` removes it.

Use `apply` when you want to try the same changes on several branches, or
when you want to be extra careful. Use `pop` for the normal case.

### Common confusion: "my new file did not get stashed!"
By default stash only shelves files Git already **tracks**.
**Tracked file** = a file Git already knows about because it was
committed before. A brand-new file you never ran `git add` on is
**untracked**, and plain `git stash` leaves it in your folder.

```bash
git stash -u        # -u = --include-untracked: also shelve new, never-added files
git stash -a        # -a = --all: also shelve files that .gitignore tells Git to ignore
```

`-a` can sweep up things like virtual environments and build folders, so
use it rarely.

### What if `pop` hits a conflict?
A **conflict** = Git cannot combine two versions of the same lines by
itself (Lesson 05). If `git stash pop` conflicts:

1. Git writes the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) into your file.
2. Git **keeps** the stash entry on the shelf (it does not drop it).
3. You fix the file, then run `git stash drop` yourself.

Nothing is lost if it goes wrong. That is the safety net.

### Doubt? "Where does the stash live? Is it a branch?"
No. It is stored inside your local `.git` folder as special commits. It
is **local only**: it is never pushed to GitHub, and it is not shared with
teammates. Stash is a private shelf.

### Mini recap
```
edit files -> git stash -> switch, work elsewhere -> switch back -> git stash pop
```

---

## `git tag` — Marking Specific Points (Releases)

### Why do we need this?
Commits are identified by hashes like `a1b2c3d`. Nobody remembers those.
When you ship version 1.0 of your test framework, you want a human name
for "exactly this snapshot" that never changes.

Analogy: a branch is like a bookmark you keep moving forward as you read.
A **tag** is like a sticky note glued onto one specific page. It stays on
that page forever, even as you keep reading.

**Tag = a permanent, named label on one specific commit.** Unlike a
branch, it never moves when you make new commits.

**How to read this:** a label under a commit points at that commit. Watch
which label moves and which stays.

BEFORE (you just tagged C; `main` is also on C):

```
  A---B---C
          ^
          main, v1.0.0
          HEAD -> main
```

Command: make two more commits (`git commit` twice)

AFTER:

```
  A---B---C---D*---E*
          ^        ^
          v1.0.0   main
                   HEAD -> main
```

**What changed:** the branch `main` moved from C to E* (and HEAD moved
with it). The tag `v1.0.0` did NOT move: it is still on C. D* and E* are
the new commits; A, B, C are unchanged.

### Creating tags

```bash
git tag v1.0.0                                 # label the commit you are on now
git tag v1.0.0 a1b2c3d                         # label a specific older commit instead
git tag -a v1.0.0 -m "First stable release"    # annotated tag (explained below)
```

`git tag` prints nothing when it succeeds. Silence here is success.

### Two kinds of tags
- **Lightweight tag** = just a name pointing at a commit. No extra information. Like a sticky note with only a title.
- **Annotated tag** = a full stored object (like a mini-commit) with its own message, the name of the person who tagged, and the date. Like a sticky note with a title, a signature and a date.

**Use annotated tags for real releases.** They carry useful metadata and
work properly with `git describe` (a command that tells you "how far is the
current commit from the nearest tag").

Python analogy: the tag name `v1.0.0` is like the version string in
`setup.py` / `pyproject.toml`. The tag is Git's version of that number,
attached to the real commit.

A convention called **semantic versioning** names versions `MAJOR.MINOR.PATCH`:
- `MAJOR` goes up when you break compatibility (`1.0.0` -> `2.0.0`),
- `MINOR` goes up when you add features safely (`1.0.0` -> `1.1.0`),
- `PATCH` goes up for bug fixes only (`1.0.0` -> `1.0.1`).

### Looking at tags

```bash
git tag                  # list all tags
git tag -l "v1.*"        # list only tags matching a pattern (* = anything)
git show v1.0.0          # show what the tag points to
```

Expected output of `git tag`:

```
v1.0.0
v1.1.0
```

Expected output of `git show v1.0.0` for an annotated tag:

```
tag v1.0.0
Tagger: Dharnish <you@example.com>
Date:   Mon Jan 5 10:12:00 2026 +0530

First stable release

commit 7b3e9a1c4d5f60718293a4b5c6d7e8f901234567
Author: Dharnish <you@example.com>
Date:   Mon Jan 5 10:10:00 2026 +0530

    Add login tests
```

Line by line:
- `tag v1.0.0` + `Tagger:` + `Date:` + message = the tag's own metadata (only annotated tags have this part).
- `commit 7b3e9a1...` = the commit the tag points to.
- The indented text = that commit's message.

For a lightweight tag you would see only the `commit ...` part, because
there is no tag metadata.

### Checking out a tag

```bash
git checkout v1.0.0
```

This shows your project exactly as it was at that release. But it puts you
in a special state called detached HEAD. That is the next section.

### Tags are not pushed automatically
Creating a tag only changes your **local** repo. GitHub has never heard of
it until you send it.

```bash
git push origin v1.0.0          # push one tag
git push origin --tags          # push ALL your local tags
```

Expected output:

```
Enumerating objects: 1, done.
To github.com:you/repo.git
 * [new tag]         v1.0.0 -> v1.0.0
```

The line `* [new tag] v1.0.0 -> v1.0.0` = "a tag that did not exist on the remote is now there."

**How to read this:** two separate repos side by side, your laptop and
GitHub. We track where the tag exists.

BEFORE (tag created locally):

```
  YOUR LAPTOP               GITHUB (origin)
  v1.0.0                    (no tag)
```

Command: `git push origin v1.0.0`

AFTER:

```
  YOUR LAPTOP               GITHUB (origin)
  v1.0.0        --->        v1.0.0*
```

**What changed:** GitHub gained the tag (marked `*` as new). Your laptop
is unchanged.

Common confusion: a normal `git push` does NOT send tags. People forget
this, then wonder why the tag is missing on GitHub.

### Deleting a tag

```bash
git tag -d v1.0.0                    # delete locally
git push origin --delete v1.0.0      # delete on the remote (GitHub) too
```

Expected output:

```
Deleted tag 'v1.0.0' (was 7b3e9a1)
To github.com:you/repo.git
 - [deleted]         v1.0.0
```

The two commands are separate because your copy and GitHub's copy are
separate. Deleting one does not delete the other.

### GitHub Releases
A **GitHub Release** = a page on GitHub built on top of a tag. It adds
release notes and downloadable files (zip, binaries).

Steps in the web page: your repo -> Releases -> "Create a new release" ->
pick (or create) a tag -> write release notes -> optionally attach built
files -> Publish. This is the standard way projects publish downloadable
versions.

---

## Detached HEAD — What It Means and Why It's Not Scary

### Why do we need this?
It appears the first time you check out a tag, and the warning text scares
everyone. Let's make it boring.

### Normal state first
Normally `HEAD` points at a **branch name**, and the branch name points at
a commit:

```
HEAD -> main -> D
```

**How to read this:** HEAD points at the branch name, and the branch name
points at the newest commit. Two hops.

```
  A---B---C---D
              ^
              main
              HEAD -> main
```

When you commit, the branch moves forward, and HEAD follows it. Your new
commits always have a branch holding on to them.

### Detached state
If you check out a tag or a raw commit hash, `HEAD` points **directly at
a commit**, with no branch in between. "Detached" = HEAD is not attached
to a branch.

**How to read this:** say commit C has the short id `abc123` and the tag
`v1.0.0`. In the AFTER picture HEAD skips the branch and points at C.

BEFORE (attached):

```
  A---B---C---D
              ^
              main
              HEAD -> main
```

Command:

```bash
git checkout v1.0.0
```

AFTER (detached):

```
  A---B---C---D
          ^   ^
          |   main
          |
          v1.0.0
          HEAD -> abc123
```

**What changed:** only HEAD moved: from "main" to the commit `abc123`
(C). Branch `main` is still on D. No commits were created or changed.

Expected output (shortened):

```
Note: switching to 'v1.0.0'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

HEAD is now at 7b3e9a1 Add login tests
```

Line by line:
- `You are in 'detached HEAD' state` = HEAD points at a commit, not a branch.
- `look around, make experimental changes` = it is safe to explore.
- `git switch -c <new-branch-name>` = Git tells you the exit: make a branch here.
- `HEAD is now at 7b3e9a1 ...` = the commit you are standing on.

The modern command to do the same thing, with the intent spelled out, is
`git switch --detach v1.0.0`.

### What is the only risk?
You can run code and even commit. But a commit made while detached has no
branch holding it. If you switch away, those commits become **orphaned**
(nothing points to them). Git keeps them for a while, and the reflog
(Lesson 06, the personal log of where HEAD has been) can recover them, but
they are easy to lose track of.

**How to read this:** you are detached at C and make a commit X. Each
branch of history gets its own row.

BEFORE (detached at C):

```
  A---B---C---D      main
          ^
          HEAD -> abc123
```

Command: `git commit` (while detached)

AFTER:

```
            X*       HEAD -> xyz789   (no branch name points here!)
           /
  A---B---C---D      main
```

**What changed:** X* is a new commit and HEAD moved onto it. No branch
moved. Now run `git switch main`:

```
            X        <- orphaned: nothing points at it any more
           /
  A---B---C---D      main
                     HEAD -> main
```

X still exists in `.git` for a while but has no name. That is the risk.

Analogy: writing a note on a loose sheet of paper instead of in your
notebook. Fine for a moment, but you will lose it unless you file it.

### Fix: make a branch if you want to keep the work

```bash
git checkout v1.0.0
git switch -c hotfix-from-v1
```

Expected output:

```
Switched to a new branch 'hotfix-from-v1'
```

Now HEAD is attached to the new branch `hotfix-from-v1`, and new commits
have a permanent home.

**How to read this:** the rescue is to put a branch label on X before you
leave it.

BEFORE (detached, X exists only under HEAD):

```
            X*       HEAD -> xyz789
           /
  A---B---C---D      main
```

Command: `git switch -c hotfix-from-v1`

AFTER:

```
            X        hotfix-from-v1
           /         HEAD -> hotfix-from-v1
  A---B---C---D      main
```

**What changed:** a new branch label `hotfix-from-v1` was created on X,
and HEAD now points at that branch (attached again). No commits changed;
X is safe because a branch holds it.

### Fix: leave without keeping anything

```bash
git switch main
```

```
Previous HEAD position was 7b3e9a1 Add login tests
Switched to branch 'main'
```

Doubt? "Did I break something by being detached?" No. Detached HEAD
changes nothing about your branches. Only commits you make while detached
are at risk.

---

## `git bisect` — Binary Search Through History to Find a Bug

### Why do we need this?
Scenario: your test suite passed 3 weeks ago. Today it fails. There are
200 commits between then and now, by several people. Which one broke it?

Reading all 200 is slow. Checking out and testing each one is slow too.

### The analogy: the number guessing game
I think of a number from 1 to 200. You guess 100. I say "higher". You
guess 150. "Lower." Each guess cuts the remaining possibilities in half.
This is **binary search**, and you know it from Python (`bisect` module).

Bisect does exactly that over commits:

```
good                                        bad
 o----o----o----o----o----o----o----o----o----o
 1                   ^                       200
                  test the middle
```

**How to read this:** a small example with 9 commits, oldest on the left.
`G` = known good, `B` = known bad, `?` = not tested yet. The commit `c6`
is the culprit, but Git does not know that yet.

START (you only know the two ends):

```
  c1--c2--c3--c4--c5--c6--c7--c8--c9
  G   ?   ?   ?   ?   ?   ?   ?   B
```

STEP 1: Git checks out the middle, `c5`. You test it: works, so `good`.

```
  c1--c2--c3--c4--c5--c6--c7--c8--c9
  G   G   G   G   G   ?   ?   ?   B
                  ^
                  tested
```

STEP 2: Git checks out the middle of what is left, `c7`. Broken, so `bad`.

```
  c1--c2--c3--c4--c5--c6--c7--c8--c9
  G   G   G   G   G   ?   B   B   B
                          ^
                          tested
```

STEP 3: only `c6` is left to test. Broken, so `bad`.

```
  c1--c2--c3--c4--c5--c6--c7--c8--c9
  G   G   G   G   G   B   B   B   B
                      ^
                      first bad commit
```

**What changed:** the `?` window shrank 7 -> 3 -> 1 -> 0 commits, roughly
halving each time. Git stops when a good commit (`c5`) sits right next to
a bad one (`c6`): the bad one is the first bad commit. Nothing in history
changed; bisect only moves HEAD (detached) around to let you test.

- Test the middle commit. If it works, the bug came later (look at the right half).
- If it is broken, the bug came at or before it (look at the left half).
- Repeat. 200 commits need only about 8 tests, because log2(200) is about 7.6.

Key words:
- **Good commit** = a commit where the thing works.
- **Bad commit** = a commit where the thing is broken.
- **First bad commit** = the very first commit in history where it is broken. That is your culprit.

### Manual bisect, step by step

Step 1: start a session.

```bash
git bisect start
```

Expected output: none (or in older Git versions, nothing at all).

Step 2: tell Git the current commit is broken.

```bash
git bisect bad
```

```
status: waiting for both good and bad commits
```

Hmm, the exact message varies by Git version. The meaning: Git has one
end of the range, and needs the other.

Step 3: tell Git a commit where it worked (a hash, tag or branch name).

```bash
git bisect good v1.0.0
```

Expected output:

```
Bisecting: 99 revisions left to test after this (roughly 7 steps)
[5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d] Refactor login helper
```

Line by line:
- `Bisecting: 99 revisions left to test after this` = about 99 commits are still candidates.
- `(roughly 7 steps)` = how many more good/bad answers you will need.
- `[5c6d7e8f...] Refactor login helper` = the middle commit Git just checked out for you. Your working directory now contains the project exactly as of that commit.

Step 4: YOU test it. Run the app, or run the failing test, whatever proves
whether it is broken here. Then answer:

```bash
git bisect good      # this commit works
# or
git bisect bad       # this commit is broken too
```

After each answer Git jumps to the next middle commit and prints a new
"Bisecting: N revisions left" line.

Step 5: the final answer. After enough rounds, Git prints:

```
a1b2c3d4e5f60718293a4b5c6d7e8f9012345678 is the first bad commit
commit a1b2c3d4e5f60718293a4b5c6d7e8f9012345678
Author: Jane Doe <jane@example.com>
Date:   Tue Dec 2 14:20:00 2025 -0500

    Change timeout handling in login

 login.py | 3 ++-
 1 file changed, 2 insertions(+), 1 deletion(-)
```

Line by line:
- `... is the first bad commit` = the culprit. Every commit before it is good, it and everything after is bad.
- The rest = that commit's author, date, message and files touched. Read the diff to see the actual bug.

Step 6: end the session. **Do not skip this.**

```bash
git bisect reset
```

```
Previous HEAD position was a1b2c3d Change timeout handling in login
Switched to branch 'main'
```

`reset` = return to the branch you started on. Without it you stay stuck at
some old commit in detached HEAD state.

### Common confusion: "good" and "bad" are about the BUG, not about the code quality
You are only answering: "At this commit, does the problem I am hunting
exist?" Answer `good` if it does not, `bad` if it does.

### Common confusion: "I made a mistake answering"
It happens. Run `git bisect log` to see your answers, or `git bisect reset`
and start over. Wrong answers lead to a wrong culprit, so be careful.

### Common confusion: "Commit cannot be tested"
If at some commit the project will not even build, use `git bisect skip`.
That tells Git "I can't judge this one, pick another."

### Automating it (the SDET superpower)
If you have a command that exits with `0` for good and a non-zero number
for bad, Git can do all the steps itself. A test runner like `pytest`
already works this way: exit code 0 = tests passed, non-zero = failed.
(Exit code = the number a program returns to say how it ended.)

```bash
git bisect start HEAD v1.0.0
git bisect run pytest tests/test_login.py
```

`git bisect start HEAD v1.0.0` reads as: "bad end is HEAD, good end is
`v1.0.0`". The first argument is the bad commit, the second is the good one.

Expected output (shortened):

```
running  'pytest' 'tests/test_login.py'
...
Bisecting: 24 revisions left to test after this (roughly 5 steps)
...
running  'pytest' 'tests/test_login.py'
...
a1b2c3d4e5f60718293a4b5c6d7e8f9012345678 is the first bad commit
...
bisect found first bad commit
```

Git ran the test at every step, judged each by the exit code, and named
the culprit with no manual work. Then `git bisect reset` to finish.

Special exit codes: `125` means "skip this commit", and codes 128 and above
abort the whole run. Everything else from 1 to 127 (except 125) means bad.

Why this matters: "it used to work, now it does not, and I don't know
why" is one of the most common debugging situations in real work.

---

## `git blame` — Who Wrote This Line, and Why

### Why do we need this?
You are staring at one confusing line. You want to know who last changed it,
when, and (ideally) why. The word "blame" sounds harsh, but the purpose
is to find context, not to find someone to shout at.

`git blame` labels every line of a file with the commit that last changed it.

```bash
git blame app.py
```

Expected output:

```
a1b2c3d4 (Jane Doe  2025-11-02 14:20:00 -0500  42) if retries > MAX_RETRIES:
```

Line by line:
- `a1b2c3d4` = the commit that last changed this line.
- `Jane Doe` = the author of that commit.
- `2025-11-02 14:20:00 -0500` = when.
- `42` = the line number in the file.
- `if retries > MAX_RETRIES:` = the line itself.

Useful flags:

```bash
git blame -L 10,20 app.py     # only lines 10 to 20, not the whole file
git blame -w app.py           # ignore whitespace-only changes
git blame -C app.py           # follow code that was moved or copied from another place
```

Why each flag exists:
- `-w` : if someone re-indented the whole file, blame would show the re-indent commit for every line. `-w` skips those and shows the commit that changed the logic.
- `-C` : if someone moved a function from another file, blame would say "the move commit wrote this". `-C` traces back to where the code really came from.

### Blame tells you who and when. The commit message tells you why.
Follow up by reading that commit:

```bash
git show a1b2c3d4              # the full diff + commit message
git log -1 a1b2c3d4            # only the message and details, no diff
```

Expected output of `git log -1 a1b2c3d4`:

```
commit a1b2c3d4e5f60718293a4b5c6d7e8f9012345678
Author: Jane Doe <jane@example.com>
Date:   Sun Nov 2 14:20:00 2025 -0500

    Cap retries at MAX_RETRIES to stop runaway CI jobs
```

That message is the "why". It is also the reason good commit messages
(Lesson 04) matter: a future teammate, often you in six months, will land
on exactly this commit through blame.

In daily work most people do not type `git blame`. VS Code's GitLens
extension and PyCharm's "Annotate" gutter show the same information inline.
Still, know the raw command. It works in a plain SSH session on a server,
on a teammate's unfamiliar editor, or when reading a CI log.

---

## Pickaxe Search — Finding *When* a String Was Added or Removed

### Why do we need this?
Scenario: when did the constant `MAX_RETRIES` first appear? The file might
have been renamed or refactored since. `git log app.py` only follows the
one path and can lose track after a rename.

**Pickaxe search** = search through the actual changes inside every commit,
not through file names. The name comes from digging through history with a
pickaxe.

```bash
git log -S"MAX_RETRIES"
```

Expected output:

```
commit a1b2c3d4e5f60718293a4b5c6d7e8f9012345678
Author: Jane Doe <jane@example.com>
Date:   Sun Nov 2 14:20:00 2025 -0500

    Cap retries at MAX_RETRIES to stop runaway CI jobs
```

`-S"text"` lists commits where the **number of times** that exact text
appears changed. So it finds the commit that added it, removed it, or
copied it. A commit that merely edited a line next to it is not listed.

```bash
git log -G"MAX_RETRIES\s*="
```

`-G"pattern"` uses a **regular expression** (a text-matching pattern, like
Python's `re`). It lists commits whose diff added or removed a line that
matches the pattern. Here `\s*` means "any amount of whitespace", so this
finds assignments like `MAX_RETRIES = 3` and `MAX_RETRIES=3`.

### Common confusion: `-S` vs `-G`
| Flag | Matches | Good for |
|---|---|---|
| `-S"text"` | exact text; the count of occurrences changed | "when was this introduced or deleted?" |
| `-G"regex"` | a regex; any changed line matches | "when did this line change in any way?" |

Add `-p` to see the diff of each match, and `--all` to search every
branch, not just the current one:

```bash
git log -S"MAX_RETRIES" -p --all
```

---

## `git worktree` — Multiple Branches Checked Out at Once

### Why do we need this?
Normally one repo folder shows one branch at a time. To change branch you
must commit or stash first (the stash section above).

Analogy: one desk, one project. A worktree is a second desk in the same
office, sharing the same filing cabinet.

**Worktree = an extra folder with a different branch checked out, sharing the same `.git` history as your original folder.** There is no second
clone and no duplicated history. (Recall Lesson 03: objects are stored by
their content, so Git has nothing to duplicate.)

**How to read this:** each row is a folder on your disk. The middle column
is the branch checked out there. The right column is where history lives.

BEFORE (one folder):

```
  FOLDER              BRANCH            HISTORY
  /path/repo          main              ---> .git   (one copy)
```

Command: `git worktree add ../repo-hotfix hotfix-branch`

AFTER:

```
  FOLDER              BRANCH            HISTORY
  /path/repo          main              ---> .git   (still ONE copy)
  /path/repo-hotfix*  hotfix-branch     ---> same .git
```

**What changed:** one new folder (`*`) with its own branch and its own
HEAD. The `.git` history is shared, not copied. Your original folder is
unchanged.

```bash
git worktree add ../repo-hotfix hotfix-branch
```

Expected output:

```
Preparing worktree (checking out 'hotfix-branch')
HEAD is now at ef56789 Fix flaky login test
```

Meaning: a new folder `../repo-hotfix` (next to your repo folder) now has
`hotfix-branch` checked out. Your original folder is untouched.

```bash
git worktree list
```

```
/path/to/repo          abcd123 [main]
/path/to/repo-hotfix   ef56789 [hotfix-branch]
```

Each line = folder, latest commit there, branch in square brackets.

When done:

```bash
git worktree remove ../repo-hotfix
```

This deletes that extra folder. The branch and its commits are kept.

### Common confusion: one rule
The same branch cannot be checked out in two worktrees at once. Git will
refuse, to prevent two folders from fighting over one branch.

### Why an SDET will like this
Run the full test suite against `main` in one worktree while you keep
editing a feature branch in another. No stash and pop, and no risk of a
half-finished edit leaking into the run you are validating. It is also
handy for keeping a manual-QA checkout of a release branch open in its own
folder while you keep developing in your main folder.

---

## Exercise

```bash
mkdir stash-tag-bisect-practice && cd stash-tag-bisect-practice && git init
```

**Stash:**
1. Create `notes.txt` with one line and commit it. Now edit `notes.txt`
   again without committing. Run `git stash push -m "my wip"`.
   - Expected: `git status` says "working tree clean", and `git stash list` shows `stash@{0}: On main: my wip` (your branch may be called `master`).
   - Now `git switch -c other`, make an unrelated commit, `git switch` back to your first branch, and run `git stash pop`.
   - Expected: your edit to `notes.txt` is back, and `git stash list` prints nothing.

**Tags:**
2. Make 3 commits. Tag the second one with
   `git tag -a v1.0.0 -m "First stable release" <hash of commit 2>`
   (find the hash with `git log --oneline`).
   - Expected: `git show v1.0.0` shows a `Tagger:` block and then the second commit, not the latest.
   - Also run `git tag` and expect to see `v1.0.0`.

**Bisect (the important one):**
3. Create `check.py` containing `import sys; sys.exit(0)`. Commit it.
   Tag that commit `good-start`.
4. Make 6 more commits (for example, add a line to a file each time). In one
   of them, in the middle (say commit 3 of 6), change `check.py` to
   `import sys; sys.exit(1)`. Keep committing normally after that, as if you
   had not noticed. Write down which commit you broke it in.
5. From the latest commit run:
   ```bash
   git bisect start HEAD good-start
   git bisect run python3 check.py
   ```
   - Expected: after about 3 steps, Git prints `<hash> is the first bad commit` followed by `bisect found first bad commit`.
6. Confirm that the hash and message match the commit you noted in step 4.
   Then run `git bisect reset`.
   - Expected: `Switched to branch 'main'` (or your branch name), and `git status` shows a clean tree on your branch, not detached.

**Bonus (optional):**
7. Run `git checkout v1.0.0`, read the detached HEAD message, then return with `git switch -`.
8. Run `git blame notes.txt` and then `git log -S"some text from your file"` to see both tools on your own repo.

---

## Recap in 5 lines
1. `git stash` shelves uncommitted changes on a private local stack; `pop` brings them back (use `-u` for new files).
2. `git tag -a v1.0.0 -m "..."` puts a permanent label on a commit; push it explicitly with `git push origin v1.0.0`.
3. Detached HEAD = HEAD points at a commit, not a branch; run `git switch -c <name>` if you want to keep new commits.
4. `git bisect` (ideally with `git bisect run <test>`) finds the first bad commit in about log2(N) tests; always finish with `git bisect reset`.
5. `git blame` tells you who and when, `git log -S` tells you when a string appeared, and `git worktree` gives you a second branch in a second folder.

Continue to [Lesson 11 — GitHub Pro Features](11-github-pro-features.md).
