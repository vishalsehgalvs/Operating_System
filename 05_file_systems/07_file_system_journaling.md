# File System Journaling

> File system journaling keeps a running log (the "journal") of what changes are about to be made before actually making them — if the system crashes mid-operation, the journal lets the OS replay or discard the incomplete work in seconds instead of scanning the entire disk for hours.

---

## Table of Contents

1. [What Is Journaling?](#1-what-is-journaling)
2. [Why Journaling Matters](#2-why-journaling-matters)
3. [How Journaling Works — Three Steps](#3-how-journaling-works--three-steps)
4. [Types of Journaling](#4-types-of-journaling)
5. [Journal Recovery Process](#5-journal-recovery-process)
6. [Benefits of Journaling](#6-benefits-of-journaling)
7. [Limitations](#7-limitations)
8. [Journaling in Common File Systems](#8-journaling-in-common-file-systems)
9. [Journaling vs Backup](#9-journaling-vs-backup)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What Is Journaling?

**Journaling** is a file system technique where all planned changes are written to a special log area (the **journal**) BEFORE being applied to the actual file system data structures.

**Diary / renovation analogy:**

```
  Before journaling:
  You start renovating a room. Power goes out mid-task.
  Nobody knows what you finished and what's half-done — chaos.

  With journaling:
  You write in your diary:
    "Plan: Move bookshelf → repaint wall → install shelf bracket → hang shelf"
  Now power goes out after "repaint wall".
  You check the diary, see "install shelf bracket" was next, and continue from there.

  The journal is the diary. The file operation is the renovation.
```

The journal is a **circular log** stored on the same disk, typically a small fixed-size area reserved for this purpose.

---

## 2. Why Journaling Matters

Without journaling, a crash mid-operation can leave the file system **inconsistent**:

```
  Example: Creating a new file (3 steps without journaling)
  Step 1: Allocate a free inode from the inode table  ← crash here → inode marked used, no directory entry, data blocks allocated but orphaned
  Step 2: Write data to allocated data blocks
  Step 3: Add directory entry linking name → inode

  The result: a file that "exists" in the inode table but has no name in any directory,
              or a directory entry pointing to an unallocated inode.

  Recovery before journaling: RUN fsck (file system check) — scans EVERY inode,
  EVERY block, EVERY directory entry on the entire disk.
  On a 4 TB drive → fsck can take hours. Server downtime = $$$.
```

With journaling, recovery takes **seconds** because only the journal needs to be replayed.

---

## 3. How Journaling Works — Three Steps

```mermaid
flowchart LR
    A[Operation begins\ne.g., create file] --> B[Step 1: Write to Journal\nRecord the planned changes]
    B --> C[Step 2: Write to File System\nApply the actual changes]
    C --> D[Step 3: Mark as Complete\nCommit entry in journal]
    D --> E[Journal entry eventually\noverwritten by new entries]
```

### Step 1: Write to Journal (Log Intent)

Before touching any inode or data block, the OS writes a **transaction** to the journal describing exactly what it intends to do.

```
  Journal transaction for "create report.txt":
  ┌─────────────────────────────────────────────────┐
  │ Transaction #1042  [PENDING]                    │
  │ Operation: CREATE FILE                          │
  │   - Allocate inode #1234                        │
  │   - Set inode fields: owner=alice, perms=644... │
  │   - Allocate data blocks: 501, 502, 503         │
  │   - Add directory entry: "report.txt" → 1234   │
  └─────────────────────────────────────────────────┘
```

This write is confirmed on disk before proceeding.

### Step 2: Write to File System (Apply Changes)

The OS now makes the actual changes: updates inode table, writes data blocks, adds directory entry.

_If a crash happens here, the journal has the complete plan → recovery can finish it._

### Step 3: Mark as Complete (Commit)

Once ALL changes are applied successfully, the journal transaction is marked `COMMITTED`.

```
  Transaction #1042  [COMMITTED] ✓ ← crash-safe now
```

Future journal entries can overwrite this space.

---

## 4. Types of Journaling

Three levels of what gets recorded in the journal:

### Metadata Journaling (Writeback mode)

- **What's logged**: only file system metadata (inodes, directory entries, allocation bitmaps)
- **What's NOT logged**: actual file data content
- **Speed**: fastest (least data written to journal)
- **Risk**: after crash, file system is structurally consistent, BUT a file being written may contain partial/old data

```
  Crash scenario with metadata journaling:
  File "document.txt" was being written with "New paragraph added"

  After recovery:
  - File exists ✓ (inode/directory intact)
  - File size is correct ✓
  - BUT content may have partial write → garbled text ✗

  File system = healthy; file content = possibly corrupt
```

### Data Journaling

- **What's logged**: both metadata AND actual file data
- **Speed**: slowest — every write goes to journal THEN to file system (data written twice)
- **Risk**: minimal — both structure and content protected

### Ordered Journaling (Default in most systems)

- **What's logged**: metadata only (like metadata journaling)
- **Extra rule**: data blocks are written to disk BEFORE the corresponding metadata is journaled
- **Effect**: ensures metadata never points to unwritten/garbage data blocks
- **Speed**: moderate — good balance of performance and protection

```
  Ordered mode guarantee:
  If file write crashes:
  - Worst case: data blocks written but not referenced by metadata
  - These are "orphan blocks" — safe to reclaim; no corruption
  - File system structure = intact; at most, you lose the file that was being written
```

| Mode                      | What's Logged       | Speed        | Protection                                                        |
| ------------------------- | ------------------- | ------------ | ----------------------------------------------------------------- |
| Writeback (metadata only) | Metadata            | Fastest      | FS structure safe; data content may corrupt                       |
| **Ordered** (default)     | Metadata + ordering | **Moderate** | FS structure safe; data loss at most is just the in-progress file |
| Data journaling           | Metadata + data     | Slowest      | Both FS structure and data content fully protected                |

---

## 5. Journal Recovery Process

After a system crash, on the next boot:

```
  1. File system driver reads the journal before mounting the FS
  2. Scans journal entries for their state:
     - PENDING (wrote intent but crashed before applying) → REPLAY
     - IN-PROGRESS (applying, crashed midway) → REPLAY to complete
     - COMMITTED → Done; no action needed
     - INCOMPLETE JOURNAL ENTRY (crash during journal write itself) → DISCARD SAFELY
  3. Replay uncommitted transactions → file system reaches consistent state
  4. Mount file system normally

  Time: seconds to minutes (vs. hours for fsck without journaling)
```

```
  Without journal: "Was block 7234 really supposed to be part of file X?"
  → Must check every block on the entire 4 TB disk to know for sure.

  With journal: "Transaction #1042 was in-progress. Apply these specific 4 changes."
  → Done in milliseconds.
```

---

## 6. Benefits of Journaling

| Benefit                 | Description                                                         |
| ----------------------- | ------------------------------------------------------------------- |
| **Fast recovery**       | Boot in seconds after crash — only journal needs replaying          |
| **Data consistency**    | File system structure stays intact regardless of when crash happens |
| **Atomicity**           | Operations either complete fully or are safely rolled back          |
| **Reduced maintenance** | No need to run hours-long fsck on every unexpected shutdown         |
| **Server uptime**       | Critical for servers where downtime = money                         |

```
  Real difference on a 4 TB server disk:

  Pre-journaling world (ext2):
  Power failure at 2 AM → fsck runs for 4+ hours at next boot → server unavailable

  Post-journaling world (ext3/4, NTFS, APFS):
  Power failure at 2 AM → journal replay takes 30 seconds at next boot → server online
```

---

## 7. Limitations

### Performance Overhead

```
  Every file operation:
  Metadata journaling: 1 extra write (journal) + 1 main write = ~1.1× overhead
  Data journaling:     1 journal write + 1 main write = ~2× overhead (write amplification)

  On SSDs with NVMe: overhead is tiny (I/O is fast)
  On HDDs: overhead more noticeable, but still worthwhile
```

### Journal Size Limits

```
  Journal is a fixed-size circular buffer (typically 128 MB on large volumes).
  If uncommitted entries fill the journal faster than they're committed:
  → File system pauses briefly (journal checkpoint) → commits pending entries → resumes

  Automatic — users don't see this, but it can cause occasional write stalls.
```

### Not a Backup

```
  Journaling protects against:   ✓ Power failures during writes
                                  ✓ OS crashes mid-operation
                                  ✓ Unexpected shutdowns

  Journaling does NOT protect against:
  ✗ Hardware failure (disk physically dies)
  ✗ Accidental deletion (the delete transaction completes perfectly)
  ✗ Ransomware (file encryption transaction completes perfectly)
  ✗ Fire/flood destroying the drive

  You still need backups for those scenarios!
```

---

## 8. Journaling in Common File Systems

| File System | Journaling Mode                               | Notes                                           |
| ----------- | --------------------------------------------- | ----------------------------------------------- |
| **ext2**    | None                                          | Old Linux FS — requires full fsck after crash   |
| **ext3**    | Ordered (default), also metadata / data modes | Added journaling to ext2                        |
| **ext4**    | Ordered (default), also metadata / data modes | Current Linux standard                          |
| **NTFS**    | Metadata journaling (`$LogFile`)              | Windows standard                                |
| **XFS**     | Metadata journaling                           | High-performance; used in RHEL/enterprise Linux |
| **APFS**    | Copy-on-write (crash-safe by design)          | macOS — CoW is a journaling alternative         |
| **Btrfs**   | Copy-on-write + explicit journal              | Advanced Linux FS with snapshots                |

**Copy-on-write (CoW) as an alternative to journaling (APFS, Btrfs):**

```
  Instead of: write to journal → overwrite in-place → commit
  CoW does:   write new data to NEW blocks → atomically update pointer → old blocks freed

  If crash between "write new blocks" and "update pointer": old data is still intact.
  Crash is always safe — pointer update is a single atomic operation.
```

---

## 9. Journaling vs Backup

```
  ┌──────────────────────────────────────────────────────┐
  │                    Comparison                        │
  ├────────────────────┬─────────────────────────────────┤
  │ Journaling         │ Backup                          │
  ├────────────────────┼─────────────────────────────────┤
  │ Short-term safety  │ Long-term safety                │
  │ Protects crashes   │ Protects hardware failure       │
  │ Automatic          │ Requires setup/scheduling       │
  │ On same disk       │ Different disk/location         │
  │ Seconds to recover │ Hours to restore                │
  │ Not for deletions  │ Can restore deleted files       │
  └────────────────────┴─────────────────────────────────┘

  Verdict: You need BOTH.
  Journaling = seatbelt (protects you during the crash)
  Backup = spare car (gets you running again after total loss)
```

---

## 10. Key Takeaways

- **Journaling** = write a log of planned changes BEFORE applying them; if crash happens, replay the log to reach consistent state
- **Without journaling**: crash → full disk scan (fsck) → hours of downtime
- **With journaling**: crash → journal replay → seconds to consistent state
- **Three-step process**: (1) write transaction to journal → (2) apply changes to FS → (3) mark transaction committed
- **Three journaling levels**: metadata-only (fastest, file data may corrupt), ordered (default — good balance), data journaling (safest, slowest — writes data twice)
- **Ordered journaling** (ext3/4 default): forces data to disk before journaling metadata → guarantees FS structure integrity; file content loss at worst
- **Recovery**: on next boot, OS replays or discards uncommitted journal transactions → FS is consistent in seconds
- **Journal = circular buffer** — transactions are overwritten once committed
- **Copy-on-write (CoW)** in APFS/Btrfs achieves the same crash-safety guarantees as journaling through a different mechanism
- **Journaling ≠ backup**: protects against crashes and power failure, NOT against hardware failure, accidental deletion, or ransomware
- Modern file systems that don't journal (ext2) are essentially obsolete for general-purpose use; all modern general-purpose file systems journal by default
