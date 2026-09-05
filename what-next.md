# What Next — Advanced Roadmap

You've finished the core course. This is where Git/GitHub mastery
continues once you're operating on real teams and larger codebases.

## 1. Branching Strategies at Scale

- **GitHub Flow** — the simple model this course implicitly taught:
  branch off `main`, PR, merge. Good for continuous deployment, small-to-
  mid teams.
- **Git Flow** — a heavier model with long-lived `develop`, `release/*`,
  and `hotfix/*` branches. Common in software with scheduled release
  cycles (versioned desktop apps, embedded firmware).
- **Trunk-Based Development** — everyone commits to `main` (or very
  short-lived branches merged within a day), leaning on feature flags
  instead of long-lived branches to hide unfinished work. What most
  high-velocity tech companies (Google, Meta-scale orgs) actually run
  internally.

## 2. Monorepos

Large orgs often keep dozens/hundreds of services in **one** Git
repository instead of many small repos. Lesson 12 already covers the
core tooling this requires — sparse-checkout, shallow clone, and partial
clone — so nobody has to download the entire codebase just to work on
one service. What's still genuinely advanced beyond that: build-system
tooling like Bazel, Nx, and Turborepo, which sit on top of a monorepo to
handle build/test *scoping* (only rebuild/retest what actually changed,
across a codebase too large to blindly rebuild everything on every
commit).

## 3. Git LFS and Large Data

Beyond the basics in Lesson 13 — if your work involves ML model weights,
datasets, or media assets, look at **DVC** (Data Version Control), which
is purpose-built for versioning large datasets alongside Git, more
flexibly than raw Git LFS.

## 4. Advanced GitHub Actions

- **Reusable workflows** and **composite actions** — stop copy-pasting
  the same CI steps across repos.
- **Environments** — staging/production deployment gates, required
  approvers before deploying.
- **Self-hosted runners** — run Actions on your own infrastructure
  instead of GitHub's shared runners (needed for private network access,
  GPU jobs, cost control at scale).
- **Matrix + `needs:`** — build genuinely complex, multi-stage pipelines
  (build → test → deploy) with proper job dependencies.

## 5. GitOps

The practice of using a Git repository as the **single source of truth**
for infrastructure/deployment state — a tool (ArgoCD, Flux) continuously
watches a repo and automatically syncs a Kubernetes cluster (or other
infra) to match whatever's committed. Every deployment becomes a Git
commit, every rollback is a `git revert`.

## 6. Git Internals, Deeper

Packfiles, `git gc`/`repack` (Lesson 03) and submodules vs subtrees
(Lesson 12) are now covered in the core course. What's still genuinely
advanced beyond that:

- **Custom merge/diff drivers** — teach Git how to intelligently
  diff/merge non-text or structured files (e.g. Jupyter notebooks,
  binary formats) via `.gitattributes` — Lesson 13 covers the basic
  line-ending-normalization use of `.gitattributes`; custom drivers for
  structured/binary formats are the next level up.
- **`git rerere` at scale** — Lesson 09 covers the basics; on very
  long-lived branches with heavy repeated rebasing, teams sometimes share
  a `rerere` cache across machines to avoid every developer re-resolving
  the same recurring conflict independently.

## 7. Security Hardening

Dependabot and secret scanning/push protection (Lesson 11) are now
covered in the core course. What's still genuinely advanced beyond that:

- Mandatory signed commits (Lesson 13) enforced via branch protection
  org-wide.
- **SBOM generation** in CI for supply-chain visibility.

## 8. Practice by Contributing

The fastest way to convert this course into real intuition: find an
active open-source project in a language/domain you care about and make
one real contribution using the fork workflow from Lesson 08. Nothing
builds Git confidence like a PR review from a stranger pointing out
something you didn't expect.
