# Lesson 03 — Under the Hood: Blobs, Trees, Commits & the Object Database

## Goal
See exactly how Git stores your project internally. This is what separates
"I follow tutorials" from "I understand Git" — and it's what lets you
calmly recover from disasters in Lesson 06, because you'll know nothing is
ever *really* lost as long as a commit was made.

## Prerequisites
[Lesson 02 — Core Concepts & Git Anatomy](02-core-concepts-and-git-anatomy.md)

## After This Lesson You Will Be Able To
- Explain what a SHA hash is and why Git uses it
- Explain the difference between a blob, a tree, and a commit object
- Inspect Git's internal object database with plumbing commands

---

## Git Is a Content-Addressable Key-Value Store

Strip away everything else, and Git is fundamentally a simple database:
you give it content, it gives you back a key. That key is a **SHA hash** —
a fingerprint computed from the content itself.

```python
# Simplified Python analogy of what Git does internally:
import hashlib

def git_hash_object(content: bytes) -> str:
    header = f"blob {len(content)}\0".encode()
    full = header + content
    return hashlib.sha1(full).hexdigest()

git_hash_object(b"print('hello world')\n")
# → '8ab686eafeb1f44702738c8b0f24f2567c36da6' (some fixed hash)
```

**Critical property: identical content always produces the identical
hash.** This is why Git storage is so efficient — if ten commits all
contain a file that never changed, Git stores that file's content
**once**, and all ten commits just point to the same hash.

There are 4 object types in Git's database, and *everything* is one of
these:

```
BLOB     →  raw file content (no filename, no metadata — just bytes)
TREE     →  a folder listing (filenames + which blob/tree hash each maps to)
COMMIT   →  a snapshot pointer: which tree, which parent commit(s), author, message
TAG      →  a named, permanent pointer to a specific commit (Lesson 10)
```

---

## Blob: Just the Bytes

A **blob** stores only the *content* of a file — never its filename. If
you rename a file without changing its content, Git stores the exact same
blob hash as before; only the *tree* (the folder listing) records the new
name.

```bash
mkdir demo && cd demo
git init
echo "hello" > file.txt
git hash-object file.txt
# → ce013625030ba8dba906f756967f9e9ca394464
```

That hash is deterministic — anyone, anywhere, who hashes a file
containing exactly "hello\n" gets that exact same SHA. This is also why
two files with identical content in the same repo (or even different
repos!) share one blob internally.

---

## Tree: A Snapshot of a Folder

A **tree** object is Git's version of a directory listing. It maps names
to hashes:

```
tree a1b2c3...
100644 blob 8ab686e...   login.py
100644 blob 3f9a2e1...   utils.py
040000 tree 9c8d7f6...   tests/        ← a subfolder is just another tree object
```

Nested folders are just trees pointing to other trees. A commit's "whole
project snapshot" is really just **one root tree** — and that tree
recursively describes everything.

---

## Commit: The Snapshot + Metadata

A **commit** object ties it all together:

```
commit 9fceb02...
tree 4b825dc...              ← the ENTIRE project snapshot at this moment
parent 5c3a1e2...            ← the previous commit (empty if this is the first ever)
author Jane Doe <jane@example.com> 1706558400 -0500
committer Jane Doe <jane@example.com> 1706558400 -0500

Fix null pointer on login timeout
```

The commit hash is computed from *all of the above* — meaning if even one
character of the commit message changes, or the tree changes, or the
parent changes, you get a **completely different hash**. This is what
makes Git history tamper-evident: you cannot secretly edit an old commit
without its hash (and every descendant commit's hash) changing.

```
A(a1b2) ← B(c3d4) ← C(e5f6) ← D(g7h8)
                                  ↑
                               HEAD → main
```

Each arrow above is really "contains the hash of," not a separate link —
the parent pointer is baked directly into the commit's own content, which
is part of why the chain is cryptographically tamper-evident.

---

## Seeing This For Real: Git Plumbing Commands

Git has two command tiers: **porcelain** (the friendly commands you'll use
99% of the time — `add`, `commit`, `push`) and **plumbing** (low-level
commands that expose the object database directly). Let's use plumbing
just this once, to prove all of the above is real, not theory.

```bash
mkdir git-internals-demo && cd git-internals-demo
git init
echo "print('v1')" > app.py
git add app.py
git commit -m "First commit"

# See the commit object itself:
git cat-file -p HEAD
# commit output shows: tree <hash>, author, committer, message

# See the tree that commit points to:
git cat-file -p HEAD^{tree}
# 100644 blob <hash>    app.py

# See the raw blob content:
git cat-file -p <blob-hash-from-above>
# print('v1')

# List every object Git has ever stored in this repo:
git cat-file --batch-check --batch-all-objects
```

Try changing `app.py` and committing again — then inspect the new commit.
You'll see a **new tree hash** (because content changed) but the *old*
blob from commit 1 still exists in the object database, untouched,
forever (until Git garbage-collects genuinely unreachable objects).

---

## Packed Objects & Garbage Collection

Everything above described **loose objects** — literally one file per
object, stored at `.git/objects/xx/yyyy...` where `xx` is the first two
hex characters of the SHA and `yyyy...` is the rest. That's simple, but
wasteful: thousands of tiny compressed files, each with its own filesystem
overhead, and no sharing of near-duplicate content between them.

```bash
git count-objects -v
# count: 47              ← loose objects sitting around right now
# size: 188               ← their total size on disk (KB)
# in-pack: 15230          ← objects already packed
# packs: 2                ← how many packfiles exist
```

A **packfile** solves this: Git bundles many objects into one file,
compresses them together, and stores most objects as **deltas** (diffs
against a similar object already in the pack) rather than full content.
This is why cloning a huge repo transfers a couple of `.pack` files, not
hundreds of thousands of individual object files — packing is exactly
what makes Git's network protocol efficient.

```bash
git gc                      # Git's own housekeeping: packs loose objects,
                              # removes genuinely unreachable ones once they're
                              # past the reflog's expiry window (Lesson 06)
git repack -ad               # manually force a full, aggressive repack
                              # (-a: pack everything into one pack, -d: delete
                              # the old redundant packs afterward)
```

Git runs a lightweight `gc` automatically every so often (roughly every
few hundred loose objects, or after certain operations) — you rarely need
to run it by hand, but knowing it exists explains why a repo that felt
sluggish after months of history sometimes gets noticeably snappier after
one.

### Dangling Commits

A commit that no branch, no tag, and nothing else points to is called
**dangling** — typically left behind by `git commit --amend`, a `rebase`,
or a `reset --hard` (Lesson 06). It's not deleted; it just has no
"handle" pointing at it anymore. As long as it's still in your reflog (or
even if it's aged out of the reflog but `gc` hasn't run yet), it's fully
recoverable.

```bash
git fsck --unreachable       # lists dangling objects, including orphaned commits
git cat-file -p <hash>       # inspect what one of them actually contains
```

This is the deeper mechanism behind Lesson 06's "nothing is ever really
lost" guarantee: a commit only truly disappears once (a) nothing
references it, reflog included, **and** (b) `git gc` actually runs and
prunes it — which by default requires it to be at least a couple of
weeks old. In practice, for anything you noticed within days, it's still
sitting on disk somewhere, findable with `fsck` even if you'd already
forgotten the exact commit hash.

---

## Why This Matters in Practice

1. **Nothing is truly deleted just by moving HEAD around.** As long as a
   commit hash exists *somewhere* Git can find it (a branch, a tag, or
   even just the reflog), the objects it points to are safe. This is the
   entire foundation of Lesson 06's "undo anything" superpowers.
2. **Two branches with the same file content share storage.** Branching
   in Git is cheap not just because of the pointer trick (Lesson 05), but
   because unchanged files across branches literally share the same blob.
3. **Commit hashes are how Git detects "have these two people converged
   on the same state."** If your local `main` and GitHub's `main` have the
   same commit hash, you are byte-for-byte, provably in sync — not just
   "looks the same."

---

## Exercise

In your `git-internals-demo` folder from above:

1. Create two files with **identical content** (e.g. both containing just
   `hello\n`) and commit them. Run `git cat-file -p HEAD^{tree}` — confirm
   both filenames point to the **same blob hash**.
2. Amend your last commit's message with `git commit --amend -m "new message"`
   and run `git log --oneline`. Notice the commit hash changed even though
   no file content changed — because the commit object's content (which
   includes the message) changed.
3. Run `git log --oneline --graph --all` in this tiny repo. It'll look
   unimpressive with one branch, but this exact command becomes essential
   once you have multiple branches (Lesson 05).

Continue to [Lesson 04 — Your First Repo](04-your-first-repo.md) to start
using this for real, end to end.
