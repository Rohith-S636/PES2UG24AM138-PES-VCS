# Building PES-VCS — A Version Control System from Scratch

**Objective:** Build a local version control system that tracks file changes, stores snapshots efficiently, and supports commit history. Every component maps directly to operating system and filesystem concepts.

**Platform:** Ubuntu 22.04

---



## Phase 1: Object Storage Foundation

**Filesystem Concepts:** Content-addressable storage, directory sharding, atomic writes, hashing for integrity

**Files:** `pes.h` (read), `object.c` (implement `object_write` and `object_read`)

### What to Implement

Open `object.c`. Two functions are marked `// TODO`:

1. **`object_write`** — Stores data in the object store.
   - Prepends a type header (`"blob <size>\0"`, `"tree <size>\0"`, or `"commit <size>\0"`)
   - Computes SHA-256 of the full object (header + data)
   - Writes atomically using the temp-file-then-rename pattern
   - Shards into subdirectories by first 2 hex chars of hash

2. **`object_read`** — Retrieves and verifies data from the object store.
   - Reads the file, parses the header to extract type and size
   - **Verifies integrity** by recomputing the hash and comparing to the filename
   - Returns the data portion (after the `\0`)

Read the detailed step-by-step comments in `object.c` before starting.

### Testing

```bash
make test_objects
./test_objects
```

The test program verifies:
- Blob storage and retrieval (write, read back, compare)
- Deduplication (same content → same hash → stored once)
- Integrity checking (detects corrupted objects)

**📸 Screenshot 1A:** Output of `./test_objects` showing all tests passing.
<br>![Screenshot 1A: Output of test_objects](./screenshot_1A.png)

**📸 Screenshot 1B:** `find .pes/objects -type f` showing the sharded directory structure.
<br>![Screenshot 1B: Object Store Sharding](./screenshot_1B.png)

---

## Phase 2: Tree Objects

**Filesystem Concepts:** Directory representation, recursive structures, file modes and permissions

**Files:** `tree.h` (read), `tree.c` (implement all TODO functions)

### What to Implement

Open `tree.c`. Implement the function marked `// TODO`:

1. **`tree_from_index`** — Builds a tree hierarchy from the index.
   - Handles nested paths: `"src/main.c"` must create a `src` subtree
   - This is what `pes commit` uses to create the snapshot
   - Writes all tree objects to the object store and returns the root hash

### Testing

```bash
make test_tree
./test_tree
```

The test program verifies:
- Serialize → parse roundtrip preserves entries, modes, and hashes
- Deterministic serialization (same entries in any order → identical output)

**📸 Screenshot 2A:** Output of `./test_tree` showing all tests passing.
<br>![Screenshot 2A: Output of test_tree](./screenshot_2A.png)

**📸 Screenshot 2B:** Pick a tree object from `find .pes/objects -type f` and run `xxd .pes/objects/XX/YYY... | head -20` to show the raw binary format.
<br>![Screenshot 2B: XXD output of Tree Object](./screenshot_2B.png)

---

## Phase 3: The Index (Staging Area)

**Filesystem Concepts:** File format design, atomic writes, change detection using metadata

**Files:** `index.h` (read), `index.c` (implement all TODO functions)

### What to Implement

Open `index.c`. Three functions are marked `// TODO`:

1. **`index_load`** — Reads the text-based `.pes/index` file into an `Index` struct.
   - If the file doesn't exist, initializes an empty index (this is not an error)
   - Parses each line: `<mode> <hash-hex> <mtime> <size> <path>`

2. **`index_save`** — Writes the index atomically (temp file + rename).
   - Sorts entries by path before writing
   - Uses `fsync()` on the temp file before renaming

3. **`index_add`** — Stages a file: reads it, writes blob to object store, updates index entry.
   - Use the provided `index_find` to check for an existing entry

`index_find` , `index_status` and `index_remove` are already implemented for you — read them to understand the index data structure before starting.

#### Expected Output of `pes status`

```
Staged changes:
  staged:     hello.txt
  staged:     src/main.c

Unstaged changes:
  modified:   README.md
  deleted:    old_file.txt

Untracked files:
  untracked:  notes.txt
```

If a section has no entries, print the header followed by `(nothing to show)`.

### Testing

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index    # Human-readable text format
```

**📸 Screenshot 3A:** Run `./pes init`, `./pes add file1.txt file2.txt`, `./pes status` — show the output.
<br>![Screenshot 3A: PES Init/Add/Status Sequences](./screenshot_3A.png)

**📸 Screenshot 3B:** `cat .pes/index` showing the text-format index with your entries.
<br>![Screenshot 3B: Cat internal text Index format](./screenshot_3B.png)

---

## Phase 4: Commits and History

**Filesystem Concepts:** Linked structures on disk, reference files, atomic pointer updates

**Files:** `commit.h` (read), `commit.c` (implement all TODO functions)

### What to Implement

Open `commit.c`. One function is marked `// TODO`:

1. **`commit_create`** — The main commit function:
   - Builds a tree from the index using `tree_from_index()` (**not** from the working directory — commits snapshot the staged state)
   - Reads current HEAD as the parent (may not exist for first commit)
   - Gets the author string from `pes_author()` (defined in `pes.h`)
   - Writes the commit object, then updates HEAD

`commit_parse`, `commit_serialize`, `commit_walk`, `head_read`, and `head_update` are already implemented — read them to understand the commit format before writing `commit_create`.

The commit text format is specified in the comment at the top of `commit.c`.

### Testing

```bash
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

You can also run the full integration test:

```bash
make test-integration
```

**📸 Screenshot 4A:** Output of `./pes log` showing three commits with hashes, authors, timestamps, and messages.
<br>![Screenshot 4A: Commit history outputs](./screenshot_4A.png)

**📸 Screenshot 4B:** `find .pes -type f | sort` showing object store growth after three commits.
<br>![Screenshot 4B: Final Objects Directory Find/Sort Sequences](./screenshot_4B.png)

**📸 Screenshot 4C:** `cat .pes/refs/heads/main` and `cat .pes/HEAD` showing the reference chain.
<br>![Screenshot 4C: Physical HEAD linking chains](./screenshot_4C.png)

**📸 Final Integration Test:** Output of full end-to-end testing sequence.
<br>![Screenshot Final: Test Integration Run](./screenshot_final.png)

---

## Phase 5 & 6: Analysis-Only Questions

The following questions cover filesystem concepts beyond the implementation scope of this lab. Answer them in writing — no code required.

### Branching and Checkout

**Q5.1:** A branch in Git is just a file in `.git/refs/heads/` containing a commit hash. Creating a branch is creating a file. Given this, how would you implement `pes checkout <branch>` — what files need to change in `.pes/`, and what must happen to the working directory? What makes this operation complex?

**Answer:**
To implement `pes checkout <branch>`:
1. **`.pes/` Internal Changes:** We must update `.pes/HEAD` to contain `"ref: refs/heads/<branch>"`. Additionally, we must load the target branch's commit hash, traverse its tree, and rebuild `.pes/index` linearly to match the target snapshot exactly.
2. **Working Directory Changes:** The working directory files must be overwritten, deleted, or safely replaced so that the physical files on disk exactly match the tree bound to the target branch. 
3. **Complexity:** The operation is complex because it requires safely modifying user files without permanently deleting untracked files or destroying unsaved/uncommitted modifications. It also requires complex recursive recursive tree-walking to expand objects into complete directory structures natively.

**Q5.2:** When switching branches, the working directory must be updated to match the target branch's tree. If the user has uncommitted changes to a tracked file, and that file differs between branches, checkout must refuse. Describe how you would detect this "dirty working directory" conflict using only the index and the object store.

**Answer:**
To detect a dirty working directory conflict natively:
1. Traverse the current `Index` to fetch all tracked files.
2. Use POSIX `stat()` on each working directory file to check its `mtime` and `size` against the cached index values.
3. If an index mismatch occurs (indicating unsaved physical modifications), read the physical file and compute its SHA-256 blob hash using `compute_hash()`.
4. Walk the *target branch's* `tree` within the object store concurrently to locate that file's hash.
5. If the uncommitted working directory hash contradicts the target branch's expected hash, it triggers a conflict state, and `checkout` safely refuses execution to prevent data loss.

**Q5.3:** "Detached HEAD" means HEAD contains a commit hash directly instead of a branch reference. What happens if you make commits in this state? How could a user recover those commits?

**Answer:**
If you execute a commit in a "Detached HEAD" state, the commit object is properly saved to `.pes/objects`, and `.pes/HEAD` is physically updated to point to the new hash. However, **no branch file** inside `.pes/refs/heads/` moves. 
If the user subsequently switches to another branch from here, the detached commit becomes unreferenced/unreachable. To recover it, the user could simply create a new branch pointer file in `refs/heads/` mapping directly to that orphaned hash (e.g., executing `git checkout -b <new-branch> <hash>`), or they could search the internal system logs (`git reflog`) for recently executed operations to scavenge the forgotten commit ID.

### Garbage Collection and Space Reclamation

**Q6.1:** Over time, the object store accumulates unreachable objects — blobs, trees, or commits that no branch points to (directly or transitively). Describe an algorithm to find and delete these objects. What data structure would you use to track "reachable" hashes efficiently? For a repository with 100,000 commits and 50 branches, estimate how many objects you'd need to visit.

**Answer:**
You would use a **Mark-and-Sweep** directed graph algorithm:
1. **Mark Phase:** Map all branch pointers in `.pes/refs/heads/` and the `.pes/HEAD` state as root nodes. Perform a recursive graph traversal (BFS/DFS) walking backwards through commit parents, extracting and parsing tree pointers, and marking all discovered child sub-trees and blobs.
2. **Sweep Phase:** Iterate sequentially through all physical files present in `.pes/objects/`. If a discovered chunk was never "marked", it is orphaned and automatically deleted natively to reclaim space.
**Data Structure:** A **Hash Set** efficiently guarantees O(1) deduplication memory tracking for each 32-byte binary hash mapping.
**Estimate:** For a repository with 100,000 commits containing around 10 modified blob items natively per commit over 50 dynamic branch chains, the BFS algorithm would natively touch/visit ** millions** of tracked objects securely throughout its index processing overhead (e.g., ~1,000,000 to ~2,000,000 raw tree/blob bounds total).

**Q6.2:** Why is it dangerous to run garbage collection concurrently with a commit operation? Describe a race condition where GC could delete an object that a concurrent commit is about to reference. How does Git's real GC avoid this?

**Answer:**
Running GC recursively during live `commit` environments executes an unchecked structural data race condition:
**The Race:** A user runs `pes commit`. The execution process creates and flushes the new `blob` securely to `.pes/objects/XX/`. Concurrently, the GC cycle wakes up natively! Because the newly minted blob has not yet actively been bound to an established `.pes/refs/heads` branch parent commit link locally, the secondary GC thread inherently views it as deeply orphaned, sweeping and deleting the fresh blob completely offline. The `commit` completes natively, tying a hard pointer locally directly to a permanently deleted element, inducing data corruption!
**Git's Solution:** Real Git inherently avoids this by implementing strict timestamp bounding! The underlying garbage collector implicitly strictly ignores any structurally orphaned object newly written dynamically within a set tolerance window structurally (usually bounds less than 2 physical weeks). Natively it safely skips active pipeline data.


