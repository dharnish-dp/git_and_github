# Lesson 01 — Why Version Control? The Problem Git Solves

## Goal
Understand *why* Git exists before learning *how* to use it.

If you skip this, every later command feels like arbitrary magic. If you
get this, every later command feels like an obvious answer to a problem you
already understand.

## Prerequisites
None. This is the first lesson of your Git journey.

## After This Lesson You Will Be Able To
- Explain what problem version control solves, in your own words
- Explain the difference between Git (the tool) and GitHub (a website)
- Explain the difference between centralized and distributed version control
- Explain what a "commit" and a "repository" are, at a basic level
- Run one command to check that Git is installed

---

## Words You Will See in This Lesson

Here are the only new terms, defined once, in plain words. Come back to
this list if you forget one.

| Term | Plain meaning | Python-world analogy |
|---|---|---|
| **Version control system (VCS)** | A tool that remembers every past version of your files. | A very smart "undo history" that survives closing the editor. |
| **Repository (repo)** | A project folder *plus* its full recorded history. | Your project folder + a hidden database of all its past states. |
| **Commit** | One saved snapshot of your project at one moment, with a note saying why. | One entry appended to a history list. |
| **Clone** | A full copy of a repository, including its history, on your machine. | Copying a whole database, not just the latest row. |
| **Remote** | A copy of the repo that lives on another computer (for example, GitHub). | A shared network drive copy. |
| **Push / Pull** | Push = send your new commits to a remote. Pull = fetch other people's new commits from a remote. | Upload / download. |
| **Branch** | A separate line of work, so experiments do not touch your main work. | A `git`-managed copy of the project you can throw away. |

You do not need to master these now. Each gets its own lesson. You only
need a rough feeling for them so the rest of this lesson makes sense.

---

## The Problem: Life Before Version Control

**Analogy.** Imagine writing an important essay in a single Word file.
You want to try a bold rewrite of the intro. You are scared of losing the
old intro. So you press "Save As" and make a copy. Then another. After a
week you have ten copies and no idea which is the good one.

Programmers did exactly this with code. Here is a script you are about to
edit risky-ly:

```
automation_suite.py
automation_suite_backup.py
automation_suite_v2.py
automation_suite_v2_FINAL.py
automation_suite_v2_FINAL_actually_final.py
automation_suite_v2_FINAL_working_before_lunch.py
```

This is "version control by hand". It fails in four specific ways.

### Failure 1: No record of why something changed

Six months later, `_v2_FINAL` tells you nothing. Which bug did it fix?
Why was it made? The file name cannot hold that answer.

### Failure 2: No safe way to combine two people's work

You and a teammate both open `automation_suite.py`. Each of you edits it.
Whoever saves last overwrites the other person's changes. Nobody gets a
warning. The lost work is simply gone.

How to read this: each row is one person; time flows left to right. Both
start from the same file, then save at different moments.

```
time --->   t1          t2          t3            t4
You:        open file   edit        edit          SAVE  (overwrites!)
Teammate:   open file   edit        SAVE          (their changes are gone)

Result: whoever saves LAST wins. Silently. No warning, no merge.
```

### Failure 3: No way to see history

"What did this function look like a month ago?" You would have to
remember which backup file contains it, if any backup does.

### Failure 4: No way to experiment safely

You want to try a risky refactor. So you copy the whole folder "just in
case". Now you have two folders. You must remember in your head which one
has which changes.

### What we actually want

A **version control system (VCS)** solves all four problems. Its job, in
one sentence:

> Track every change to a set of files over time, record who changed what
> and why, and let several people work on the same files without
> overwriting each other.

Git is a VCS. It is the most widely used one in the world.

---

## Python Analogy: Manual Snapshots vs Git

**Why do we need this analogy?** You already know how to hold state in
Python. Seeing the manual version makes it clear what Git automates.

```python
# Without version control, you track changes by hand, and badly:
history = []
data = [1, 2, 3]
history.append(list(data))   # manual snapshot #1
data.append(4)
history.append(list(data))   # manual snapshot #2
data[0] = 99
history.append(list(data))   # manual snapshot #3
# Question: WHY did I change data[0] to 99? Who did it? When? Unknown.
```

Notice what is missing from `history`: no author, no time, no reason.

Git does the same job for your whole project folder, and it also stores:

```
one commit = a snapshot of all your files
           + the author's name
           + the date and time
           + a message (the "why")
```

Git's core promise: **you can always go back, and you will always know
who changed things and why.**

> **Common confusion: "Is a commit the same as saving a file?"**
> No. Saving writes your file to disk, overwriting the old content. A
> commit is a *separate, permanent record* of the saved files at that
> moment. You save many times; you commit when you reach a state worth
> remembering. Think: saving = Ctrl+S, committing = "bookmark this
> moment forever, with a note".

---

## Centralized vs Distributed Version Control

**Why do we need this?** Git was designed to fix weaknesses in older
tools. Knowing those weaknesses explains why Git works the way it does,
including why GitHub is not "the" repository.

### Older tools: centralized

Tools before Git (CVS, Subversion/SVN, Perforce) were **centralized**.
That means one central server holds the full history. Your computer holds
only the current files.

**Analogy.** A public library with one master book. You borrow a page to
read. To see any older edition, you must walk to the library.

How to read this: one server at the top holds all history; each developer
below holds only the current files and must ask the server for anything else.

```
CENTRALIZED (old model: SVN, CVS)

               +------------------+
               |  Central Server  |  <- the ONE copy of full history
               |  (has history)   |
               +------------------+
                 ^      ^      ^
                 |      |      |     every history request goes up
                 v      v      v
            +-------+ +-------+ +-------+
            | Dev A | | Dev B | | Dev C |  <- current files ONLY
            +-------+ +-------+ +-------+     (no history)
```

Two problems:

- **No server access means no work.** Offline, VPN down, or server outage:
  you cannot commit, view history, or do anything version-control related.
- **Single point of failure.** If the server dies and has no separate
  backup, the whole history is gone.

### Git: distributed

Git is **distributed**. Every clone of a repository contains the *entire*
history: every commit ever made, back to the very first one.

**Analogy.** Every student gets a full photocopy of the whole textbook,
not just the page they are reading. If one copy burns, many others exist.

How to read this: four equal boxes; each holds the full history. The
arrows (<--->) mean "can send commits to, and receive commits from".

```
DISTRIBUTED (Git model): every copy is a FULL repo

  Dev A laptop     Dev B laptop     Dev C laptop     GitHub
  +-----------+    +-----------+    +-----------+    +-----------+
  | FULL repo |<-->| FULL repo |<-->| FULL repo |<-->| FULL repo |
  | + history |    | + history |    | + history |    | + history |
  +-----------+    +-----------+    +-----------+    +-----------+
```

(In daily practice people usually sync through GitHub, not directly with
each other. The diagram shows that any copy *could* talk to any other.)

### Key insight

GitHub is not "the" repository. It is *one more full copy* that everyone
agrees to treat as the shared, official one.

If GitHub disappeared tomorrow, every developer with a clone would still
have the complete project history on their laptop.

> **Doubt? "Where is my repository, physically?"**
> Inside your project folder, in a hidden sub-folder named `.git`. That
> hidden folder *is* the repository (the history database). Your visible
> files are just the current snapshot taken out of it. You will explore
> this folder in Lessons 02 and 03.
>
> The important point for now: your local `.git` folder is already a
> complete, real repository. You do not need GitHub for it to work.

---

## Git vs GitHub: The Distinction That Confuses Everyone

**Why do we need this?** Beginners often say "Git" when they mean
"GitHub", and the reverse. They are different things, and mixing them up
makes later lessons confusing.

**Analogy.**
- Git is the *rules of chess*. The rules work anywhere, with no club.
- GitHub is a *chess club* that hosts your games, lets members comment on
  each other's moves, and runs tournaments.

You can play chess without a club. You can use Git without GitHub.

| | Git | GitHub |
|---|---|---|
| What is it | A command-line **program** on your computer that tracks changes to files | A **website/service** that stores Git repositories online |
| Made by | Linus Torvalds (2005), for developing the Linux kernel | A company (now owned by Microsoft) |
| Works offline? | Yes. All core Git work needs no internet. | No. It is a website; you need a connection. |
| Alternatives | None that are widely used (Git won) | GitLab, Bitbucket, self-hosted Gitea, and others |
| Analogy | The rules of chess | A chess club with a leaderboard and tournaments |

You could use Git for your whole life and never touch GitHub. GitHub adds:

- A place to **push** your repo (send your commits online) so others, or
  your other computers, can **pull** it (download the new commits).
- **Pull Requests**: a page where you propose a change and teammates
  review it before it is accepted.
- **Issues**: a tracker for bugs and feature requests.
- **Actions**: automation that runs scripts (such as your test suite)
  when something happens in the repo. This is called CI/CD (continuous
  integration / continuous delivery): automatic build and test runs.
- **Social features**: stars, forks (your own copy of someone else's
  repo), following, discovery.

> **Common confusion: "If I delete my GitHub repo, is my code gone?"**
> Not if you have a clone on your laptop. That clone holds the full
> history. You could create a new GitHub repo and push it all back. The
> reverse also holds: if your laptop dies but you pushed to GitHub, you
> can clone it again.

---

## What Git Actually Tracks

**Why do we need this?** Most beginners picture Git as a list of
"what lines changed" (a diff). That picture is slightly wrong, and it
causes confusion later.

A **diff** is a description of what changed between two versions
(for example "line 12 removed, line 13 added").

Git does *not* mainly store diffs. Git stores **a full snapshot** of your
project at each commit. To avoid wasting space, files that did not change
are not copied again. The new snapshot just points at the existing copy.

**Analogy.** Taking a photo of your desk each evening. If your stapler
did not move, the photo still shows the stapler. You do not need to
re-manufacture the stapler; you just see it in the new photo.

You will go deep on this in Lesson 03. For now, remember this model:

> A Git repository is a **timeline of snapshots**. Each snapshot is called
> a **commit**. You can jump to any point on the timeline, compare any two
> points, or start a new side timeline (a branch) from any point.

**Diagram legend** (used in every lesson):

```
A---B---C      letters (A, B, C...) = commits, joined by dashes
oldest ... newest   time flows LEFT to RIGHT (newest commit on the right)
main           a label = a branch name
^              points at the commit that the label refers to
HEAD -> main   HEAD = "where you are now" (which branch you are on)
```

> Note: internally, Git stores in each commit a link to its *parent* (the
> commit before it). In our drawings we do not draw those links as arrows.
> We simply let time flow left to right.

How to read this: each letter is one commit, oldest on the left, newest on
the right.

```
  A---B---C---D
  |   |   |   |
  |   |   |   +-- "added reporting"
  |   |   +------ "fixed a bug"
  |   +---------- "added a login test"
  +-------------- "first version"
```

Each letter is one commit: a snapshot plus author, time, and a message.

Now the idea of a **branch**. A branch is just a movable label pointing at
one commit. This is the one branching picture you need for now (Lesson 05
covers it fully).

How to read this: `main` is a label sitting on commit D, the newest one.
HEAD says you are currently "on" main.

```
  A---B---C---D
              ^
              |
            main
              |
        HEAD -> main
```

A side timeline is a second label whose commits split off from the main
line. Here `feature` has two commits (E, F) that `main` does not have.

How to read this: each branch is on its own row, with its name at the END
of its row. The `/` shows where the side timeline split off (after B).

```
            E---F        feature
           /
  A---B---C---D          main
```

What changed: nothing is overwritten. Commits A, B, C are shared by both
branches. Only E and F belong to `feature`; only D belongs to `main`.

---

## Hands-On: Check That Git Is Installed

Most of this lesson is conceptual. One tiny command proves your setup is
ready for Lesson 02.

Open a terminal and run:

```bash
git --version
```

Expected output (your version number will differ):

```
git version 2.43.0
```

Line by line:

- `git version` : Git is installed and answered you.
- `2.43.0` : the installed version. Any 2.x version is fine for this
  course.

If you see `command not found: git` (zsh) or similar, Git is not
installed. Install it from https://git-scm.com/downloads (on macOS, you
can also run `xcode-select --install`), then run the command again.

> **Doubt? "Why did a command that starts with `git` work with no
> internet?"**
> Because Git is a program installed on your own computer. It never needs
> a network for this. That fact is the Git-versus-GitHub difference from
> earlier in action.

---

## Exercise

This lesson is mostly conceptual. Answer these out loud or in your head.
Check your answer against the hints.

1. In two sentences, explain to an imaginary coworker why "keep copies of
   the folder" stops working as a project grows.
   *Hint: mention at least two of: no record of why, silent overwriting,
   no history browsing, hard to experiment.*

2. If GitHub went offline for a day, what could you still do with a repo
   you already cloned?
   *Answer: nearly everything: commit, create branches, view history,
   compare versions. You could not push or pull, because those need a
   remote to talk to.*

3. Name one thing GitHub gives you that plain Git does not.
   *Answer: any of Pull Requests, Issues, Actions, or online hosting.*

4. Run `git --version` and confirm it prints a version.
   *Expected: a line starting with `git version`.*

5. True or false: "Git stores each commit mainly as a list of changed
   lines." *Answer: false. Git stores full snapshots, reusing unchanged
   files to save space.*

---

## Recap in 5 Lines

1. Copying files by hand has no history, no "why", no safe teamwork, and
   no safe experiments. A version control system fixes that.
2. Git is a free program on your computer; GitHub is a website that hosts
   Git repositories. They are not the same thing.
3. Git is distributed: every clone holds the full history, so GitHub is
   just one more copy, not the only copy.
4. A repository is a timeline of snapshots; each snapshot is a commit
   (snapshot + author + time + message).
5. All core Git work happens offline on your own machine.

Move to [Lesson 02](02-core-concepts-and-git-anatomy.md) once you can
answer the exercises confidently.
