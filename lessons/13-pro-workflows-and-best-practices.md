# Lesson 13 — Pro Workflows & Best Practices

## Goal
The habits, configuration, and troubleshooting instincts that mark the
top 1% of Git users: commit hygiene, `.gitignore` mastery, hooks,
signing commits, and how to stay calm and methodical when something goes
badly wrong.

## Prerequisites
[Lesson 12 — Submodules, Subtrees & Large-Repo Tooling](12-submodules-subtrees-and-large-repos.md),
and really all previous lessons — this is the capstone.

## After This Lesson You Will Be Able To
- Configure Git properly for daily professional use
- Write `.gitignore` files that actually work
- Automate checks with Git hooks
- Sign commits for verified authorship
- Diagnose and recover from real-world Git disasters methodically

---

## `.gitignore` — Keeping Junk Out of Your Repo

Never commit: build artifacts, dependency folders, secrets, OS/editor
clutter, logs. `.gitignore` tells Git to simply not track these.

```gitignore
# Python
__pycache__/
*.pyc
.venv/
venv/
*.egg-info/

# Environment / secrets — NEVER commit these
.env
.env.local
*.pem
*.key

# Editor/OS
.DS_Store
.vscode/
.idea/

# Build output
dist/
build/
*.log
```

Rules:
- One pattern per line. `#` starts a comment. `!pattern` **un-ignores** a
  previously-ignored pattern (useful for exceptions).
- `.gitignore` only affects **untracked** files. If a file is already
  tracked, adding it to `.gitignore` does nothing until you also run
  `git rm --cached <file>` to stop tracking it (the file stays on disk,
  it just leaves Git's tracking).
- GitHub maintains great starter templates per language:
  https://github.com/github/gitignore — use one instead of writing from
  scratch.
- Global ignores (apply to every repo on your machine, e.g. `.DS_Store`
  everywhere): `git config --global core.excludesfile ~/.gitignore_global`

### If You Already Committed a Secret

**Do not just delete it in a new commit** — Lesson 03 taught you the old
blob still exists in history forever, retrievable by anyone with the
repo. You must actually remove it from history (`git filter-repo` is the
modern recommended tool, replacing the older `filter-branch`) **and**
rotate/revoke the leaked credential immediately — treat any leaked
secret as compromised regardless of whether you scrub history.

```bash
# using git-filter-repo (install via: brew install git-filter-repo)
git filter-repo --path secrets.env --invert-paths     # removes it from EVERY commit, forever
git push --force        # rewrites remote history — coordinate with your team first!
```

---

## Git Config Worth Setting Up Once

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main         # modern default, but good to be explicit
git config --global core.editor "code --wait"        # use VS Code for commit messages, rebases, etc.
git config --global pull.rebase false                # explicit choice: merge on pull (or `true` to always rebase)
git config --global fetch.prune true                  # auto-clean deleted remote branches on fetch
git config --global color.ui auto                    # colored output (usually on by default already)

# Aliases — huge time savers:
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.last "log -1 HEAD"
```

After the alias setup: `git lg` gives you the full graph view from
Lesson 05 with four keystrokes.

Per-repo overrides (e.g. a different email for work vs personal projects)
go in the repo's local config, which always wins over global:

```bash
cd work-project
git config user.email "you@company.com"     # no --global → only applies here
```

---

## Git Hooks — Automating Checks Locally

Hooks are scripts in `.git/hooks/` that run automatically at specific
points. They're **not** committed by default (they live outside the
tracked file tree) — for team-shared hooks, use a tool like
[pre-commit](https://pre-commit.com/) or [Husky](https://typicode.github.io/husky/)
that manages this for you.

```bash
# .git/hooks/pre-commit  (make executable: chmod +x .git/hooks/pre-commit)
#!/bin/sh
pytest -q || { echo "Tests failed — commit blocked."; exit 1; }
```

Common hook points: `pre-commit` (before a commit is created — great for
linting/tests), `commit-msg` (validate message format), `pre-push`
(before pushing — great final gate).

Using `pre-commit` (the popular Python-based framework) instead, so hooks
are versioned and shared with your team via `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.4.2
    hooks:
      - id: black
  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
```

```bash
pip install pre-commit
pre-commit install       # sets up the git hook for everyone who runs this once
```

---

## `.gitattributes` — Line Endings and File Handling

Windows uses `CRLF` (`\r\n`) to end lines; Mac and Linux use `LF`
(`\n`). If one teammate is on Windows and commits a file with CRLF
endings while everyone else uses LF, `git diff` can show **every single
line as changed** even though nothing meaningful happened — pure noise
that buries the one real line someone actually needs to review.

The per-person fix is `core.autocrlf`:

```bash
git config --global core.autocrlf true    # Windows: LF→CRLF on checkout, CRLF→LF on commit
git config --global core.autocrlf input   # Mac/Linux: CRLF→LF on commit, leaves checkout alone
```

The problem with `core.autocrlf`: it's a personal setting. It only helps
if *everyone* on the team remembers to set it correctly for their own
OS — one person who doesn't will still poison history for everyone else.
The team-wide, repo-level fix is a **committed `.gitattributes` file**,
so the behavior is defined once, for the repo, regardless of anyone's
personal config:

```gitattributes
# .gitattributes
* text=auto              # let Git auto-detect text files and normalize their line endings
*.sh text eol=lf          # shell scripts must always keep LF, even checked out on Windows
*.bat text eol=crlf       # batch scripts must always keep CRLF, even checked out on Mac/Linux
*.png binary              # never treat as text — no line-ending conversion, no text-style diffing
```

You've actually already seen this exact file: the `git lfs track`
command earlier in this lesson writes its rules into `.gitattributes`
too (`*.psd filter=lfs diff=lfs merge=lfs -text`) — it's one file serving
double duty as both "how should Git normalize line endings" and "which
files does LFS manage," and it's worth committing early on any repo with
contributors on more than one OS, before the first CRLF-noise diff ever
happens.

---

## Signed Commits — Verified Authorship

Anyone can `git config user.name "Linus Torvalds"` and commit under a
fake identity — Git doesn't verify names/emails by default. **GPG or SSH
commit signing** cryptographically proves a commit really came from you.

```bash
# SSH signing (simpler, reuses your existing SSH key from Lesson 07):
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true      # sign every commit automatically

git commit -m "Signed commit"     # now shows "Verified" badge on GitHub
```

Add the same key as a **"Signing Key"** (not just an auth key) in GitHub
Settings → SSH and GPG keys for the "Verified" badge to actually appear
on GitHub. Many companies now require signed commits on `main` via branch
protection (Lesson 11).

---

## Commit Message Conventions at Scale: Conventional Commits

Many teams standardize commit prefixes so tooling can auto-generate
changelogs and version bumps:

```
feat: add search filter to results page
fix: guard against None response in fetch_status()
docs: update README setup instructions
refactor: extract retry logic into its own module
test: add coverage for empty-input edge case
chore: bump requests to 2.32
```

Tools like `semantic-release` parse these prefixes to auto-determine
whether a release is a major/minor/patch version bump — worth adopting
even without the tooling, purely for readability of `git log`.

---

## Handling Large Files: Git LFS

Git is fundamentally bad at large binary files (videos, datasets, model
weights) — every version of a big file is stored in full (Lesson 03),
bloating the repo forever. **Git LFS** (Large File Storage) replaces
large files with lightweight pointers, storing actual content separately.

```bash
git lfs install
git lfs track "*.psd"          # or *.mp4, *.onnx, etc.
git add .gitattributes          # this file records what LFS tracks — must be committed
git add design.psd
git commit -m "Add design file via LFS"
```

---

## Disaster Recovery Playbook — Staying Calm

When something goes wrong, work through this in order, every time:

1. **Stop. Don't run more commands yet.** Most Git disasters get worse
   because someone panics and runs three more commands trying to fix it,
   compounding the mess.
2. **`git status`** — what state are you actually in right now?
3. **`git log --oneline --all --graph`** — what does history actually
   look like? Is the "lost" commit visible anywhere?
4. **`git reflog`** (Lesson 06) — if a commit or branch seems to have
   vanished, this is almost always where it still is.
5. **If you're mid-merge/rebase/cherry-pick and it's a mess:**
   `git merge --abort` / `git rebase --abort` / `git cherry-pick --abort`
   — these safely return you to your exact starting point.
6. **Before any `--hard` reset or `--force` push**, ask: "could this
   erase something someone else needs, or something not yet recoverable
   via reflog?" If genuinely unsure, make a safety branch first:
   `git branch backup-before-i-mess-this-up`
7. **When truly stuck**, it's fine to just re-clone a fresh copy of the
   remote into a new folder and manually copy your uncommitted work over
   — a completely valid "nuclear option" that costs you nothing except a
   few minutes, and guarantees a clean slate.

---

## The Top-1% Checklist

- [ ] Commits are small, focused, and have imperative-mood messages
- [ ] `.gitignore` is set up before the first commit, not after a secret
      leaks
- [ ] Feature branches, never direct commits to `main`
- [ ] PRs are small enough to review in one sitting
- [ ] `git status` before anything destructive
- [ ] Rebase only unpushed/personal history; revert (never reset) for
      shared history mistakes
- [ ] `--force-with-lease`, never bare `--force`
- [ ] Branch protection + CI required checks on every shared repo
- [ ] Know `reflog` exists and trust it instead of panicking
- [ ] Comfortable enough with the object model (Lesson 03) that no Git
      command feels like unexplainable magic anymore

---

## Exercise — Capstone

Set up one real repo, end to end, using everything from this course:

1. `git init`, write a proper `.gitignore` for your language, initial
   commit, connect to a new GitHub repo via SSH.
2. Set up `git config` aliases and enable SSH commit signing.
3. Add a GitHub Actions workflow running at least a linter.
4. Turn on branch protection requiring PRs + passing checks.
5. Create a feature branch, make several small, well-messaged commits,
   clean them up with interactive rebase, open a PR via `gh pr create`,
   review it yourself, and merge with squash.
6. Deliberately create a merge conflict, resolve it properly.
7. Deliberately run a `git reset --hard` you regret, then recover fully
   using `git reflog` — no help, no notes, from memory.

If you can do all seven from memory without looking anything up, you're
genuinely in the top 1% of Git users — most professional engineers never
go past steps 1 and 5.

---

Course complete. See [What Next](../what-next.md) for where advanced Git
practice goes from here (monorepos, trunk-based development, GitOps).
