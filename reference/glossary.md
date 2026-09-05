# Git & GitHub Glossary

Every term used across the course, defined plainly. Lesson references
point to where each concept is taught in depth.

**Blob** — A Git object storing raw file *content* only, no filename or
metadata. [Lesson 03](../lessons/03-under-the-hood-git-objects.md)

**Branch** — A movable pointer (just a text file containing a commit
hash) to the tip of a line of development. [Lesson 05](../lessons/05-branching-and-merging.md)

**Branch protection rule** — A GitHub setting that enforces requirements
(PR review, passing CI, no force-push) on a specific branch, usually
`main`. [Lesson 11](../lessons/11-github-pro-features.md)

**Cherry-pick** — Applying one specific commit's changes onto a different
branch, as a new commit. [Lesson 09](../lessons/09-rebase-cherry-pick-squash.md)

**Clone** — Downloading a complete copy of a remote repository, including
its full history, to your machine. [Lesson 07](../lessons/07-remotes-and-github-basics.md)

**CODEOWNERS** — A file that automatically assigns reviewers based on
which files a PR changes. [Lesson 11](../lessons/11-github-pro-features.md)

**Commit** — A permanent snapshot of the entire project at a point in
time, plus metadata (author, message, parent commit, timestamp) and a
unique SHA hash. [Lesson 02](../lessons/02-core-concepts-and-git-anatomy.md), [Lesson 03](../lessons/03-under-the-hood-git-objects.md)

**Conflict** — What happens when Git can't automatically combine two
sets of changes to the same lines of a file during a merge/rebase/
cherry-pick, requiring manual resolution. [Lesson 05](../lessons/05-branching-and-merging.md)

**Dangling commit** — A commit object that still exists in the object
database but that no branch, tag, or reflog entry currently points to;
recoverable until `git gc` eventually prunes it. [Lesson 03](../lessons/03-under-the-hood-git-objects.md)

**Dependabot** — A GitHub feature that automatically opens PRs bumping
dependencies with known security vulnerabilities to a patched version.
[Lesson 11](../lessons/11-github-pro-features.md)

**Detached HEAD** — A state where `HEAD` points directly at a commit
instead of at a branch; any new commits made here need a branch created
immediately or they risk being orphaned. [Lesson 10](../lessons/10-stash-tags-bisect.md)

**Fast-forward merge** — A merge where the target branch simply advances
its pointer to match the source branch, because no divergent commits
exist; no merge commit is created. [Lesson 05](../lessons/05-branching-and-merging.md)

**Fetch** — Downloading new commits from a remote without merging them
into your current branch. Always safe. [Lesson 07](../lessons/07-remotes-and-github-basics.md)

**Fork** — Your own personal copy of someone else's GitHub repository,
used when you don't have direct write access to the original.
[Lesson 08](../lessons/08-collaboration-pull-requests.md)

**Gitattributes** — A `.gitattributes` file that tells Git how to treat
specific paths: line-ending normalization, marking files binary, or
wiring up custom diff/merge drivers. [Lesson 13](../lessons/13-pro-workflows-and-best-practices.md)

**GitHub Actions** — GitHub's built-in CI/CD system; runs scripts
automatically in response to repo events, defined in `.github/workflows/`.
[Lesson 11](../lessons/11-github-pro-features.md)

**GitHub Pages** — Free static-site hosting served directly from a
repo's branch/folder, at `username.github.io/repo`.
[Lesson 11](../lessons/11-github-pro-features.md)

**Runner** — The machine (GitHub-hosted VM, or a self-hosted one you
register) that actually executes a GitHub Actions job. [Lesson 11](../lessons/11-github-pro-features.md)

**HEAD** — A pointer to whatever commit you currently have checked out;
almost always points at a branch, which in turn points at a commit.
[Lesson 02](../lessons/02-core-concepts-and-git-anatomy.md)

**Hook** — A script that runs automatically at a specific point in Git's
workflow (e.g. before a commit is created). [Lesson 13](../lessons/13-pro-workflows-and-best-practices.md)

**Index** — Another name for the staging area; also the literal filename
(`.git/index`) where it's stored on disk. [Lesson 02](../lessons/02-core-concepts-and-git-anatomy.md)

**Interactive rebase** — Using `git rebase -i` to edit, reorder, squash,
or drop commits before sharing them. [Lesson 09](../lessons/09-rebase-cherry-pick-squash.md)

**Merge-base** — The most recent commit that two branches have in
common; what Git diffs each branch against to compute a three-way
merge. [Lesson 05](../lessons/05-branching-and-merging.md)

**Merge commit** — A commit with two (or more) parents, created when
combining two branches whose histories have diverged. [Lesson 05](../lessons/05-branching-and-merging.md)

**Merge queue** — A GitHub feature that serializes and re-tests PRs
against each other before merging, so two green PRs can't combine into
a broken `main`. [Lesson 11](../lessons/11-github-pro-features.md)

**Merge strategy** — The algorithm Git uses to combine two branches'
changes at a merge-base; `ort` is the modern default, `recursive` is
its predecessor, and `octopus` merges more than two branches at once.
[Lesson 05](../lessons/05-branching-and-merging.md)

**Origin** — The conventional (but not mandatory) nickname for a
repository's primary remote. [Lesson 07](../lessons/07-remotes-and-github-basics.md)

**Packfile** — A single compressed file holding many Git objects
together (delta-compressed against each other), replacing many
individual loose objects for efficient storage/transfer. [Lesson 03](../lessons/03-under-the-hood-git-objects.md)

**Partial clone** — A clone (`--filter=blob:none`) that fetches full
commit/tree history immediately but defers downloading file contents
until they're actually needed. [Lesson 12](../lessons/12-submodules-subtrees-and-large-repos.md)

**Pickaxe** — Nickname for `git log -S`/`-G`, which searches commit
history for the commit(s) that introduced or removed a given string or
regex pattern. [Lesson 10](../lessons/10-stash-tags-bisect.md)

**Pull** — `git fetch` followed immediately by a merge (or rebase, if
configured) of the fetched changes into your current branch.
[Lesson 07](../lessons/07-remotes-and-github-basics.md)

**Pull Request (PR)** — A GitHub feature for proposing that commits from
one branch be merged into another, with room for review and discussion.
[Lesson 08](../lessons/08-collaboration-pull-requests.md)

**Push** — Uploading your local commits to a remote, advancing the
remote's branch pointer to match yours. [Lesson 07](../lessons/07-remotes-and-github-basics.md)

**Push protection** — A GitHub feature that blocks a `git push` outright
when it detects a credential-shaped secret in the incoming commits,
before the secret ever enters remote history. [Lesson 11](../lessons/11-github-pro-features.md)

**Rebase** — Rewriting a branch's commits so they appear to have been
created starting from a different (usually more current) base commit,
producing new commit hashes. [Lesson 09](../lessons/09-rebase-cherry-pick-squash.md)

**Reflog** — A local, personal log of everywhere `HEAD` has pointed;
the primary tool for recovering "lost" commits. [Lesson 06](../lessons/06-undoing-things.md)

**Remote** — A nickname for another copy of the repository, typically
hosted elsewhere (e.g. GitHub). [Lesson 07](../lessons/07-remotes-and-github-basics.md)

**Remote-tracking branch** — A local bookmark (e.g. `origin/main`)
recording where a remote branch was, as of your last fetch/pull.
[Lesson 07](../lessons/07-remotes-and-github-basics.md)

**Rerere** — "Reuse recorded resolution": a Git feature that remembers
how you resolved a conflict and automatically reapplies that same
resolution if the identical conflict recurs (common across repeated
rebases). [Lesson 09](../lessons/09-rebase-cherry-pick-squash.md)

**Reset** — Moving a branch pointer to a different commit, with three
modes (`--soft`, `--mixed`, `--hard`) controlling what happens to staged
changes and working files. [Lesson 06](../lessons/06-undoing-things.md)

**Revert** — Creating a new commit that undoes the effect of an earlier
commit, without rewriting history — safe on shared branches.
[Lesson 06](../lessons/06-undoing-things.md)

**SHA (hash)** — A fixed-length fingerprint computed from an object's
content; identical content always produces an identical hash.
[Lesson 03](../lessons/03-under-the-hood-git-objects.md)

**Shallow clone** — A clone (`--depth N`) that only downloads the most
recent N commits' history instead of the entire project history.
[Lesson 12](../lessons/12-submodules-subtrees-and-large-repos.md)

**Sparse-checkout** — A setting that limits which paths of a repo are
actually materialized into the working directory, while the full
history stays available. [Lesson 12](../lessons/12-submodules-subtrees-and-large-repos.md)

**Squash** — Combining multiple commits into one, either during an
interactive rebase or via GitHub's "Squash and merge" PR option.
[Lesson 08](../lessons/08-collaboration-pull-requests.md), [Lesson 09](../lessons/09-rebase-cherry-pick-squash.md)

**Stage / staging area** — The holding area where you specify exactly
which changes should be included in your next commit. [Lesson 02](../lessons/02-core-concepts-and-git-anatomy.md)

**Stash** — A stack for temporarily shelving uncommitted changes so you
can switch branches or contexts cleanly. [Lesson 10](../lessons/10-stash-tags-bisect.md)

**Submodule** — A reference embedding another Git repository inside
yours at a specific path, pinned to one exact commit of that other
repo. [Lesson 12](../lessons/12-submodules-subtrees-and-large-repos.md)

**Subtree** — A way of merging another repository's history directly
into a subfolder of yours as regular commits, avoiding submodules'
separate-pointer bookkeeping. [Lesson 12](../lessons/12-submodules-subtrees-and-large-repos.md)

**Tag** — A permanent, non-moving pointer to a specific commit, typically
used to mark releases. [Lesson 10](../lessons/10-stash-tags-bisect.md)

**Three-way merge** — A merge requiring an actual merge commit, because
both branches have diverging commits since their common ancestor.
[Lesson 05](../lessons/05-branching-and-merging.md)

**Tree** — A Git object representing a directory listing: names mapped
to blob or tree hashes. [Lesson 03](../lessons/03-under-the-hood-git-objects.md)

**Upstream** — Conventionally, the nickname for the original repository
you forked from (as opposed to `origin`, your own fork).
[Lesson 08](../lessons/08-collaboration-pull-requests.md)

**Working directory** — The actual files on disk that you see and edit;
the only one of Git's "three trees" that lives outside the `.git`
database. [Lesson 02](../lessons/02-core-concepts-and-git-anatomy.md)

**Worktree** — A second working directory, backed by the same `.git`
database, letting you check out a different branch in a separate
folder without stashing or cloning again. [Lesson 10](../lessons/10-stash-tags-bisect.md)
