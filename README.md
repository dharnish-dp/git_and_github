# Git & GitHub Mastery — Complete Learning Guide

A complete, lesson-by-lesson course that takes you from "never opened a
terminal for Git" to "top 1% Git/GitHub proficiency." Every lesson includes
concepts, mental models (with Python analogies), diagrams, real commands to
run, and hands-on exercises.

No prior Git knowledge assumed. By the end, you'll understand not just
*which commands to type*, but *what Git is actually doing under the hood* —
which is the difference between someone who follows tutorials and someone
who can fix any Git disaster calmly.

---

## Lessons

| Lesson | Topic | What You'll Never Fear Again |
|--------|-------|-------------------------------|
| [Lesson 01](lessons/01-why-version-control.md) | Why Version Control? The Problem Git Solves | "Should I zip a backup of this folder before editing?" |
| [Lesson 02](lessons/02-core-concepts-and-git-anatomy.md) | Core Concepts: The Three Trees & Git's Anatomy | Working dir vs staging vs commit — what actually moves where |
| [Lesson 03](lessons/03-under-the-hood-git-objects.md) | Under the Hood: Blobs, Trees, Commits & the Object Database | "What even IS a commit hash?" |
| [Lesson 04](lessons/04-your-first-repo.md) | Your First Repo: init, add, commit, status, diff, log | Starting any repo from zero, confidently |
| [Lesson 05](lessons/05-branching-and-merging.md) | Branching & Merging | Merge conflicts — and why branches are basically free |
| [Lesson 06](lessons/06-undoing-things.md) | Undoing Things: checkout, restore, reset, revert, reflog | "I think I just lost my work" panic |
| [Lesson 07](lessons/07-remotes-and-github-basics.md) | Remotes & GitHub Basics: clone, push, pull, fetch | push vs pull vs fetch confusion |
| [Lesson 08](lessons/08-collaboration-pull-requests.md) | Collaboration: Forks, Pull Requests & Code Review | Contributing to a team repo or open source project |
| [Lesson 09](lessons/09-rebase-cherry-pick-squash.md) | Rebase, Cherry-Pick & Squashing History | Rewriting history without breaking everything |
| [Lesson 10](lessons/10-stash-tags-bisect.md) | Stash, Tags & Bisect | Context-switching mid-task, releases, and hunting bugs |
| [Lesson 11](lessons/11-github-pro-features.md) | GitHub Pro Features: Actions, Branch Protection, CLI | Setting up CI/CD and repo governance |
| [Lesson 12](lessons/12-submodules-subtrees-and-large-repos.md) | Submodules, Subtrees & Large-Repo Tooling | Embedding one repo in another, and staying fast in a huge repo |
| [Lesson 13](lessons/13-pro-workflows-and-best-practices.md) | Pro Workflows & Best Practices | Commit hygiene, `.gitignore`, hooks, signing, disaster recovery |

---

## What Comes After This Course

- [What Next — Advanced Roadmap](what-next.md) — monorepos, trunk-based dev, Git LFS, GitOps, advanced CI/CD

---

## Quick Reference

- [Cheat Sheet](reference/cheatsheet.md) — the commands you'll type every single day
- [Glossary](reference/glossary.md) — every term defined in plain English

---

## How This Course Works

- **Lessons build on each other** — do them in order the first time through.
- **Every lesson has hands-on exercises** — Git is a *muscle-memory* skill.
  Reading about branching will not make you good at branching. Typing the
  commands will.
- **Diagrams over memorization** — Git makes sense once you understand its
  data model (a graph of snapshots). Once that clicks, every command becomes
  obvious instead of a magic incantation.
- **Mistakes are part of the curriculum** — Lesson 06 exists specifically so
  you can break things safely and learn how to undo *anything*. Professionals
  aren't people who never make Git mistakes; they're people who aren't
  scared when they do.

## Setup Check

Before Lesson 01, confirm Git is installed and you have a GitHub account:

```bash
git --version        # should print something like git version 2.4x.x
```

If it's missing on macOS: `brew install git` (or install Xcode Command Line
Tools, which bundles Git: `xcode-select --install`).

Then set your identity once, globally — Git stamps every commit with this:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Create a free account at https://github.com if you don't have one — you'll
need it starting in Lesson 07.
