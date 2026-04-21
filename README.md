# PES-VCS Lab Report
**Name:** Parth Kale
**SRN:** PES1UG24AM184

---

## Phase 5: Branching and Checkout

### Q5.1: How would you implement `pes checkout <branch>`?

To implement `pes checkout <branch>`, the following steps are needed:

1. **Read the target branch file** from `.pes/refs/heads/<branch>` to get the commit hash.
2. **Read the commit object** to get its tree hash.
3. **Walk the tree recursively** and restore every file in the working directory to match the tree's blobs.
4. **Update `.pes/HEAD`** to contain `ref: refs/heads/<branch>`.

What makes this complex:
- You must **delete files** that exist in the current branch but not in the target branch.
- You must **create directories** that don't exist yet.
- If the user has **uncommitted changes**, you risk overwriting their work.
- You need to update the index to match the new branch's tree so `status` works correctly.

---

### Q5.2: How would you detect a "dirty working directory" conflict?

To detect conflicts before switching branches:

1. For each file in the **current index**, compute the SHA-256 of the file on disk.
2. Compare it to the hash stored in the index entry.
3. If they differ, the file has **uncommitted local changes**.
4. Check if that same file **differs between the two branches' trees**.
5. If both are true — the file is locally modified AND differs between branches — refuse checkout and report the conflict.

This only requires the index and object store — no extra metadata needed.

---

### Q5.3: What happens in "detached HEAD" state?

When HEAD contains a commit hash directly instead of a branch reference (e.g., `ref: refs/heads/main`), you are in **detached HEAD** state.

If you make commits in this state:
- New commits are created and chained correctly with parent pointers.
- HEAD is updated to point to the new commit hash directly.
- However, **no branch pointer is updated**, so these commits are not reachable from any branch.

To recover these commits, a user can:
1. Note the commit hash shown after each `commit` command.
2. Create a new branch pointing to that hash: `git branch recovery <hash>`.
3. Or use `git reflog` (in real Git) to find recent commit hashes.

Without doing this, the commits become **unreachable** and will eventually be deleted by garbage collection.

---

## Phase 6: Garbage Collection

### Q6.1: Algorithm to find and delete unreachable objects

**Algorithm (Mark and Sweep):**

1. **Mark phase:** Start from all branch refs in `.pes/refs/heads/`. For each branch, walk the commit chain. For each commit, record its tree hash. For each tree, recursively record all blob and subtree hashes. Use a **hash set** (e.g., a hash table keyed by the 64-char hex hash) to track all reachable object IDs.

2. **Sweep phase:** Walk all files under `.pes/objects/`. For each file, reconstruct its hash from the path. If the hash is **not in the reachable set**, delete the file.

**Data structure:** A hash set of 64-character hex strings is ideal — O(1) average lookup and insertion.

**Estimate for 100,000 commits, 50 branches:**
- Each commit references 1 tree; each tree references ~10 blobs on average.
- Total objects visited ≈ 100,000 commits + 100,000 trees + ~500,000 blobs = **~700,000 objects**.
- Starting from 50 branches, the walk converges quickly since commits share history.

---

### Q6.2: Race condition between GC and concurrent commit

**The race condition:**

1. A commit operation runs `tree_from_index()` and creates new blob objects in the store.
2. Before the commit object is written (and before HEAD is updated), GC runs.
3. GC scans all refs — the new blobs are not yet reachable from any ref.
4. GC deletes the new blobs as "unreachable."
5. The commit operation then writes the commit object referencing the now-deleted blobs.
6. The repository is now **corrupt** — the commit points to missing objects.

**How Git avoids this:**
- Git uses a **grace period** — objects newer than 14 days are never deleted by GC, giving in-progress operations time to complete.
- Git also writes a `FETCH_HEAD` or lock file during operations to signal that a transaction is in progress.
- The object is written to disk **before** any ref points to it, so a partial commit leaves orphan objects (harmless) rather than missing ones.
- GC uses file modification timestamps to skip recently created objects.

---
## Screenshots

### Phase 1
- **1A:** `./test_objects` output showing all tests passing
![1a](./screenshots/1a.png)
- **1B:** `find .pes/objects -type f` showing sharded directory structure
![1a](./screenshots/1b.png)

### Phase 2
- **2A:** `./test_tree` output showing all tests passing
![1a](./screenshots/2a.png)
- **2B:** `xxd` of a raw tree object (first 20 lines)
![1a](./screenshots/2b.png)
### Phase 3
- **3A:** `pes init` → `pes add` → `pes status` sequence
![1a](./screenshots/2a.png)
- **3B:** `cat .pes/index` showing the text-format index
![1a](./screenshots/2b.png)
### Phase 4
- **4A:** `pes log` output with three commits
![1a](./screenshots/2a.png)
- **4B:** `find .pes -type f | sort` showing object store 
growth
![1a](./screenshots/2b.png)
- **4C:** `cat .pes/refs/heads/main` and `cat .pes/HEAD`
![1a](./screenshots/5a.png)
- **Final:** Full `make test-integration` output
![1a](./screenshots/6a.png)