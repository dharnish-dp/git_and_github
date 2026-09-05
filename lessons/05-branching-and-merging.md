# Lesson 05 — Branching & Merging

## Goal
Understand branches as what they really are (movable pointers, not
folder copies), and master merging — including resolving conflicts without
fear.

## Prerequisites
[Lesson 04 — Your First Repo](04-your-first-repo.md)

## After This Lesson You Will Be Able To
- Create, switch, and delete branches confidently
- Explain fast-forward vs three-way merges
- Resolve a merge conflict from scratch, calmly
- Read `git log --graph` output for multi-branch history

---

## What a Branch Actually Is

Recall from Lesson 03: a commit is an object with a hash, pointing to a
parent. A **branch is nothing but a text file containing one commit
hash** — literally, `.git/refs/heads/main` contains a single line: a SHA.

```bash
cat .git/refs/heads/main
# 3f2a91c8e4b7d6a5f0e9c8b7a6f5e4d3c2b1a0f9
```

That's it. That's a branch. When you commit, Git makes a new commit object
and **updates that text file** to point to the new commit's hash. This is
why creating a branch is instantaneous regardless of project size — it's
one small file write, not a folder copy.

```
Before:                          After committing on main:

A ← B ← C                        A ← B ← C ← D
        ↑                                    ↑
     main, HEAD                           main, HEAD
```

`HEAD` (Lesson 02) usually points *at a branch*, and the branch points at
a commit — an extra layer of indirection that's exactly what makes
"switch branches" and "make a new commit" two cleanly separate operations.

```
HEAD → main → commit C
```

---

## Creating and Switching Branches

```bash
git branch feature-login          # create a new branch pointing at current commit — does NOT switch to it
git switch feature-login          # switch to it (modern command)
# — or, the classic all-in-one —
git checkout -b feature-login     # create AND switch in one step (older, still everywhere in the wild)

git branch                        # list all local branches, * marks current
git branch -a                     # include remote-tracking branches too
git branch -d feature-login       # delete a branch (safe — refuses if unmerged commits would be lost)
git branch -D feature-login       # force delete (dangerous — use if you're sure)
```

`git switch` and `git restore` were introduced in Git 2.23 specifically to
un-confuse `git checkout`, which historically did *both* "switch branches"
and "discard file changes" — two very different, easily-confused
operations. Prefer `switch`/`restore` going forward; know `checkout` exists
because most tutorials and older teammates still use it.

Right after creating a branch:

```
A ← B ← C
        ↑ ↖
      main  feature-login
        ↑
      HEAD (still on main, until you switch)
```

After `git switch feature-login` and one commit:

```
A ← B ← C ← D           ← feature-login, HEAD
        ↑
      main
```

`main` didn't move. `feature-login` advanced. This is the entire point:
**branches let multiple lines of work exist independently, in the same
repo, at the same time**, without interfering with each other.

---

## Merging: Bringing Branches Back Together

```bash
git switch main                 # go to the branch you want to merge INTO
git merge feature-login         # bring feature-login's commits into main
```

There are two fundamentally different outcomes, and understanding *why*
each happens is the key to demystifying merges:

### Fast-Forward Merge

If `main` hasn't moved *at all* since you branched off it, merging is
trivial: Git just slides `main`'s pointer forward to match
`feature-login`. No new commit is created.

```
Before:              A ← B ← C ← D    ← feature-login
                              ↑
                            main

After (fast-forward): A ← B ← C ← D    ← feature-login, main
```

### Three-Way Merge

If `main` *also* got new commits while you were working on your branch
(very common — teammates push things, or you fixed a hotfix directly on
main), Git can't just slide the pointer — the histories diverged. Git
creates a new **merge commit** with **two parents**, combining both
lines:

```
        E ← F                ← main  (main moved forward too)
       /     \
A ← B ← C     M               ← main, HEAD (after merge)
       \     /
        D ← ... 
              (feature-login had its own commits)

Actually, drawn more precisely:

A ← B ─┬─ C ← D               ← feature-login
       └─ E ← F ← M           ← main (M = merge commit, two parents: F and D)
```

Git compares the two branch tips against their **common ancestor** (here,
commit B) and combines the changes from both sides automatically — as
long as they don't touch the same lines.

---

## Merge Conflicts: What They Are and How to Resolve Them

A conflict happens when **both branches changed the same lines of the same
file** in different ways — Git genuinely cannot guess which version you
want, so it stops and asks you.

```bash
git switch main
git merge feature-login
# Auto-merging config.py
# CONFLICT (content): Merge conflict in config.py
# Automatic merge failed; fix conflicts and then commit the result.
```

Open the conflicted file. Git has inserted conflict markers directly into
it:

```python
<<<<<<< HEAD
TIMEOUT = 30
=======
TIMEOUT = 60
>>>>>>> feature-login
```

- Everything between `<<<<<<< HEAD` and `=======` is **your current
  branch's** version (main, in this case).
- Everything between `=======` and `>>>>>>> feature-login` is the
  **incoming branch's** version.

Resolution steps:

```bash
# 1. Edit the file by hand — delete the markers, keep whichever code
#    (or a combination) is actually correct:
TIMEOUT = 60   # decided the higher timeout was right

# 2. Tell Git you resolved it:
git add config.py

# 3. Check if there are more conflicted files:
git status

# 4. Once all conflicts are resolved and staged, complete the merge:
git commit
# (Git pre-fills a merge commit message for you — usually fine as-is)
```

**Abort at any point** if it's going wrong and you want to start over:

```bash
git merge --abort   # cancels the merge, returns everything to pre-merge state
```

This is completely safe — nothing is lost, you're just backing out.

---

## Conflict-Reading Tip: Use a Merge Tool

Raw conflict markers are fine for small conflicts. For bigger ones,
configure a visual merge tool once:

```bash
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
git mergetool
```

Most IDEs (VS Code, PyCharm) also detect conflict markers automatically
and show inline "Accept Current / Accept Incoming / Accept Both" buttons —
use them, there's no shame in it.

---

## Merge Strategies & Useful Merge Flags

### Finding the Common Ancestor Yourself

Remember commit B from the three-way merge diagram above — the point
where the two branches split? Git finds that automatically, but you can
ask for it directly, which is genuinely useful when you want to see
"everything that happened on this branch" without noise from `main`:

```bash
git merge-base main feature-login
# → b2c3d4e...   (the exact commit both branches share as their last common point)

git log b2c3d4e..feature-login --oneline    # only feature-login's own commits
```

### Controlling Whether a Merge Commit Gets Created

```bash
git merge --ff-only feature-login   # succeed ONLY if a fast-forward is possible;
                                      # otherwise abort with no changes made at all —
                                      # great in scripts/CI where a surprise merge
                                      # commit would be unwanted
git merge --no-ff feature-login     # force a real merge commit EVEN when a
                                      # fast-forward would have worked — keeps a
                                      # visible record that "this was a feature
                                      # branch," which some teams prefer on `main`
```

### Conflict Shortcuts: `-X ours` / `-X theirs`

```bash
git merge -X ours feature-login     # on every conflicting hunk, silently keep
                                      # YOUR side's version
git merge -X theirs feature-login   # on every conflicting hunk, silently keep
                                      # THEIR side's version
```

These still perform a real merge (non-conflicting changes from both sides
are combined normally) — they only decide the winner *when* there's a
conflict. This is different from `git merge -s ours`, which discards the
other branch's changes entirely and just records that it was merged
without integrating any of its content. `-X ours`/`-X theirs` are rarely
what you actually want for real code (you're silently throwing away a
teammate's logic), but they're the right call for machine-generated files
that regenerate cleanly anyway, like a lockfile you're about to
regenerate right after the merge regardless.

### Which Strategy Is Actually Running?

Since Git 2.33, the default merge strategy for a normal two-branch merge
is **`ort`** (replacing the older **`recursive`**, still selectable with
`git merge -s recursive` if you ever need it for compatibility). You
don't need to think about this day to day — `ort` is just a faster,
more correct rewrite of the same idea. One strategy you might actually
reach for deliberately is **octopus**, which merges *more than two*
branches into a single merge commit in one shot (`git merge b1 b2 b3`) —
rare, but occasionally used when cutting a release branch that combines
several already-independent feature branches at once.

---

## Visualizing Branch History

```bash
git log --oneline --graph --all --decorate
```

```
* 4d5e6f (HEAD -> main) Merge branch 'feature-login'
|\
| * 3c4d5e (feature-login) Add password reset flow
| * 2b3c4d Add login form validation
* | 1a2b3c Hotfix: patch XSS in comment field
|/
* 0f1e2d Initial commit
```

Read this bottom to top: one shared ancestor, two branches diverging, one
merge commit reuniting them. This exact command is what you'll run
constantly to *see* the shape of history instead of guessing.

---

## Naming and Branching Strategy (Preview)

You'll see full team workflows in Lesson 08, but as a habit starting now:

```bash
git switch -c feature/add-retry-logic     # feature/ prefix
git switch -c fix/null-pointer-login       # fix/ or bugfix/ prefix
git switch -c chore/upgrade-deps           # chore/ for maintenance
```

Never do serious work directly on `main`. Always branch first — branches
are free, so there's no reason not to.

---

## Exercise

```bash
mkdir branch-practice && cd branch-practice && git init
echo "v1" > file.txt && git add . && git commit -m "Initial commit"
```

1. Create branch `feature-a`, edit `file.txt` to say `v1 + feature A`,
   commit.
2. Switch back to `main`, create branch `feature-b`, edit `file.txt` to
   say `v1 + feature B` (touching the *same line*), commit.
3. Switch to `main` and merge `feature-a` first — confirm it's a clean
   fast-forward (`git log --graph` will show a straight line).
4. Now merge `feature-b` into `main` — this **will** conflict, since both
   branches edited the same line. Resolve it by hand, keeping both
   changes combined into one sensible line. Commit the merge.
5. Run `git log --oneline --graph --all` and identify the merge commit and
   its two parents.

Continue to [Lesson 06 — Undoing Things](06-undoing-things.md) — arguably
the most valuable lesson for building real confidence with Git.
