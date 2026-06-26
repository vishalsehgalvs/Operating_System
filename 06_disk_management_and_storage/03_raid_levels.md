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

## 11. Code Examples

> Working code that demonstrates RAID striping, mirroring, and XOR parity in practice.

### C++ — Simple Version
Shows how a file is split (striped) across disks in RAID 0 — block-by-block visualization.

```cpp
// RAID 0 (Striping) Simulator
// Data is split into fixed-size blocks and distributed round-robin across disks.
// Read/write throughput scales with disk count; zero fault tolerance.

#include <iostream>
#include <vector>
#include <string>

int main() {
    const int NUM_DISKS   = 3;   // 3 disks in the RAID 0 array
    const int STRIPE_SIZE = 1;   // 1 block per stripe chunk (simplest case)

    // Each disk holds a sequence of blocks
    std::vector<std::vector<std::string>> disks(NUM_DISKS);

    // File to store: 9 blocks labeled A through I
    std::vector<std::string> file_blocks = {"A","B","C","D","E","F","G","H","I"};

    std::cout << "RAID 0 Write — " << NUM_DISKS << " disks, stripe size = "
              << STRIPE_SIZE << " block\n\n";

    std::cout << "Block | Data | -> Disk (Round-Robin)\n";
    std::cout << "------|------|---------------------\n";

    for (int i = 0; i < (int)file_blocks.size(); i++) {
        int disk_idx = (i / STRIPE_SIZE) % NUM_DISKS;  // which disk gets this block
        int stripe   = i / (STRIPE_SIZE * NUM_DISKS);  // which stripe row

        disks[disk_idx].push_back(file_blocks[i]);

        std::cout << "  [" << i << "]  |  " << file_blocks[i]
                  << "   |  Disk " << disk_idx
                  << "  (stripe " << stripe << ")\n";
    }

    // Print the final physical layout stripe by stripe
    int max_blocks = 0;
    for (auto& d : disks) max_blocks = std::max(max_blocks, (int)d.size());

    std::cout << "\nPhysical disk layout:\n";
    std::cout << "Stripe | Disk0 | Disk1 | Disk2\n";
    std::cout << "-------|-------|-------|------\n";

    for (int s = 0; s < max_blocks; s++) {
        std::cout << "  " << s << "    ";
        for (auto& d : disks) {
            std::cout << " |  " << (s < (int)d.size() ? d[s] : "-") << "  ";
        }
        std::cout << "\n";
    }

    std::cout << "\nRead advantage: all 3 disks can transfer simultaneously -> 3x bandwidth\n";
    std::cout << "Risk: ONE disk failure = entire file LOST (no parity, no mirror)\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style
RAID 5 write with rotating XOR parity, plus recovery of a failed disk.

```cpp
// RAID 5: Distributed Parity with XOR — 1 Disk Failure Tolerance
//
// Layout (4 disks, parity position rotates per stripe):
//   Stripe 0: [D0] [D1] [D2] [P ]   <- parity on disk 3
//   Stripe 1: [D0] [D1] [P ] [D3]   <- parity on disk 2
//   Stripe 2: [D0] [P ] [D2] [D3]   <- parity on disk 1
//   Stripe 3: [P ] [D1] [D2] [D3]   <- parity on disk 0
//
// XOR math: P = D0 ^ D1 ^ D2  =>  any one missing = XOR of the rest

#include <iostream>
#include <vector>
#include <cassert>
#include <cstdint>
#include <numeric>

using Byte = uint8_t;

// XOR all values in a list — the core of RAID parity
Byte xor_all(const std::vector<Byte>& vals) {
    Byte p = 0;
    for (Byte v : vals) p ^= v;
    return p;
}

int main() {
    const int N = 4;   // 4 disks: 3 data + 1 parity (position rotates)

    // 4 stripes of raw data (3 data bytes per stripe — one per data disk)
    std::vector<std::vector<Byte>> data = {
        {0xAA, 0xBB, 0xCC},   // stripe 0
        {0x11, 0x22, 0x33},   // stripe 1
        {0xFF, 0x0F, 0xF0},   // stripe 2
        {0x12, 0x34, 0x56},   // stripe 3
    };

    // disks[d][s] = byte stored on disk d, stripe s
    std::vector<std::vector<Byte>> disks(N);

    // ── Write phase ───────────────────────────────────────────────────────────
    std::cout << "RAID 5 Write Phase\n";
    std::cout << "Stripe | Disk0  | Disk1  | Disk2  | Disk3  | Parity on\n";
    std::cout << std::string(60, '-') << "\n";

    for (int s = 0; s < (int)data.size(); s++) {
        Byte parity    = xor_all(data[s]);             // parity = D0 ^ D1 ^ D2
        int  parity_d  = (N - 1 - s % N);              // parity disk rotates

        // Build the full stripe: insert parity at the rotating position
        std::vector<Byte> stripe(data[s]);
        stripe.insert(stripe.begin() + parity_d, parity);

        for (int d = 0; d < N; d++) disks[d].push_back(stripe[d]);

        printf("  %d    |  0x%02X  |  0x%02X  |  0x%02X  |  0x%02X  | Disk %d (P=0x%02X)\n",
               s, stripe[0], stripe[1], stripe[2], stripe[3], parity_d, parity);
    }

    // ── Simulate Disk 1 failure and recovery ──────────────────────────────────
    const int FAILED_DISK = 1;
    std::cout << "\nDISK " << FAILED_DISK << " FAILED — recovering via XOR...\n";
    std::cout << "Stripe | Disk0  | Disk1     | Disk2  | Disk3  | Recovered\n";
    std::cout << std::string(65, '-') << "\n";

    for (int s = 0; s < (int)data.size(); s++) {
        // Collect all surviving bytes (skip the failed disk)
        std::vector<Byte> survivors;
        for (int d = 0; d < N; d++)
            if (d != FAILED_DISK) survivors.push_back(disks[d][s]);

        Byte recovered = xor_all(survivors);   // XOR of survivors = missing disk

        assert(recovered == disks[FAILED_DISK][s]);   // verify correctness

        printf("  %d    |  0x%02X  | [DEAD]    |  0x%02X  |  0x%02X  |  0x%02X OK\n",
               s, disks[0][s], disks[2][s], disks[3][s], recovered);
    }

    std::cout << "\nXOR math: surviving D0 ^ D2 ^ D3(parity) = recovered D1\n";
    std::cout << "RAID 5 can rebuild any single disk from the remaining data.\n";
    return 0;
}
```

### Python — Simple Version
Visualizes RAID 0 file striping and RAID 1 mirroring side by side.

```python
# RAID 0 (Striping) and RAID 1 (Mirroring) — Side-by-Side Comparison
# Shows how the same file is distributed differently on each RAID level.

def raid0_write(file_blocks, num_disks):
    """Distribute blocks round-robin across disks (RAID 0 striping)."""
    disks = [[] for _ in range(num_disks)]
    for i, block in enumerate(file_blocks):
        disk_idx = i % num_disks   # round-robin: 0, 1, 2, 0, 1, 2, ...
        disks[disk_idx].append(block)
    return disks

def raid1_write(file_blocks, num_disks=2):
    """Write every block to ALL disks (RAID 1 mirroring)."""
    return [list(file_blocks) for _ in range(num_disks)]

def show_layout(disks, label):
    """Print stripe-by-stripe disk layout."""
    print(f"\n{label}")
    max_blocks = max(len(d) for d in disks)
    header = "Stripe  " + "".join(f" Disk{i}  " for i in range(len(disks)))
    print(header)
    print("-" * len(header))
    for s in range(max_blocks):
        row = f"  {s}     "
        for disk in disks:
            row += f"  [{disk[s]}]  " if s < len(disk) else "   --  "
        print(row)

    total_stored = sum(len(d) for d in disks)
    unique_blocks = max(len(d) for d in disks)
    efficiency = 100 * len(disks[0]) / total_stored   # % of raw capacity used
    print(f"\nDisks: {len(disks)} x {unique_blocks} blocks stored")
    print(f"Capacity efficiency: {efficiency:.0f}%")

# 9-block file
file_blocks = ["A", "B", "C", "D", "E", "F", "G", "H", "I"]

print("=" * 45)
print(f"File: {file_blocks} ({len(file_blocks)} blocks)")

raid0 = raid0_write(file_blocks, num_disks=3)
show_layout(raid0, "RAID 0 (Striping, 3 disks) — Speed, no safety")

raid1 = raid1_write(file_blocks, num_disks=2)
show_layout(raid1, "RAID 1 (Mirroring, 2 disks) — Safety, 50% capacity")

print("\nRAID 0: one disk fails -> ALL data lost. Best for temp scratch space.")
print("RAID 1: one disk fails -> other disk has full copy. Best for boot drives.")
```

### Python — Medium Level
RAID 5 write with XOR parity computation, then recovery simulation from a single disk failure.

```python
# RAID 5: Distributed Parity with XOR — write, layout, and disk recovery
# Core math: parity = D0 ^ D1 ^ D2 ... so any one missing value = XOR of rest.

from functools import reduce

def xor_all(values):
    """XOR all integers in the list — the RAID 5 parity formula."""
    return reduce(lambda a, b: a ^ b, values)

class RAID5:
    """
    Simplified RAID 5: N disks, parity rotates left by one position per stripe.
    Each stripe: (N-1) data blocks + 1 parity block distributed across all disks.
    """
    def __init__(self, num_disks):
        self.n = num_disks
        self.disks = [[] for _ in range(num_disks)]

    def parity_disk(self, stripe_idx):
        """Which disk holds parity for this stripe? Rotates each stripe."""
        return (self.n - 1 - stripe_idx % self.n)

    def write(self, data_blocks):
        """Write one stripe. Needs exactly n-1 data values."""
        assert len(data_blocks) == self.n - 1
        s          = len(self.disks[0])   # current stripe index
        p_disk     = self.parity_disk(s)
        parity     = xor_all(data_blocks)

        # Insert parity at the rotating position
        stripe = list(data_blocks)
        stripe.insert(p_disk, parity)

        for d, block in enumerate(stripe):
            self.disks[d].append(block)
        return s, p_disk, parity

    def read(self, stripe_idx, failed_disk=None):
        """Read a stripe, reconstructing missing data via XOR if needed."""
        stripe = [
            self.disks[d][stripe_idx] if d != failed_disk else None
            for d in range(self.n)
        ]
        if failed_disk is not None:
            stripe[failed_disk] = xor_all([b for b in stripe if b is not None])
        return stripe

    def show(self):
        print(f"\nRAID 5 disk layout ({self.n} disks):")
        header = f"{'Stripe':<8}" + "".join(f"{'Disk'+str(d):<11}" for d in range(self.n))
        print(header)
        print("-" * len(header))
        for s in range(len(self.disks[0])):
            p = self.parity_disk(s)
            row = f"  {s}     "
            for d in range(self.n):
                val   = self.disks[d][s]
                label = f"0x{val:02X}{'(P)' if d == p else '   '}"
                row  += f" {label:<10}"
            print(row)

# ── Demo ─────────────────────────────────────────────────────────────────────
raid = RAID5(num_disks=4)   # 4 disks: 3 data + 1 parity per stripe (rotating)

data_stripes = [
    [0xAA, 0xBB, 0xCC],
    [0x11, 0x22, 0x33],
    [0xFF, 0x0F, 0xF0],
    [0x12, 0x34, 0x56],
]

print("Writing 4 stripes to RAID 5:")
for stripe_data in data_stripes:
    s, p_disk, parity = raid.write(stripe_data)
    print(f"  Stripe {s}: data={[f'0x{b:02X}' for b in stripe_data]}, "
          f"parity=0x{parity:02X} on Disk {p_disk}")

raid.show()

# Simulate Disk 1 failure and reconstruct each stripe
print("\n" + "=" * 50)
print("DISK 1 FAILED — reconstructing all stripes:")
all_ok = True
for s in range(len(raid.disks[0])):
    original  = [raid.disks[d][s] for d in range(raid.n)]
    recovered = raid.read(s, failed_disk=1)
    ok = (recovered == original)
    all_ok = all_ok and ok
    print(f"  Stripe {s}: {[f'0x{b:02X}' for b in recovered]}  match={ok}")

print(f"\nAll stripes recovered correctly: {all_ok}")
print("XOR math: D0 ^ D2 ^ D3(parity) = missing D1 — RAID 5 survives 1 disk loss.")
```

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
