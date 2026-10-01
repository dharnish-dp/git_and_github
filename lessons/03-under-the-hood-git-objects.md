# Lesson 03 — Under the Hood: Blobs, Trees, Commits & the Object Database

## Goal
See exactly how Git stores your project internally. This is what separates
"I follow tutorials" from "I understand Git". It also lets you stay calm
when you recover from mistakes in Lesson 06, because you will know that
nothing is ever *really* lost once a commit has been made.

## Prerequisites
[Lesson 02 — Core Concepts & Git Anatomy](02-core-concepts-and-git-anatomy.md)

## After This Lesson You Will Be Able To
- Explain what a SHA hash is and why Git uses it
- Explain the difference between a blob, a tree, and a commit object
- Inspect Git's internal object database with plumbing commands
- Explain loose objects, packfiles, and dangling commits

## Words You Will Meet in This Lesson

Each is explained in detail later. This list is only a preview.

| Word | Plain meaning |
|------|---------------|
| **Object** | One stored item inside Git's database (a file's content, a folder listing, or a commit). |
| **Hash / SHA** | A fingerprint (a long string of letters and digits) computed from some content. |
| **Blob** | An object that holds the content of one file. |
| **Tree** | An object that holds a folder listing. |
| **Commit** | A saved snapshot of your whole project, plus a note about who saved it and why. |
| **Object database** | The folder `.git/objects/` where all objects are kept. |

---

## 1. Git Is a Lookup Table (a Key-Value Store)

### Why do we need this?
Before we meet blobs, trees, and commits, you need one mental model that
explains all of them. Everything else in this lesson is a special case of it.

### Analogy: a Python dict
You know this already:

```python
store = {}
store["some-key"] = "some value"
print(store["some-key"])     # some value
```

Git's internal storage is exactly a `dict`. Content goes in as the value.
Git computes the key for you. Nobody picks the key.

### How does Git compute the key?
It runs the content through a **hash function**.

- A **hash function** takes any content and produces a fixed-length
  fingerprint of it.
- Git's fingerprint is called a **SHA-1 hash** (SHA = "Secure Hash
  Algorithm"). In practice it is a 40-character string of hex digits
  (`0-9` and `a-f`).
- Same content always gives the same fingerprint. Different content gives
  a different fingerprint.

Here is a close Python version of what Git does. You can run it:

```python
import hashlib

def git_hash_object(content: bytes) -> str:
    header = f"blob {len(content)}\0".encode()   # Git adds a small label first
    return hashlib.sha1(header + content).hexdigest()

print(git_hash_object(b"hello\n"))
```

Expected output:

```
ce013625030ba8dba906f756967f9e9ca394464a
```

Line by line:
- `header` is a tiny label, `blob 6\0`. It says "this is a blob and it is
  6 bytes long". Git puts this label in front of the content before hashing.
- `hashlib.sha1(...)` is the fingerprint machine.
- `.hexdigest()` turns the fingerprint into the 40-character text you see.

### Why identical content means identical hash matters
This is the most important property in this lesson:

> Identical content always produces the identical hash.

Three consequences:
1. If ten commits all contain a file that never changed, Git stores that
   file's content **once**. All ten commits point to the same hash.
2. Two files with the same content in one repo share one stored copy.
3. If content changes by even one character, the hash changes completely.

**Common confusion: "Is the hash random, like a UUID?"**
No. A UUID is random, so making one twice gives two different values.
A Git hash is computed from the content, so making it twice gives the same
value. Think of it as `key = f(content)`, not `key = random()`.

**Common confusion: "Can two different files get the same hash?"**
In theory yes, in practice no. With 40 hex digits there are far more
possible hashes than there are atoms you could ever use for storage. Git
treats "same hash" as "same content".

### The four kinds of object
Everything Git stores is one of four object types:

```
BLOB     ->  the content of one file (no filename, no metadata, just bytes)
TREE     ->  a folder listing (names, each mapped to a blob or another tree)
COMMIT   ->  a snapshot: which tree, which parent commit, who, when, why
TAG      ->  a permanent, named label on a commit (Lesson 10)
```

We take them one at a time, in order: blob, tree, commit. We leave tag for
Lesson 10.

---

## 2. Blob: Just the Bytes

### Why do we need this?
Git has to store the contents of your files somewhere. The blob is that
storage unit.

### Analogy: a book with no cover
Imagine photocopying the pages of a book but leaving off the cover and the
title. You have the content only. You cannot tell what the book was
"called". A blob is that: content with no name.

### Plain explanation
A **blob** (short for "binary large object", though you can just read it
as "file contents") stores only the *content* of a file. It never stores
the filename.

Where does the filename live? In the *tree*, which comes next. So if you
rename a file without changing its content, the blob stays identical. Only
the tree changes.

### Try it
`git init` creates an empty repository (an empty project folder that Git
tracks). Then we ask Git what hash a file would get.

```bash
mkdir demo && cd demo
git init
echo "hello" > file.txt
git hash-object file.txt
```

Expected output:

```
Initialized empty Git repository in /your/path/demo/.git/
ce013625030ba8dba906f756967f9e9ca394464a
```

Line by line:
- The first line confirms Git created its hidden `.git` folder.
- `git hash-object file.txt` computes the hash of the file's content.
  It only *prints* the hash. It does **not** store anything yet.
- `echo "hello"` writes `hello` plus a newline (`\n`). The hash is of
  `hello\n`, which is exactly what our Python function above hashed. Note
  the output matches.

That hash is the same on every computer in the world for a file that
contains exactly `hello` and a newline.

**Doubt? "If a blob has no name, how does Git know which file is which?"**
The *tree* tells it. A tree says "the name `file.txt` maps to blob
`ce0136...`". That is the next section.

---

## 3. Tree: A Snapshot of a Folder

### Why do we need this?
Blobs have no names and no folders. We need something that says "this
blob is called `login.py` and lives in this folder".

### Analogy: a table of contents
A tree is a table of contents. Each row has a name and points to the real
content (a blob) or to a sub-table (another tree).

### Plain explanation
A **tree** object is Git's version of a directory listing. Each row holds
four things: file mode, type, hash, name.

```
100644 blob 75d9766...   login.py
100644 blob 3f9a2e1...   utils.py
040000 tree 9c8d7f6...   tests
```

Column meaning:
- `100644` is the **file mode**: a normal, non-executable file. (`100755`
  means executable. `040000` means a directory.)
- `blob` or `tree` says what the hash points to.
- The hash says where the content lives in the database.
- The last column is the name.

A subfolder (`tests` above) is just another tree. So a tree can point to
other trees, which point to other trees, and so on.

Diagram legend (used in this lesson):

```
  --label-->   "contains the hash of": the object on the left holds the
               hash of the object on the right; the label says what the
               link means (a filename, "tree", "parent")
  *            a new object created by the command just run
  BEFORE/AFTER the state around a command shown between them
```

How to read this: start at the root tree and follow each arrow; the label
on the arrow is the name stored in the tree row.

```
root tree
  --login.py-------> blob 75d9766...
  --utils.py-------> blob 3f9a2e1...
  --tests/---------> tree 9c8d7f6...
                       --test_login.py--> blob a1b2c3...
                       --conftest.py-----> blob d4e5f6...
```

A project snapshot is therefore **one root tree**, which describes
everything beneath it.

**Common confusion: "Does Git store a copy of my whole folder for every
commit?"**
Logically yes, physically no. Every commit points to a root tree, so each
commit *describes* the whole project. But unchanged files reuse the same
blob hashes, so only changed content is stored again.

---

## 4. Commit: The Snapshot + Who, When, Why

### Why do we need this?
A root tree is a snapshot, but it has no author, no date, no message, and
no idea what came before. The commit adds all of that.

### Analogy: a photo with a label on the back
The tree is the photo of the project. The commit is the label on the back:
"taken by Jane, on this date, reason: fix login bug, previous photo is
this one."

### Plain explanation
A **commit** is a saved snapshot of your whole project. In the database it
is a small text object like this:

```
tree 9f18d5e072c841dadc0a6e238fa57bb3cca04001
parent 5c3a1e2...
author Jane Doe <jane@example.com> 1706558400 -0500
committer Jane Doe <jane@example.com> 1706558400 -0500

Fix null pointer on login timeout
```

Line by line:
- `tree` is the root tree, meaning the entire project at this moment.
- `parent` is the commit that came right before this one. The very first
  commit in a repo has no `parent` line.
- `author` is who wrote the change. The number `1706558400` is the time
  in seconds since 1 Jan 1970 (a "Unix timestamp"). `-0500` is the
  timezone.
- `committer` is who recorded the commit. Usually the same person as
  the author. They differ when someone applies another person's patch.
- The blank line, then the **commit message**.

### Why a change anywhere changes the hash
The commit's hash is computed from the entire text above. So if any of
these change, the commit's hash changes completely:
- the message
- the tree (meaning any file changed)
- the parent
- the author or time

The parent hash is written *inside* the commit text. So a commit's hash
depends on its parent's hash, which depends on *its* parent's hash, and
so on. Changing an old commit therefore changes every commit after it.
This is why Git history is **tamper-evident** (you cannot secretly edit
the past without everyone seeing that the hashes no longer match).

How to read this: time flows left to right; the oldest commit is on the
left. The short ids in brackets stand for each commit's hash.

```
  A(a1b2)---B(c3d4)---C(e5f6)---D(g7h8)
                                   ^
                                 main
                                 HEAD -> main
```

Note: Git internally stores, inside each commit, the hash of its parent
(C contains B's hash, B contains A's hash). We draw time flowing left to
right, so that link points "backwards" in reality. There is no separate
link record. The parent hash is simply part of the child's content.

What happens if you edit old commit B (before and after):

```
BEFORE:   A(a1b2)---B(c3d4)---C(e5f6)---D(g7h8)
```

```
  (edit B's message)
```

```
AFTER:    A(a1b2)---B*(x9y8)---C*(p7q6)---D*(m5n4)
```

What changed: B gets a new hash, so C (which contains B's hash) must also
change, and so must D. `A` is the only commit that stays the same. (The ids
here are illustrative.)

(`HEAD` is a pointer to the commit you are currently on, and `main` is a
branch, a movable label on a commit. Both were covered in Lesson 02.)

**Doubt? "I only changed the commit message. Why did the hash change?"**
Because the message is part of the commit text that gets hashed. You will
prove this yourself in the exercise.

### The big picture in one diagram

How to read this: start at the commit on the left and follow each labelled
arrow to the object it points to. Arrow label = what that link means.

```
commit 0844913
  --tree---> tree 9f18d5e --app.py--> blob e59f059
             (root folder)            (file content)
  --parent-> previous commit (none for the first commit)
```

What this shows: a commit points to exactly one root tree (the snapshot)
and to its parent commit (or none for the first commit). The tree points
to blobs (file contents) and to sub-trees.

---

## 5. Seeing This For Real: Git Plumbing Commands

### Why do we need this?
So far this is theory. In this section you will look at the real objects
and see that nothing above was made up.

### Porcelain vs plumbing
Git has two tiers of commands:
- **Porcelain** are the friendly everyday commands: `add`, `commit`,
  `push`.
- **Plumbing** are low-level commands that work directly on the object
  database. You rarely use them, but they let us look inside.

(The names are a joke: porcelain is the shiny toilet, plumbing is the
pipes behind it.)

### Step 1: Make a tiny repo with one commit

```bash
mkdir git-internals-demo && cd git-internals-demo
git init
echo "print('v1')" > app.py
git add app.py
git commit -m "First commit"
```

`git add app.py` puts the file in the **staging area** (a waiting room for
changes you plan to include in the next commit). `git commit` then saves
the snapshot.

Expected output of the commit:

```
[main (root-commit) 0844913] First commit
 1 file changed, 1 insertion(+)
 create mode 100644 app.py
```

- `root-commit` means this is the first commit, so it has no parent.
- `0844913` is the start of the commit's hash. Yours will differ because
  your name and the time differ.

### Step 2: Look at the commit object
`git cat-file -p` prints an object. (`cat-file` = show an object, `-p` =
"pretty-print" it.) `HEAD` means "the commit I am on".

```bash
git cat-file -p HEAD
```

Expected output (your hashes, name and time will differ):

```
tree 9f18d5e072c841dadc0a6e238fa57bb3cca04001
author J <j@e.com> 1790863616 +0530
committer J <j@e.com> 1790863616 +0530

First commit
```

This is exactly the structure from section 4. There is no `parent` line
because it is the first commit.

### Step 3: Look at the tree the commit points to
`HEAD^{tree}` is Git's way of saying "the tree inside the commit HEAD".

```bash
git cat-file -p 'HEAD^{tree}'
```

(The quotes stop your shell from treating the braces specially. This
matters in zsh.)

Expected output:

```
100644 blob e59f0597d775f35cd6a0de643da6842730edbde9	app.py
```

Meaning: one normal file called `app.py`, whose content is the blob
`e59f059...`.

### Step 4: Look at the blob
Use the blob hash from the previous output. Git accepts a shortened hash
as long as it is unambiguous.

```bash
git cat-file -p e59f059
```

Expected output:

```
print('v1')
```

That is your file's content, pulled back out of the database by its
fingerprint.

### Step 5: List every object in the repo

```bash
git cat-file --batch-check --batch-all-objects
```

Expected output:

```
0844913c54186226439f47dd7b6c3daefa5234d2 commit 135
9f18d5e072c841dadc0a6e238fa57bb3cca04001 tree 34
e59f0597d775f35cd6a0de643da6842730edbde9 blob 12
```

Each line is: full hash, object type, size in bytes. One commit, one
tree, one blob. That is the entire database for a one-file, one-commit
project.

### Step 6: Make a second commit and compare

```bash
echo "print('v2')" > app.py
git add app.py
git commit -m "Second commit"
git cat-file -p HEAD
git cat-file -p 'HEAD^{tree}'
```

What you will see:
- The commit now has a `parent` line pointing at the first commit.
- The tree has a **new hash**, because `app.py` changed.
- `app.py` points to a **new blob**.
- The *old* blob from commit 1 (`e59f059...`) is still in the database.
  Run the `--batch-all-objects` command again to see 6 objects.

Old objects stay until Git garbage-collects objects nothing points to
(explained in section 6).

How to read this: each row is one object. `--tree-->` and `--blob-->`
mean "this object contains the hash of that object". Arrows are labelled
with what they mean.

```
BEFORE  (after commit 1)

commit 0844913 --tree--> tree 9f18d5e --app.py--> blob e59f059 (v1)
```

```
  $ echo "print('v2')" > app.py ; git add app.py ; git commit
```

```
AFTER  (after commit 2)

commit 0844913 --tree--> tree 9f18d5e --app.py--> blob e59f059 (v1)
     ^
     '--parent-------------.
                           |
commit (new)* --tree--> tree (new)* --app.py--> blob (new)* (v2)
```

What changed: three new objects (`*`): commit, tree, blob. The old three
are untouched and still in the database. That is why you count 6 objects.
Only the newest commit has a `parent` line pointing back to the first.

---

## 6. Where Objects Live: Loose Objects and Packfiles

### Why do we need this?
Now you know *what* is stored. This section covers *where* and *how*, and
explains two things you will see in real life: `git gc` and why clones are
fast.

### Loose objects
When you commit, each new object is written as its own small compressed
file. These are called **loose objects**. The path comes from the hash:
the first 2 characters become a folder name, the other 38 become the
filename.

```bash
ls .git/objects
```

Expected output:

```
08  9f  e5  info  pack
```

Folders `08`, `9f`, `e5` match the first two characters of our commit,
tree, and blob hashes. For example the commit lives at
`.git/objects/08/44913c54186226439f47dd7b6c3daefa5234d2`.

Loose objects are simple but wasteful at scale: thousands of tiny files,
and no sharing between near-duplicates (for example two versions of a
10,000-line file that differ by one line).

### Check how many you have

```bash
git count-objects -v
```

Example output for a bigger, older repo:

```
count: 47
size: 188
in-pack: 15230
packs: 2
size-pack: 9012
prune-packable: 0
garbage: 0
size-garbage: 0
```

- `count` is the number of loose objects right now.
- `size` is their total size on disk, in KB.
- `in-pack` is the number of objects already inside packfiles.
- `packs` is how many packfiles exist.
- `size-pack` is the packfiles' total size in KB.

In our tiny demo repo you will see `count: 3` (then 6) and `in-pack: 0`.

### Packfiles
A **packfile** is a single file (`.pack`) that bundles many objects
together. Git compresses them together and stores most of them as
**deltas** (a delta is "this object equals that similar object, plus these
small differences", like a diff instead of a full copy).

Analogy: instead of keeping 50 full printed copies of a report that
differ by a few lines, keep one full copy and 49 sticky notes of edits.

Packing also explains why cloning (downloading a full copy of a repo) is
fast. Git sends a few `.pack` files, not hundreds of thousands of
individual object files.

### Garbage collection

```bash
git gc
```

**Garbage collection (`gc`)** is Git's housekeeping. It packs loose
objects and deletes objects that are unreachable and old enough. (What
"unreachable" means is explained next.) After it, `git count-objects -v`
will show the objects moved from `count` to `in-pack`.

```bash
git repack -ad
```

This is the manual, aggressive version. `-a` packs everything into one
pack. `-d` deletes the old packs that are now redundant.

Git also runs a light `gc` automatically every so often, so you rarely
run it by hand.

---

## 7. Dangling Commits: Why Nothing Is Really Lost

### Why do we need this?
Later you will run commands like `git commit --amend` or `git reset
--hard` that seem to "destroy" commits. This section explains why they
do not, and how to find them again.

### Analogy: a library book with no catalog card
A commit is a book on a shelf. A branch, tag, or `HEAD` is a catalog
card that points to it. If you throw away the catalog card, the book is
still on the shelf. It is just hard to find. It is only removed when the
library does a clean-up later.

### Plain explanation
A **dangling commit** is a commit that still exists in the database but
that no branch, no tag, and nothing else points to. It is typically left
behind by:
- `git commit --amend` (replaces the last commit with a new one)
- `git rebase` (re-creates commits; Lesson 09)
- `git reset --hard` (moves a branch backward; Lesson 06)

The **reflog** is Git's private diary of where `HEAD` and your branches
have pointed recently (Lesson 06). Commits still listed there are also
safe.

How to read this: history runs left to right, and `main` is a label on
one commit. Here is a dangling commit, before and after
`git commit --amend`.

```
BEFORE amend:

      A---B---C
              ^
            main
            HEAD -> main
```

```
  $ git commit --amend -m "better message"
```

```
AFTER amend:

      A---B---C         (old C: dangling, nothing points at it)
          \
           C*           (new commit, same parent B, new hash)
           ^
         main
         HEAD -> main
```

What changed: the `main` label moved from `C` to the new `C*`. `A` and `B`
are unchanged. `C` was not deleted; it just has no label pointing at it
any more, so it is "dangling" until `git gc` removes it.

### Finding them

```bash
git fsck --unreachable
```

`fsck` = "file system check". It walks the database and lists objects
nothing points to. Example output:

```
unreachable commit 7d2f9a1c3b5e4f60718293a4b5c6d7e8f9012345
unreachable blob 3fa6d1c2b8e94075a1b2c3d4e5f60718293a4b5c
```

Each line is: `unreachable`, the object type, the full hash. To see what one
contains:

```bash
git cat-file -p 7d2f9a1
```

This prints the commit (tree, author, message), so you can recognise it.
Once you know the hash, Lesson 06 shows how to get it back onto a branch.

### When is a commit really gone?
Only when both are true:
1. Nothing references it, including the reflog.
2. `git gc` runs **and** the object is old enough. By default that is
   roughly two weeks (unreachable objects are kept for a grace period).

So anything you noticed within days is almost certainly still on disk.

**Common confusion: "Unreachable commit means deleted, right?"**
No. "Unreachable" means "no pointer leads to it". The data is still there
until `gc` removes it.

---

## 8. Why This Matters in Practice

1. **Moving pointers does not delete data.** Moving `HEAD` or a branch
   never deletes commits. As long as some pointer (branch, tag, or
   reflog entry) leads to a commit, its objects are safe. This is the
   base of Lesson 06.
2. **Branches are cheap.** A branch is just a tiny pointer (Lesson 05),
   and unchanged files on different branches share the same blob.
3. **Hashes prove two copies are identical.** If your local `main` and
   GitHub's `main` show the same commit hash, they are byte-for-byte the
   same, because the commit hash covers the tree, which covers every file.

A Python-test analogy: a commit hash is like a checksum of your whole test
suite folder. Equal checksums mean equal suites, with no need to compare
files one by one.

---

## Exercise

Work in your `git-internals-demo` folder.

### Part 1: Same content, one blob
1. Create two files with identical content and commit them:
   ```bash
   echo "hello" > a.txt
   echo "hello" > b.txt
   git add a.txt b.txt
   git commit -m "Add two identical files"
   git cat-file -p 'HEAD^{tree}'
   ```
2. Expected: the tree lists `a.txt` and `b.txt` with the **same blob
   hash** (`ce01362...`), proving Git stored the content only once:
   ```
   100644 blob ce013625030ba8dba906f756967f9e9ca394464a	a.txt
   100644 blob e59f0597d775f35cd6a0de643da6842730edbde9	app.py   (or the v2 hash)
   100644 blob ce013625030ba8dba906f756967f9e9ca394464a	b.txt
   ```
   (The `app.py` line will have the hash of whichever version you have.)

### Part 2: Amending changes the hash
1. Run `git log --oneline` and note the top hash.
2. Run `git commit --amend -m "new message"`.
3. Run `git log --oneline` again.
4. Expected: the commit hash is different, although no file changed. The
   message is part of the commit object, so changing it produces a new
   object. Run `git cat-file -p HEAD^{tree}` (quoted) and notice the tree
   hash is unchanged.

### Part 3: Find the old commit
1. Run `git fsck --unreachable`.
2. Expected: the pre-amend commit may appear as `unreachable commit ...`.
   It might not show if the reflog still references it. In that case
   `git reflog` will list it instead. Either way it still exists.
3. Run `git cat-file -p <that-hash>` to read its old message.

### Part 4: See the history drawing
Run `git log --oneline --graph --all`. Expected output is a simple
straight line of commits with one `*` each. It looks plain now, but this
command becomes essential once you have several branches (Lesson 05).

---

## Recap in 5 Lines
1. Git is a lookup table: the key is a hash computed from content, the value is the content.
2. A **blob** is file content (no name), a **tree** is a folder listing, a **commit** is a snapshot plus author, message and parent.
3. Changing anything changes the hash, and a commit's hash includes its parent's hash, so history is tamper-evident.
4. Objects are stored loose at first, later bundled into compressed **packfiles** by `git gc`.
5. "Deleted" commits are usually just **dangling** (unreachable), and stay recoverable with `git fsck` and the reflog until `gc` prunes them.

Continue to [Lesson 04 — Your First Repo](04-your-first-repo.md) to start
using this for real, end to end.
