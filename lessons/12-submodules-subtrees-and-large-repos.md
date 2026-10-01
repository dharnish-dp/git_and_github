# Lesson 12 — Submodules, Subtrees & Large-Repo Tooling

## Goal
Learn how to put one Git repository inside another when you genuinely
need to (shared test libraries, shared firmware/hardware code). Understand
why submodules have a reputation for confusing teams. Then learn the tools
(`sparse-checkout`, shallow clone, partial clone) that keep very large
repos fast to work in.

## Prerequisites
[Lesson 11 — GitHub Pro Features](11-github-pro-features.md)

## After This Lesson You Will Be Able To
- Explain what a submodule actually is (a commit pointer, not a copy of
  files) and avoid its two classic beginner traps
- Add, clone, and update a submodule correctly
- Use `git subtree` as an alternative, and know which one fits which
  situation
- Judge when the right answer is neither, and restructure instead
- Use sparse-checkout and shallow/partial clone to work efficiently in a
  large repo

## Words You Already Know (quick refresher)
- **Repository (repo)** = a project folder plus its full saved history.
- **Commit** = a saved snapshot of your project. Each one has a unique
  ID called a **hash** (a long string like `a1b2c3d...`).
- **Parent repo** = the repo that *contains* another repo (new word for
  this lesson; it is just a label, not a special Git feature).
- **HEAD** = Git's "you are here" marker (Lesson 02).
- **Detached HEAD** = HEAD points at a bare commit, not at a branch
  (Lesson 10). We revisit it below.

---

## The Problem: One Repo Needs Code From Another Repo

**Why do we need this?** Real projects share code. If you did not have a
plan for sharing, you would copy files around and they would slowly rot.

**Python analogy:** you wrote a helper module `test_utils.py`. Three
different test projects need it. How do you get it into all three?

Picture it concretely. You maintain a shared `test-utils` library
(device fixtures, retry helpers, a mock hardware interface). Three
separate project repos all need it. You have three options:

```
Option 1: Copy-paste the code into each repo.
   Works today. Breaks when test-utils gets a bugfix: now there are
   3 copies to fix by hand, and they drift apart.

Option 2: Publish test-utils as a real installable package
   (PyPI, a private package index, an internal artifact repo) and
   `pip install` it like any other dependency.
   USUALLY THE RIGHT ANSWER. Versioned, upgradeable, no Git tricks.

Option 3: Embed the other repo inside yours, using a submodule
   or a subtree (the two Git features in this lesson).
   For when option 2 is not possible: the shared code is not a clean
   installable package, or you must make coordinated changes to both
   repos as one piece of work.
```

This lesson is about option 3. Keep option 2 in your back pocket. Most
of the pain people associate with submodules is really the pain of using
option 3 where option 2 would have been easier.

---

## Git Submodules: A Repo Inside a Repo

### Step 1: The idea, with an analogy

**Analogy:** a bookmark in a shared document. Your notebook does not
contain a copy of the other person's book. It contains a sticky note:
"See *that book*, edition printed on March 3rd, page 1."

**Python analogy:** it is like a `requirements.txt` line
`shared-test-utils==1.4.2`. Your repo does not hold the library's code.
It holds the exact version to fetch.

**Plain explanation.** A submodule is **a pointer to one specific commit
of another repo**, placed at a chosen folder path. It is not a copy of
the files.

Diagram legend (same in every lesson):

```
  A---B---C     commits; oldest on the LEFT, newest on the RIGHT
  main          branch name sits at the END of its row; ^ marks the commit
                a label points at; HEAD is shown as "HEAD -> main"
  D*            a NEW commit, created by the command shown above it
  /  and  \     a branch forking off, or merging back in
  --->  :  v    a link from one thing to another
```

**How to read this:** two separate repos, each drawn as its own row of
commits. The dotted link down from the parent's commit `P3` to the child's
commit `a1b2c3d` is the submodule: the parent stores only that hash.

```
 PARENT    P1---P2---P3                      main-repo
                     :
                     : P3 records: libs/shared-test-utils = a1b2c3d
                     v
 CHILD     c1---c2---a1b2c3d---d4---d5       shared-test-utils
                     ^
                     pinned (d4 and d5 exist, but the parent ignores them)
```

What to notice: the child repo keeps growing (`d4`, `d5`) but the parent
still points at `a1b2c3d` until someone deliberately moves the pin.

"Pinned" means "fixed to one exact commit, so everybody gets the same
version." The parent repo stores only that one hash. It knows nothing
else about what is inside `shared-test-utils`.

### Step 2: Add a submodule

**Why do we need this command?** It records the pointer and downloads
the other repo into the folder.

```bash
git submodule add https://github.com/org/shared-test-utils.git libs/shared-test-utils
```

Expected output:

```
Cloning into '/home/you/main-repo/libs/shared-test-utils'...
remote: Enumerating objects: 42, done.
remote: Counting objects: 100% (42/42), done.
remote: Total 42 (delta 10), reused 40 (delta 9), pack-reused 0
Receiving objects: 100% (42/42), 8.10 KiB | 4.05 MiB/s, done.
Resolving deltas: 100% (10/10), done.
```

Line by line:
- `Cloning into '...'` = Git **cloned** (made a full download copy of)
  the other repo into the folder `libs/shared-test-utils`.
- The `remote:` / `Receiving objects` / `Resolving deltas` lines are
  normal download progress. Nothing is wrong.

Now look at what changed:

```bash
git status
```

```
On branch main
Changes to be committed:
	new file:   .gitmodules
	new file:   libs/shared-test-utils
```

Two things were staged (staged = queued for your next commit):

1. **`.gitmodules`** is a plain text file in your repo root. It records
   each submodule's folder path and URL. It is a normal tracked file,
   committed like any other.
2. **`libs/shared-test-utils`** is a single special entry: a **gitlink**.
   A gitlink is a tree entry that holds a commit hash instead of file
   content. (Lesson 03 covered blobs = file contents and trees = folders.
   A gitlink is a third kind of entry.)

Commit them:

```bash
git commit -m "Add shared-test-utils as a submodule, pinned to current main"
```

```
[main 3e4f5a6] Add shared-test-utils as a submodule, pinned to current main
 2 files changed, 4 insertions(+)
 create mode 100644 .gitmodules
 create mode 160000 libs/shared-test-utils
```

Line by line:
- `create mode 100644 .gitmodules` = a normal file (mode `100644` means
  "regular file").
- `create mode 160000 libs/shared-test-utils` = mode `160000` is Git's
  code for "this is a gitlink, not a file or folder."

You can see the gitlink directly:

```bash
git ls-tree HEAD libs/
```

```
160000 commit a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4	libs/shared-test-utils
```

Reading it: `160000` (mode) `commit` (the entry type) then the pinned
commit hash, then the path. Your parent repo's history only changes when
that one hash changes.

> **Common confusion:** "Is `libs/shared-test-utils` a folder or not?"
> On your disk, yes, it is a normal-looking folder full of files. But to
> the *parent* repo, it is one line holding one hash. The files inside
> belong to the *other* repo. Rule of thumb: inside that folder, Git
> talks to the submodule's repo; outside it, Git talks to the parent.

### Step 3: Trap #1, a plain `git clone` leaves submodules empty

**Why do we need to know this?** Nearly everyone hits this on day one.

Cloning a repo that has submodules does **not** download their content
automatically. The parent only stores a pointer and a URL. Git waits for
you to ask.

```bash
git clone https://github.com/org/main-repo.git
ls libs/shared-test-utils
```

```
(no output: the folder exists but is empty)
```

Fix it in one step when cloning:

```bash
git clone --recurse-submodules https://github.com/org/main-repo.git
```

Or fix an already-cloned repo:

```bash
git submodule update --init --recursive
```

```
Submodule 'libs/shared-test-utils' (https://github.com/org/shared-test-utils.git) registered for path 'libs/shared-test-utils'
Cloning into '/home/you/main-repo/libs/shared-test-utils'...
Submodule path 'libs/shared-test-utils': checked out 'a1b2c3d4e5f67890a1b2c3d4e5f67890a1b2c3d4'
```

What the flags mean:
- `--init` = "register the submodule from `.gitmodules` (first time only)."
- `--recursive` = "also do this for submodules inside submodules."
- `update` = "check out the exact commit the parent has pinned."

Last line: `checked out 'a1b2...'` confirms the pinned commit is now on disk.

> **Doubt?** "A teammate says `libs/` is empty for them." Almost always
> the cause is this trap. Send them the `git submodule update --init
> --recursive` command.

### Step 4: Trap #2, submodules start in detached HEAD

**Why do we need to know this?** It is how people accidentally lose
commits.

Recall from Lesson 10: normally HEAD points at a *branch* (a moving
label on the latest commit). **Detached HEAD** means HEAD points at a
bare commit with no branch label. Any new commit you make there has no
label, so it is easy to lose.

A submodule is checked out at exactly the pinned commit, so it always
starts detached. Confirm it:

```bash
cd libs/shared-test-utils
git status
```

```
HEAD detached at a1b2c3d
nothing to commit, working tree clean
```

`HEAD detached at a1b2c3d` is the warning. To work safely, switch to a
real branch *first*:

```bash
git switch main
```

```
Previous HEAD position was a1b2c3d Add retry helper
Switched to branch 'main'
Your branch is behind 'origin/main' by 3 commits, and can be fast-forwarded.
```

"Behind by 3 commits" means the pinned commit was older than the latest
`main`. Pull to catch up, then go back to the parent and record the new
pointer:

```bash
git pull
cd ../..
git status
```

```
On branch main
Changes not staged for commit:
	modified:   libs/shared-test-utils (new commits)
```

`(new commits)` means: the submodule now sits on a newer commit than the
parent has pinned. Record the new pin:

```bash
git add libs/shared-test-utils     # stages the UPDATED pointer, not any files
git commit -m "Bump shared-test-utils to latest main"
```

```
[main 7b8c9d0] Bump shared-test-utils to latest main
 1 file changed, 1 insertion(+), 1 deletion(-)
```

The one changed line is the hash inside the gitlink: old hash out, new
hash in.

**How to read this:** the same two rows as before. The parent's newest
commit decides which child commit is pinned. Older parent commits keep
their old pin forever.

BEFORE (parent pins `a1b2c3d`):

```
 PARENT    P1---P2---P3                      main-repo
                     :
                     v  P3 pins a1b2c3d
 CHILD     c1---c2---a1b2c3d---d4---d5       shared-test-utils
                     ^
                     pinned by P3
```

Command (in the parent, after `git pull` inside the submodule):
`git add libs/shared-test-utils && git commit -m "Bump ..."`

AFTER (new parent commit `P4*` pins `d5`):

```
 PARENT    P1---P2---P3---P4*                main-repo
                          :
                          : P4* records: libs/shared-test-utils = d5
 CHILD     c1---c2---a1b2c3d---d4---d5       shared-test-utils
                     ^              ^
                     |              pinned by P4* (new)
                     pinned by P3 (old, still true for P3)
```

What changed: one new parent commit `P4*`. The child repo did not change
at all. Checking out `P3` again would pin `a1b2c3d` again.

### Step 5: Teammates do not update automatically

**Why do we need to know this?** It is the most common "why is my
submodule not updating?" question.

**Analogy:** you changed the sticky note to "edition 5". Your teammate
got your updated notebook, but their desk still has edition 4 of the
book open. They must go fetch edition 5.

After a teammate pulls your parent-repo commit, their submodule folder
still shows the *old* commit. They need:

```bash
git pull
git submodule update --init --recursive
```

This is deliberate. It makes builds reproducible: nobody's submodule
silently drifts to a newer version behind their back. The fix is always
the same two steps: pull the parent, then update the submodules.

> **Common confusion: `git pull` vs `git submodule update`.**
> `git pull` updates the *parent repo* (including the pin). `git
> submodule update` moves the *submodule's files* to match that pin.
> Different jobs. You need both.

---

## Git Subtree: The "Merge It In" Alternative

### Step 1: The idea

**Analogy:** instead of a sticky note pointing to a book, you photocopy
the chapters you need and bind them into your own notebook.

**Plain explanation.** `git subtree` **copies** the other repo's files
into yours as ordinary tracked files, so they live in your history.
There is no `.gitmodules`, no gitlink, and no detached HEAD. Anyone who
runs a plain `git clone` gets everything immediately.

### Step 2: Add a subtree

**Why do we need this command?** It performs the one-time import.

```bash
git subtree add --prefix=libs/shared-test-utils \
    https://github.com/org/shared-test-utils.git main --squash
```

Breaking down the arguments:
- `--prefix=libs/shared-test-utils` = the folder in your repo where the
  files will land.
- the URL = the repo to import from.
- `main` = which branch of that repo to import.
- `--squash` = collapse the other repo's whole history into **one**
  commit (squash = combine many commits into one). Without it, every
  commit from the other repo would appear one-by-one in your `git log`.
  Omit `--squash` only if you truly want that full history in your log.

Expected output:

```
git fetch https://github.com/org/shared-test-utils.git main
From https://github.com/org/shared-test-utils
 * branch            main       -> FETCH_HEAD
Added dir 'libs/shared-test-utils'
```

`Added dir '...'` confirms the files are now real tracked content in
your repo. Run `git log --oneline -3` and you will see a "Squashed
'libs/shared-test-utils/' content from commit ..." commit plus a merge
commit.

**How to read this:** `A---B---C` is your repo. `s1---s2---s3` is the
shared repo. The subtree command merges the shared repo's content into a
subfolder of your repo, so the merge commit `M*` has two parents.

BEFORE (two unrelated repos):

```
 your repo:            A---B---C                   main
 shared-test-utils:    s1---s2---s3                main   (separate repo)
```

Command: `git subtree add --prefix=libs/shared-test-utils <url> main --squash`

AFTER with `--squash` (the shared history is collapsed into one commit):

```
                     S*                  s1+s2+s3 squashed into ONE commit
                       \
      A---B---C--------M*                main
```

AFTER without `--squash` (every shared commit comes along):

```
         s1---s2---s3                    shared history, copied in
                      \
      A---B---C-------M*                 main
```

What changed: in both pictures `M*` is a merge commit whose files now
include the real `libs/shared-test-utils/` folder. Only the amount of
shared history pulled into your `git log` differs.

### Step 3: Pull updates from upstream

**Upstream** = the original repo you imported from.

```bash
git subtree pull --prefix=libs/shared-test-utils \
    https://github.com/org/shared-test-utils.git main --squash
```

**How to read this:** each pull repeats the same pattern: one new squash
commit from the shared repo, merged into your `main`.

BEFORE (`S1`/`M1` came from the first `subtree add`, `D` is your own work):

```
                     S1
                       \
      A---B---C--------M1---D                main
```

AFTER `git subtree pull`:

```
                     S1        S2*
                       \         \
      A---B---C--------M1---D-----M2*        main
```

What changed: two new commits, `S2*` (the newer shared code, squashed) and
`M2*` (the merge that lands it in `libs/shared-test-utils/`).

### Step 4: Push your local edits back upstream

You can send changes you made inside `libs/shared-test-utils` back to the
original repo. This is possible with a subtree, unlike plain
copy-paste:

```bash
git subtree push --prefix=libs/shared-test-utils \
    https://github.com/org/shared-test-utils.git a-new-branch-to-open-a-pr-from
```

Push to a **new branch** (the last argument) so you can open a pull
request there, rather than writing straight into their `main`.

### Submodule vs Subtree, Honestly

| | Submodule | Subtree |
|---|---|---|
| What's stored | A pointer (pinned commit hash) | Actual copied files + history |
| Clone experience | Empty folder unless `--recurse-submodules` | Works immediately, no flags |
| "What version are we on?" | Trivial: it's the pinned commit | No built-in answer; you'd have to dig through log messages |
| Repo size | Small (just a pointer) | Grows, because you store real content |
| Contributing back upstream | Normal: it's a regular clone at that path | Works via `subtree push`, but history gets murkier with more back-and-forth |
| Team confusion factor | High: detached HEAD, empty-clone traps | Low for consumers, but merge history can get messy for maintainers |

> **Doubt?** "Which one should I pick?" If people mostly *consume* the
> shared code and rarely edit it, subtree is less confusing. If you need
> an exact, visible version pin and often edit the shared repo too,
> submodule fits better. If you are unsure, go to the next section first.

---

## When to Prefer Neither

**Why do we need this section?** Knowing when *not* to use a tool is
part of mastering it.

Be as honest here as the rebase golden rule was in Lesson 09. If you
constantly fight submodules or subtrees (teammates forgetting
`--recurse-submodules`, endless "why didn't my submodule update"
questions, subtree merges producing baffling diffs), the *architecture*
is probably wrong. You are not missing a clever flag.

Two real fixes, in order of preference:

1. **Publish the shared code as a versioned package** (PyPI, internal
   package index, private registry) and depend on it the normal way
   (`pip install`). This is almost always cleaner than embedding one Git
   repo inside another.
2. **If the code always changes together with the code that uses it**,
   it belongs in the *same* repo. A small, deliberate **monorepo** (one
   repo holding several related projects) is better than two repos glued
   together.

Most large tech orgs avoid submodules for exactly this reason: they
either invest in package management, or go fully monorepo (see
[What Next](../what-next.md) for more on monorepo tooling). Submodules
and subtrees are worth knowing because you will meet them in the wild.
Reaching for them by default is rarely the top-1% move.

---

## Working Efficiently in a Large Repo

**Why do we need this?** This is a separate problem in the same
neighborhood. A huge repo (many files, long history, or both) makes a
plain `git clone` slow, even if you only touch one small corner.

**Analogy:** downloading an entire 500-video playlist when you want to
watch three videos. There are three independent ways to download less.
They can be combined.

Terms first:
- **History** = the full chain of past commits.
- **Blob** = the stored content of one version of one file (Lesson 03).
- **Working directory** = the actual files you see and edit on disk.

**Baseline: what a plain `git clone` downloads.** How to read this: the top
row is the commit line (oldest left, newest right). Each row below it is one
file, with one mark per commit showing whether that commit's version of the
file was downloaded.

```
  X   downloaded          .   NOT downloaded (skipped)
  [C1]   a commit that was NOT downloaded
```

```
 COMMITS    C1---C2---C3---C4---C5---C6     main
                                     ^
                                     HEAD -> main
 app.py     X    X    X    X    X    X
 util.py    X    X    X    X    X    X
 manual.pdf X    X    X    X    X    X
```

Every commit and every version of every file comes down. That is the slow
part on a huge repo. The three tools below each skip a different piece.

### Tool 1: Shallow clone (less history)

A shallow clone downloads only the most recent commits, not the whole
chain.

```bash
git clone --depth 1 <url>       # only the latest commit, fast
```

```
Cloning into 'big-repo'...
remote: Enumerating objects: 1820, done.
Receiving objects: 100% (1820/1820), 12.4 MiB | 9.1 MiB/s, done.
```

`--depth 1` = "only 1 commit deep." It looks tiny because older commits
are simply not there. Confirm:

```bash
git log --oneline
```

```
c0ffee1 (grafted, HEAD -> main, origin/main) Latest change
```

`grafted` means "history was cut off here on purpose."

How to read this: compare with the baseline above. Commits in brackets
were never downloaded, so their file versions are gone too.

AFTER `git clone --depth 1`:

```
 COMMITS    [C1]-[C2]-[C3]-[C4]-[C5]-C6      main
                                     ^
                                     HEAD -> main
 app.py      .    .    .    .    .   X
 util.py     .    .    .    .    .   X
 manual.pdf  .    .    .    .    .   X
```

What changed: you keep only `C6` and the files as they are in `C6`. Your
working directory still has every file; only the past is missing.

Need more later?

```bash
git fetch --deepen=50           # pull 50 more commits of history
git fetch --unshallow           # or get all the rest
```

BEFORE (depth 1):

```
 COMMITS    [C1]-[C2]-[C3]-[C4]-[C5]-C6      main
```

Command: `git fetch --deepen=2`

AFTER (`C4*` and `C5*` are newly downloaded):

```
 COMMITS    [C1]-[C2]-[C3]-C4*--C5*--C6      main
```

Trade-off: commands that walk history (`git log`, `git blame`,
`git bisect` from Lesson 10) have nothing past what you fetched.
Great for a quick CI build. Poor as your daily-driver clone.

### Tool 2: Partial clone (less file content)

A partial clone keeps the **full commit history** but downloads file
*contents* lazily, only when you actually check out or touch that file.

```bash
git clone --filter=blob:none <url>
```

- `--filter=blob:none` = "skip all blobs (file contents) for now."
  Commit and folder information still comes down in full.
- When you check out a branch, Git fetches just the blobs it needs. You
  may see a short extra download at that moment. That is the lazy
  fetching at work, not an error.

How to read this: same marks as the baseline. Here every commit is
present (no brackets), but most file versions are skipped.

AFTER `git clone --filter=blob:none`, once `main` is checked out:

```
 COMMITS    C1---C2---C3---C4---C5---C6     main
                                     ^
                                     HEAD -> main
 app.py     .    .    .    .    .    X
 util.py    .    .    .    .    .    X
 manual.pdf .    .    .    .    .    X
```

What changed: the commit line is complete, so `git log` works offline.
Only the `C6` versions (needed for checkout) were fetched. Run `git blame
app.py` or `git show C2:app.py` and Git fetches that old version then.

This is the better middle ground for daily work: real history for
`log`/`blame`/`bisect`, without downloading every historical version of
every file.

> **Common confusion: shallow vs partial.**
> Shallow = *fewer commits* (history is cut). Partial = *all commits* but
> *fewer file contents* (fetched on demand). Shallow loses history;
> partial does not.

### Tool 3: Sparse-checkout (less on disk)

Sparse-checkout is for when you work in only one or two subfolders of a
huge repo. The rest never appears in your working directory.

```bash
git clone --filter=blob:none --no-checkout <url>
cd repo
git sparse-checkout init --cone     # cone mode: modern, fast, recommended
git sparse-checkout set services/my-service libs/shared-test-utils
git checkout main
```

Line by line:
- `--no-checkout` = clone, but do not put any files in the working
  directory yet (so we can choose first).
- `init --cone` = turn on sparse mode. **Cone mode** means you list
  whole folders (plus their parent folders), which is fast and simple.
- `set ...` = the folders you want.
- `git checkout main` = now materialize (write to disk) only those.

Check what is active:

```bash
git sparse-checkout list
```

```
libs/shared-test-utils
services/my-service
```

```bash
ls
```

```
libs  services
```

Only your chosen folders (and their parents) exist. Other top-level
folders are missing from disk but still in history, so `git log`,
`blame`, and branch switching work correctly. To go back to everything:
`git sparse-checkout disable`.

How to read this: the commit line is complete (history is untouched). The
file tree shows which folders are written to your disk.

```
 COMMITS    C1---C2---C3---C4---C5---C6     main
                                     ^
                                     HEAD -> main

 repo/                            on disk?
   services/my-service/           YES   (in your sparse set)
   libs/shared-test-utils/        YES   (in your sparse set)
   docs/                          no    (in history, not written to disk)
   firmware/                      no    (in history, not written to disk)
```

What changed: nothing about commits. Only the working directory shrinks,
and (with `--filter=blob:none`) the skipped folders' files are never
downloaded either.

Pairing sparse-checkout with partial clone gives a light working
directory *and* a light initial download. This is the standard
combination for a genuinely large monorepo.

| Tool | Less of... | History intact? |
|---|---|---|
| Shallow (`--depth`) | commits | No, cut |
| Partial (`--filter=blob:none`) | file contents downloaded | Yes |
| Sparse-checkout | files on disk | Yes |

---

## Exercise

Setup. Use `-b main` so the branch name is predictable:

```bash
mkdir submodule-practice && cd submodule-practice
mkdir shared-lib && cd shared-lib && git init -b main
echo "def helper(): return 42" > helper.py
git add . && git commit -m "Initial shared-lib commit"
cd ..
mkdir main-project && cd main-project && git init -b main
echo "readme" > README.md && git add . && git commit -m "Initial main-project commit"
```

**Important note about local paths.** Newer Git versions (2.38.1+) block
submodule operations that use a plain local-folder path, for security.
Since this exercise uses local folders, add
`-c protocol.file.allow=always` to submodule commands below. It only
applies to that one command. Also use the absolute path, so the URL
saved in `.gitmodules` works from any clone:

```bash
LIB="$(cd ../shared-lib && pwd)"    # absolute path to shared-lib
```

1. Add the submodule and commit it:
   ```bash
   git -c protocol.file.allow=always submodule add "$LIB" libs/shared-lib
   git commit -m "Add shared-lib submodule"
   cat .gitmodules
   git ls-tree HEAD libs/
   ```
   Expected: `.gitmodules` shows `[submodule "libs/shared-lib"]` with
   `path = libs/shared-lib` and `url = /your/path/shared-lib`. The
   `ls-tree` line starts with `160000 commit` followed by a hash and
   `libs/shared-lib`.

2. Clone `main-project` into a fresh folder **without**
   `--recurse-submodules`:
   ```bash
   cd ..
   git clone main-project clone-plain
   ls clone-plain/libs/shared-lib
   ```
   Expected: `ls` prints nothing (empty folder). Then populate it:
   ```bash
   cd clone-plain
   git -c protocol.file.allow=always submodule update --init
   ls libs/shared-lib
   ```
   Expected: `helper.py` now appears.

3. Inside that submodule checkout, prove detached HEAD, fix it, commit,
   and bump the pointer:
   ```bash
   cd libs/shared-lib
   git status                # expect: HEAD detached at <hash>
   git switch main
   echo "def extra(): return 1" >> helper.py
   git commit -am "Add extra helper"
   cd ../..
   git status                # expect: modified: libs/shared-lib (new commits)
   git add libs/shared-lib
   git commit -m "Bump shared-lib"
   ```
   Expected: `git log --oneline -1` shows "Bump shared-lib". Running
   `git ls-tree HEAD libs/` now shows a different hash than in step 1.

4. Switch to a subtree. Go back to the original `main-project`, remove
   the submodule cleanly, and add a subtree:
   ```bash
   cd ../main-project
   git submodule deinit -f libs/shared-lib
   git rm -f libs/shared-lib
   rm -rf .git/modules/libs/shared-lib
   git commit -m "Remove submodule"
   git subtree add --prefix=libs/shared-lib "$LIB" main --squash
   cd ..
   git clone main-project clone-subtree
   ls clone-subtree/libs/shared-lib
   ```
   Expected: `helper.py` is already there, with no extra commands.
   `.gitmodules` no longer exists in the new clone.

5. (Optional, needs internet) Compare download times:
   ```bash
   time git clone --filter=blob:none --depth 50 <some large public GitHub repo URL> fast-clone
   time git clone <same URL> full-clone
   ```
   Expected: the first finishes noticeably faster and `git log --oneline
   | wc -l` in it shows about 50 lines, while the full clone shows many
   more.

---

## Recap in 5 Lines
1. A submodule is a pinned-commit pointer to another repo; a subtree is a
   real copy of its files in your repo.
2. Submodule traps: clones come up empty (use `--recurse-submodules`) and
   start in detached HEAD (switch to a branch before committing).
3. After pulling the parent, run `git submodule update --init --recursive`
   to move submodule files to the new pin.
4. Often the best answer is neither: publish a package, or merge into a
   monorepo.
5. For big repos: shallow = fewer commits, partial = lazy file contents,
   sparse-checkout = fewer files on disk; combine partial + sparse.

Continue to [Lesson 13 — Pro Workflows & Best Practices](13-pro-workflows-and-best-practices.md).
