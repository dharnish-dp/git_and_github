# Lesson 12 — Submodules, Subtrees & Large-Repo Tooling

## Goal
Learn how to embed one Git repository inside another when you genuinely
need to (shared test libraries, shared firmware/hardware code), understand
why submodules have a well-earned reputation for confusing teams, and pick
up the tools (`sparse-checkout`, shallow/partial clone) that keep large
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

---

## The Problem: One Repo Needs Code From Another Repo

Picture this, as an SDET/automation engineer: you maintain a shared
`test-utils` library — device fixtures, retry helpers, a mock hardware
interface — and three separate project repos all need it. You have three
real options:

```
1. Copy-paste the code into each repo.
   → Works today. Breaks the moment test-utils gets a bugfix, because
     now there are 3 copies to update by hand, and they will drift.

2. Publish test-utils as a real installable package (PyPI, a private
   package index, an internal artifact repo) and `pip install` it like
   any other dependency.
   → USUALLY THE RIGHT ANSWER. Versioned, upgradeable, no Git trickery.

3. Embed the other repo's actual Git history directly into yours, via
   a submodule or a subtree.
   → For when option 2 isn't feasible: the shared code isn't a clean
     installable package, or you need to make coordinated commits that
     touch both repos as part of the same piece of work.
```

This lesson is entirely about option 3 — but keep option 2 in your back
pocket. Most of the pain people associate with Git submodules is really
the pain of using option 3 in situations where option 2 would have been
simpler.

---

## Git Submodules — A Repo Inside a Repo

The core idea: **a submodule is a commit pointer, not a copy of files.**
Your parent repo doesn't store `test-utils`'s files in its own object
database (Lesson 03) — it stores one thing: "at this path, use `test-utils`
pinned to exactly this commit hash."

```bash
git submodule add https://github.com/org/shared-test-utils.git libs/shared-test-utils
git commit -m "Add shared-test-utils as a submodule, pinned to current main"
```

Two things just happened:

1. Git created (or updated) a `.gitmodules` file in your repo root,
   recording the submodule's path and URL — this file IS a normal
   tracked file, committed like any other.
2. Git added a special **gitlink** entry to your tree (Lesson 03) at
   `libs/shared-test-utils` — not a blob, not a tree, but a raw commit
   hash. Run `git ls-tree HEAD libs/` and you'll see something like:

```
160000 commit a1b2c3d4e5f6...    shared-test-utils
```

`160000` is the special mode for "this is a gitlink, not a file or
folder." Your parent repo's own history only ever changes when that one
line changes — it has no idea what's *inside* `test-utils` beyond that
one pinned hash.

### Trap #1: Plain `git clone` Leaves Submodules Empty

Cloning a repo that has submodules does **not** automatically fetch
their content — you get an empty folder at `libs/shared-test-utils`
unless you ask for it explicitly:

```bash
git clone --recurse-submodules https://github.com/org/main-repo.git

# already cloned without it? fix it after the fact:
git submodule update --init --recursive
```

This trips up nearly everyone the first time. If a teammate says "the
`libs/` folder is just empty for me," this is almost always the cause.

### Trap #2: Submodules Start in Detached HEAD

When you (or `submodule update`) check out a submodule, it lands in
**detached HEAD** (Lesson 10) — `HEAD` points straight at the pinned
commit, not at a branch. If you start editing and committing inside the
submodule without switching to a branch first, those new commits have no
home and are easy to lose track of.

```bash
cd libs/shared-test-utils
git switch main            # submodules start detached — switch to a real branch before committing
git pull
cd ../..
git add libs/shared-test-utils     # stages the UPDATED pointer, not any files
git commit -m "Bump shared-test-utils to latest main"
```

**Critical:** everyone else who already has this parent repo checked out
still sees the *old* pinned commit — even after they `git pull` the
parent repo itself — until they separately run `git submodule update`.
This is deliberate (it's what makes builds reproducible: nobody's
submodule silently drifts to a newer commit behind their back), but it's
the single most common source of "why isn't my submodule updating?"
confusion. The fix is always the same two-step: pull the parent, then
`git submodule update --init --recursive`.

---

## Git Subtree — The "Just Merge It In" Alternative

`git subtree` solves the same problem completely differently: it actually
**copies** the other repo's files and history into yours as ordinary
tracked files. No `.gitmodules`, no gitlink, no separate checkout state,
no detached-HEAD gotcha. Anyone who clones your repo with a plain
`git clone` — no special flags at all — gets everything immediately.

```bash
git subtree add --prefix=libs/shared-test-utils \
    https://github.com/org/shared-test-utils.git main --squash
```

`--squash` collapses `test-utils`'s entire history into a single commit
when importing it, so your repo's `git log` doesn't balloon with someone
else's history. Omit it only if you genuinely want the full imported
history to appear commit-by-commit in your own log.

Pulling upstream updates later:

```bash
git subtree pull --prefix=libs/shared-test-utils \
    https://github.com/org/shared-test-utils.git main --squash
```

Pushing local changes back upstream — yes, this is possible, unlike a
plain copy-paste:

```bash
git subtree push --prefix=libs/shared-test-utils \
    https://github.com/org/shared-test-utils.git a-new-branch-to-open-a-pr-from
```

### Submodule vs Subtree, Honestly

| | Submodule | Subtree |
|---|---|---|
| What's stored | A pointer (pinned commit hash) | Actual copied files + history |
| Clone experience | Empty folder unless `--recurse-submodules` | Works immediately, no flags |
| "What version are we on?" | Trivial — it's the pinned commit | No built-in answer; you'd have to dig |
| Repo size | Small (just a pointer) | Grows — you're storing real content |
| Contributing back upstream | Normal — it's just a regular clone at that path | Works via `subtree push`, but history gets murkier the more back-and-forth happens |
| Team confusion factor | High — detached HEAD, empty-clone traps | Low for consumers, but the *merge history* can get messy for maintainers |

---

## When to Prefer Neither

Be honest with yourself the same way you were about rebase's golden rule
in Lesson 09: if you find yourself fighting submodules or subtrees
constantly — teammates forgetting `--recurse-submodules`, endless "why
didn't my submodule update" questions, subtree merges producing baffling
diffs — that is usually a sign the *architecture* is wrong, not that
you're missing some clever flag.

Two real fixes, in order of preference:

1. **Actually publish the shared code as a versioned package** (a PyPI
   package, an internal package index, a private registry) and depend on
   it the normal way. This is almost always cleaner than embedding one
   Git repo inside another.
2. **If the code always changes together with the code that consumes
   it**, that's a sign it should live in the *same* repo — a small,
   deliberate monorepo for those specific pieces — rather than two repos
   glued together with submodules or subtrees.

Most large tech orgs avoid submodules for exactly this reason: they
either invest in real package management, or go fully monorepo (see
[What Next](../what-next.md) for more on monorepo tooling). Submodules
and subtrees are tools worth knowing well — you'll meet them in the
wild — but reaching for them by default is rarely the top-1% move.

---

## Working Efficiently in a Large Repo

Separate problem, same neighborhood: once a repo gets large — many
files, deep history, or both (very common in a real monorepo, or any
old codebase) — a plain `git clone` and full checkout can get slow even
if you only ever touch one small corner of it. Three independent tools,
often combined:

**Shallow clone** — only recent history, not the full commit graph:

```bash
git clone --depth 1 <url>       # just the latest commit, fast
git fetch --deepen=50           # need more history later? pull 50 more commits
git fetch --unshallow           # or just get the rest of it, fully
```

Trade-off: commands that walk history — `git log`, `git blame`,
`git bisect` (Lesson 10) — have nothing to walk past what you fetched.
Fine for a quick build or CI checkout; not fine as your daily-driver
clone.

**Partial clone** — full commit history/metadata, but file *content* is
fetched lazily, only when you actually check out or touch that file:

```bash
git clone --filter=blob:none <url>
```

This is the better middle ground for day-to-day work: real history for
`log`/`blame`/`bisect`, without downloading every historical version of
every file up front.

**Sparse-checkout** — for when you only work in one subfolder of a huge
repo, and don't want the rest cluttering your working directory at all:

```bash
git clone --filter=blob:none --no-checkout <url>
cd repo
git sparse-checkout init --cone     # cone mode: modern, fast, recommended
git sparse-checkout set services/my-service libs/shared-test-utils
git checkout main
```

`--cone` mode checks out only the folders you list (plus their parent
folders) — the full commit history is still there underneath, so
`git log`, `blame`, and branch switching all still work correctly; you've
only trimmed what's actually materialized on disk. Pairing sparse-checkout
with partial clone gives you both a light working directory *and* a light
initial transfer — the standard combination for working comfortably in a
genuinely large monorepo.

---

## Exercise

```bash
mkdir submodule-practice && cd submodule-practice
mkdir shared-lib && cd shared-lib && git init
echo "def helper(): return 42" > helper.py
git add . && git commit -m "Initial shared-lib commit"
cd ..
mkdir main-project && cd main-project && git init
echo "readme" > README.md && git add . && git commit -m "Initial main-project commit"
```

1. Add `../shared-lib` as a submodule of `main-project` at
   `libs/shared-lib`, commit it, then inspect `.gitmodules` and run
   `git ls-tree HEAD libs/` to see the gitlink entry (mode `160000`) for
   yourself.
2. Clone `main-project` into a fresh folder **without**
   `--recurse-submodules`. Confirm `libs/shared-lib` is empty. Then run
   `git submodule update --init` inside it and confirm it populates.
3. Inside that submodule checkout, confirm you're in detached HEAD
   (`git status` will say so), `git switch main`, make a small edit,
   commit it, `cd` back up to `main-project`, and commit the updated
   submodule pointer.
4. Remove the submodule and instead add the same `shared-lib` via
   `git subtree add --prefix=libs/shared-lib ../shared-lib main --squash`.
   Clone this new version of `main-project` into yet another fresh
   folder with a plain `git clone` — no extra flags — and confirm
   `libs/shared-lib/helper.py` is already there with zero extra commands.
5. (Optional) If you have internet access, try
   `git clone --filter=blob:none --depth 50 <some large public GitHub repo URL>`
   and time it against a full `git clone` of the same repo to feel the
   difference directly.

Continue to [Lesson 13 — Pro Workflows & Best Practices](13-pro-workflows-and-best-practices.md).
