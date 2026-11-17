# Lab: Git Basics, Seen in My Own Repo

**Started:** 2026-11-17 · **Map:** [Git](README.md)

Hands-on lessons that follow the map's **Minimum Mental Model**. The [map](README.md) holds the ideas; this file holds what I ran and what I saw. Read-only lessons use this repo (`learning-map`); lessons that change things use a throwaway sandbox repo in `/tmp`.

| # | Lesson | Map idea | Status |
| - | ------ | -------- | ------ |
| 1 | The three areas | Working tree → index → repository | ✅ Done |
| 2 | A commit is a snapshot with a hash | Snapshot, content hash, commit graph | ✅ Done |
| 3 | A branch is just a pointer | Refs and HEAD | ✅ Done |
| 4 | Local vs shared | Remote is a stale view; push and fetch | Next |
| 5 | Merge and conflict | Merge base; conflict is textual | |
| 6 | Nothing is lost (almost) | Reflog; revert over rewrite | |

---

## Lesson 1: The three areas

```
Working tree          Index (staging area)          Repository
(files I edit)   ──git add──▶  (next snapshot)  ──git commit──▶  (history)
```

- **Working tree**: the normal files in Finder and my editor. A scratchpad.
- **Index**: a *draft* of the next commit. I choose what goes in.
- **Repository**: the saved history (commits).

**A commit records the index, not my folder.** `git add` decides *what*; `git commit` decides *when*.

### Reading `git status --short`

```
 M maps/README.md
^^
||
|└─ second column: working tree vs index → M = changed, not staged
└── first column:  index vs last commit  → space = nothing staged
```

| Symbol | Meaning |
| ------ | ------- |
| `M ` | Staged: the index differs from the last commit |
| ` M` | Not staged: the working tree differs from the index |
| `MM` | Both: some changes staged, more changes not staged |
| `??` | Untracked: Git doesn't know the file at all |

Real example: while committing the Tailscale map, `maps/README.md` showed `MM`. Only the Tailscale line was staged; 4 lines for other maps stayed in the working tree. The commit took the 1 staged line. Afterwards it showed ` M`: the commit had emptied the index, and the working tree hadn't changed.

### Comparing the areas

| Command | Compares | What I saw |
| ------- | -------- | ---------- |
| `git status --short` | All three areas | ` M maps/README.md`, plus `??` for the Economics, Server, and Early Childhood folders |
| `git diff --stat` | Working tree ↔ index | `maps/README.md` with 4 insertions |
| `git diff --cached --stat` | Index ↔ last commit | Nothing: the index was empty |

- `git diff` ignores `??` files. It only compares files Git already tracks.
- Files that are the same in all three areas don't appear in `git status` at all. No news is good news.

### Untracked means unprotected

`??` files exist only in my folder: not in the index, not in any commit. If I delete one by mistake, Git can't bring it back. This is the map's misconception "Git is a backup system": Git only protects what I commit (and push).

---

## Lesson 2: A commit is a snapshot with a hash

### History is a chain that points backwards

`git log --oneline -6`:

```
531c14f Add SSH shortcut and Claude Code path sections to lab notes
e5e3906 Add sshTests, exit node lesson, and cleanup to lab notes
a6cf3f2 Add restart test section to server lock-down lab notes
6e282dd Add Tailscale SSH section to server lock-down lab notes
cd0cd85 Add Tailscale learning map and server lock-down lab notes
5bd51ff Format Jobs to be Done tables
```

```
5bd51ff ◀── cd0cd85 ◀── 6e282dd ◀── a6cf3f2 ◀── e5e3906 ◀── 531c14f  ◀── HEAD, main
(older)                                                         (newest)
```

A new commit knows its parent; a parent never knows its children. History only grows at the end: an append-only graph. The short codes are the first 7 of 40 hash characters.

### Inside a commit: no diff

`git cat-file -p HEAD`:

```
tree 33e4f3c57b3cde042bd03ac7e79b0bc3a624db90
parent e5e3906c408b88b78fdd4cd96799bb9e114ae59f
author Weicheng Jiang <email> 1794901355 -0800
committer Weicheng Jiang <email> 1794901355 -0800

Add SSH shortcut and Claude Code path sections to lab notes
```

| Line | Meaning |
| ---- | ------- |
| `tree` | The snapshot: the top folder of the project at this moment |
| `parent` | The commit before (the backward arrow) |
| `author` | Who wrote the change, with time (seconds since 1970) and time zone |
| `committer` | Who put it into history; usually the same person |
| message | After the empty line |

A commit = **snapshot + who, when, why, and what came before**. The diffs on GitHub are *computed* by comparing two trees; they are not stored.

### Why history is tamper-evident

The commit's ID is a hash of exactly that text, including the `parent` line. Change one byte in an old commit → its hash changes → the next commit's `parent` line changes → its hash changes → every later ID changes. Nobody can quietly edit history.

### Inside the snapshot: trees and blobs

`git cat-file -p HEAD^{tree}` (the tree of the HEAD commit):

```
040000 tree 004b111…   .claude
100644 blob 719900f…   .gitignore
100644 blob 2c5f7ee…   CLAUDE.md
100644 blob c765384…   README.md
040000 tree d241ce4…   maps
040000 tree 530b1ce…   prompts
```

```
100644 blob 2c5f7ee…   CLAUDE.md
──┬─── ─┬── ───┬────   ───┬─────
  │     │      │          └ name
  │     │      └ hash of the contents
  │     └ blob = a file's contents, tree = a folder
  └ mode: 100644 = normal file, 040000 = folder
```

- A **blob** has no name. Names live in **trees**.
- A snapshot is a tree of trees.
- `.claude` and `.gitignore` start with a dot, so `ls` hides them, but Git tracks them (`ls -a` shows them).

`git cat-file -p HEAD:maps` (the `maps` folder in the HEAD snapshot) listed every committed map, but **not** `economics`, `server`, or `early-childhood-education-through-ai`: untracked files are not part of history.

### The hash is a fingerprint of the contents

In that listing, `maps/README.md` was blob `6b8e183`. When the one-line version of that file was staged earlier, the diff said `index 1690fec..6b8e183`: the same hash. Same contents → same hash, on any computer.

### Unchanged files are stored once

`git rev-parse HEAD:CLAUDE.md 5bd51ff:CLAUDE.md` printed the **same hash twice**:

```
commit 5bd51ff ─▶ tree ─▶ CLAUDE.md ─┐
commit cd0cd85 ─▶ tree ─▶ CLAUDE.md ─┤
     …                               ├──▶ blob 2c5f7ee  (stored once)
commit 531c14f ─▶ tree ─▶ CLAUDE.md ─┘
```

Every commit is a full snapshot, but a new commit only stores blobs and trees whose contents are new. This is **content addressing**: an object's address is its content's hash.

**Typo note:** I first typed `CLAUDE.cd`. Git said `path 'CLAUDE.cd' does not exist` — a clear error, unlike `systemctl`, which just says `inactive` for a misspelled service.

### Lesson 2 summary

| I saw | It shows |
| ----- | -------- |
| `git cat-file -p HEAD` | A commit = tree + parent + author + message. No diff. |
| `HEAD^{tree}`, `HEAD:maps` | A snapshot = trees (folders) pointing to blobs (file contents) |
| `maps/README.md` = `6b8e183` | The hash is a fingerprint of the exact contents |
| `CLAUDE.md` same in 6 commits | Unchanged files are stored once and shared |
| No `economics` in the tree | Untracked files aren't part of history |

**A commit is a picture of everything, linked to the picture before it. Git never stores the same content twice.**

---

## Lesson 3: A branch is just a pointer

Practiced in a sandbox repo, `/tmp/git-lab`, with one file (`notes.txt`) and two commits (`ac2f828 First commit`, `434ff38 Second commit`). Every command used `git -C /tmp/git-lab`: `-C` means "run Git as if I were in this folder", so I could stay in my real project.

### What `main` and `HEAD` really are

Git keeps its data in the hidden `.git` folder.

```
cat /tmp/git-lab/.git/HEAD              →  ref: refs/heads/main
cat /tmp/git-lab/.git/refs/heads/main   →  434ff383cc3a742a6d789619aa0099049a78cf52
```

- `HEAD` holds **the name of a branch**, not a commit: a pointer to a pointer.
- A branch is **one line, one hash** — a 41-byte file. No copy of files, no list of commits. The newest commit is enough, because every commit knows its parent.

```
HEAD ──▶ main ──▶ 434ff38 (Second) ──▶ ac2f828 (First)
```

### Creating a branch copies nothing

```
git -C /tmp/git-lab branch experiment
cat /tmp/git-lab/.git/refs/heads/experiment   →  434ff383cc3a742a6d789619aa0099049a78cf52
```

The same hash as `main`. Creating a branch wrote one new 41-byte file. That's why branches are nearly free, whether the project has 1 file or 10,000. Most Git commands print nothing when they succeed.

### Switching changes `HEAD`

```
git -C /tmp/git-lab switch experiment   →  Switched to branch 'experiment'
cat /tmp/git-lab/.git/HEAD              →  ref: refs/heads/experiment
```

`git branch` creates a pointer; `git switch` moves `HEAD` to it.

### Committing moves only the branch `HEAD` points to

```
echo "an idea" >> /tmp/git-lab/notes.txt && git -C /tmp/git-lab commit -q -am "Try an idea"
cat .git/refs/heads/main .git/refs/heads/experiment
  →  434ff383…   (main: unchanged)
  →  eb43ea84…   (experiment: new commit)
```

`git commit` (1) creates a new commit whose parent is the current commit, and (2) moves **the branch `HEAD` points to**. `main` didn't notice anything. Branches keep work apart by moving only one pointer, not by copying files.

### Seeing the pointers

```
git -C /tmp/git-lab log --oneline --graph --all --decorate

* eb43ea8 (HEAD -> experiment) Try an idea
* 434ff38 (main) Second commit
* ac2f828 First commit
```

- Without `--decorate`, the names were missing: Git only adds them by itself in a normal terminal.
- It's a **straight line**, not a fork: `main` just points to an older commit on the same line. A fork appears only when both branches get their own new commits (Lesson 5).
- `ac2f828` has no name but is safe: it's **reachable** by following parent arrows from `main` or `experiment` (Lesson 6).

```
ac2f828 ◀── 434ff38 ◀── eb43ea8
(First)     (Second)    (Try an idea)
              ▲              ▲
             main        experiment ◀── HEAD
```

### Switching rewrites the working tree

```
git -C /tmp/git-lab switch main && cat /tmp/git-lab/notes.txt
  →  hello
     world
```

"an idea" disappeared from the file — `switch` rewrote the working tree to match `main`'s snapshot. It's not lost: it's in commit `eb43ea8`, which `experiment` points to. Each branch is a different version of the project, and `switch` swaps the folder between them.

### Lesson 3 summary

| I did | What really happened |
| ----- | -------------------- |
| `cat .git/HEAD` | `ref: refs/heads/main` — HEAD points to a branch, not a commit |
| `cat .git/refs/heads/main` | One hash — a branch is a 41-byte file |
| `git branch experiment` | A new file with the same hash; nothing copied |
| `git switch experiment` | The `HEAD` file changed to `experiment` |
| `git commit` | New commit; only the branch HEAD points to moved |
| `git switch main` | `HEAD` changed, and the working tree was rewritten to match `main` |

**A branch is a sticky note with a commit hash on it. `HEAD` is a sticky note saying which sticky note I'm using. Committing moves my sticky note forward.**
