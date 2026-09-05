# Git & GitHub Cheat Sheet

Commands you'll type daily, grouped by task. See the lesson linked in each
section header for full explanations.

## Setup ([Lesson 07](../lessons/07-remotes-and-github-basics.md), [Lesson 13](../lessons/13-pro-workflows-and-best-practices.md))

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
ssh-keygen -t ed25519 -C "you@example.com"
ssh -T git@github.com                          # test SSH auth
```

## Starting a Repo ([Lesson 04](../lessons/04-your-first-repo.md), [07](../lessons/07-remotes-and-github-basics.md))

```bash
git init                                        # new repo, no history
git clone git@github.com:user/repo.git          # copy an existing repo, with full history
git remote add origin <url>                     # connect a local repo to a remote
git remote -v                                    # list remotes
```

## Everyday Loop ([Lesson 04](../lessons/04-your-first-repo.md))

```bash
git status                     # what's changed, what's staged
git diff                       # unstaged changes (working dir vs staging)
git diff --staged              # staged changes (staging vs last commit)
git add <file>                 # stage a file
git add .                      # stage everything in and below current dir
git add -p                     # stage interactively, hunk by hunk
git commit -m "message"        # commit staged changes
git commit -am "message"       # stage all TRACKED modified files + commit (skips new files!)
git log --oneline              # compact history
git log --oneline --graph --all --decorate   # visual branch graph — alias this!
```

## Undoing ([Lesson 06](../lessons/06-undoing-things.md))

```bash
git restore <file>              # discard unstaged edits to a file
git restore --staged <file>     # unstage (keep the edits)
git commit --amend -m "msg"     # rewrite last commit's message
git commit --amend --no-edit    # add staged changes to last commit, keep same message
git reset --soft HEAD~1         # undo last commit, keep changes staged
git reset --mixed HEAD~1        # undo last commit, keep changes unstaged (default mode)
git reset --hard HEAD~1         # undo last commit, DISCARD its changes entirely
git revert <hash>                # safely undo a commit by adding a new counter-commit (safe on shared branches)
git reflog                       # recovery map — find any "lost" commit/branch
```

## Branching & Merging ([Lesson 05](../lessons/05-branching-and-merging.md))

```bash
git branch                      # list local branches
git branch -a                   # list local + remote-tracking branches
git switch <branch>              # switch to existing branch
git switch -c <branch>           # create + switch in one step
git branch -d <branch>           # delete branch (safe)
git branch -D <branch>           # force delete branch
git merge <branch>               # merge <branch> into current branch
git merge --ff-only <branch>     # merge ONLY if it can fast-forward, else fail (keeps history linear on purpose)
git merge --no-ff <branch>       # force a merge commit even if a fast-forward was possible (preserves feature boundary)
git merge -X ours <branch>       # on conflict, auto-prefer CURRENT branch's version of conflicting hunks
git merge -X theirs <branch>     # on conflict, auto-prefer INCOMING branch's version of conflicting hunks
git merge-base main feature      # find the common ancestor commit of two branches
git merge --abort                # bail out of a messy merge, return to pre-merge state
```

## Remotes ([Lesson 07](../lessons/07-remotes-and-github-basics.md))

```bash
git fetch origin                 # download new commits, don't merge (always safe)
git pull                         # fetch + merge in one step
git pull --rebase                # fetch + rebase instead of merge
git push origin main             # push local main to origin's main
git push -u origin <branch>      # push + set up tracking (first push of a new branch)
git push --force-with-lease      # safely overwrite remote history after amend/rebase
git branch -vv                   # see tracking status + ahead/behind counts
git config --global credential.helper cache      # cache HTTPS creds in memory for a while (default 15 min)
git config --global credential.helper osxkeychain  # macOS: store HTTPS creds permanently in Keychain
```

## Rebase & History Cleanup ([Lesson 09](../lessons/09-rebase-cherry-pick-squash.md))

```bash
git rebase main                       # replay current branch's commits on top of main
git rebase -i HEAD~4                  # interactively edit/squash/reorder last 4 commits
git rebase --continue                 # after resolving a rebase conflict
git rebase --abort                    # bail out entirely, return to pre-rebase state
git cherry-pick <hash>                # apply one specific commit onto current branch
git merge --squash <branch>           # combine a branch's changes into one uncommitted change
git commit --fixup <hash>             # make a "fixup!" commit targeting an earlier commit
git commit --squash <hash>            # make a "squash!" commit targeting an earlier commit (keeps its message too)
git rebase -i --autosquash HEAD~6     # auto-reorders fixup!/squash! commits onto their targets during the todo list
git config --global rebase.autosquash true   # make --autosquash the default for every interactive rebase
git config --global rerere.enabled true      # remember conflict resolutions and auto-replay them next time
git rerere status                     # see which files rerere has recorded a resolution for
```

## Stash, Tags, Bisect ([Lesson 10](../lessons/10-stash-tags-bisect.md))

```bash
git stash                        # shelve uncommitted changes
git stash pop                    # reapply + remove most recent stash
git stash list                   # see all stashed entries
git tag -a v1.0.0 -m "message"   # annotated tag (use for releases)
git push origin --tags           # push all tags to remote
git bisect start HEAD <good-commit>
git bisect run <test-command>    # automated binary search for the bad commit
git bisect reset                 # finish, return to original HEAD
```

## Blame, Pickaxe & Worktrees ([Lesson 10](../lessons/10-stash-tags-bisect.md))

```bash
git blame <file>                 # who last touched each line, and in which commit
git blame -L 10,20 <file>        # only blame lines 10-20
git blame -w <file>              # ignore whitespace-only changes when attributing lines
git blame -C <file>              # also detect lines moved/copied from other files
git log -S"needle"               # find commits that added or removed the exact string "needle"
git log -G"regex"                # find commits whose diff matches a regex pattern
git worktree add ../hotfix main  # check out "main" into a second working directory, no stashing needed
git worktree list                # see all worktrees attached to this repo
git worktree remove ../hotfix    # clean up a worktree you're done with
```

## GitHub CLI ([Lesson 11](../lessons/11-github-pro-features.md))

```bash
gh auth login
gh repo create <name> --public --clone
gh pr create --fill
gh pr list
gh pr checkout <number>
gh pr diff <number>
gh pr merge <number> --squash
gh issue create --title "..."
gh run watch                     # live-tail the current CI run
```

## Submodules, Subtrees & Large Repos ([Lesson 12](../lessons/12-submodules-subtrees-and-large-repos.md))

```bash
git submodule add <url> <path>              # embed another repo at <path>, pinned to one commit
git clone --recurse-submodules <url>         # clone a repo AND fetch its submodules' content in one step
git submodule update --init --recursive      # populate submodules after a plain clone/pull
git subtree add --prefix=<path> <url> <branch> --squash   # merge another repo's history in as a subfolder
git subtree pull --prefix=<path> <url> <branch> --squash  # pull upstream subtree changes into <path>
git subtree push --prefix=<path> <url> <branch>            # push local <path> changes back upstream
git clone --depth 1 <url>                    # shallow clone — only the latest commit, much faster
git fetch --unshallow                        # convert a shallow clone into a full one later
git clone --filter=blob:none <url>           # partial clone — full history, file contents fetched on demand
git sparse-checkout init --cone              # only materialize a subset of the working directory
git sparse-checkout set services/api docs    # choose which paths to actually check out
```

## Internals & Maintenance ([Lesson 03](../lessons/03-under-the-hood-git-objects.md))

```bash
git count-objects -v             # see how many loose vs packed objects exist, and their size
git gc                           # garbage-collect: pack loose objects, prune old unreachable ones
git repack -ad                   # repack everything into one fresh packfile, dropping redundant packs
git fsck --unreachable           # list commits/blobs no branch, tag, or reflog entry points to anymore
```

## Line Endings (`.gitattributes`, [Lesson 13](../lessons/13-pro-workflows-and-best-practices.md))

```gitattributes
* text=auto           # normalize line endings for anything Git detects as text
*.sh text eol=lf       # force shell scripts to always use LF, even on Windows checkouts
*.png binary            # never try to diff/normalize binary files
```

## Diagnosing Where You Are

```bash
git status                       # always start here
git log --oneline -5             # last 5 commits
git branch -vv                   # tracking + ahead/behind
git remote -v                    # what remotes exist and where they point
git diff HEAD                    # everything changed since last commit
```
