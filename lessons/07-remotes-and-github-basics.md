# Lesson 07 — Remotes & GitHub Basics: clone, push, pull, fetch

## Goal
Connect your local Git knowledge to GitHub. Understand exactly what
push/pull/fetch each do (they're commonly confused), set up authentication
properly, and never fear force-push again by knowing when it's actually
dangerous.

## Prerequisites
[Lesson 06 — Undoing Things](06-undoing-things.md). You'll also need a
free GitHub account (https://github.com).

## After This Lesson You Will Be Able To
- Clone a repo, or connect an existing local repo to GitHub
- Explain the difference between fetch, pull, and push
- Set up SSH authentication (no more typing passwords)
- Understand remote-tracking branches

---

## What Is a "Remote"?

A **remote** is just a nickname Git gives to another copy of the
repository, usually hosted somewhere else (GitHub, GitLab, a teammate's
machine, even another folder on your own disk). The default nickname for
your main remote is `origin` — but that's only a convention, not a
special keyword.

```bash
git remote -v
# origin  https://github.com/yourname/your-repo.git (fetch)
# origin  https://github.com/yourname/your-repo.git (push)
```

You can have multiple remotes (e.g. `origin` for your fork, `upstream` for
the original project you forked from — this becomes essential in
Lesson 08).

---

## Two Ways to Get Started

### Option A: Clone an Existing GitHub Repo

```bash
git clone https://github.com/someuser/some-repo.git
cd some-repo
```

This downloads the **entire history** (every commit, ever — remember,
Git is distributed, Lesson 01) and automatically sets up `origin` pointing
back to that URL.

### Option B: Connect a Local Repo You Already Have

If you've been following along and have a local repo from earlier
lessons with no remote yet:

1. On GitHub: click **New repository**, give it a name, do **not**
   initialize with a README (since you already have local commits — this
   avoids an unnecessary conflict).
2. GitHub shows you the exact commands, essentially:

```bash
git remote add origin https://github.com/yourname/your-repo.git
git branch -M main               # ensure your local branch is named "main"
git push -u origin main          # push, and set up tracking (see below)
```

---

## SSH vs HTTPS — Set This Up Once, Never Type a Password Again

GitHub supports two ways to authenticate: HTTPS (with a token) or SSH
(with a key pair). **SSH is the professional default** — set it up once
per machine.

```bash
# 1. Generate a key pair (if you don't already have one):
ssh-keygen -t ed25519 -C "you@example.com"
# press Enter through the prompts (default location is fine); optionally set a passphrase

# 2. Copy your PUBLIC key:
cat ~/.ssh/id_ed25519.pub

# 3. On GitHub: Settings → SSH and GPG keys → New SSH key → paste it in

# 4. Test it:
ssh -T git@github.com
# "Hi yourname! You've successfully authenticated..."
```

Now clone/add remotes using the SSH URL instead of HTTPS:

```
git@github.com:yourname/your-repo.git      ← SSH (no password prompts, ever)
https://github.com/yourname/your-repo.git   ← HTTPS (needs a Personal Access Token as your "password")
```

If you already have a remote set up with HTTPS and want to switch:

```bash
git remote set-url origin git@github.com:yourname/your-repo.git
```

**Never put a real password in the HTTPS URL or in any Git prompt** —
GitHub disabled password auth for Git operations years ago specifically
because of this. If you must use HTTPS, generate a **Personal Access
Token** (Settings → Developer settings → Personal access tokens) and use
that as the password when prompted.

---

## Credential Helpers — Stop Re-Entering Your Token

If you're on HTTPS, typing your Personal Access Token on every single
`push`/`pull` gets old fast. A **credential helper** is a small program
Git delegates credential storage to, instead of asking you every time.

```bash
# Cache it in memory for a while (simplest option, no install needed):
git config --global credential.helper 'cache --timeout=3600'   # remembers for 1 hour
```

Better: use your OS's native secure storage, so it's remembered
indefinitely (until you explicitly revoke it):

```bash
# macOS — stores the token in your Keychain:
git config --global credential.helper osxkeychain
```

The modern cross-platform recommendation is **Git Credential Manager
(GCM)** — it works identically on Windows, macOS, and Linux, and can
handle GitHub's browser-based OAuth login flow directly, so you never
even generate a raw token by hand:
https://github.com/git-ecosystem/git-credential-manager

```bash
git config --global credential.helper manager     # after installing GCM
```

Once any of these is set, Git prompts for your token **once**, stores it
via the helper, and reuses it silently on every future HTTPS operation —
until it expires or you revoke it on GitHub.

**If you followed this lesson's recommendation and set up SSH instead,**
none of this applies to you — SSH keys authenticate silently every time,
with nothing to cache in the first place.

---

## `fetch` vs `pull` vs `push` — The Confusion, Resolved

```
                          origin/main  (remote-tracking branch — a LOCAL
                              │          bookmark of where the remote's
                              │          main branch was, last time you checked)
    ┌─────────────────────────┼─────────────────────────┐
    │         YOUR MACHINE     │      GITHUB (origin)     │
    │                          │                          │
    │   main ──git push──────► │ ◄────── main             │
    │   main ◄──git pull────── │ ────── main               │
    └──────────────────────────┴──────────────────────────┘
```

- **`git fetch`** — downloads new commits from the remote, and updates
  your local `origin/main` bookmark — but does **NOT** touch your actual
  `main` branch or your working files. Completely safe, always. Think of
  it as "check for updates without applying them."
- **`git pull`** — is literally `git fetch` **followed by** `git merge
  origin/main` (or a rebase, if configured — see Lesson 09). It fetches
  *and* immediately integrates the changes into your current branch.
- **`git push`** — uploads your local commits to the remote, advancing
  the remote branch to match yours.

```bash
git fetch origin              # safe, always — just look, don't touch
git log origin/main           # inspect what's new without merging yet
git pull                       # fetch + merge, in one step (most common)
git pull --rebase              # fetch + rebase instead of merge (cleaner history — Lesson 09)
git push origin main           # push your main to origin's main
git push                       # shorthand, once tracking is set up (below)
```

**Beginner-safe habit:** if you're ever unsure what a `pull` will do,
`fetch` first and inspect with `git log main..origin/main` (shows exactly
what commits are incoming that you don't have yet) before merging.

---

## Tracking Branches

When you run `git push -u origin main` (the `-u` is `--set-upstream`),
you're telling Git: "remember that my local `main` corresponds to
`origin`'s `main`." After that, plain `git push` and `git pull` (no
arguments) know exactly where to go.

```bash
git branch -vv
# main   3f2a91c [origin/main] Add retry logic
#                  ↑ shows tracking relationship + whether you're ahead/behind
```

If it says `[origin/main: ahead 2]`, you have 2 local commits not yet
pushed. `[origin/main: behind 3]` means the remote has 3 commits you
don't have locally yet. This is one of the most useful things to glance at
before pushing or pulling.

For any *new* branch you create locally and want to share:

```bash
git push -u origin feature/my-branch    # first push, sets up tracking
git push                                 # every push after that
```

---

## Force Push — Understanding the Actual Danger

```bash
git push --force            # DANGEROUS if others have pulled the commits you're overwriting
git push --force-with-lease # SAFER — refuses if the remote has commits you don't know about
```

Force-push is needed whenever you've rewritten local history (via
`reset`, `amend`, or `rebase` — Lesson 09) and need to overwrite what's on
the remote to match. The danger: if a teammate already pulled the
*old* commits, force-pushing new ones causes their local history to
diverge silently, leading to confusing conflicts or, worse, someone
accidentally force-pushing the old commits back.

**Rule of thumb:**
- Force-pushing to **your own personal feature branch** that nobody else
  is using — totally fine, common, expected (especially after rebasing).
- Force-pushing to **`main`/`master`** or any shared branch — almost
  always wrong, and most professional teams configure GitHub to outright
  block it (branch protection — Lesson 11).
- Always prefer `--force-with-lease` over plain `--force` — it double
  checks the remote hasn't changed since you last fetched, protecting you
  from clobbering a teammate's just-pushed work you haven't seen yet.

---

## Exercise

1. Create a fresh local repo, make 2-3 commits, and push it to a **new**
   GitHub repository using the SSH method above.
2. On GitHub's website, use the "Edit" pencil icon to make a tiny change
   directly in the browser and commit it there (this simulates a
   teammate's change).
3. Locally, run `git fetch` then `git log main..origin/main` — confirm you
   can see the incoming commit *before* merging it.
4. Run `git pull` and confirm your local file now matches.
5. Make a local commit, then deliberately run `git commit --amend` to
   change its message, then `git push` (expect it to be **rejected** —
   Git protects you from silently overwriting remote history!). Fix it
   properly with `git push --force-with-lease`.

Continue to [Lesson 08 — Collaboration: Forks, Pull Requests & Code Review](08-collaboration-pull-requests.md).
