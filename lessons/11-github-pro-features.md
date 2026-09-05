# Lesson 11 — GitHub Pro Features: Actions, Branch Protection, CLI

## Goal
Go beyond basic push/pull into what makes GitHub a full engineering
platform: automated CI/CD with Actions, repo governance with branch
protection, and driving all of it from the terminal with the GitHub CLI.

## Prerequisites
[Lesson 10 — Stash, Tags & Bisect](10-stash-tags-bisect.md)

## After This Lesson You Will Be Able To
- Write a basic GitHub Actions workflow that runs tests on every PR
- Configure branch protection rules so `main` can't be broken accidentally
- Use `gh` (GitHub CLI) to create PRs, review, and manage issues without
  leaving the terminal
- Know what CODEOWNERS, Issues, and Projects are for
- Explain what GitHub Pages, Actions runners, and GitHub's pricing tiers
  actually are

---

## GitHub Actions — CI/CD, Defined in Your Repo

GitHub Actions runs scripts automatically in response to repo events
(push, PR opened, schedule, etc.). Workflows live in
`.github/workflows/*.yml`.

```yaml
# .github/workflows/test.yml
name: Run Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # clones your repo onto the runner
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: pytest
```

Anatomy:
- **`on:`** — the trigger(s). Common ones: `push`, `pull_request`,
  `schedule` (cron syntax), `workflow_dispatch` (manual "Run workflow"
  button).
- **`jobs:`** — one or more independent units of work; each runs on a
  fresh virtual machine (`runs-on:`).
- **`steps:`** — sequential actions within a job. `uses:` runs a
  pre-built, reusable action (from the Actions Marketplace); `run:` runs
  a raw shell command.

Once this file exists on `main`, every PR automatically gets a ✅/❌ status
check — this is the mechanism that powers "all checks must pass before
merging" (see Branch Protection, below).

### A Slightly Richer Example: Matrix Testing

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt
      - run: pytest
```

This runs the entire job **three times in parallel**, once per Python
version — catching version-specific bugs before they reach anyone.

### Secrets

Never hardcode API keys/passwords in a workflow file (they're public if
your repo is public!). Store them in **Settings → Secrets and variables →
Actions**, then reference them:

```yaml
- run: deploy.sh
  env:
    API_KEY: ${{ secrets.PROD_API_KEY }}
```

---

## Branch Protection Rules — Making `main` Un-Breakable

**Settings → Branches → Add branch protection rule**, targeting `main`
(or `master`, or any pattern like `release/*`). Common rules a
professional team enables:

- **Require a pull request before merging** — nobody, including admins
  (optionally), can push straight to `main`.
- **Require approvals** — e.g. at least 1 or 2 reviewers must approve.
- **Require status checks to pass before merging** — your GitHub Actions
  workflow(s) above must succeed; a red ❌ physically blocks the merge
  button.
- **Require branches to be up to date before merging** — forces you to
  merge/rebase in the latest `main` first (Lesson 09), preventing "it
  passed CI but broke when combined with someone else's change."
- **Require conversation resolution before merging** — every review
  comment thread must be marked resolved.
- **Do not allow force pushes** / **Do not allow deletions** — protects
  history integrity on this branch specifically (directly addresses the
  force-push danger from Lesson 07).

This is the actual mechanism that turns "please don't push directly to
main" from a social convention (that gets violated under deadline
pressure) into something GitHub enforces automatically.

On a busy repo, branch protection alone still has a gap: two PRs can each
pass CI individually, get approved, and merge one after the other — yet
*combined*, they break `main`, because CI only ever validated each PR
against the `main` that existed before the other one landed. A **merge
queue** closes this gap by serializing merges: it re-runs CI against the
actual, up-to-date result of each merge, one at a time, before it's
allowed to land. Enable it under Settings → Branches → your branch
protection rule → "Require merge queue" — worth turning on the moment a
repo has enough contributors that same-day merge collisions become a
real (not theoretical) occurrence.

---

## GitHub's Security Suite: Dependabot, Code Scanning & Secret Scanning

Branch protection and CI keep `main` from breaking functionally — this
suite is about keeping it from becoming a *security* liability, and it's
mainstream, often default-on-for-public-repos functionality, not an
exotic add-on.

### Dependabot

Dependabot opens automated PRs in two situations: a dependency you use
has a disclosed vulnerability (**security updates** — Settings → Code
security → Dependabot), or a dependency is simply out of date (**version
updates**, configured explicitly via a committed file):

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
```

Once this exists, Dependabot opens a PR (reviewed like any other) every
time a newer `pip` package version is available — you stay current
without anyone manually babysitting `requirements.txt`.

### Code Scanning (CodeQL)

Code scanning runs static analysis on every push/PR and reports findings
as annotations right on the diff — the same visual spot as a failed CI
check. GitHub's setup wizard (Security → Code scanning → Set up) usually
generates the workflow for you at `.github/workflows/codeql.yml`; you
rarely hand-write it. Findings range from real vulnerabilities (SQL
injection, hardcoded credentials) to lower-confidence lint-style
warnings — treat it as another required status check, same as tests.

### Secret Scanning + Push Protection

GitHub scans repos for recognizable credential patterns (cloud provider
keys, common API token formats) both retroactively across existing
history and, with **push protection** enabled, *before* a push is even
accepted — the push is flat-out rejected with the offending line and
file named, before the secret ever enters history at all. This is the
mechanism that prevents Lesson 13's "If You Already Committed a Secret"
disaster-recovery scenario from happening in the first place on
GitHub-hosted repos — though it's not a substitute for `.gitignore`
discipline (Lesson 13) or for actually rotating a secret the moment you
suspect it leaked; push protection only catches patterns it recognizes,
and there's no substitute for never committing secrets locally to begin
with.

All three are enabled per-repository under **Settings → Code security**;
org admins can also enforce them account-wide so individual repos can't
opt out.

---

## CODEOWNERS — Automatic Review Assignment

A `.github/CODEOWNERS` file automatically requests review from the right
people whenever specific files/folders change:

```
# .github/CODEOWNERS
*.py            @backend-team
/frontend/      @frontend-team
/infra/         @platform-team
/docs/          @tech-writers
```

Combine with branch protection's "Require review from Code Owners" to
guarantee, e.g., infra changes always get a platform engineer's eyes
before merging — without anyone having to remember to tag them manually.

---

## Issues — Lightweight Tracking

GitHub Issues are for bugs, feature requests, and tasks. Key features:
- **Labels** (`bug`, `enhancement`, `good first issue`)
- **Assignees**
- **Milestones** (group issues into a release/sprint)
- **Linking to PRs** — writing `Closes #42` in a PR description
  auto-closes issue #42 when that PR merges (mentioned in Lesson 08)
- **Issue templates** — `.github/ISSUE_TEMPLATE/bug_report.md` pre-fills
  a structured form for reporters

## Projects — Kanban/Sprint Boards

GitHub Projects gives you a Kanban-style board (To Do / In Progress /
Done, or fully custom columns) built directly from your Issues and PRs —
useful if your team doesn't already live in Jira/Azure DevOps for this.

---

## GitHub the Platform: Hosting, Runners & Pricing

Three practical questions beginners have about GitHub-as-a-product,
answered briefly.

### GitHub Pages — Free Static Site Hosting From a Repo

**Settings → Pages** turns a branch (or a `/docs` folder, or a GitHub
Actions deploy step) into a live website at
`https://yourname.github.io/repo-name` — no server to manage. Works for
plain HTML/CSS/JS or a static-site generator's build output (Jekyll,
Hugo, MkDocs). Common uses: project documentation, a portfolio, or —
like this course — rendering a `lessons/` folder as browsable pages.
Custom domains are supported too.

### What Machine Your Actions Workflow Actually Runs On

`runs-on: ubuntu-latest` (or `windows-latest`, `macos-latest`) spins up a
**fresh, disposable virtual machine** hosted by GitHub for that one job,
then throws it away when the job ends — nothing persists between runs
unless you explicitly cache or upload it. GitHub-hosted runners have
fixed, modest specs (a few CPU cores, a few GB RAM) that are free up to a
monthly minutes quota, then billed per minute. For heavier needs (GPUs,
more RAM, access to a private network) teams run **self-hosted
runners** — your own machine registered to run Actions jobs instead of
GitHub's.

### GitHub Pricing — What's Actually Free

The **Free** plan (what you've used all course) includes unlimited
public *and* private repos, Pages, Issues/Projects, and a monthly Actions
minutes allowance (more for public repos than private). Paid tiers
(**Team**, **Enterprise**) mainly add: more Actions minutes/storage,
org-wide enforcement of branch protection and the security suite above,
SSO/SAML, and audit logs — governance features for organizations, not
things an individual developer is missing day to day. Exact limits and
prices change; check github.com/pricing for current numbers rather than
memorizing them.

---

## The GitHub CLI (`gh`) — Never Leave the Terminal

```bash
brew install gh          # macOS
gh auth login            # one-time setup, opens browser to authenticate
```

Daily-driver commands:

```bash
gh repo create my-project --public --clone     # create a new repo AND clone it, in one step
gh repo view --web                              # open current repo in browser

gh pr create --title "Add search filter" --body "Closes #42"   # open a PR from your current branch
gh pr create --fill                             # auto-fill title/body from your commits
gh pr list                                      # see open PRs
gh pr view 57                                   # view PR #57's details, right in the terminal
gh pr view 57 --web                             # ...or open it in the browser
gh pr checkout 57                               # check out someone else's PR branch locally to test it
gh pr diff 57                                   # see the diff without opening a browser
gh pr merge 57 --squash                         # merge it (respects branch protection rules)
gh pr review 57 --approve -b "LGTM, nice work"   # approve, right from the terminal

gh issue create --title "Bug: crash on empty input"
gh issue list --label bug
gh issue close 42

gh run list                                     # see recent GitHub Actions runs
gh run watch                                    # live-tail the currently running workflow
gh run view --log-failed                        # see exactly why a failed run failed, without clicking through the UI
```

`gh` is genuinely faster than the browser for most day-to-day PR/issue
tasks once it's muscle memory — especially `gh pr checkout` for quickly
testing a teammate's branch locally.

---

## Exercise

1. In one of your existing repos, add `.github/workflows/test.yml` running
   any trivial command (even just `echo "hello CI"` if you have no real
   tests yet) triggered on `push` and `pull_request`. Push it, open a PR
   from a branch, and watch the check appear and go green on the PR page.
2. Turn on branch protection for `main`: require a PR before merging and
   require that status check to pass. Try pushing directly to `main` and
   confirm GitHub rejects it.
3. Install `gh`, authenticate, and recreate the whole PR flow from
   Lesson 08's exercise using only `gh pr create`, `gh pr view`, and
   `gh pr merge` — no browser at all.
4. Add a `.github/CODEOWNERS` file assigning yourself as owner of the
   whole repo (`* @yourusername`), open a PR, and confirm you're
   auto-requested as a reviewer.

Continue to [Lesson 12 — Submodules, Subtrees & Large-Repo Tooling](12-submodules-subtrees-and-large-repos.md).
