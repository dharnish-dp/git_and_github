# Lesson 11 — GitHub Pro Features: Actions, Branch Protection, CLI

## Goal
Go beyond basic push/pull into what makes GitHub a full engineering
platform: automated CI/CD with Actions, repo governance with branch
protection, and driving all of it from the terminal with the GitHub CLI.

In plain words: so far GitHub has been a place to store your code and
open pull requests. In this lesson GitHub becomes a **robot
teammate** that runs your tests for you, a **bouncer** that stops bad
code from entering `main`, and a **command-line tool** you can drive
without opening a browser.

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

## Quick Vocabulary (used throughout this lesson)

Terms you already know, as a refresher:

- **Repository (repo)** = a project folder whose full history Git tracks.
- **Commit** = a saved snapshot of your project.
- **Branch** = a movable label pointing at a commit; a separate line of work.
- **`main`** = the branch that holds the "official, working" version of the project.
- **Push** = upload your local commits to GitHub.
- **Pull request (PR)** = a request on GitHub saying "please merge my
  branch into `main`", with a page for review and discussion.
- **Merge** = combining one branch's work into another.

New terms are defined where they first appear.

---

## GitHub Actions — CI/CD, Defined in Your Repo

### Why do we need this?

Picture yourself as an SDET. A teammate opens a PR. Someone has to pull
the branch, create a virtualenv, install dependencies and run `pytest`.
If people forget, broken code reaches `main`.

**Analogy:** think of a factory conveyor belt with an automatic
inspector at the end. Every item (your code change) goes past the
inspector. Nobody has to remember to inspect.

GitHub Actions is that inspector.

### Two acronyms, defined

- **CI (Continuous Integration)** = every time someone changes code,
  automatically build it and run its tests, so problems show up within
  minutes.
- **CD (Continuous Delivery/Deployment)** = after the tests pass,
  automatically ship the code (upload a package, deploy a website, etc.).

Both are just "scripts that run automatically". **GitHub Actions** is
GitHub's built-in system for running those scripts.

### What is a workflow file?

A **workflow** is a recipe written in a file. It says: "when THIS
happens, do THESE steps." Workflow files are written in **YAML**
(a plain-text format based on indentation, similar to a Python dict
written without braces). They must live in the folder
`.github/workflows/` inside your repo.

Because the file is inside your repo, it is committed and versioned like
any other file. Anyone who clones the repo gets the same recipe.

### Your first workflow

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

Read it from top to bottom, in plain English:

| Line | Meaning |
|------|---------|
| `name: Run Tests` | A label shown in the GitHub UI. Pick anything. |
| `on:` | The **trigger**: which events start this workflow. |
| `push: branches: [main]` | Run whenever someone pushes commits to `main`. |
| `pull_request: branches: [main]` | Run whenever a PR targeting `main` is opened or updated. |
| `jobs:` | The list of work to do. |
| `test:` | The name of one job (you choose it). |
| `runs-on: ubuntu-latest` | Which kind of machine to run on (a Linux virtual machine). |
| `steps:` | The ordered to-do list inside the job. |
| `uses: actions/checkout@v4` | Use a ready-made action that downloads your repo onto the machine. |
| `uses: actions/setup-python@v5` | Use a ready-made action that installs Python. |
| `with: python-version: "3.12"` | The input (argument) given to that action. |
| `run: pip install ...` | Run a plain shell command, exactly as you would in a terminal. |
| `run: pytest` | Run your tests. If this command fails, the job fails. |

**Python analogy:**
- `uses:` is like `import` plus calling a library function someone else wrote.
- `run:` is like `subprocess.run(...)`: you give it a shell command.
- `with:` is like keyword arguments: `setup_python(python_version="3.12")`.

### Vocabulary: trigger, job, step, runner

- **Trigger (`on:`)** = the event that starts the workflow. Common ones:
  - `push` (someone pushed commits)
  - `pull_request` (a PR was opened or updated)
  - `schedule` (a timer, written in **cron** syntax, a compact time
    pattern such as "every night at 2 AM")
  - `workflow_dispatch` (a manual "Run workflow" button in the GitHub UI)
- **Job** = one group of steps that runs on one machine. If you define
  several jobs, they run independently (in parallel by default).
- **Step** = one item in a job. Steps run one after another, in order.
  If one step fails, the remaining steps are skipped by default.
- **Runner** = the machine that executes a job (explained further below).

### Picture it: what happens after you push

**Diagram legend** (same in every lesson):

```
+-------+   a box = one step, place or thing
--->  v   arrows show the direction things flow
A---B---C commits: oldest on the LEFT, newest on the RIGHT
D*        an asterisk marks a NEW commit
/  \      a branch forking off, or merging back in
PASS/FAIL labels on arrows show which path is taken
```

How to read this: follow the arrows left to right; each box is handed
to the next one.

```
+-----------+   +-------------+   +-----------+   +---------------+
| EVENT     |-->| WORKFLOW    |-->| JOB       |-->| RUNNER (VM)   |
| push / PR |   | test.yml    |   | "test"    |   | ubuntu-latest |
+-----------+   +-------------+   +-----------+   +---------------+
```

Now zoom into the runner. How to read this: the steps run in order,
and the last one decides pass or fail, which is reported back to the PR.

```
   STEPS on the runner (run in order, left to right)

+----------+   +--------------+   +-------------+   +--------+
| checkout |-->| setup-python |-->| pip install |-->| pytest |
+----------+   +--------------+   +-------------+   +--------+
                                                     |
                     +--- exit 0 --------------------+--- exit != 0 --+
                     v                                                v
                job = PASS                                       job = FAIL
                     |                                                |
                     v                                                v
         +---------------------+                      +---------------------+
         | PR: green check     |                      | PR: red X           |
         | merge allowed       |                      | merge blocked if    |
         +---------------------+                      | check is required   |
                                                      +---------------------+
```

What happened: the event started the workflow, the runner executed each
step, and the pass/fail result flowed back onto the PR as a status check.

### Doubt? `uses` vs `run` — what is the difference?

- `run:` executes **a command you write**.
- `uses:` executes **a packaged, reusable action written by someone
  else** (found in the Actions Marketplace, GitHub's public catalog of
  such actions). The text after `@` is its **version**, like pinning
  `requests==2.31` in `requirements.txt`.

Why not just `run: apt install python`? You can, but `setup-python`
handles versions, caching and PATH for you. Think "use a library"
instead of "write it yourself".

### Doubt? "Where do I see it run?"

After you push the file, open your repo on GitHub and click the
**Actions** tab. You see one row per run. Click a row, then a job, and
you see the live log, exactly like a terminal.

For a PR, near the bottom of the page you will see something like:

```
 ✅ All checks have passed
   ✅ Run Tests / test (pull_request)   Successful in 34s
```

or, if `pytest` failed:

```
 ❌ Some checks were not successful
   ❌ Run Tests / test (pull_request)   Failing after 28s
```

That ✅/❌ is called a **status check**: a pass/fail result that an
outside system (here, Actions) reports back onto your PR. This is the
mechanism behind "all checks must pass before merging" in Branch
Protection, below.

### Doubt? "Does the workflow file need to be on `main` first?"

For `push` triggers, the workflow file must exist on the branch you
push. For `pull_request` triggers, GitHub uses the workflow file from
the PR's merged result, so the file can be introduced in the very PR you
are testing. In practice: commit the file on a branch, open a PR, and
the check appears on that PR.

### A Slightly Richer Example: Matrix Testing

**Why do we need this?** Your code may work on Python 3.12 but break on
3.10. Testing locally on only the version you have installed won't
reveal that.

**Analogy:** `pytest.mark.parametrize`. You write the test once, and
pytest runs it for each parameter. A **matrix** does the same for an
entire job.

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

New pieces:
- `strategy: matrix:` defines a list of values. Here the variable is
  `python-version`, with three values.
- `${{ matrix.python-version }}` is a **placeholder** (GitHub calls it
  an expression). GitHub replaces it with the current value for each copy.

Result: the whole job runs **three times in parallel**, once per Python
version. On the PR you will see three checks:

```
 ✅ Run Tests / test (3.10) (pull_request)
 ✅ Run Tests / test (3.11) (pull_request)
 ❌ Run Tests / test (3.12) (pull_request)
```

If 3.12 is red, you have found a version-specific bug before any user
did.

Note the quotes around `"3.10"`. Without quotes, YAML reads `3.10` as the
number 3.1, which would select the wrong Python. Quote version numbers.

### Secrets

**Why do we need this?** Some steps need credentials: an API key, a
password, a deploy token. If you type them into the YAML file, they
become part of your commit history. In a public repo, anyone can read
them.

**Analogy:** a secret is like a locked key cabinet. The workflow may
borrow a key when it runs, but the key is never printed on the recipe.

A **secret** is an encrypted value stored by GitHub, outside your code.

1. Open your repo on GitHub.
2. Go to **Settings -> Secrets and variables -> Actions**.
3. Click **New repository secret**, give it a name (e.g.
   `PROD_API_KEY`) and paste the value.

Then reference it by name in the workflow:

```yaml
- run: deploy.sh
  env:
    API_KEY: ${{ secrets.PROD_API_KEY }}
```

Line by line:
- `run: deploy.sh` runs your script.
- `env:` sets **environment variables** (named values a program can read,
  like `os.environ["API_KEY"]` in Python) for that step only.
- `${{ secrets.PROD_API_KEY }}` is replaced by GitHub with the real value
  while the job runs. If the value ever appears in the log, GitHub
  automatically hides it as `***`.

Inside `deploy.sh` (or a Python script) you read it the normal way:
`os.environ["API_KEY"]`.

---

## Branch Protection Rules — Making `main` Un-Breakable

### Why do we need this?

Without any rules, anyone with write access can push broken code straight
to `main`. "Please don't do that" is a social agreement, and social
agreements get broken at 6 PM on release day.

**Analogy:** a building with a security guard at the door. The guard
checks that you have an ID badge (approval) and that the inspection
sticker is green (tests passed). It does not matter who you are; the
rules apply.

A **branch protection rule** is a GitHub setting that enforces rules on
a specific branch.

### How to set one up

1. Repo page -> **Settings** -> **Branches**.
2. Click **Add branch protection rule**.
3. In "Branch name pattern", type `main` (or a pattern such as
   `release/*`, where `*` matches anything).
4. Tick the rules you want and click **Create**.

(GitHub also has a newer feature called **Rulesets**, found in
Settings -> Rules, that does the same job with more flexibility. The
concepts below apply to both.)

### The common rules, one at a time

- **Require a pull request before merging**
  Nobody can push commits directly to `main`. All changes must come
  through a PR. (Admins can optionally be exempted or included.)

- **Require approvals**
  A PR needs at least N teammates (e.g. 1 or 2) to click "Approve"
  before it can merge.

- **Require status checks to pass before merging**
  Picks specific checks (such as your "Run Tests" workflow) that must be
  green. A red ❌ makes the Merge button unclickable. You choose which
  checks are required by name, and a check appears in the list only after
  it has run at least once in the repo.

- **Require branches to be up to date before merging**
  Your PR branch must contain the latest `main`. If `main` moved on, you
  must merge or rebase `main` into your branch (Lesson 09) and re-run
  tests. This prevents "it passed CI but broke when combined with
  someone else's change."

- **Require conversation resolution before merging**
  Every review comment thread must be marked "Resolved".

- **Do not allow force pushes** / **Do not allow deletions**
  Nobody can rewrite `main`'s history (the danger from Lesson 07's
  force-push section) or delete the branch.

### Picture it: the gate every PR must pass

How to read this: go top to bottom. Each diamond-like box is one rule;
"no" sends you to BLOCKED, "yes" moves you down to the next rule.

```
      PR opened
          |
          v
+----------------------+  NO   +-------------------------------+
| Required checks      |------>| BLOCKED: fix code, push again |
| all green?           |       | (checks re-run automatically) |
+----------------------+       +-------------------------------+
          | YES
          v
+----------------------+  NO   +-------------------------------+
| Enough approvals?    |------>| BLOCKED: wait for a reviewer  |
+----------------------+       +-------------------------------+
          | YES
          v
+----------------------+  NO   +-------------------------------+
| Comments resolved,   |------>| BLOCKED: resolve threads /    |
| branch up to date?   |       | update branch, re-run checks  |
+----------------------+       +-------------------------------+
          | YES
          v
+----------------------+
| Merge button ENABLED |
+----------------------+
```

When the merge happens, `main` changes like this (merge commit shown):

BEFORE:
```
            E---F            add-search
           /
      A---B---C---D          main
```

AFTER:
```
            E---F---\        add-search
           /         \
      A---B---C---D---M*     main
```

What changed: `main` gained one new commit, `M*`, which joins the
`add-search` work (`E`, `F`) with `main`'s own commits. Nothing before
it was rewritten.

### What you see when a rule blocks you

If you try `git push origin main` directly on a protected branch:

```
$ git push origin main
remote: error: GH006: Protected branch update failed for refs/heads/main.
remote: error: Changes must be made through a pull request.
To github.com:you/repo.git
 ! [remote rejected] main -> main (protected branch hook declined)
error: failed to push some refs to 'github.com:you/repo.git'
```

Line by line:
- `GH006` is GitHub's error code for "protected branch".
- `Changes must be made through a pull request.` tells you exactly which
  rule triggered.
- `[remote rejected]` means GitHub (the remote) refused. Your local copy
  is unchanged and nothing was lost.

The fix is the normal workflow: create a branch, push it, open a PR.

### Doubt? "Does branch protection run my tests?"

No. Two separate pieces work together:
1. **Actions** runs the tests and reports a ✅/❌ status check.
2. **Branch protection** says "that check must be ✅ or merging is blocked".

Actions is the inspector. Branch protection is the rule that says
"nothing ships without the inspector's green sticker."

### Merge queue — fixing a gap that remains

Even with everything above, one gap exists on a busy repo. Example:

How to read this: history runs left to right; each PR branch forks off
`main` at `A` and is tested only against `A`.

BEFORE (no queue, both PRs tested only against A):
```
            B                pr-1 (adds function foo)   -> green
           /
      A                      main
           \
            C                pr-2 (deletes helper foo needs) -> green
```

Each PR is green on its own. Merge PR 1, then PR 2, and now `main`
contains both changes together, which nobody ever tested. `main` breaks.

AFTER (no queue, both merged):
```
      A---B*---C*            main   <- B+C together: never tested, RED
```

What changed: two individually green changes landed back to back, and
the untested combination broke `main`.

A **merge queue** is a line that PRs wait in. For each PR in turn, GitHub
builds the result of "latest main + this PR", runs the required checks on
that combined result, and merges only if it is green. One at a time, so
every merge is tested against the real, current `main`.

How to read this: the queue is on top (front merges first); below it,
each test builds on the result of the previous one.

```
  Queue:  [ PR 1 ] -> [ PR 2 ] -> [ PR 3 ]        (front merges first)

  Test 1:  A---B           = main + PR 1            -> GREEN, merge
  Test 2:  A---B---C       = main + PR 1 + PR 2     -> RED, PR 2 ejected
  Test 3:  A---B---D       = main + PR 1 + PR 3     -> GREEN, merge
```

AFTER (with queue), `main` only ever gets commits that were tested:
```
      A---B*---D*            main
```

What changed: PR 1 and PR 3 merged, each proven green on top of the
real `main`. PR 2's conflict with PR 1 was caught in the queue, so
`main` never went red.

Enable it under Settings -> Branches -> your branch protection rule ->
"Require merge queue". Turn it on when your repo has enough contributors
that two people merging on the same day is common. For a solo project it
is unnecessary.

---

## GitHub's Security Suite: Dependabot, Code Scanning & Secret Scanning

### Why do we need this?

Branch protection and CI keep `main` from breaking functionally. This
suite keeps it from becoming a *security* problem. These features are
mainstream, and often on by default for public repos.

Three tools, each answering one question:

| Tool | Question it answers |
|------|---------------------|
| Dependabot | "Are the libraries I use outdated or vulnerable?" |
| Code scanning (CodeQL) | "Does MY code contain security bugs?" |
| Secret scanning | "Did someone commit a password or key?" |

### Dependabot

A **dependency** is a third-party library your project needs (every
line in `requirements.txt`). **Dependabot** is a GitHub bot that opens
PRs to update dependencies for you. It works in two modes:

1. **Security updates**: a library you use has a publicly reported
   vulnerability, and Dependabot opens a PR upgrading to the fixed
   version. Enable at Settings -> Code security -> Dependabot.
2. **Version updates**: the library simply has a newer version. You turn
   this on by committing a config file:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
```

Line by line:
- `version: 2` is the config file format version (always 2 today).
- `package-ecosystem: "pip"` says "watch Python packages". Other values
  include `npm`, `docker`, and `github-actions`.
- `directory: "/"` says where your `requirements.txt` lives (repo root).
- `interval: "weekly"` says check once a week.

Dependabot's PRs look like any other PR ("Bump requests from 2.31.0 to
2.32.0"). Your Actions tests run on them, so you see whether the upgrade
breaks anything before merging.

### Code Scanning (CodeQL)

**Static analysis** = a tool that reads your code (without running it)
looking for suspicious patterns. **CodeQL** is GitHub's static analysis
engine. It flags issues such as SQL injection (user input ending up
inside a database query) or hardcoded credentials.

Results appear as annotations on the PR diff, in the same place as a
failed check. Setup is a wizard: Security tab -> Code scanning -> Set
up. It generates `.github/workflows/codeql.yml` for you, which is just
another workflow. You rarely write it by hand.

Findings vary from real vulnerabilities to low-confidence warnings, so
treat it as another required check and use judgment on each finding.

### Secret Scanning + Push Protection

**Secret scanning** looks through your repo for text that matches known
credential formats (cloud provider keys, common API token formats).
- Normally it scans existing history after the fact.
- With **push protection** enabled, GitHub checks *before* accepting a
  push. If it spots a secret, it rejects the push and names the file and
  line, so the secret never enters history.

Why this matters: once a secret is in a pushed commit, it must be
treated as stolen, even if you delete it later, because history keeps it
(this is Lesson 13's "If You Already Committed a Secret" scenario).
Push protection prevents that scenario on GitHub-hosted repos.

It is not a complete defence:
- It only catches patterns it recognises.
- It does not replace `.gitignore` discipline (Lesson 13).
- If you suspect a leak, rotate (replace) the secret immediately.

All three are enabled per repository under **Settings -> Code security**.
Organization admins can enforce them for every repo in the org.

---

## CODEOWNERS — Automatic Review Assignment

### Why do we need this?

On a team, some files need specific experts to review them. Relying on
the PR author to remember who to tag does not scale.

**Analogy:** a mail-sorting room with a lookup table: "anything
addressed to the infra department goes to Platform team."

A `.github/CODEOWNERS` file is that lookup table. Each line is:
`file pattern` then one or more owners (`@username` or `@org/team-name`).

```
# .github/CODEOWNERS
*.py            @backend-team
/frontend/      @frontend-team
/infra/         @platform-team
/docs/          @tech-writers
```

Reading it:
- `*.py` matches every Python file anywhere, so `@backend-team` is
  requested for review when a `.py` file changes.
- `/frontend/` (leading slash) means the `frontend` folder at the repo root.
- Lines starting with `#` are comments.

When a PR touches matching files, GitHub automatically adds those
owners as requested reviewers.

How to read this: each changed file in the PR (left) is matched against
the CODEOWNERS lines (middle), and the matching owner is requested (right).

```
 FILES IN THE PR        CODEOWNERS LINE          REVIEW REQUESTED FROM
+----------------+    +-------------------+    +----------------------+
| app/login.py   |--->| *.py    backend   |--->| @backend-team        |
| infra/db.tf    |--->| /infra/ platform  |--->| @platform-team       |
| docs/intro.md  |--->| /docs/  writers   |--->| @tech-writers        |
+----------------+    +-------------------+    +----------------------+
                                                         |
                                                         v
                                        PR cannot merge until each one
                                        approves (if "Require review
                                        from Code Owners" is on)
```

If several patterns match a file, the **last matching line wins**, so put
general rules first and specific ones later.

Combine with the branch protection option **"Require review from Code
Owners"**. Then an infra change cannot merge without a platform
engineer's approval, and nobody had to tag them manually.

---

## Issues — Lightweight Tracking

An **Issue** is a ticket on GitHub: a bug report, feature request or
task, each with its own numbered page (`#42`). If you use Jira, think of
it as a lighter Jira ticket living next to the code.

Key features:
- **Labels**: coloured tags (`bug`, `enhancement`, `good first issue`)
  for filtering.
- **Assignees**: who is responsible.
- **Milestones**: group issues into a release or sprint.
- **Linking to PRs**: writing `Closes #42` in a PR description
  auto-closes issue #42 when that PR merges (from Lesson 08).
  `Fixes #42` and `Resolves #42` work too.
- **Issue templates**: `.github/ISSUE_TEMPLATE/bug_report.md` pre-fills a
  structured form (steps to reproduce, expected result, actual result),
  which SDETs will appreciate.

## Projects — Kanban/Sprint Boards

A **Kanban board** is a set of columns (To Do / In Progress / Done)
with cards that move left to right as work progresses. **GitHub
Projects** builds such a board from your Issues and PRs. It is useful if
your team does not already use Jira or Azure DevOps for this.

---

## GitHub the Platform: Hosting, Runners & Pricing

Three practical questions beginners have about GitHub as a product.

### GitHub Pages — Free Static Site Hosting From a Repo

A **static site** is a website made of fixed files (HTML, CSS,
JavaScript) with no server-side code. **GitHub Pages** hosts one for you.

Steps: **Settings -> Pages**, then choose a source: a branch, a `/docs`
folder, or a GitHub Actions deploy step. GitHub publishes it at:

```
https://yourname.github.io/repo-name
```

No server to manage. It works for plain HTML or the output of a
static-site generator (a tool that turns Markdown into a website, e.g.
Jekyll, Hugo, MkDocs). Common uses: project documentation, a portfolio,
or rendering a `lessons/` folder like this course as browsable pages.
Custom domains are supported.

How to read this: left to right, from your push to the live website.

```
+-----------+   +-------------+   +--------------+   +------------------+
| push to   |-->| Actions     |-->| build site   |-->| deploy to Pages  |
| main      |   | workflow    |   | (mkdocs etc) |   |                  |
+-----------+   +-------------+   +--------------+   +------------------+
                                        |                     |
                                  FAIL: stop,            PASS: live at
                                  old site stays   yourname.github.io/repo
```

### What Machine Your Actions Workflow Actually Runs On

A **runner** is the computer that executes your job.

`runs-on: ubuntu-latest` (or `windows-latest`, `macos-latest`) asks
GitHub for a **fresh, disposable virtual machine** (a simulated computer
running inside a bigger one) for that one job. When the job ends, GitHub
throws the machine away.

**Analogy:** a hotel room that is wiped clean after each guest. Your
job can install anything it likes, but nothing is left for the next run
unless you explicitly **cache** it (save files for reuse) or **upload
artifacts** (save output files, like test reports, to attach to the run).

Points to know:
- GitHub-hosted runners have fixed, modest specs (a few CPU cores, a few
  GB of RAM).
- They are free up to a monthly allowance of minutes, then billed per
  minute.
- A **self-hosted runner** is your own machine (laptop, company server,
  cloud VM) that you register with GitHub so it runs your jobs instead
  of GitHub's. Teams use these for GPUs, more RAM, or access to a private
  network. You then manage that machine yourself.

### GitHub Pricing — What's Actually Free

The **Free** plan (what you have used all course) includes:
- unlimited public and private repos
- Pages, Issues, Projects
- a monthly Actions minutes allowance (larger for public repos than
  private ones)

Paid tiers (**Team**, **Enterprise**) mainly add more Actions
minutes/storage, organization-wide enforcement of branch protection and
the security suite, SSO/SAML (company single sign-on), and audit logs
(a record of who did what). These are governance features for
organizations, not things an individual developer is missing day to day.

Exact limits and prices change; check github.com/pricing rather than
memorizing numbers.

---

## The GitHub CLI (`gh`) — Never Leave the Terminal

### Why do we need this?

Doing everything in a browser means constant switching between terminal
and web page. **`gh`** is GitHub's official command-line tool, so you
can do PR/issue/Actions tasks right where you already type `git`.

### `gh` vs `git` — common confusion

- `git` manages your **local repository** and syncs commits with a remote.
  It knows nothing about PRs or issues.
- `gh` talks to **GitHub's features**: PRs, issues, Actions runs.
  It is a separate program.

Analogy: `git` is `pytest`-style core tooling; `gh` is like a CLI
wrapper around a web service's REST API (in fact, that is what it is).

### Setup (once)

```bash
brew install gh          # macOS
gh auth login            # one-time setup, opens browser to authenticate
```

`brew` is the macOS package manager. `gh auth login` asks a few
questions (GitHub.com, HTTPS or SSH, log in via browser). Typical
finish:

```
$ gh auth login
? Where do you use GitHub? GitHub.com
? What is your preferred protocol for Git operations? HTTPS
? How would you like to authenticate GitHub CLI? Login with a web browser
! First copy your one-time code: ABCD-1234
- Press Enter to open github.com in your browser...
✓ Authentication complete.
✓ Logged in as yourname
```

You paste the one-time code into the browser page, approve, and `gh` is
authorised. Verify any time with `gh auth status`.

### Repos

```bash
gh repo create my-project --public --clone     # create a new repo AND clone it, in one step
gh repo view --web                              # open current repo in browser
```

- `gh repo create my-project --public --clone` makes a new public repo
  on GitHub and then runs `git clone` so you get a local folder named
  `my-project`.
  Expected:
  ```
  ✓ Created repository yourname/my-project on GitHub
  ✓ Cloned repository yourname/my-project
  ```
- `--web` means "open this in the browser instead of printing it".

### Pull requests

```bash
gh pr create --title "Add search filter" --body "Closes #42"   # open a PR from your current branch
gh pr create --fill                             # auto-fill title/body from your commits
gh pr list                                      # see open PRs
gh pr view 57                                   # view PR #57's details, right in the terminal
gh pr view 57 --web                             # ...or open it in the browser
gh pr checkout 57                               # check out someone else's PR branch locally to test it
gh pr diff 57                                   # see the diff without opening a browser
gh pr merge 57 --squash                         # merge it (respects branch protection rules)
gh pr review 57 --approve -b "LGTM, nice work"   # approve, right from the terminal
```

Explained, with sample output:

`gh pr create` (run from the branch you want merged; push it first, or
`gh` will offer to push):
```
$ gh pr create --title "Add search filter" --body "Closes #42"
Creating pull request for add-search-filter into main in yourname/my-project

https://github.com/yourname/my-project/pull/57
```
The last line is the URL of the new PR (number 57).

`gh pr create --fill` reuses your commit message as the title and body
so you don't type them again.

`gh pr list`:
```
Showing 2 of 2 open pull requests in yourname/my-project

#57  Add search filter    add-search-filter   about 1 minute ago
#55  Fix login timeout    fix-login-timeout   about 2 days ago
```
Columns: PR number, title, branch name, age.

`gh pr checkout 57` downloads that PR's branch and switches to it, so
you can run its tests locally. Very handy for an SDET reviewing a
teammate's change:
```
$ gh pr checkout 57
Switched to branch 'add-search-filter'
```

`gh pr merge 57 --squash` merges PR 57 using **squash merge** (all the
PR's commits combined into a single commit on `main`, from Lesson 09).
Other options: `--merge` (normal merge commit), `--rebase`. If branch
protection blocks it (e.g. checks failing), `gh` refuses with a message
and nothing is merged. Add `--auto` to merge automatically once the
checks pass.

`gh pr review 57 --approve -b "..."`: `-b` is the review comment body.

### Issues

```bash
gh issue create --title "Bug: crash on empty input"
gh issue list --label bug
gh issue close 42
```

Sample `gh issue list --label bug` output:
```
Showing 1 of 1 open issue in yourname/my-project that matches your search

#42  Bug: crash on empty input   bug   about 3 hours ago
```

### Actions runs

```bash
gh run list                                     # see recent GitHub Actions runs
gh run watch                                    # live-tail the currently running workflow
gh run view --log-failed                        # see exactly why a failed run failed, without clicking through the UI
```

Sample `gh run list`:
```
STATUS  TITLE               WORKFLOW    BRANCH             EVENT         ID          ELAPSED
✓       Add search filter   Run Tests   add-search-filter  pull_request  9123456789  34s
X       Fix login timeout   Run Tests   fix-login-timeout  push          9123456001  28s
```

`✓` passed, `X` failed. For a failed run, `gh run view --log-failed`
prints only the log of the failing steps, so you see the `pytest` error
right in your terminal. For a test engineer this replaces most
click-through-the-UI debugging.

`gh` is genuinely faster than the browser for day-to-day PR and issue
work once it is muscle memory, especially `gh pr checkout` for testing
a teammate's branch.

---

## Exercise

1. **Create a workflow.** In one of your existing repos, add
   `.github/workflows/test.yml` running any trivial command (even
   just `echo "hello CI"` if you have no real tests yet) triggered on
   `push` and `pull_request`. Push it on a branch, open a PR, and watch
   the check appear.
   *Expected result:* the PR page shows "All checks have passed" with a
   green ✅ next to your workflow name. If it is red, click "Details" and
   read the log to find the typo (YAML indentation is the usual cause).
2. **Protect `main`.** Settings -> Branches -> add a rule for `main`:
   require a PR before merging and require that status check to pass.
   Then try `git push origin main` directly.
   *Expected result:* GitHub rejects it with `GH006: Protected branch
   update failed`.
3. **Terminal-only PR flow.** Install `gh`, run `gh auth login`, and
   repeat Lesson 08's PR exercise using only `gh pr create`,
   `gh pr view`, and `gh pr merge`, with no browser.
   *Expected result:* `gh pr create` prints a PR URL, and `gh pr merge`
   finishes with a message that the PR was merged (the check from
   exercise 1 must be green first, or merging is refused).
4. **CODEOWNERS.** Add `.github/CODEOWNERS` with `* @yourusername`,
   open a PR from another branch.
   *Expected result:* you appear under "Reviewers" as a requested
   reviewer. (GitHub does not request a review from the PR author
   themselves, so on a solo repo you may see no request; test with a
   second account or a teammate if so.)

---

## Recap in 5 Lines
1. GitHub Actions runs the workflow files in `.github/workflows/` on a disposable runner machine and reports a ✅/❌ status check on your PR.
2. Secrets keep passwords out of your code; branch protection makes GitHub refuse merges that break your rules (PR required, approvals, green checks).
3. A merge queue tests each PR against the real latest `main`; Dependabot, CodeQL and secret scanning guard dependencies, your code and credentials.
4. CODEOWNERS auto-requests the right reviewers; Issues and Projects track work; Pages hosts static sites; runners and pricing decide where and how much your jobs cost.
5. `gh` is the terminal tool for GitHub features (PRs, issues, runs), while `git` handles your local repo.

Continue to [Lesson 12 — Submodules, Subtrees & Large-Repo Tooling](12-submodules-subtrees-and-large-repos.md).
