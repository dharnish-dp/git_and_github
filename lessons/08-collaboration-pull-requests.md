# Lesson 08 — Collaboration: Forks, Pull Requests & Code Review

## Goal
Learn how real teams and open-source projects use GitHub day to day:
forking, branching, opening Pull Requests, and reviewing code. You will
use this workflow in every job.

## Prerequisites
[Lesson 07 — Remotes & GitHub Basics](07-remotes-and-github-basics.md)

Quick refresher of words from earlier lessons (you need them here):

- **Repository (repo)** = a project folder plus its full saved history.
- **Commit** = a saved snapshot of your project at one moment.
- **Branch** = a named line of work. It is a label pointing at a commit.
- **Remote** = a copy of your repo that lives somewhere else (usually GitHub).
- **origin** = the default nickname for the remote you cloned from.
- **Push** = upload your new commits to a remote.
- **Fetch** = download new commits from a remote, without changing your files.
- **Merge** = combine the work of one branch into another branch.

## After This Lesson You Will Be Able To
- Contribute to any open-source project via fork + Pull Request
- Contribute to a team repo you have direct write access to
- Review a teammate's PR usefully
- Keep your branch up to date with `main` cleanly

---

## 1. The Big Idea: Why Pull Requests Exist

**Why do we need this?** On a team, many people change the same code.
If everyone pushed straight into `main` (the branch that holds the
"official" version), one bad change would break everyone.

**Analogy (Python / test automation).** Think of a CI pipeline for your
test framework. Nobody edits the shared framework directly on the
server. You write a change on your own, someone reviews it, tests run,
and only then does it go into the shared code. A Pull Request is that
gate.

**Plain explanation.** A **Pull Request (PR)** is a page on GitHub where
you say: "I made these commits on my branch. Please review them, and if
they look good, pull them into `main`."

The name is confusing, so here is the doubt answered up front.

> **Doubt: "Why is it called a *pull* request? I am pushing my code."**
> You *push* your branch to GitHub. That is step one and it needs no
> permission. Then you *request* that the project owners *pull* (take)
> your changes into their branch. So "pull request" means "please pull my
> work in." GitLab calls the same thing a "Merge Request", which is a
> clearer name.

A PR is **not** a Git feature. It is a **GitHub feature** built on top of
Git. Git itself only knows about commits and branches. The PR page, the
comments, and the merge button all live on GitHub.

---

## 2. Two Collaboration Models

**Why do we need two?** It depends on one question: *do you have
permission to push to the project's repo?*

- If **yes** (you are on the team), you push a branch straight to it.
- If **no** (a stranger on the internet), you cannot. You make your own
  copy and ask them to accept changes from it.

**Analogy.** A shared office whiteboard. Colleagues can write on it
directly (shared-repo). A visitor cannot touch it, so they photocopy it,
write on the copy, and hand the copy to the owner (fork).

| | **Fork-based** | **Shared-repo (direct branch)** |
|---|---|---|
| When | Open source, or you have no write access | Internal team repos where you are a collaborator |
| You push to | Your own copy (the fork) on GitHub | The shared repo itself, on a feature branch |
| PR opened | From your fork's branch to the original repo's `main` | From your feature branch to `main`, same repo |

Most companies use shared-repo internally. Open source uses fork-based.
Both end with the same final step: a **Pull Request**.

> **Common confusion: "Fork vs clone vs branch?"**
> - **Branch** = a separate line of work *inside one repo*.
> - **Clone** = download a repo from GitHub to your computer (Lesson 07).
> - **Fork** = make a *new copy of the repo on GitHub, under your account*.
>
> A fork happens on GitHub's servers. A clone happens on your laptop.
> You usually do both: fork first, then clone your fork.

---

## 3. Fork-Based Workflow (Open Source Style)

We will build this in small steps. The target project is
`original-owner/project`. Your GitHub username is `yourname`.

### Step 1: Fork on GitHub

On the project's GitHub page, click **Fork** (top right), then **Create
fork**. GitHub makes a complete copy at `github.com/yourname/project`.

That copy is yours. You can push to it freely. The original is
untouched.

### Step 2: Clone YOUR fork

**Why?** You need the code on your laptop to edit it. Clone *your* fork
because you can push to it. Cloning the original would leave you unable
to push.

```bash
git clone git@github.com:yourname/project.git
cd project
```

Expected output:

```
Cloning into 'project'...
remote: Enumerating objects: 312, done.
remote: Counting objects: 100% (312/312), done.
Receiving objects: 100% (312/312), 85.40 KiB | 2.10 MiB/s, done.
Resolving deltas: 100% (140/140), done.
```

Line by line:
- `Cloning into 'project'...` — Git is creating a folder named `project`.
- `Enumerating / Counting objects` — GitHub is counting the saved items
  (commits, files) to send you.
- `Receiving objects` — the download itself (312 items, 85 KiB).
- `Resolving deltas` — Git is rebuilding files stored as "differences",
  a space-saving trick. You do not need to act on it.

Git automatically names your fork `origin`. Check:

```bash
git remote -v
```

```
origin  git@github.com:yourname/project.git (fetch)
origin  git@github.com:yourname/project.git (push)
```

Each remote shows twice: one URL for downloading (fetch), one for
uploading (push). They are normally the same.

### Step 3: Add the ORIGINAL repo as a second remote called `upstream`

**Why do we need this?** After you fork, your copy is frozen in time. The
original project keeps getting new commits from other people. Your fork
does not receive them automatically. You need a second bookmark that
points at the original, so you can download its new work.

**Analogy.** Your fork is a photocopy. `upstream` is the address of the
original document, so you can re-copy the updates.

`upstream` is just a *convention* (a widely used nickname). Git does not
treat it specially. You could call it `original`, but everyone says
`upstream`.

```bash
git remote add upstream git@github.com:original-owner/project.git
git remote -v
```

```
origin    git@github.com:yourname/project.git (fetch)
origin    git@github.com:yourname/project.git (push)
upstream  git@github.com:original-owner/project.git (fetch)
upstream  git@github.com:original-owner/project.git (push)
```

- `origin` = **your fork**. You read from it and write to it.
- `upstream` = **the original repo**. You normally only read from it. You
  probably cannot push there.

`git remote add` prints nothing when it works. No news is good news.

### Step 4: Create a branch and make your change

**Why a branch?** Keep `main` clean. One branch per change makes one
clean PR.

```bash
git switch -c fix/typo-in-readme
```

```
Switched to a new branch 'fix/typo-in-readme'
```

`switch -c` means "create a branch with this name, and move onto it."

Edit a file (for example fix a typo in `README.md`), then:

```bash
git add .
git commit -m "Fix typo in README"
```

- `git add .` = put all changed files in the **staging area** (the
  "ready to be saved" box).
- `git commit -m "..."` = save a snapshot with that message.

Expected output:

```
[fix/typo-in-readme 3f9a1c2] Fix typo in README
 1 file changed, 1 insertion(+), 1 deletion(-)
```

- `fix/typo-in-readme` = the branch you committed on.
- `3f9a1c2` = the short ID (hash) of your new commit.
- `1 insertion(+), 1 deletion(-)` = one line replaced (one removed, one
  added).

### Step 5: Push to YOUR fork (origin), not upstream

**Why origin?** You almost certainly lack write permission on upstream.
Pushing there would be rejected.

```bash
git push -u origin fix/typo-in-readme
```

Expected output:

```
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Writing objects: 100% (3/3), 310 bytes | 310.00 KiB/s, done.
remote: Create a pull request for 'fix/typo-in-readme' on GitHub by visiting:
remote:      https://github.com/yourname/project/pull/new/fix/typo-in-readme
To github.com:yourname/project.git
 * [new branch]      fix/typo-in-readme -> fix/typo-in-readme
branch 'fix/typo-in-readme' set up to track 'origin/fix/typo-in-readme'.
```

Line by line:
- `Writing objects` — uploading your new commit.
- `remote: Create a pull request...` — GitHub itself prints a link to
  start your PR. Handy.
- `* [new branch] A -> A` — the branch did not exist on GitHub; now it does.
- `set up to track` — because of `-u`, your local branch now remembers
  its GitHub twin. Next time, plain `git push` is enough.

> **Doubt: "What does `-u` mean?"** It stands for "set upstream"
> (here "upstream" means "the remote branch to track", a different use of
> the word than the `upstream` remote above; yes, the term is overloaded).
> You only need `-u` the first time you push a new branch.

### Step 6: Open the Pull Request on GitHub

Open the link from the push output (or go to the original repo and click
**Compare & pull request**). Check the direction:

```
base repository: original-owner/project   base: main
head repository: yourname/project         compare: fix/typo-in-readme
```

- **base** = where the changes should go (the destination).
- **head / compare** = where the changes come from (your branch).

Think `base <- head`: "merge head into base." Then write a title and
description, and click **Create pull request**.

### Diagram legend (used in every lesson)

```
A---B---C       commits; oldest on the LEFT, newest on the RIGHT
main            a branch label; written at the END of its row
HEAD -> main    HEAD is a separate label: "you are here, on main"
D*              asterisk = commit created by the command just run
A'              prime = a copy of A with a new ID (rebase)
--->            things moving: a download, an upload, a request
```

Note: internally Git stores a link from each commit back to its parent.
We draw time flowing left to right instead, because it is easier to read.

### Whole fork flow at a glance

**How to read this:** three places (columns), five numbered actions
(rows, top to bottom). Each arrow is labelled with the command (or
button) and points to where the data goes.

```
  UPSTREAM              YOUR FORK             YOUR LAPTOP
  original-owner/       yourname/             ~/project
  project               project               (local repo)
  (GitHub)              (GitHub)
  remote: upstream      remote: origin
  |                     |                     |
  |-- fork ------------>|                     |   (1) Fork button
  |                     |                     |
  |                     |-- git clone -------->   (2) copy to laptop
  |                     |                     |
  |                     |<-- git push --------|   (3) upload your branch
  |                     |                     |
  |<-- PR --------------|                     |   (4) open Pull Request
  |                     |                     |
  |-- git fetch upstream --------------------->   (5) get new upstream work
  |                     |                     |
```

What each arrow means:
- (1) `fork` and (4) `PR` happen on the GitHub website, not in a terminal.
- (2) `clone` and (3) `push` go between GitHub and your laptop.
- (3) you push to **your fork** (`origin`), never to upstream.
- (5) `fetch upstream` goes straight from the original to your laptop. It
  skips your fork, which is why you later also push `main` to `origin`.

### Keeping your fork in sync with upstream

**Why do we need this?** While your PR waits, the original project
changes. If you start new work from an old copy, you will hit conflicts
(two edits to the same lines). So refresh regularly.

```bash
git fetch upstream
git switch main
git merge upstream/main
git push origin main
```

Step by step:

1. `git fetch upstream` — download the original's new commits. Safe:
   it does **not** touch your files or branches.
2. `git switch main` — move to your local `main`.
3. `git merge upstream/main` — bring the original's `main` into yours.
   `upstream/main` is Git's local memory of where the original's `main`
   was at your last fetch.
4. `git push origin main` — update your fork's `main` on GitHub.

Example output of step 1 and 3:

```
remote: Enumerating objects: 9, done.
From github.com:original-owner/project
 * [new branch]      main       -> upstream/main
```
```
Updating 3a1b2c3..9d8e7f6
Fast-forward
 README.md | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

`Fast-forward` means your `main` had no extra commits of its own, so Git
just moved the label forward. No merge commit was needed.

**How to read this:** two rows of history, your local `main` and Git's
memory of the original's `main` (`upstream/main`), before and after.

BEFORE (after `git fetch upstream`, you are on `main`):

```
      A---B---C                 main
              ^
            HEAD -> main
      A---B---C---D---E         upstream/main
```

Command: `git merge upstream/main`

AFTER:

```
      A---B---C---D---E         main
                      ^
                    HEAD -> main
      A---B---C---D---E         upstream/main
```

What changed: only the `main` label moved, from C to E (a fast-forward).
D and E were already downloaded by the fetch; no new commit was created.
A, B, C are unchanged. (You could also
use `git rebase upstream/main`; see Lesson 09.)

> **Common confusion: "`fetch` vs `merge` vs `pull`?"**
> `fetch` downloads. `merge` combines. `git pull` = fetch + merge in one
> step. In this lesson we type them separately so each idea is visible.

---

## 4. Shared-Repo Workflow (Typical Team/Company Style)

**Why is it shorter?** You have write access, so no copy is needed.

```bash
git clone git@github.com:company/project.git
cd project
git switch -c feature/add-search-filter
# ... edit files, commit ...
git push -u origin feature/add-search-filter
```

Then on GitHub, open a PR from `feature/add-search-filter` to `main`,
same repo.

The only structural difference: **one remote (`origin`) instead of
two**, since `origin` is the company repo itself.

> **Doubt: "If I can push to the company repo, can I break `main`?"**
> Yes, unless the team adds **branch protection** (a GitHub rule that
> forces PRs, reviews, and passing tests before merging to `main`;
> Lesson 11). That is why you work on a feature branch and never commit
> to `main` directly.

---

## 5. Opening a Good Pull Request

**How to read this:** the life of one PR, top to bottom. The loop at
step 3 repeats until reviewers are happy.

```
  [1] git push your feature branch
        |
        v
  [2] Open the PR     (base: main  <-  head: your branch)
        |
        v
  [3] Review + CI checks  <-------------------------+
        |                                           |
        v                                           |
      changes requested? --- yes ---> edit, commit, git push
        | no                          (same branch; PR updates itself)
        v
  [4] Approved and checks green
        |
        v
  [5] Press merge (merge commit, squash, or rebase; see section 7)
        |
        v
  [6] Delete the branch; locally: git switch main, then git pull
```

**What a PR page gives you:**

- A **diff view**: every changed line, file by file. (A **diff** = a
  before/after comparison, with removed lines in red and added in green.)
- A place for **discussion and inline comments** on specific lines.
- **Status checks**: automatic test results from CI (Continuous
  Integration = a robot that runs your tests on every push; Lesson 11).
- A **merge button**, usable once approved and checks pass.

**What makes a PR easy to review and merge fast:**

1. **Keep it small and focused.** One logical change per PR. A 20-line PR
   is reviewed in 5 minutes. A 2,000-line PR waits a week.
2. **Write a real description.** Say what changed, why, and how you
   tested it. Many repos have a **PR template** (a pre-filled checklist
   that appears in the description box). Fill it in.
3. **Link the issue.** An **issue** is a GitHub ticket (a bug or task).
   Writing `Closes #142` in the description links issue 142 and makes
   GitHub close it automatically when the PR merges.
4. **Review your own diff first.** Read it on GitHub before asking others.
   You will catch half your own mistakes.
5. **Reply to every comment**, even with "Done, fixed in latest commit."
   Otherwise reviewers wonder if you saw it.

A sample description:

```
## What
Add a search filter to the users page.

## Why
Users could not find accounts in long lists (see #142).

## How I tested
- Added 3 pytest cases for empty, partial, and no-match queries.
- Ran the full suite locally: 128 passed.

Closes #142
```

---

## 6. Reviewing Someone Else's PR

**Why review?** A second pair of eyes finds bugs, teaches the team, and
keeps code consistent. (You already do this as an SDET when you question
a test.)

On the PR's **Files changed** tab:

- Click a line number (or the blue `+` beside it) to leave an **inline
  comment** on exactly that line.
- Click **Start a review** instead of "Add single comment". Your comments
  are held privately as pending and sent together as one review. This
  avoids spamming the author with a notification per comment.
- When done, click **Review changes** and choose one of three verdicts:

| Verdict | Meaning |
|---|---|
| **Comment** | Feedback only. No approval or rejection. |
| **Approve** | Looks good; ready to merge. |
| **Request changes** | Blocks merging until addressed (if branch protection requires it; Lesson 11). |

**What to look for** (beyond "does it work"):

- Does it match the codebase's existing patterns and conventions?
- Are there tests, and do they cover real risk, not only the happy path?
- Will the next reader understand it in six months?
- Any security concerns (input validation, leaked secrets, injection)?

Good review comments are **specific and kind**. "This will throw if
`user` is `None`. Should we guard it?" beats "this is wrong."

---

## 7. Merge Strategies on GitHub

**Why does this matter?** When a PR is approved, you press the green
button. GitHub offers three ways to merge. Each leaves a different
history on `main`, and history is what you will read for years when
debugging.

Set up a tiny example. Your PR has three commits, A, B, C (messy: "wip",
"fix typo", "actually fix"). Meanwhile `main` has gained commit M.

**How to read this:** `main` is the bottom row, `feature` is the row
above it. The feature branch split off at L; `main` then moved on to M.

BEFORE (the same starting point for all three strategies):

```
            A---B---C        feature
           /
      K---L---M              main
```

### 7a. Create a merge commit

Keeps every commit, and adds one extra commit that joins the two lines.

Button: **Create a merge commit**

AFTER:

```
            A---B---C        feature
           /         \
      K---L---M-------X*     main      X = merge commit
```

What changed: `main` moved from M to the new commit X, which has **two
parents**: M and C. A, B, C now also belong to `main`'s history, with
their original IDs. The `feature` label did not move.

- Preserves exact history. Default option.
- `git log --oneline --graph` shows a visible "braid".

### 7b. Squash and merge

Combines ALL PR commits into **one new commit** on `main`.

Button: **Squash and merge**

AFTER:

```
            A---B---C        feature   (unchanged, now unused)
           /
      K---L---M---S*         main      S = A+B+C combined
```

What changed: `main` moved from M to S, a brand-new commit with ONE
parent (M). It holds the final code of A+B+C together. A, B, C are **not**
part of `main`; only `feature` (and the PR page) still points at them.
There is no line joining `feature` to `main`: Git sees them as unrelated.

After you delete the `feature` branch, the picture on `main` is simply:

```
      K---L---M---S          main
```

- Your messy "wip" commits disappear from `main`. One commit per feature.
- Many teams default to this. It does not matter how messy your working
  commits were.
- Downside: individual commits are lost from `main` (the branch still
  shows them on the PR page).

### 7c. Rebase and merge

Replays each commit (A, B, C) one by one on top of `main`, with new IDs,
and no merge commit.

Button: **Rebase and merge**

AFTER:

```
            A---B---C                feature   (old copies, now unused)
           /
      K---L---M---A'*-B'*-C'*        main      A' = copy of A, etc.
```

What changed: `main` moved from M to C'. Three new commits (A', B', C')
were created, each with one parent, in a straight line. They have the
same code changes and messages as A, B, C but **different IDs**. The
originals A, B, C are untouched but no longer on `main`.

- Keeps individual commits but gives a perfectly straight (linear)
  history.
- The commits are *copies*, so their hashes change. We cover rebase
  mechanics fully in Lesson 09.

| Strategy | Result on `main` | Use when |
|---|---|---|
| **Create a merge commit** | All commits + one join commit | You want the exact history |
| **Squash and merge** | One commit per PR | Messy "wip" commits; want clean history |
| **Rebase and merge** | All commits, straight line, no join | You want each commit kept and a linear history |

> **Common confusion: "Does squash delete my work?"**
> No. The code changes are all there, inside the one squashed commit.
> Only the *separate commit steps* are collapsed.

> **Doubt: "After a squash merge, Git says my branch is behind/diverged."**
> Correct. Squash makes a brand-new commit S that Git does not see as the
> same as A, B, C. After merging a PR, switch to `main`, pull, and delete
> the old feature branch. Start the next change from a fresh branch.

### 7d. Squash merge vs normal merge: what is the difference and the benefit?

This is the most common doubt, so here is a direct comparison. Both end
with the **same files** on `main`. Only the **history** is different.

Say the feature branch has 4 commits: C="wip", D="fix typo", E="oops",
F="final".

```
NORMAL MERGE (merge commit)

      A---B-----------M      main      M = merge commit, 2 parents
           \         /
            C---D---E---F    feature

  main's history now contains C, D, E, F and M.


SQUASH MERGE

      A---B---S*             main      S = C+D+E+F combined, 1 parent
           \
            C---D---E---F    feature   (still exists, NOT part of main)

  main's history contains only S.
```

| | Normal merge | Squash merge |
|---|---|---|
| Commits added to `main` | All of them + 1 merge commit | Exactly 1 |
| "wip" / "oops" commits visible on `main` | Yes | No |
| History shape | Branches and joins (a braid) | Straight line |
| Does Git know the branch was merged? | Yes | No (new unrelated commit) |
| `git branch -d feature` works afterwards | Yes | No, needs `-D` |

**Benefits of squash merge**

- **Clean history.** One commit per PR, so `git log` on `main` reads like
  a changelog.
- **Messy commits are hidden.** "wip", "fix typo", "oops" never reach `main`.
- **Easy revert.** To undo the whole feature: `git revert S`. With a
  normal merge you must revert a merge commit with `-m 1`, or several
  commits.
- **Easier bisecting.** Every commit on `main` is a complete, working
  feature, never a half-finished step.
- **Clear PR-to-commit mapping.** One PR = one commit.

**Benefits of normal merge**

- **Full history is kept.** You can see how the work evolved and who
  wrote which part.
- **Fine-grained blame and bisect.** You can find the exact small commit
  that introduced a bug.
- **Git records the merge.** No surprises if you keep working on the
  same branch.

**Downsides of squash merge**

- The detailed commits and per-commit authorship are collapsed into one
  on `main`.
- Git thinks the branch is still unmerged. If you keep building on the
  old branch you can get repeated conflicts. **Delete the branch after
  squashing** and start new work from a fresh `main`.

**Which should you use?**

- **Squash:** the usual choice for PR-based teams with small features and
  messy working commits. It is the default on many projects.
- **Normal merge:** long-lived branches (for example `develop` into
  `main`), or when every commit is meaningful and worth keeping.
- **Rebase and merge:** when you want to keep each commit but also want a
  straight line with no merge commit.

(The local command version, `git merge --squash`, is in
[Lesson 05](05-branching-and-merging.md).)

---

## 8. Keeping Your Branch Updated While a PR Is Open

**Why do we need this?** On busy repos, `main` moves while your PR is
waiting. GitHub may say "This branch is out-of-date with the base branch"
or show conflicts. You need to bring `main`'s new commits into your
branch.

```bash
git switch feature/add-search-filter
git fetch origin
git merge origin/main
```

- `git fetch origin` — download new commits from GitHub (safe, changes
  nothing of yours).
- `git merge origin/main` — bring them into your branch.

**How to read this:** `feature` is the top row, `origin/main` the bottom
row. `main` gained M and N while your PR was open.

BEFORE (you are on `feature`, after `git fetch origin`):

```
            A---B            feature
           /    ^
      K---L---M---N          origin/main
                HEAD -> feature
```

Command: `git merge origin/main`

AFTER:

```
            A---B---X*       feature
           /       /
      K---L---M---N          origin/main
                  HEAD -> feature (at X)
```

What changed: only `feature` moved, from B to the new merge commit X
(parents: B and N). A and B are unchanged; `origin/main` did not move.

Expected output when it works:

```
Merge made by the 'ort' strategy.
 src/users.py | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
```

- `Merge made by the 'ort' strategy` — Git combined the two lines and
  created a merge commit. (`ort` is just the name of Git's current
  combining algorithm; you do not need to choose it.)

If the same lines were changed on both sides, you get a **conflict**
(Git cannot decide which version to keep):

```
Auto-merging src/users.py
CONFLICT (content): Merge conflict in src/users.py
Automatic merge failed; fix conflicts and then commit the result.
```

Resolve it as taught in Lesson 05: open the file, choose the right code,
remove the `<<<<<<<` markers, then `git add` the file and `git commit`.
Finally push:

```bash
git push
```

```
To github.com:company/project.git
   4b5c6d7..8e9f0a1  feature/add-search-filter -> feature/add-search-filter
```

The PR page updates by itself. It tracks the branch, not a frozen copy.

> **Doubt: "Do I need to create a new PR after pushing more commits?"**
> No. A PR follows the branch. Every new push to the same branch appears
> in the same PR automatically. This is how you address review feedback.

(You can use `git rebase origin/main` instead for a straighter history.
Same BEFORE as above, but the AFTER looks like this:

```
                            A'*-B'*   feature
                           /
      K---L---M---N                   origin/main
```

`feature` now starts from N. A' and B' are new copies of A and B; the old
A and B are abandoned.
Lesson 09 explains exactly when rebase is the better choice. One
warning now: rebase rewrites commits, so after rebasing a branch already
pushed you need `git push --force-with-lease`. Lesson 09 covers why.)

---

## 9. Draft PRs

**Why do we need this?** Sometimes you want early feedback or want CI to
run, but you are not ready for formal review.

**Analogy.** Sending a colleague a rough test plan with "WIP, don't
review yet, just sanity-check the direction."

A **Draft Pull Request** is a PR marked "not ready." Reviewers are not
asked to review, and it cannot be merged. On the "Create pull request"
button, click the small arrow and choose **Create draft pull request**.
When ready, click **Ready for review**. This is underused. Use it
liberally.

---

## Exercise

1. Find a small open-source repo (search `good first issue` at
   github.com/topics/good-first-issue), or use your own repo with a
   second GitHub account or a friend.
2. Fork it, clone **your fork**, add `upstream`, create a branch, make a
   trivial documentation fix, and push to your fork.
   - Expected: `git remote -v` lists both `origin` (your fork) and
     `upstream` (original). `git push -u origin <branch>` ends with
     `set up to track 'origin/<branch>'`.
3. Open a Pull Request from your fork to the original repo. Write a real
   description (what, why, how tested). Even if never merged, the
   practice is the point.
   - Expected: the PR page shows "base: main <- compare: your-branch".
4. On one of your own repos, simulate review: open a PR to yourself, use
   **Start a review**, leave at least 2 inline comments, submit with
   **Comment**.
   - Expected: both comments appear together, under one review.
5. Make three toy PRs, each with 3 small commits. Merge one with each
   strategy (merge commit, squash, rebase). After each, run on `main`:
   ```bash
   git pull
   git log --oneline --graph
   ```
   - Expected: merge commit shows a braided graph and a "Merge pull
     request #N" line; squash shows a single new commit; rebase shows a
     straight line with all 3 commits.

---

## Recap in 5 Lines
1. A Pull Request is a GitHub page asking to merge your branch into another; it tracks the branch, so new pushes update it.
2. No write access: fork, clone your fork (`origin`), add the original as `upstream`, push to `origin`, open the PR.
3. With write access: push a feature branch to `origin` and open the PR within the same repo.
4. Good PRs are small, described well, linked to an issue, self-reviewed; good reviews are specific and kind.
5. Merge commit keeps everything, squash gives one commit per PR, rebase gives a straight line; update your branch with `fetch` + `merge origin/main`.

Continue to [Lesson 09 — Rebase, Cherry-Pick & Squashing History](09-rebase-cherry-pick-squash.md).
