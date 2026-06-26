# File Allocation Methods: Contiguous, Linked, and Indexed

> File allocation methods decide how an OS assigns disk blocks to files — contiguous packs blocks together for speed, linked chains blocks with pointers for flexibility, and indexed uses a lookup table for both random access and fragmentation-free storage; modern systems like ext4 and NTFS use advanced indexed variants.

---

## Table of Contents

1. [What Is File Allocation?](#1-what-is-file-allocation)
2. [Contiguous Allocation](#2-contiguous-allocation)
3. [Linked Allocation](#3-linked-allocation)
4. [Indexed Allocation](#4-indexed-allocation)
5. [Method Comparison](#5-method-comparison)
6. [Understanding Fragmentation](#6-understanding-fragmentation)
7. [Modern File System Approaches](#7-modern-file-system-approaches)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What Is File Allocation?

When you save a file, the OS must decide **which disk blocks to assign to it** and **how to track them**. This is file allocation.

**Storage box analogy:**

```
  Imagine a long row of numbered storage boxes (disk blocks).
  A file needs some boxes — but which ones? How does the OS remember?
  The three allocation methods are three different strategies for answering this.
```

Three primary strategies:

1. **Contiguous** — reserve a run of consecutive blocks
2. **Linked** — use any free blocks; each block points to the next
3. **Indexed** — use any free blocks; a dedicated index block lists them all

---

## 2. Contiguous Allocation

**Each file occupies a set of consecutive (adjacent) disk blocks.**

```
  Directory entry stores: start block + length

  File_A: start=0, length=3
  File_B: start=3, length=5
  File_C: start=8, length=4

  [0][1][2] ─── File_A
  [3][4][5][6][7] ─── File_B
  [8][9][10][11] ─── File_C
  [12][13]... ─── Free

  To read block 2 of File_B: jump to block 3+2 = block 5. Done.
```

**Movie theater analogy:** Reserve 5 consecutive seats for your group — everyone sits together, easy to coordinate.

### Advantages

- **Sequential access**: blazing fast — disk head moves in one direction
- **Direct (random) access**: address = start + offset — computed instantly, no traversal
- **Minimal metadata**: only start block and length needed in directory entry

### Disadvantages

- **External fragmentation**: as files are created/deleted, free space becomes scattered gaps — cannot fit a new file even if total free space is enough
- **File growth problem**: can't extend a file if adjacent blocks are taken
- **Must know size in advance** or over-allocate (wastes space)
- **Compaction** (defrag) is the only fix — expensive and time-consuming

**Where it's still used:** CD-ROMs, DVDs — files written once, never grow.

---

## 3. Linked Allocation

**Each disk block contains data AND a pointer to the next block.** Directory stores only the first block address.

```
  Directory entry stores: first block only

  File_D: start=2

  [2]: Data_D1 │ → 5     (block 2 data; next = block 5)
  [5]: Data_D2 │ → 9     (block 5 data; next = block 9)
  [9]: Data_D3 │ → -1    (block 9 data; EOF — no next)

  To read File_D: follow chain 2 → 5 → 9.
```

**Treasure hunt analogy:** Each clue (block) leads to the next location. You must follow the whole chain to find the end.

### Advantages

- **No external fragmentation**: any free block can be used — scattered space is fine
- **Files can grow easily**: just attach a new block anywhere to the end of the chain
- No need to know file size in advance

### Disadvantages

- **Sequential-only access**: to read block N, must traverse N-1 pointers (each possibly a disk seek)
- **Pointer overhead**: each block uses some bytes for a pointer instead of data
- **Reliability risk**: one corrupted pointer breaks the entire rest of the file
- **No direct/random access** efficiency

### FAT Variation

**FAT (File Allocation Table)** improves on linked allocation by moving ALL pointers into a single in-memory table:

```
  FAT table (in memory):
  Index: 2  3  4  5  6  7  8  9  10
  Value: 5  ─  ─  9  ─  ─  ─  EOF ─

  File_D: FAT[2] = 5, FAT[5] = 9, FAT[9] = EOF

  Benefits:
  - No pointer overhead within data blocks
  - Faster traversal (table is in RAM)
  - Still no external fragmentation
```

---

## 4. Indexed Allocation

**A dedicated index block holds an array of pointers to all the file's data blocks.** Directory entry points to the index block.

```
  Directory entry: File_F → index block at 3

  Index block [3]:
  [0] → 7    (data block 0 of file)
  [1] → 2    (data block 1)
  [2] → 11   (data block 2)
  [3] → 5    (data block 3)
  [4] → -1   (no more blocks)

  Disk blocks:  [2]: Data_F2  [5]: Data_F4  [7]: Data_F1  [11]: Data_F3

  To read block 2: look at index[2] = 11 → go straight to block 11.
```

**Book table of contents analogy:** The index page tells you exactly which disk "page" holds each chapter — no need to read the whole book to find chapter 7.

### Advantages

- **Direct/random access**: look up index[N] → jump directly to any block
- **No external fragmentation**: blocks can be anywhere on disk
- **Supports dynamic file growth**: just add entries to the index block

### Disadvantages

- **Index block overhead**: even a 1-block file needs a full index block
- **2 disk reads minimum**: one for index block + one for data block
- **Max file size**: limited by how many pointers fit in one index block

### Multi-Level Indexing

For large files, index blocks point to other index blocks:

```
  Single indirect:   inode → [index block] → [data blocks]       (4 MB range)
  Double indirect:   inode → [index of indexes] → [indexes] → [data]  (4 GB)
  Triple indirect:   3 levels deep                              (4 TB+)

  This is exactly how Unix/Linux inodes work!
```

---

## 5. Method Comparison

| Aspect                 | Contiguous               | Linked                         | Indexed                   |
| ---------------------- | ------------------------ | ------------------------------ | ------------------------- |
| Sequential access      | ⭐⭐⭐ Excellent         | ⭐⭐ Good                      | ⭐⭐ Good                 |
| Random (direct) access | ⭐⭐⭐ Excellent         | ⭐ Poor (chain walk)           | ⭐⭐⭐ Excellent          |
| External fragmentation | ⚠️ Yes — major issue     | ✅ None                        | ✅ None                   |
| Internal fragmentation | ⚠️ Possible (overalloc)  | ⚠️ Pointer overhead            | ⚠️ Index block waste      |
| File growth            | ❌ Difficult (relocate)  | ✅ Easy (append block)         | ✅ Easy (add index entry) |
| Reliability            | ✅ Good                  | ❌ One bad pointer = lost tail | ✅ Good                   |
| Metadata overhead      | Minimal (start + len)    | Pointer per block              | Index block(s)            |
| Best use case          | Read-only media (CD/DVD) | Sequential streaming           | **General-purpose**       |

---

## 6. Understanding Fragmentation

### External Fragmentation (Contiguous allocation problem)

```
  Disk after some creates and deletes:
  [File_A][File_A][Free][Free][Free][File_C][File_C]

  Want to allocate File_D needing 4 blocks:
  Free blocks: positions 2, 3, 4 = only 3 consecutive → FAIL

  Even if we had 4 free blocks scattered: positions 2, 3, 4, 6
  No 4 consecutive → still FAIL for contiguous allocation.
  Fix: compaction (move File_C to close the gap) — expensive!
```

### Internal Fragmentation (All fixed-block systems)

```
  Block size: 4 KB
  File size: 10 KB

  Blocks allocated: ceil(10 / 4) = 3 blocks = 12 KB allocated
  Wasted space: 12 - 10 = 2 KB per file

  With 10,000 files averaging 2 KB waste = 20 MB wasted in total.

  Smaller blocks → less internal fragmentation BUT more metadata blocks to manage.
```

| Fragmentation Type | Where it happens         | Cause                                   | Fix                                          |
| ------------------ | ------------------------ | --------------------------------------- | -------------------------------------------- |
| External           | Between allocated blocks | Free space scattered by create/delete   | Compaction, or use non-contiguous allocation |
| Internal           | Inside allocated blocks  | Block granularity larger than file size | Smaller block size (trade-off)               |

---

## 7. Modern File System Approaches

**ext4 (Linux):** Uses **extents** — a contiguous allocation descriptor stored in an indexed structure. An extent = (start block, length). Small files use direct extents; large files use a B-tree of extents. This gives contiguous-like performance without the all-or-nothing fragility.

**NTFS (Windows):** Uses **MFT runs** — similar to extents. Each run describes a contiguous range. The MFT record (like an indexed structure) lists all runs. Small files store their data directly inside the MFT record itself.

**APFS (macOS):** Copy-on-write + extent-based allocation with cloneability.

```
  Modern systems never use pure linked allocation.
  Most use indexed allocation with extent optimization:
  → Fast random access (indexed)
  → Reduced fragmentation (extents = contiguous runs)
  → Flexible growth (more extents as needed)
```

**Defragmentation:** Useful for HDDs (moving parts = seek time matters). **Do NOT defrag SSDs** — no moving parts, random access is as fast as sequential, and unnecessary writes reduce SSD lifespan.

---

## 8. Key Takeaways

- **Contiguous allocation** = file occupies consecutive blocks; excellent sequential/random access but suffers from external fragmentation and can't grow easily — used for CD/DVD
- **Linked allocation** = each block has a pointer to next; no fragmentation, easy growth, but terrible random access and vulnerable to pointer corruption — used as FAT file system
- **FAT (File Allocation Table)** improves linked allocation by centralizing all pointers in an in-memory table
- **Indexed allocation** = a dedicated index block maps file offset → disk block; best balance of random access, no fragmentation, and growth support — used by most modern file systems
- **Multi-level indexing** (single/double/triple indirect) handles very large files from a compact inode structure
- **External fragmentation** = free space scattered in gaps between files → can't fit new files even with enough total free space (contiguous problem)
- **Internal fragmentation** = last block of every file has unused padding (fixed-block-size problem)
- **Extents** = modern optimization: one indexed entry describes a whole contiguous run of blocks → fewer index entries, better sequential performance
- **Never defrag SSDs** — no performance gain, reduces drive lifespan
