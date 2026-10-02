# Lesson 07 — Remotes & GitHub Basics: clone, push, pull, fetch

## Goal
Connect your local Git knowledge to GitHub. Understand exactly what
`push`, `pull`, and `fetch` each do (they are commonly confused), set up
authentication properly, and know when force-push is actually dangerous.

## Prerequisites
[Lesson 06 — Undoing Things](06-undoing-things.md). You also need a
free GitHub account (https://github.com).

Quick reminders of words from earlier lessons:
- **Repository (repo)** = a project folder plus its full saved history
  (stored in the hidden `.git` folder).
- **Commit** = a saved snapshot of your whole project, with a message and
  an author.
- **Branch** = a movable label that points at one commit (usually the
  newest one of a line of work). `main` is the default branch name.

## After This Lesson You Will Be Able To
- Clone a repo, or connect an existing local repo to GitHub
- Explain the difference between fetch, pull, and push
- Set up SSH authentication (no more typing passwords)
- Read remote-tracking branches and "ahead/behind" messages
- Decide when a force-push is safe

---

## What Is a "Remote"?

**Why do we need this?** So far, all your work lives on your laptop. If
the laptop dies, the work is gone. And nobody else can see it. We need a
second copy somewhere else, and a way to talk to it.

**Analogy.** Think of your phone's contacts. You do not dial a phone
number from memory; you tap a name like "Mom". A **remote** is a
contact-list entry for another copy of your repository. The name is a
nickname; the "phone number" is a URL.

**Plain explanation.**
- A **remote** = a nickname + URL that points to another copy of the repo.
  That copy is usually on GitHub, but it could be a teammate's machine or
  even another folder on your disk.
- The usual nickname for your main remote is **`origin`**. This is only a
  habit. You could call it `banana` and Git would not mind.
- **GitHub** = a website that hosts Git repositories and adds
  collaboration tools around them. **Git** is the tool on your laptop;
  **GitHub** is one company's server. They are not the same thing.

```
   YOUR COMPUTER                            GITHUB (origin)
  +-----------------+                     +-----------------+
  | your repo       |   "origin" = URL    | the hosted repo |
  | A---B---C       | ------------------> | A---B---C       |
  +-----------------+                     +-----------------+
```

Diagram legend (used in every diagram from here on):

```
  A---B---C     commits; oldest on the LEFT, newest on the RIGHT
  main          branch label at the END of its row (points at last commit)
  origin/main   remote-tracking branch: your saved photo of GitHub
  D*            asterisk = commit new/moved by the command just run
  /  \          a fork or a merge; each branch gets its own row
  ahead/behind  ahead N = commits only you have; behind N = only remote has
```

Note: Git internally stores each commit's link to its parent, but we draw
time flowing left to right.

List your remotes:

```bash
git remote -v
```

Expected output:

```
origin  https://github.com/yourname/your-repo.git (fetch)
origin  https://github.com/yourname/your-repo.git (push)
```

Line by line:
- `origin` = the nickname.
- The URL = where that nickname points.
- `(fetch)` = the URL used when downloading from it.
- `(push)` = the URL used when uploading to it. Usually the same URL, so
  you see two lines. `-v` means "verbose" (show the URLs, not just names).

If you have no remotes yet, the command prints nothing.

> **Common confusion: "Is `origin` a special Git word?"**
> No. It is just the default nickname `git clone` picks. You can have
> several remotes, for example `origin` (your copy) and `upstream` (the
> original project you copied from). That becomes important in
> [Lesson 08](08-collaboration-pull-requests.md).

---

## Two Ways to Get Started

There are two situations:
- A) The project already exists on GitHub and you want it on your laptop.
- B) The project exists on your laptop and you want it on GitHub.

### Option A: Clone an Existing GitHub Repo

**Why do we need this?** To get a working copy of someone's project.

**Analogy.** Photocopying a whole book, including every earlier edition,
not just the latest page.

**Plain explanation.** **Clone** = download a complete copy of a remote
repo, with its full history (every commit ever made). Git is
"distributed", meaning every copy holds everything (Lesson 01). Clone
also creates the `origin` remote for you.

```bash
git clone https://github.com/someuser/some-repo.git
cd some-repo
```

Expected output of the first command:

```
Cloning into 'some-repo'...
remote: Enumerating objects: 128, done.
remote: Counting objects: 100% (128/128), done.
remote: Compressing objects: 100% (74/74), done.
remote: Total 128 (delta 41), reused 120 (delta 38), pack-reused 0
Receiving objects: 100% (128/128), 32.10 KiB | 1.60 MiB/s, done.
Resolving deltas: 100% (41/41), done.
```

Line by line:
- `Cloning into 'some-repo'...` = Git creates a new folder named after
  the repo.
- `remote: ...` lines = messages from the GitHub server as it prepares
  the data.
- `Receiving objects` = your laptop downloading the saved snapshots and
  file contents (Git calls these "objects", see Lesson 03).
- `Resolving deltas` = Git rebuilding files that were stored as
  "differences from an older version" to save space.

Then `cd some-repo` moves your terminal into the new folder.

Check that `origin` exists:

```bash
git remote -v
# origin  https://github.com/someuser/some-repo.git (fetch)
# origin  https://github.com/someuser/some-repo.git (push)
```

### Option B: Connect a Local Repo You Already Have

**Why do we need this?** You made commits locally (from earlier lessons)
and now want a backup and a shareable copy on GitHub.

Steps:

1. On GitHub: click **New repository**, give it a name, and do **not**
   tick "Add a README". Why? Your laptop already has commits. If GitHub
   also creates a first commit, the two histories start differently and
   you create an avoidable conflict. An empty GitHub repo avoids that.
2. GitHub then shows you the commands. They are these three:

```bash
git remote add origin https://github.com/yourname/your-repo.git
git branch -M main
git push -u origin main
```

Explained one by one:

| Command | What it does |
|---|---|
| `git remote add origin <url>` | Adds a contact-list entry: nickname `origin` -> this URL. Prints nothing when it works. |
| `git branch -M main` | Renames your current branch to `main` (`-M` = rename, even if the name is taken). Older Git versions defaulted to `master`; GitHub now expects `main`. |
| `git push -u origin main` | Uploads your `main` branch to `origin`, and remembers the link (the `-u`, explained below). |

Expected output of the push:

```
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Writing objects: 100% (9/9), 812 bytes | 812.00 KiB/s, done.
Total 9 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:yourname/your-repo.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

Line by line:
- `Writing objects` = uploading your snapshots.
- `To github.com:...` = the destination.
- `* [new branch] main -> main` = "your local `main` became a new branch
  called `main` on the remote". Format is `local -> remote`.
- `branch 'main' set up to track 'origin/main'` = the `-u` worked
  (explained in "Tracking Branches" below).

---

## How GitHub Works and What `origin` Really Is (Step by Step)

**Why do we need this?** Most confusion in this lesson comes from not
knowing *where things live*. Once you can picture exactly what is on
GitHub, what is on your laptop, and what Git creates when you clone, every
command below becomes obvious.

### What GitHub actually is

GitHub is two things stacked together:

1. **A Git repository on a server.** The same kind of repo you have on
   your laptop (commits, branches, tags), but with no working folder, just
   the history. (Git calls this a *bare* repository.)
2. **A website around it.** Pull requests, issues, Actions, permissions.
   Those extras exist only on GitHub; they are not part of Git.

So "pushing to GitHub" simply means "copying commits into the repo that
lives on GitHub's server".

```
   GITHUB (server)                          YOUR LAPTOP
  +-----------------------------+          +-----------------------------+
  | Git repo (history only)     |          | Git repo (.git folder)      |
  |   branches: main            |  <-----> |   + your working folder     |
  |             feature-x       |  clone   |     (the files you edit)    |
  |             bugfix-y        |  fetch   |                             |
  | + website: PRs, issues ...  |  push    |                             |
  +-----------------------------+          +-----------------------------+
```

### The three places a branch can exist

This is the key idea. The same project has up to **three kinds of branch
pointers**:

| Pointer | Where it lives | Who can move it | Example |
|---|---|---|---|
| **Remote branch** | On GitHub's server | Whoever pushes (you or a teammate) | `main` on GitHub |
| **Remote-tracking branch** | On your laptop, inside `.git` | Only `fetch`, `pull`, `push` | `origin/main` |
| **Local branch** | On your laptop, inside `.git` | You: commit, merge, reset | `main` |

Rule of thumb: **`origin/...` is Git's saved photo of the GitHub row.**
It exists because Git works offline and never changes your own branches
behind your back.

### What `git clone` does, step by step

Suppose GitHub has three branches: `main`, `feature-x`, `bugfix-y`.

```bash
git clone https://github.com/someuser/some-repo.git
```

Git performs these steps in order:

1. **Makes the folder and an empty repo inside it** (`some-repo/.git`),
   like `git init`.
2. **Saves the remote.** It records the nickname `origin` and the URL in
   `.git/config`:

   ```
   [remote "origin"]
       url = https://github.com/someuser/some-repo.git
       fetch = +refs/heads/*:refs/remotes/origin/*
   ```

   The `fetch = ...` line is a copy rule: "every branch on the remote
   (`refs/heads/*`) is photographed into `refs/remotes/origin/*` on my
   laptop". This rule is **why `origin/<branch>` appears automatically**.
3. **Downloads everything**: every commit of **every** branch, plus all
   file contents. Not only `main`.
4. **Creates one `origin/<branch>` bookmark for each remote branch.**
5. **Creates ONE local branch**: the remote's default branch (usually
   `main`), pointing at the same commit as `origin/main`. It also writes
   the tracking link into `.git/config`:

   ```
   [branch "main"]
       remote = origin
       merge = refs/heads/main
   ```

6. **Checks out that branch**: fills your folder with its files and
   points `HEAD` at `main`.

Result right after the clone:

```
    GITHUB (origin)
    A---B---C                     main
    A---B---C---D                 feature-x
    A---B---E                     bugfix-y

    YOUR COMPUTER
    A---B---C                     main                 (local branch, HEAD is here)
    A---B---C                     origin/main          (photo)
    A---B---C---D                 origin/feature-x     (photo)
    A---B---E                     origin/bugfix-y      (photo)
```

**What changed:** your laptop now holds all the commits, but only `main`
is a real local branch. `feature-x` and `bugfix-y` exist only as
`origin/...` photos. Verify:

```bash
git branch        # local branches only
git branch -r     # remote-tracking branches only
git branch -a     # both
```

Expected output:

```
* main                            <- git branch

  origin/HEAD -> origin/main      <- git branch -r
  origin/bugfix-y
  origin/feature-x
  origin/main
```

`origin/HEAD -> origin/main` just records "the remote's default branch is
`main`".

### Starting work on an existing remote branch

You want to work on `feature-x`, which only exists as `origin/feature-x`:

```bash
git switch feature-x
```

Expected output:

```
branch 'feature-x' set up to track 'origin/feature-x'.
Switched to a new branch 'feature-x'
```

What Git did: it saw no local `feature-x`, found exactly one
`origin/feature-x`, so it **created a local `feature-x` at the same commit
and linked it to `origin/feature-x`**. (Long form:
`git switch -c feature-x --track origin/feature-x`.)

```
    YOUR COMPUTER
    A---B---C                     main
    A---B---C---D                 feature-x            (NEW local branch, HEAD)
    A---B---C---D                 origin/feature-x     (photo, unchanged)
```

### Creating your own brand-new local branch

```bash
git switch -c my-work
```

Now only your laptop knows about `my-work`:

```
    YOUR COMPUTER
    A---B---C                     main
    A---B---C                     origin/main
    A---B---C                     my-work              (HEAD) local only

    GITHUB
    A---B---C                     main                 (no my-work here)
```

- There is **no** `origin/my-work`, because `origin/...` only mirrors
  branches that exist on GitHub.
- There is no upstream yet, so plain `git push` fails with "no upstream
  branch".

You commit twice (D, E) and then publish it:

```bash
git push -u origin my-work
```

What happens in order:

1. Git uploads commits D and E to GitHub.
2. GitHub **creates the branch `my-work`** on its side.
3. Git **creates `origin/my-work`** on your laptop (photo of what it just
   created).
4. `-u` writes the tracking link into `.git/config`.

```
    YOUR COMPUTER
    A---B---C---D---E             my-work              (HEAD)
    A---B---C---D---E             origin/my-work       (NEW photo)

    GITHUB
    A---B---C---D---E             my-work              (NEW branch)
```

**What changed:** the three rows are now in sync for `my-work`. From here,
plain `git push` and `git pull` know where to go.

### When the remote changes later

| Event on GitHub | What your laptop shows after `git fetch` |
|---|---|
| A teammate pushes to `main` | `origin/main` moves forward; your `main` stays (now "behind") |
| A teammate creates branch `feature-z` | New photo `origin/feature-z` appears; no local branch is created |
| A teammate deletes `bugfix-y` | `origin/bugfix-y` stays as a stale photo until you run `git fetch --prune` |

Tidy stale photos:

```bash
git fetch --prune
git config --global fetch.prune true   # optional: do it on every fetch
```

### Summary: who creates what

| Action | Creates `origin/<x>`? | Creates local `<x>`? | Creates `<x>` on GitHub? |
|---|---|---|---|
| `git clone` | Yes, for every remote branch | Only the default branch | No |
| `git fetch` | Yes, for new remote branches; updates existing ones | No | No |
| `git switch <x>` (exists only as `origin/<x>`) | No | Yes, tracking `origin/<x>` | No |
| `git switch -c <x>` | No | Yes | No |
| `git push -u origin <x>` | Yes | No (already exists) | Yes |

> **Common confusion: "I cloned, but `git branch` shows only `main`.
> Where are the other branches?"**
> They are downloaded, but shown as `origin/...` photos. Use
> `git branch -r` to see them and `git switch <name>` to create a local
> branch from one.

---

## SSH vs HTTPS — Set This Up Once, Never Type a Password Again

**Why do we need this?** GitHub must know who you are before it accepts
uploads. There are two ways to prove it.

**Analogy.** HTTPS with a token is like showing an ID card at the door
every time. SSH is like having a key cut once: the door recognises your
key automatically.

**Plain explanation.**
- **HTTPS** = URLs starting with `https://`. Needs a **Personal Access
  Token (PAT)**: a long random string you generate on GitHub that acts as
  a password for Git.
- **SSH** = URLs starting with `git@github.com:`. Uses a **key pair**:
  two linked files. The **private key** stays secret on your laptop. The
  **public key** is safe to share; you give it to GitHub. GitHub can then
  check "this person holds the matching private key" without any password
  being sent. (Python analogy: like signing with a secret and letting
  others verify with a public key.)

```
 Your laptop                         GitHub
 ~/.ssh/id_ed25519      (private)    your account has:
 ~/.ssh/id_ed25519.pub  (public) --> a copy of the PUBLIC key
```

**SSH is the professional default.** Set it up once per machine.

### Step 1: Generate a key pair

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

- `ssh-keygen` = the tool that makes key pairs.
- `-t ed25519` = the key type (a modern, secure, short one).
- `-C "you@example.com"` = a label (comment) so you can recognise the key
  later. It does not need to be a real login.

Expected prompts:

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/Users/you/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/you/.ssh/id_ed25519
Your public key has been saved in /Users/you/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:abc123... you@example.com
```

Press Enter at the first prompt (default location is fine). A
**passphrase** is an optional password protecting the private key file.
Setting one is more secure; leaving it empty is more convenient.

Two files now exist: `id_ed25519` (private, never share) and
`id_ed25519.pub` (public, shareable).

### Step 2: Copy your PUBLIC key

```bash
cat ~/.ssh/id_ed25519.pub
```

Expected output (one long line):

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... you@example.com
```

Copy that whole line. Make sure the filename ends in `.pub`.

> **Common confusion: "Which file do I paste into GitHub?"**
> Only the one ending in `.pub`. Never paste or share the file without
> `.pub`; that is your private key.

### Step 3: Add it to GitHub

In the browser: **Settings -> SSH and GPG keys -> New SSH key**, give it a
title (such as "My MacBook"), paste the line, and save.

### Step 4: Test it

```bash
ssh -T git@github.com
```

Expected output:

```
Hi yourname! You've successfully authenticated, but GitHub does not provide shell access.
```

This is a success message. "No shell access" only means GitHub will not
let you open a terminal on its servers; that is normal. The first time,
you may see a question "Are you sure you want to continue connecting
(yes/no)?". Type `yes`; this records GitHub's identity on your laptop.

### Step 5: Use SSH URLs

```
git@github.com:yourname/your-repo.git      <- SSH (no password prompts)
https://github.com/yourname/your-repo.git  <- HTTPS (needs a token)
```

If a remote already uses HTTPS, change it:

```bash
git remote set-url origin git@github.com:yourname/your-repo.git
git remote -v
# origin  git@github.com:yourname/your-repo.git (fetch)
# origin  git@github.com:yourname/your-repo.git (push)
```

`set-url` means "keep the nickname `origin`, but change the address it
points to".

**Never put a real account password in an HTTPS URL or a Git prompt.**
GitHub stopped accepting account passwords for Git operations years ago.
If you use HTTPS, generate a Personal Access Token (Settings -> Developer
settings -> Personal access tokens) and paste that when Git asks for a
"password".

---

## Credential Helpers — Stop Re-Entering Your Token

**Why do we need this?** If you chose HTTPS, Git asks for your token on
every `push` and `pull`. That gets annoying fast.

**Plain explanation.** A **credential helper** is a small program that
Git hands your login details to, so it can store them and give them back
later. You then type the token once.

```bash
# Option 1: remember in memory for 1 hour (no install needed)
git config --global credential.helper 'cache --timeout=3600'
```

- `git config` = change a Git setting.
- `--global` = apply to every repo for your user (not just this one).
- `credential.helper` = the setting name.
- `cache --timeout=3600` = keep it in memory for 3600 seconds.

Better: use your operating system's secure storage, so it is remembered
until you revoke it.

```bash
# Option 2: macOS Keychain
git config --global credential.helper osxkeychain
```

The modern cross-platform choice is **Git Credential Manager (GCM)**. It
works on Windows, macOS, and Linux and can log you in through your
browser, so you never create a raw token by hand:
https://github.com/git-ecosystem/git-credential-manager

```bash
git config --global credential.helper manager     # after installing GCM
```

Check what is set:

```bash
git config --global credential.helper
# osxkeychain
```

Once a helper is set, Git asks for your token **once**, stores it through
the helper, and reuses it until it expires or you revoke it on GitHub.

> **Doubt? "Do I need this if I use SSH?"**
> No. SSH keys authenticate silently every time, so there is nothing to
> cache. Credential helpers only matter for HTTPS.

---

## `fetch` vs `pull` vs `push` — The Confusion, Resolved

Three commands, three different jobs. We will add them one at a time.

### New idea 1: the remote-tracking branch (`origin/main`)

**Why do we need this?** Git works offline. When you are not connected,
Git still needs a way to remember "what did GitHub's `main` look like the
last time I checked?"

**Analogy.** A photo of a departures board. The photo is on your phone
(local), but it shows what the board looked like at that moment. The
real board may have changed since.

**Plain explanation.** `origin/main` is a **remote-tracking branch**: a
read-only bookmark stored on your laptop that records where GitHub's
`main` was the last time you talked to GitHub. You never commit on it
directly. Git updates it only when you fetch, pull, or push.

**How to read this:** three rows: your own `main`, your photo `origin/main`, and the real GitHub `main`.

```
    YOUR COMPUTER
    A---B---C                     main         (row 1: your branch)
    A---B---C                     origin/main  (row 2: photo of row 3)
    GITHUB (origin)
    A---B---C                     main         (row 3: the real one)
```

So on your laptop there are **two different things** with similar names:
`main` (your own branch) and `origin/main` (your photo of GitHub's).

### New idea 2: `git fetch` — look, don't touch

**Analogy.** Checking the departures board and taking a new photo. You
learn what changed, but you do not board any train.

**Plain explanation.** `git fetch` downloads new commits from the remote
and updates `origin/main`. It does **not** change your own `main` or your
working files. It is always safe.

```bash
git fetch origin
```

Expected output when there is news:

```
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 270 bytes | 90.00 KiB/s, done.
From github.com:yourname/your-repo
   3f2a91c..8b4d7e2  main       -> origin/main
```

Line by line:
- `From github.com:...` = where it downloaded from.
- `3f2a91c..8b4d7e2` = your photo moved from commit `3f2a91c` to
  `8b4d7e2` (short commit IDs).
- `main -> origin/main` = the remote's `main` was copied into your
  `origin/main` bookmark.

If there is nothing new, fetch prints nothing at all.

**How to read this:** three rows per picture: your `main`, your photo `origin/main`, and the real GitHub `main`. Time flows left to right.

BEFORE `git fetch` (a teammate pushed D and E; your photo is stale):
```
    YOUR COMPUTER
    A---B---C                     main
    A---B---C                     origin/main (stale photo)
    GITHUB (origin)
    A---B---C---D---E             main (real)

    main vs origin/main: ahead 0, behind 0  (looks in sync, but stale)
```

Command: `git fetch origin`

AFTER:
```
    YOUR COMPUTER
    A---B---C                     main
    A---B---C---D*--E*            origin/main (photo updated)
    GITHUB (origin)
    A---B---C---D---E             main (real)

    main vs origin/main: ahead 0, behind 2
```

**What changed:** only the `origin/main` pointer moved (C -> E). D* and E*
are new on your laptop. Your `main` and your files did not change.

Now look at what arrived, without merging:

```bash
git log main..origin/main --oneline
```

Expected output:

```
8b4d7e2 Fix typo in README
a1c90d3 Add login test
```

Reading `main..origin/main`: "commits that are in `origin/main` but **not**
in my `main`". In plain words, the incoming commits. `--oneline` prints
each commit on one line (short ID + message). Empty output means nothing
is incoming.

### New idea 3: `git pull` — fetch, then merge

**Why do we need this?** After looking, you usually want the new work in
your branch.

**Plain explanation.** `git pull` = `git fetch` **followed by**
`git merge origin/main` (merge = combine another line of commits into
your current branch, Lesson 05). It downloads and then immediately
applies the changes to your current branch. With `--rebase` it replays
your commits on top instead of merging (Lesson 09).

```bash
git pull
```

Expected output:

```
Updating 3f2a91c..8b4d7e2
Fast-forward
 README.md | 2 +-
 tests/test_login.py | 10 ++++++++++
 2 files changed, 11 insertions(+), 1 deletion(-)
```

Line by line:
- `Updating 3f2a91c..8b4d7e2` = your `main` moves from the old commit to
  the new one.
- `Fast-forward` = your branch had nothing new of its own, so Git just
  slid the label forward. No merge commit needed.
- The file list = which files changed and how many lines were added or
  removed.

If you had local commits that the remote did not have, Git would create a
merge commit instead (and could show a conflict, Lesson 05).

**How to read this:** same three rows. First the fast-forward case (you have no commits of your own).

BEFORE `git pull` (right after a fetch; you are behind by 2):
```
    YOUR COMPUTER
    A---B---C                     main
    A---B---C---D*--E*            origin/main (photo updated)
    GITHUB (origin)
    A---B---C---D---E             main (real)

    main vs origin/main: ahead 0, behind 2
```

Command: `git pull`   (= `git fetch`, then `git merge origin/main`)

AFTER:
```
    YOUR COMPUTER
    A---B---C---D*--E*            main (moved C -> E)
    A---B---C---D---E             origin/main
    GITHUB (origin)
    A---B---C---D---E             main (real)

    main vs origin/main: ahead 0, behind 0
```

**What changed:** only your `main` label slid forward (C -> E). D* and E*
now belong to your branch. No new commit was created (fast-forward).

If you also have your own commit F, the histories have diverged (ahead 1,
behind 2) and pull must create a merge commit M*.

BEFORE (diverged):
```
    YOUR COMPUTER
                F                 main
              /
    A---B---C
              \
                D---E             origin/main
    GITHUB (origin)
    A---B---C---D---E             main (real)

    main vs origin/main: ahead 1, behind 2
```

Command: `git pull`

AFTER:
```
    YOUR COMPUTER
                D---E             origin/main
              /      \
    A---B---C---F-----M*          main
    GITHUB (origin)
    A---B---C---D---E             main (real)

    main vs origin/main: ahead 2, behind 0
```

**What changed:** your `main` moved to the new merge commit M* (parents F
and E). `origin/main` and GitHub did not change. You are now ahead 2
(F and M*, the two commits GitHub lacks), so the next step is a push.

### New idea 4: `git push` — upload

**Plain explanation.** `git push` uploads your local commits to the
remote and moves the remote branch forward to match yours.

```bash
git push origin main
```

Expected output:

```
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Writing objects: 100% (3/3), 310 bytes | 310.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0), pack-reused 0
To github.com:yourname/your-repo.git
   8b4d7e2..c5e9f10  main -> main
```

`8b4d7e2..c5e9f10 main -> main` = the remote's `main` moved from
`8b4d7e2` to `c5e9f10`. Format: `old..new  local -> remote`.

`git push origin main` reads as: "push my `main` to the remote named
`origin`, onto its branch `main`."

**How to read this:** you made D and E locally; GitHub has not seen them yet.

BEFORE `git push`:
```
    YOUR COMPUTER
    A---B---C---D---E             main
    A---B---C                     origin/main
    GITHUB (origin)
    A---B---C                     main (real)

    main vs origin/main: ahead 2, behind 0
```

Command: `git push origin main`

AFTER:
```
    YOUR COMPUTER
    A---B---C---D---E             main
    A---B---C---D*--E*            origin/main (photo updated)
    GITHUB (origin)
    A---B---C---D*--E*            main (real)

    main vs origin/main: ahead 0, behind 0
```

**What changed:** the real GitHub `main` moved (C -> E) and Git moved your
photo `origin/main` to match. D* and E* are now on GitHub. Your `main` did
not move.

### All together

**How to read this:** arrows show which command moves data in which direction.

```
  YOUR COMPUTER                          GITHUB (origin)
  +----------------------+               +------------------+
  | main                 | --- push ---> | main (real)      |
  |   ^                  |               |                  |
  |   | merge            |               |                  |
  | origin/main (photo)  | <-- fetch --- |                  |
  +----------------------+               +------------------+
    git pull = git fetch + git merge
    git push also moves your photo origin/main to match
```

| Command | Talks to GitHub? | Changes `origin/main`? | Changes your `main`/files? |
|---|---|---|---|
| `git fetch` | downloads | yes | **no** |
| `git pull` | downloads | yes | **yes** (merge) |
| `git push` | uploads | yes (to match) | no |

Python analogy: `fetch` is like `git log` on a remote (read-only),
`pull` is like updating your dependencies and applying them immediately,
and `push` is like publishing a package release.

> **Common confusion: "Is `git pull` safer than `git fetch`?"**
> The opposite. `fetch` cannot hurt anything. `pull` changes your branch
> and can cause merge conflicts. If unsure what a pull will bring, fetch
> first and inspect.

```bash
git fetch origin                       # safe: just look
git log main..origin/main --oneline    # what is incoming?
git pull                               # fetch + merge (most common)
git pull --rebase                      # fetch + rebase (cleaner history, Lesson 09)
git push origin main                   # upload main to origin's main
git push                               # shorthand once tracking is set up
```

**Beginner-safe habit:** when unsure, `fetch` first, then run
`git log main..origin/main` to see exactly which commits you do not have
yet, and only then merge.

---

## Tracking Branches

**Why do we need this?** Without extra setup, Git does not know which
remote branch your local `main` should talk to, so you must type
`git push origin main` every time.

**Analogy.** Speed-dial. Set the number once, then press one button.

**Plain explanation.** `-u` is short for `--set-upstream`. In
`git push -u origin main` you tell Git: "my local `main` corresponds to
`origin`'s `main`; remember that." The remote branch is then called the
**upstream** of your local branch. After this, plain `git push` and
`git pull` work with no arguments.

See the relationships:

```bash
git branch -vv
```

Expected output:

```
* main 3f2a91c [origin/main] Add retry logic
  feature/login 9d12ab4 [origin/feature/login: ahead 2] Add login form
```

Line by line:
- `*` = the branch you are on.
- `main` = local branch name; `3f2a91c` = the commit it points to.
- `[origin/main]` = its upstream. Nothing extra inside the brackets means
  you are in sync.
- `[origin/feature/login: ahead 2]` = you have 2 local commits that are
  not on the remote yet. Time to push.
- `behind 3` would mean the remote has 3 commits you do not have. Time to
  pull.
- `ahead 2, behind 3` means both; you must integrate before pushing.
- Trailing text = the latest commit message.

Glance at this before every push or pull. (Ahead/behind counts are based
on your last fetch, so run `git fetch` first for fresh numbers.)

**How to read this:** the `branch -vv` numbers as a picture (diverged: ahead 2, behind 3).

```
    YOUR COMPUTER
                F---G             main          (ahead 2)
              /
    A---B---C
              \
                D---E---H         origin/main   (behind 3)
```

**What changed:** nothing; this only shows what the numbers count. Ahead =
commits only you have (F, G). Behind = commits only the remote has (D, E, H).

For a brand-new local branch you want to share:

```bash
git push -u origin feature/my-branch    # first push: creates it remotely and sets tracking
git push                                 # every push after that
```

> **Common confusion: "Why did Git say 'The current branch has no
> upstream branch'?"**
> You created a branch locally and ran plain `git push`. Git does not
> know where to send it. Run the `git push -u origin <branch>` command
> that Git suggests in the error message, once.

---

## Force Push — Understanding the Actual Danger

**Why do we need this?** Sometimes you rewrite history (for example
`git commit --amend` or a rebase). Rewriting creates **new** commits with
new IDs that replace old ones. The remote still has the old ones, so a
normal push is refused.

**Analogy.** A shared Google Doc. You printed a copy, edited the copy,
and now want to overwrite the online doc with your version. If a
coworker edited the online doc meanwhile, you would erase their work.

First, see the refusal (this is Git protecting you):

```
To github.com:yourname/your-repo.git
 ! [rejected]        main -> main (non-fast-forward)
error: failed to push some refs to 'github.com:yourname/your-repo.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart.
```

`non-fast-forward` means: the remote branch has commits that your branch
does not contain in its history, so moving the remote label to yours
would throw commits away. Git refuses by default.

**How to read this:** a teammate pushed D and E to GitHub and you never fetched. You committed F.

BEFORE `git push` (your stale photo says ahead 1, but the real remote moved):
```
    YOUR COMPUTER
    A---B---C---F                 main
    A---B---C                     origin/main (stale photo)
    GITHUB (origin)
    A---B---C---D---E             main (real)

    main vs origin/main (stale): ahead 1, behind 0
```

Command: `git push origin main`   -> ! [rejected] (non-fast-forward)

AFTER (nothing moved; the push was refused):
```
    YOUR COMPUTER
    A---B---C---F                 main
    A---B---C                     origin/main (stale photo)
    GITHUB (origin)
    A---B---C---D---E             main (real)

    GitHub still ends at E; F never reached it
```

**What changed:** nothing. To move GitHub `main` from E to F, Git would have
to drop D and E, so it refuses. Fix: `git fetch` (photo becomes C---D---E,
so you see ahead 1, behind 2), then `git pull` (merge as above), then push.

To overwrite on purpose:

```bash
git push --force            # DANGEROUS: overwrites whatever is on the remote
git push --force-with-lease # SAFER: refuses if the remote changed since you last fetched
```

**How to read this:** you amended C into a new commit C' (new id, same place in history).

BEFORE `git push --force-with-lease`:
```
    YOUR COMPUTER
            C'                    main (amended commit)
          /
    A---B
          \
            C                     origin/main
    GITHUB (origin)
    A---B---C                     main (real)

    main vs origin/main: ahead 1, behind 1
```

Command: `git push --force-with-lease origin main`

AFTER:
```
    YOUR COMPUTER
    A---B---C'*                   main
    A---B---C'*                   origin/main
    GITHUB (origin)
    A---B---C'*                   main (real)

    old C: on no branch any more (orphaned)

    main vs origin/main: ahead 0, behind 0
```

**What changed:** the real GitHub `main` label jumped from C to C'* (not a
fast-forward) and `origin/main` followed. Old C is on no branch now;
anyone who had pulled C now disagrees with GitHub.

**The real danger.** If a teammate already pulled the *old* commits and
you force-push new ones, their local history no longer matches the
remote. They get confusing conflicts, or they may push the old commits
back and silently undo your fix.

**Why `--force-with-lease` is safer.** It means "overwrite, but only if
the remote branch is still exactly what I last saw." If a teammate pushed
something new that you have not fetched, it refuses. Plain `--force`
would erase their work without warning.

Expected output on success:

```
Enumerating objects: 5, done.
...
To github.com:yourname/your-repo.git
 + 8b4d7e2...d41f0aa feature/login -> feature/login (forced update)
```

`+` and `(forced update)` tell you history was overwritten. The three dots
`...` (instead of two) signal a non-fast-forward update.

**Rule of thumb:**
- Your **own personal feature branch** that nobody else uses: force-push
  is fine and common (especially after a rebase).
- **`main`/`master`** or any shared branch: almost always wrong. Most
  teams turn on GitHub **branch protection** (a setting that blocks force
  pushes and requires reviews; Lesson 11).
- Always prefer `--force-with-lease` over `--force`.

> **Common confusion: "Does force-push delete my teammate's commits?"**
> It moves the remote branch label to your commit. Commits not reachable
> from that label are no longer visible on the branch. They are not
> instantly destroyed, but they are hard to find and recover.

---

## Exercise

Expected results are given for each step.

1. Create a fresh local repo, make 2-3 commits, and push it to a **new**
   empty GitHub repository using the SSH method above.
   *Expected:* the push ends with `branch 'main' set up to track
   'origin/main'`, and the files appear on the GitHub page.
2. On GitHub's website, use the "Edit" pencil icon on a file to change
   one line and commit it in the browser (this simulates a teammate).
   *Expected:* GitHub shows a new commit at the top of the history.
3. Locally, run `git fetch`, then `git log main..origin/main --oneline`.
   *Expected:* fetch prints a `main -> origin/main` line; the log shows
   exactly 1 incoming commit, and your file on disk is still unchanged.
4. Run `git pull`.
   *Expected:* a `Fast-forward` message, and your local file now matches
   the browser edit. `git log main..origin/main` now prints nothing.
5. Make a local commit and push it. Then run `git commit --amend` to
   change its message, and `git push`.
   *Expected:* the second push is **rejected** with
   `! [rejected] ... (non-fast-forward)`. Fix it with
   `git push --force-with-lease`; expected output ends with
   `(forced update)`.
6. Run `git branch -vv` before step 5's amend and after.
   *Expected:* `[origin/main]` in sync before pushing; after the amend
   you see `ahead 1, behind 1`, which is why the push was rejected.

## Recap in 5 lines
1. A **remote** is a nickname (usually `origin`) for a URL of another copy of your repo.
2. `origin/main` is your local photo of GitHub's `main`; `fetch` refreshes the photo only, `pull` = fetch + merge, `push` uploads.
3. `git push -u origin <branch>` sets the upstream so plain `git push`/`git pull` work later.
4. SSH (private key on your laptop, public key on GitHub) removes password prompts; credential helpers do the same for HTTPS.
5. Force-push only on your own branches, and prefer `--force-with-lease`.

Continue to [Lesson 08 — Collaboration: Forks, Pull Requests & Code Review](08-collaboration-pull-requests.md).
