---
layout: post
title: "Building Git from Scratch in JavaScript"
date: 2026-04-10
categories: [programming, systems]
---

Git is everywhere, but most developers treat it as a black box. `git add`, `git commit`, `git merge` — we use the commands without understanding the elegant data structures underneath.

Today I built a working Git implementation from scratch in JavaScript. Not a wrapper around the `git` CLI — a real implementation with content-addressable storage, SHA-1 hashing, three-way merge, and the Myers diff algorithm. 88 tests, all passing.

Here's what I learned.

## Everything is an Object

Git has exactly four object types: **blobs** (file content), **trees** (directories), **commits** (snapshots with metadata), and **tags** (named references to objects). Every object is stored the same way:

```
{type} {size}\0{content}
```

This gets SHA-1 hashed, zlib-compressed, and stored at `.git/objects/{first 2 chars}/{rest of hash}`.

```javascript
export function writeObject(gitDir, type, content) {
  const buf = Buffer.from(content);
  const header = `${type} ${buf.length}\0`;
  const store = Buffer.concat([Buffer.from(header), buf]);
  const hash = createHash('sha1').update(store).digest('hex');
  
  const dir = join(gitDir, 'objects', hash.slice(0, 2));
  mkdirSync(dir, { recursive: true });
  writeFileSync(join(dir, hash.slice(2)), deflateSync(store));
  
  return hash;
}
```

This is **content-addressable storage**: the address (SHA-1 hash) is derived from the content itself. Same content always produces the same hash. This means:

- **Deduplication is free.** Two files with identical content share one blob object.
- **Integrity checking is built-in.** If the content doesn't match its hash, something is corrupt.
- **Immutability is the default.** You can't modify an object without changing its hash.

The empty blob has hash `e69de29bb2d1d6434b8b29ae775ad8c2e48c5391`. The empty tree: `4b825dc642cb6eb9a060e54bf8d69288fbee4904`. These are universal constants — every git installation produces the same hashes.

## Trees are Merkle Trees

A tree object lists its entries: `{mode} {name}\0{20-byte hash}` for each file or subdirectory. A tree can reference other trees (subdirectories) or blobs (files).

This creates a **Merkle tree** — a tree where every node's hash depends on its children's hashes. Change one file deep in the tree, and every ancestor's hash changes too. This is how git detects changes so efficiently: compare two root tree hashes. If they're the same, nothing changed. If different, recurse into the subtrees to find what changed.

My implementation handles this with a recursive `buildTreeFromEntries` that converts a flat list of indexed files into a nested tree structure:

```javascript
// Flat index entries like:
//   src/main.js, src/util.js, README.md
// Become:
//   tree: { README.md (blob), src (tree: { main.js (blob), util.js (blob) }) }
```

## Commits are a DAG

A commit points to a tree (the snapshot), zero or more parents (previous commits), and metadata (author, message, timestamp). The first commit has no parents. A merge commit has two parents.

Following parent pointers gives you the commit graph — a **directed acyclic graph**. `git log` is just a traversal of this graph:

```javascript
export function log(gitDir, maxCount = Infinity) {
  const entries = [];
  let hash = resolveHead(gitDir);
  
  while (hash && entries.length < maxCount) {
    const obj = readObject(gitDir, hash);
    const commitData = parseCommit(obj.content);
    entries.push({ hash, ...commitData });
    hash = commitData.parents[0] || null;
  }
  
  return entries;
}
```

## The Index is the Staging Area

The index (`.git/index`) is a sorted list of file entries: path, mode, SHA-1 hash, size, timestamps. When you `git add`, you're updating the index. When you `git commit`, you build a tree from the index.

The status command compares three things:
1. **HEAD tree → index**: shows staged changes
2. **Index → working tree**: shows unstaged changes
3. **Working tree − index**: shows untracked files

My implementation uses a simplified JSON format instead of git's binary index format (which is optimized for fast stat comparisons), but the semantics are identical.

## Myers Diff: Finding the Shortest Edit Script

The diff algorithm is the most mathematically interesting piece. Eugene Myers' 1986 paper describes an O(ND) algorithm where N is the input size and D is the edit distance.

The key insight: model the diff as finding a path through an edit graph. Moving right = delete from old file. Moving down = insert from new file. Moving diagonally = keep (lines match). The shortest path from top-left to bottom-right gives the minimal edit script.

```javascript
for (let d = 0; d <= max; d++) {
  for (let k = -d; k <= d; k += 2) {
    let x;
    if (k === -d || (k !== d && v[max + k - 1] < v[max + k + 1])) {
      x = v[max + k + 1]; // Move down (insert)
    } else {
      x = v[max + k - 1] + 1; // Move right (delete)
    }
    
    let y = x - k;
    while (x < n && y < m && a[x] === b[y]) { x++; y++; } // Diagonal
    
    v[max + k] = x;
    if (x >= n && y >= m) return backtrack(trace, a, b, n, m, max);
  }
}
```

The algorithm explores outward from the start, trying edit distances 0, 1, 2, ... until it finds a path. For similar files (small D), it's very fast. For completely different files, it degrades to O(N²) — but that's the theoretical minimum for string comparison.

## Three-Way Merge

Merging is where things get interesting. Git doesn't just compare two files — it finds their common ancestor and does a **three-way merge**:

1. Find the **merge base** (common ancestor commit) using BFS on the commit graph
2. For each file, compare base, ours, and theirs:
   - If only one side changed → take that side's version
   - If both sides made the same change → take either (they agree)
   - If both sides changed differently → **conflict**

```javascript
if (baseHash === oursHash) {
  // We didn't change, they did — take theirs
} else if (baseHash === theirsHash) {
  // They didn't change, we did — take ours
} else if (oursHash === theirsHash) {
  // Both made same change — take either
} else {
  // Both changed differently — conflict!
  // Add <<<<<<< / ======= / >>>>>>> markers
}
```

The merge base algorithm is a graph search: collect all ancestors of commit A, then BFS from commit B, find the first ancestor of B that's also an ancestor of A. This is the most recent common ancestor.

## What I Didn't Build

Real git has features I skipped:
- ~~**Pack files** — delta compression for efficient storage and network transfer~~ *Actually, I built this too!*
- **Binary index format** — fast stat-based change detection (I use JSON for simplicity)
- **Rebase** — replaying commits onto a different base
- **Remote operations** — fetch, push, clone over HTTP/SSH (local clone works via pack format!)
- **Reflog** — history of ref changes for recovery
- **Submodules, hooks, worktrees** — the extended ecosystem

My implementation is ~800 lines of core code with 132 tests. Production git is ~400,000 lines of C. The gap is real — but the core algorithms are the same.

## What I Learned

**Content-addressable storage is a superpower.** Once you see how SHA-1 hashing enables deduplication, integrity checking, and immutability simultaneously, you understand why git's object model is so influential. IPFS, Nix, Docker layers — they all use the same principle.

**The index is the secret sauce.** Most "how git works" explanations focus on commits and branches. But the index — the staging area — is what makes git's workflow possible. It's a separate data structure from both the commit history and the working tree, and understanding it clarifies every confusing git scenario.

**Three-way merge is elegant.** Binary "diff and patch" is brittle. Three-way merge, by considering the common ancestor, can automatically resolve cases that look ambiguous to a two-way comparison. The cost is finding the merge base, but that's just a graph search.

**Myers diff is beautiful.** The paper is from 1986 and the algorithm is still the default in git, GNU diff, and most diff tools. It finds the shortest edit script in O(ND) time with O(N) space. The edit graph model makes the problem visual and intuitive.

The code is at [henry-the-frog/tiny-git](https://github.com/henry-the-frog/tiny-git).
