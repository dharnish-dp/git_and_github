# Lesson 01 — Why Version Control? The Problem Git Solves

## Goal
Understand *why* Git exists before learning *how* to use it. If you skip
this, every command later feels like arbitrary magic. If you get this,
every command later feels obvious.

## Prerequisites
None. This is lesson zero of your Git journey.

## After This Lesson You Will Be Able To
- Explain what problem version control solves, in your own words
- Explain the difference between Git (the tool) and GitHub (a website)
- Explain the difference between centralized and distributed version control

---

## The Problem: Life Before Version Control

Imagine you're editing a Python script. You're about to try something risky,
so you do what everyone does before Git existed:

```
automation_suite.py
automation_suite_backup.py
automation_suite_v2.py
automation_suite_v2_FINAL.py
automation_suite_v2_FINAL_actually_final.py
automation_suite_v2_FINAL_working_before_lunch.py
```

This is version control by hand. It fails in every way that matters:

- **No record of *why* something changed.** Six months later, `_v2_FINAL`
  tells you nothing about what problem it fixed.
- **No safe way to combine two people's changes.** If you and a teammate
  both edit `automation_suite.py`, whoever saves last silently overwrites
  the other's work.
- **No way to see history.** What did this function look like a month ago?
  You'd have to remember which backup file has it, if any.
- **No way to experiment safely.** Want to try a risky refactor? Copy the
  whole folder first "just in case." Now you have two folders to keep in
  sync in your head.

Version control systems (VCS) exist to solve exactly this: **track every
change to a set of files over time, with a record of who changed what and
why, and let multiple people work on the same files without stepping on
each other.**

---

## Centralized vs Distributed VCS

There were VCS tools before Git (CVS, Subversion/SVN, Perforce). They were
**centralized**: one central server holds *the* history. Your machine only
holds the current files, not the full history.

```
CENTRALIZED (old model — SVN, CVS)

        ┌──────────────┐
        │ Central Server│   ← the ONE copy of full history
        │  (has history) │
        └──────┬────────┘
        ┌──────┼────────┐
        ▼      ▼        ▼
    Dev A   Dev B    Dev C     ← each only has current checkout
   (no history)  (no history) (no history)
```

Problems with this model:
- No server access (offline, VPN down, server outage) = you can't commit,
  can't see history, can't do *anything* version-control related.
- The server is a single point of failure. Lose it, lose everything (unless
  backed up separately).

Git is **distributed**. Every clone of a repository contains the *entire*
history — every commit, ever made, going back to the first one.

```
DISTRIBUTED (Git model)

    Dev A              Dev B              Dev C            GitHub
┌───────────┐      ┌───────────┐      ┌───────────┐    ┌───────────┐
│ FULL repo │◄────►│ FULL repo │◄────►│ FULL repo │◄──►│ FULL repo │
│ + history │      │ + history │      │ + history │    │ + history │
└───────────┘      └───────────┘      └───────────┘    └───────────┘
```

**Key insight:** GitHub is not "the" repository. It's just *one more clone*
that everyone agrees to treat as the shared source of truth. If GitHub
vanished tomorrow, every developer with a clone still has the entire
project history on their laptop. This is a fundamentally different
guarantee than centralized systems ever offered.

This is the single most important mental shift for beginners: **your local
`.git` folder already IS a complete, real repository** — with full history —
even before you ever connect it to GitHub. GitHub is a *hosting and
collaboration layer* on top of Git, not Git itself.

---

## Git vs GitHub — The Distinction That Confuses Everyone

| | Git | GitHub |
|---|---|---|
| What is it | A command-line **program** that tracks changes to files | A **website/service** that hosts Git repositories |
| Made by | Linus Torvalds (2005), for Linux kernel development | A company (now owned by Microsoft) |
| Works offline? | Yes — 100% of Git works with no internet | No — needs a server to host on |
| Alternatives | (there are none really — Git won) | GitLab, Bitbucket, self-hosted Gitea, etc. |
| Analogy | The *rules of chess* | A *chess club* that also runs tournaments, has a leaderboard (Issues, Pull Requests, Actions) |

You could use Git for your entire life and never touch GitHub — just
committing to a local repo on your own machine. GitHub adds:
- A place to **push** your repo so others (or your other machines) can pull it
- **Pull Requests** — a UI for proposing and reviewing changes
- **Issues** — bug/feature tracking
- **Actions** — CI/CD automation
- **Social features** — stars, forks, following, discovery

---

## Python Analogy: Git Is Like `git log` for a List

If you've ever wanted "undo" beyond one step, or wanted to know *who*
introduced a bug and *why*, you've wanted what Git gives you natively.

```python
# Without version control, you track state changes manually, badly:
history = []
data = [1, 2, 3]
history.append(list(data))   # manual snapshot #1
data.append(4)
history.append(list(data))   # manual snapshot #2
data[0] = 99
history.append(list(data))   # manual snapshot #3 — and WHY did I change it? who knows.

# Git does this automatically, with metadata, for your entire project:
# each commit = a snapshot + author + timestamp + message (the "why")
```

Git's core promise: **you can always go back**, and you'll always know
*who* and *why*.

---

## What Git Actually Tracks

A common misconception: Git tracks *changes* (diffs) as its primary unit.
It doesn't — Git tracks **full snapshots** of your project at each commit,
but stores them efficiently (unchanged files are just referenced, not
duplicated). We go deep on this in Lesson 03. For now, the mental model:

> A Git repository is a timeline of snapshots. Each snapshot is called a
> **commit**. You can jump to any point on that timeline, compare any two
> points, or branch off a new timeline from any point.

---

## Exercise

No commands yet — this lesson is conceptual. Answer these for yourself
(no need to write it down, just make sure you can articulate it):

1. Explain to an imaginary coworker, in two sentences, why "just keep
   copies of the folder" doesn't scale as a project grows.
2. If GitHub went offline for a day, what could you still do on your own
   machine with a Git repo you already have cloned? (Answer: nearly
   everything — commit, branch, view history, diff — just not push/pull.)
3. Name one thing GitHub gives you that plain Git does not.

Move to [Lesson 02](02-core-concepts-and-git-anatomy.md) once you can
answer these confidently.
