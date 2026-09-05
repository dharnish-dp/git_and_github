# Lesson 08 — Collaboration: Forks, Pull Requests & Code Review

## Goal
Learn how real teams and open-source projects actually use GitHub day to
day: forking, branching strategy, opening Pull Requests, and reviewing
code — the workflow you'll use in every job.

## Prerequisites
[Lesson 07 — Remotes & GitHub Basics](07-remotes-and-github-basics.md)

## After This Lesson You Will Be Able To
- Contribute to any open source project via fork + Pull Request
- Contribute to a team repo you have direct write access to
- Review a teammate's PR usefully
- Keep your branch up to date with `main` cleanly

---

## Two Collaboration Models

| | **Fork-based** | **Shared-repo (direct branch)** |
|---|---|---|
| When | Open source, or you don't have write access | Internal team repos where you're a collaborator |
| You push to | Your own copy (fork) on GitHub | Directly to the shared repo, on a feature branch |
| PR opened | From your fork's branch → original repo's `main` | From your feature branch → `main`, same repo |

Most companies use the shared-repo model internally. Open source (and many
companies for external contributors) uses fork-based. Both funnel into the
exact same final step: a **Pull Request**.

---

## Fork-Based Workflow (Open Source Style)

```bash
# 1. On GitHub: click "Fork" on the project you want to contribute to.
#    This creates YOUR OWN copy at github.com/yourname/project

# 2. Clone YOUR fork (not the original):
git clone git@github.com:yourname/project.git
cd project

# 3. Add the ORIGINAL repo as a second remote, conventionally named "upstream":
git remote add upstream git@github.com:original-owner/project.git
git remote -v
# origin    → your fork      (fetch/push)
# upstream  → original repo  (fetch/push — but you usually only pull from it)

# 4. Create a branch, make your change:
git switch -c fix/typo-in-readme
# ... edit files ...
git add . && git commit -m "Fix typo in README"

# 5. Push to YOUR fork (origin), not upstream (you likely don't have write access there):
git push -u origin fix/typo-in-readme

# 6. On GitHub: open a Pull Request FROM your fork's branch TO the original repo's main branch.
```

**Keeping your fork in sync with upstream** (do this regularly — the
original project moves on without you):

```bash
git fetch upstream
git switch main
git merge upstream/main        # or: git rebase upstream/main (Lesson 09)
git push origin main           # update your fork's main on GitHub too
```

---

## Shared-Repo Workflow (Typical Team/Company Style)

```bash
git clone git@github.com:company/project.git
cd project
git switch -c feature/add-search-filter    # branch off main
# ... work, commit ...
git push -u origin feature/add-search-filter
# On GitHub: open a Pull Request FROM feature/add-search-filter TO main, SAME repo.
```

The only structural difference from the fork model: one remote (`origin`)
instead of two, since you already have write access.

---

## Opening a Good Pull Request

A Pull Request (PR) is a request: "please review and merge these commits
into this branch." On GitHub, a PR gives you:

- A **diff view** of every changed line, file by file
- A place for **discussion and inline comments** on specific lines
- **Status checks** (CI test results — Lesson 11)
- A **merge button**, once approved and checks pass

**What makes a PR easy to review (and get merged fast):**

1. **Keep it small and focused.** One logical change per PR. A 20-line PR
   gets reviewed in 5 minutes; a 2,000-line PR gets ignored for a week.
2. **Write a real description.** What changed, why, how you tested it.
   Most repos have a PR template (a pre-filled checklist) — use it.
3. **Link the issue/ticket it addresses**, e.g. `Closes #142` in the
   description — GitHub auto-closes the issue when the PR merges.
4. **Self-review before requesting review.** Read your own diff on GitHub
   first. You'll catch half your own mistakes.
5. **Respond to every comment** — even if just "Done" or "Good point,
   fixed in latest commit" — don't leave reviewers wondering if you saw it.

---

## Reviewing Someone Else's PR

On the PR's "Files changed" tab:
- Click any line number to leave an **inline comment**.
- Use **"Start a review"** to batch multiple comments into one submission
  instead of spamming individual notifications.
- Choose one of three verdicts when submitting your review:
  - **Comment** — feedback, no explicit approval/rejection
  - **Approve** — looks good, ready to merge
  - **Request changes** — blocks merging until addressed (if branch
    protection requires it — Lesson 11)

**What to actually look for** (beyond "does it work"):
- Does it match the codebase's existing patterns/conventions?
- Are there tests, and do they cover the actual risk (not just happy path)?
- Is anything here going to be confusing to the next person reading it in
  six months?
- Any security concerns (input validation, secrets, injection risks)?

Good review comments are **specific and kind**: "This will throw if
`user` is `None` — should we guard it?" beats "this is wrong."

---

## Merge Strategies on GitHub

When you click the green merge button, GitHub offers three strategies —
know the difference, because it changes what `main`'s history looks like
afterward:

| Strategy | What Happens | When to Use |
|---|---|---|
| **Create a merge commit** | Standard 3-way merge (Lesson 05); keeps all individual commits + adds a merge commit | Default; preserves exact history |
| **Squash and merge** | Combines ALL commits in the PR into **one** commit on `main` | PRs with messy "wip", "fix typo" commits — keeps `main`'s history clean, one commit per feature |
| **Rebase and merge** | Replays each commit individually onto `main`, no merge commit at all | Want individual commits preserved but a perfectly linear history |

Many teams default to **squash and merge** for exactly this reason: it
doesn't matter how messy your commits were while developing — `main`
only ever sees one clean commit per PR. We cover rebase mechanics fully
in Lesson 09.

---

## Keeping Your Branch Updated While a PR Is Open

If `main` moves while your PR is still open (very common on active
repos), update your branch before it can be merged:

```bash
git switch feature/add-search-filter
git fetch origin
git merge origin/main          # brings main's new commits into your branch
# resolve any conflicts (Lesson 05), then:
git push
```

(Or `git rebase origin/main` for a cleaner, linear history — see
Lesson 09 for exactly when rebase is the better choice here.)

---

## Draft PRs

If you want early feedback before it's ready, or want CI to run without
signaling "review me now," open it as a **Draft Pull Request**. Convert
to a normal PR ("Ready for review") when it's actually done. This is a
genuinely underused feature — use it liberally.

---

## Exercise

1. Find a small open-source repo (even a "good first issue"-tagged one,
   searchable at github.com/topics/good-first-issue) or use one of your
   own repos with a second GitHub account/friend if available.
2. Fork it, clone your fork, add `upstream`, create a branch, make a
   trivial documentation fix, and push to your fork.
3. Open a Pull Request from your fork to the original repo. Write a
   real description (even if the maintainer never merges it — the
   practice is what matters).
4. On one of your own personal repos, simulate review: open a PR to
   yourself, use "Start a review," leave at least 2 inline comments, and
   try all three merge strategies (on different toy PRs) to see how
   `main`'s history differs afterward with `git log --oneline --graph`.

Continue to [Lesson 09 — Rebase, Cherry-Pick & Squashing History](09-rebase-cherry-pick-squash.md).
