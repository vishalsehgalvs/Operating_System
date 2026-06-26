# RAID Levels Explained: Beginner Guide to Disk Arrays

> RAID (Redundant Array of Independent Disks) combines multiple physical disks into one logical unit to gain either better performance, fault tolerance, or both — but RAID is **not** a substitute for backups.

---

## Table of Contents

1. [What is RAID?](#1-what-is-raid)
2. [Core RAID Concepts: Striping, Mirroring, Parity](#2-core-raid-concepts-striping-mirroring-parity)
3. [RAID 0 — Striping (Speed Only)](#3-raid-0--striping-speed-only)
4. [RAID 1 — Mirroring (Safety Only)](#4-raid-1--mirroring-safety-only)
5. [RAID 5 — Striping with Distributed Parity](#5-raid-5--striping-with-distributed-parity)
6. [RAID 6 — Double Parity](#6-raid-6--double-parity)
7. [RAID 10 — Mirror + Stripe](#7-raid-10--mirror--stripe)
8. [Full Comparison Table](#8-full-comparison-table)
9. [Hardware vs Software RAID](#9-hardware-vs-software-raid)
10. [RAID Rebuild Process](#10-raid-rebuild-process)
11. [Common Misconceptions](#11-common-misconceptions)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What is RAID?

RAID combines multiple physical hard disks so the OS sees them as a single logical drive. Depending on the level chosen, RAID can:

- **Protect data** against disk failures (fault tolerance)
- **Increase read/write speed** (performance)
- **Do both** (with different trade-offs)

```
  Physical view:         [Disk 1] [Disk 2] [Disk 3] [Disk 4]
                              ↓       ↓       ↓       ↓
  OS view:                    [  Logical Drive — 1 big disk  ]
```

**Important:** RAID protects against **hardware failure** only. It does NOT protect against:

- Accidental file deletion
- Ransomware/malware corrupting files
- Natural disasters
- User errors

Always combine RAID with backups.

---

## 2. Core RAID Concepts: Striping, Mirroring, Parity

| Concept       | What it means                                                  | Benefit                                           |
| ------------- | -------------------------------------------------------------- | ------------------------------------------------- |
| **Striping**  | Data split into blocks and distributed across multiple disks   | Higher read/write speed (parallel I/O)            |
| **Mirroring** | Exact copy of data written to 2+ disks simultaneously          | Fault tolerance — if one disk dies, copy survives |
| **Parity**    | Math checksum stored alongside data to reconstruct a lost disk | Fault tolerance using less space than mirroring   |

**Parity explained simply:**

```
  Data blocks: A = 1010, B = 1100
  Parity (XOR): P = 0110

  If disk storing A fails:
  A = B XOR P = 1100 XOR 0110 = 1010  ← reconstructed!

  Parity lets you recover 1 lost disk without storing a full copy.
```

---

## 3. RAID 0 — Striping (Speed Only)

**Method:** Data is split into equal-size chunks and written to all disks simultaneously.

```
  File: [Block 1] [Block 2] [Block 3] [Block 4]
              ↓         ↓         ↓         ↓
  Disk 1: [Block 1]  [Block 3]
  Disk 2: [Block 2]  [Block 4]

  Both disks write at the SAME TIME → twice the speed of one disk.
```

**Real-life analogy:** Two cashiers at checkout — customers split between them, twice as fast as one cashier. But if either cashier disappears, no one can complete any transaction.

```
  With two 1 TB disks in RAID 0:
  ┌──────────────────┐
  │ Usable: 2 TB     │  ← full capacity
  │ Speed:  2×       │  ← doubled
  │ Safety: NONE     │  ← if 1 disk dies, ALL data lost
  └──────────────────┘
```

| Property          | RAID 0                                         |
| ----------------- | ---------------------------------------------- |
| Minimum disks     | 2                                              |
| Usable capacity   | 100% (sum of all disks)                        |
| Fault tolerance   | None                                           |
| Read performance  | Very high                                      |
| Write performance | Very high                                      |
| Use case          | Video editing scratch disks, gaming, temp data |

**Warning:** Never use RAID 0 for important data. One disk failure = total data loss.

---

## 4. RAID 1 — Mirroring (Safety Only)

**Method:** Every write goes to ALL disks simultaneously. Every disk has an identical copy.

```
  Write "Hello.docx" to RAID 1 (two disks):

  Disk 1: [Hello.docx — full copy]
  Disk 2: [Hello.docx — full copy]   ← identical mirror

  Disk 1 dies? → Disk 2 has full copy. System keeps running!
```

**Real-life analogy:** You write a letter and instantly make a photocopy. Store the copy in a different drawer. If the original burns, the copy is intact.

```
  With two 1 TB disks in RAID 1:
  ┌──────────────────┐
  │ Usable: 1 TB     │  ← half wasted on mirror
  │ Speed:  reads ≈ 2×, writes ≈ 1× │
  │ Safety: 1 disk failure survives  │
  └──────────────────┘
```

| Property          | RAID 1                                     |
| ----------------- | ------------------------------------------ |
| Minimum disks     | 2                                          |
| Usable capacity   | 50%                                        |
| Fault tolerance   | 1 disk failure                             |
| Read performance  | Slightly better (read from both)           |
| Write performance | Same as single disk (must write twice)     |
| Use case          | Boot drives, critical databases, small NAS |

---

## 5. RAID 5 — Striping with Distributed Parity

**Method:** Data AND parity are striped across ALL disks. No single disk holds only parity — parity rotates across all disks each stripe.

```
  Three-disk RAID 5 example:

  Stripe 1:  Disk1: [A1]      Disk2: [A2]      Disk3: [Parity(A1,A2)]
  Stripe 2:  Disk1: [B1]      Disk2: [Parity(B1,B2)] Disk3: [B2]
  Stripe 3:  Disk1: [Parity(C1,C2)] Disk2: [C1] Disk3: [C2]

  Parity ROTATES → no single "parity disk" bottleneck
```

**If Disk 2 dies:**

```
  Lost: A2, Parity(B), C1

  Reconstruct A2: Disk1[A1] XOR Disk3[Parity(A)] → A2
  Reconstruct C1: Disk1[Parity(C)] XOR Disk3[C2] → C1

  System keeps running! Replace dead disk → array rebuilds automatically.
```

```
  With three 1 TB disks in RAID 5:
  ┌──────────────────────────────┐
  │ Usable: 2 TB (n-1 disks)    │
  │ Speed:  good reads, moderate writes  │
  │ Safety: 1 disk failure       │
  └──────────────────────────────┘
```

| Property          | RAID 5                                     |
| ----------------- | ------------------------------------------ |
| Minimum disks     | 3                                          |
| Usable capacity   | (n-1) disks worth                          |
| Fault tolerance   | 1 disk failure                             |
| Read performance  | Good (parallel striping)                   |
| Write performance | Moderate (parity calculation overhead)     |
| Use case          | File servers, NAS, general-purpose storage |

---

## 6. RAID 6 — Double Parity

**Method:** Like RAID 5, but with TWO independent parity blocks per stripe. Can survive TWO simultaneous disk failures.

```
  Four-disk RAID 6 layout (simplified):

  Disk 1: [A]       [P1]
  Disk 2: [B]       [P2]
  Disk 3: [P1]      [C]
  Disk 4: [P2]      [D]

  P1 = first parity, P2 = second parity (different XOR calculation)

  If Disk 1 AND Disk 2 both fail → two parities still reconstruct A and B
```

**Why needed?** As disk sizes grow (4TB, 8TB, 16TB), RAID rebuilds take longer (hours or days). During that time, another disk is more likely to fail. RAID 6 tolerates that second failure.

```
  With four 1 TB disks in RAID 6:
  ┌──────────────────────────────┐
  │ Usable: 2 TB (n-2 disks)    │
  │ Speed:  good reads, slower writes (2x parity calc) │
  │ Safety: 2 simultaneous disk failures │
  └──────────────────────────────┘
```

| Property          | RAID 6                                          |
| ----------------- | ----------------------------------------------- |
| Minimum disks     | 4                                               |
| Usable capacity   | (n-2) disks worth                               |
| Fault tolerance   | 2 disk failures                                 |
| Read performance  | Good                                            |
| Write performance | Slower (double parity calculation)              |
| Use case          | Large storage arrays, enterprise data, archival |

---

## 7. RAID 10 — Mirror + Stripe

**Method:** Pair disks into mirrored sets (RAID 1), then stripe data across those mirrored pairs (RAID 0). Combines the safety of RAID 1 with the speed of RAID 0.

```
  Four disks → two mirrored pairs → stripe across pairs:

  [Disk A] ← mirror → [Disk B]     [Disk C] ← mirror → [Disk D]
       ↓                                 ↓
   Pair 1 (mirror)                   Pair 2 (mirror)
       └──────────────────┬─────────────┘
                    RAID 0 STRIPE

  Write "data": half to Pair 1, half to Pair 2 (striped)
                each half is mirrored within its pair

  Pair 1 example: Disk A dies → Disk B still has complete copy of Pair 1 data
                 If BOTH Disk A and Disk B die → Pair 1 completely lost
```

```
  With four 1 TB disks in RAID 10:
  ┌──────────────────────────────┐
  │ Usable: 2 TB (50%)          │
  │ Speed:  Excellent (R+W both) │
  │ Safety: Multiple failures OK │
  │         (as long as not BOTH │
  │          disks in same pair) │
  └──────────────────────────────┘
```

| Property          | RAID 10                                                      |
| ----------------- | ------------------------------------------------------------ |
| Minimum disks     | 4                                                            |
| Usable capacity   | 50%                                                          |
| Fault tolerance   | Multiple failures (no complete pair can be lost)             |
| Read performance  | Excellent                                                    |
| Write performance | Excellent                                                    |
| Use case          | High-performance databases, enterprise apps, trading systems |

---

## 8. Full Comparison Table

| RAID        | Min Disks | Usable Capacity | Fault Tolerance | Read Speed | Write Speed | Best For                    |
| ----------- | --------- | --------------- | --------------- | ---------- | ----------- | --------------------------- |
| **RAID 0**  | 2         | 100%            | None            | Very High  | Very High   | Speed-only, non-critical    |
| **RAID 1**  | 2         | 50%             | 1 failure       | Good       | Normal      | Small critical arrays       |
| **RAID 5**  | 3         | (n−1)/n         | 1 failure       | Good       | Moderate    | General servers, NAS        |
| **RAID 6**  | 4         | (n−2)/n         | 2 failures      | Good       | Slow        | Large arrays, high risk     |
| **RAID 10** | 4         | 50%             | Multiple\*      | Excellent  | Excellent   | Databases, high performance |

\*RAID 10 can survive multiple failures as long as no complete mirrored pair is lost.

**Usable capacity formulas:**

$$\text{RAID 0: } \text{Capacity} = n \times d$$

$$\text{RAID 1: } \text{Capacity} = d$$

$$\text{RAID 5: } \text{Capacity} = (n-1) \times d$$

$$\text{RAID 6: } \text{Capacity} = (n-2) \times d$$

$$\text{RAID 10: } \text{Capacity} = \frac{n}{2} \times d$$

Where $n$ = number of disks, $d$ = capacity per disk.

---

## 9. Hardware vs Software RAID

| Aspect                   | Hardware RAID                                   | Software RAID                    |
| ------------------------ | ----------------------------------------------- | -------------------------------- |
| **Implementation**       | Dedicated RAID controller card with own CPU/RAM | OS manages RAID using system CPU |
| **Performance**          | Excellent (off-loads CPU)                       | Good on modern hardware          |
| **Cost**                 | High (controller card expensive)                | Free (built into OS)             |
| **Hot-swap**             | Yes (enterprise controllers)                    | OS-dependent                     |
| **Battery-backed cache** | Yes (protects against power loss)               | No                               |
| **Portability**          | Low (needs same controller model)               | High (any OS can read it)        |
| **Best for**             | Enterprise servers, data centers                | Home users, small business, VMs  |

**Linux software RAID:** `mdadm` — supports RAID 0/1/5/6/10, very reliable.

**Windows:** Storage Spaces or Dynamic Disk mirroring.

---

## 10. RAID Rebuild Process

When a disk fails in a fault-tolerant RAID:

```
  1. ARRAY ENTERS DEGRADED MODE
     - Still functional (if 1 disk fails in RAID 5/1/6/10)
     - No redundancy during this period
     - Performance may drop (rebuilding on the fly)

  2. ADMIN REPLACES FAILED DISK
     - Hot-swap: no shutdown needed (hardware RAID / some software RAID)
     - Cold-swap: system must be shut down

  3. REBUILD BEGINS
     - Controller reads surviving disks
     - Recalculates data for new disk (using parity or mirror)
     - Writes reconstructed data to new disk
     - Time: hours to DAYS for large disks

  4. ARRAY RETURNS TO NORMAL
     - Full redundancy restored
     - Normal performance returns
```

**Risk window:** During rebuild, NO redundancy. If a second disk fails (RAID 5), all data is lost. Best practice: avoid heavy I/O during rebuild.

---

## 11. Common Misconceptions

### Misconception 1: "RAID is a backup"

```
  WRONG:  RAID mirrors all writes, including bad ones.

  Scenario: Virus encrypts your files.
            RAID 1 faithfully mirrors the encrypted files.
            Both disks now have encrypted (unusable) data.

  RAID ≠ Backup. Always use 3-2-1 backup rule alongside RAID.
```

### Misconception 2: "More disks always means better protection"

```
  Adding more disks to RAID 5 increases capacity, BUT:
  - Larger array = longer rebuild time
  - Longer rebuild = more time with no redundancy
  - More disks = higher probability of a second failure during rebuild

  For very large needs, multiple smaller RAID arrays > one huge RAID array.
```

### Misconception 3: "All RAID levels improve both read and write speed"

| RAID Level | Read Speed        | Write Speed                |
| ---------- | ----------------- | -------------------------- |
| RAID 0     | ↑ Much faster     | ↑ Much faster              |
| RAID 1     | ↑ Slightly faster | → Same as single disk      |
| RAID 5     | ↑ Faster          | ↓ Slower (parity overhead) |
| RAID 6     | ↑ Faster          | ↓↓ Slower (double parity)  |
| RAID 10    | ↑ Much faster     | ↑ Faster                   |

Only RAID 0 and RAID 10 improve both reads AND writes significantly.

---

## 12. Key Takeaways

- **RAID** combines multiple physical disks into one logical unit for performance, fault tolerance, or both — not a replacement for backups
- **Three building blocks:** Striping (speed), Mirroring (copy), Parity (math-based redundancy)
- **RAID 0:** Fastest, zero fault tolerance — one disk dies, everything gone
- **RAID 1:** Mirror copy on 2 disks, survives 1 failure — uses 50% capacity
- **RAID 5:** Stripes data + parity across 3+ disks, survives 1 failure — best general-purpose balance of space, speed, and safety
- **RAID 6:** Like RAID 5 but double parity — survives 2 failures — for large arrays where rebuild risk is high
- **RAID 10:** Mirror pairs then stripe — survives multiple failures — best performance + safety, but 50% capacity efficiency
- **Hardware RAID** is faster and has more features but costs more; **Software RAID** (e.g., Linux mdadm) is free and portable
- **Rebuild window** is the most dangerous time — no redundancy, I/O stress on surviving disks
- **3-2-1 backup rule always applies alongside RAID:** 3 copies of data, on 2 different media, with 1 off-site
