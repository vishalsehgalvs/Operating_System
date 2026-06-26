# Cache Mapping: Direct vs Associative Techniques

> **Cache mapping** decides where each block of main memory is stored inside the much smaller cache — **direct mapping** assigns every memory block a fixed single cache slot (simple, fast, but prone to conflicts), while **associative mapping** lets any block go anywhere in cache (flexible, higher hit rate, but needs expensive parallel-search hardware).

---

## Table of Contents

1. [What is Cache Memory?](#1-what-is-cache-memory)
2. [Cache Organization Basics](#2-cache-organization-basics)
3. [Direct Mapping](#3-direct-mapping)
4. [Associative (Fully Associative) Mapping](#4-associative-fully-associative-mapping)
5. [Direct vs Associative — Comparison](#5-direct-vs-associative--comparison)
6. [Real-World Analogy](#6-real-world-analogy)
7. [Practical Considerations](#7-practical-considerations)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What is Cache Memory?

**Cache memory** is a small, very fast layer of storage sitting between the CPU and main RAM. Because RAM is orders of magnitude slower than the CPU, the cache holds copies of recently or frequently used data so the CPU can get them quickly.

```
  Access speed hierarchy:

  CPU Registers  ~  0.3 ns    (1 cycle)
  L1 Cache       ~  1 ns      (4 cycles)
  L2 Cache       ~  3 ns      (12 cycles)
  L3 Cache       ~  10 ns     (40 cycles)
  RAM            ~  60 ns     (200+ cycles)
  SSD            ~  100 µs    (100,000 cycles)
  HDD            ~  10 ms     (10,000,000 cycles)
```

**Cache hit:** CPU asks for data → it's already in cache → retrieved quickly.  
**Cache miss:** CPU asks for data → not in cache → must fetch from RAM and store a copy in cache.

The **hit rate** (% of accesses served from cache) determines how much the cache actually speeds things up. The **mapping technique** directly controls the hit rate by determining which data can coexist in cache.

---

## 2. Cache Organization Basics

Both cache and main memory are divided into fixed-size chunks:
- **Cache line / cache block:** the smallest unit transferred between RAM and cache (typically 64 bytes on modern CPUs)
- Main memory has **many more blocks** than cache can hold

**Example:** Main memory has 1024 blocks; cache can hold only 64 blocks. Mapping determines how 1024 possible blocks compete for 64 slots.

### Memory Address Breakdown

A memory address is split into three fields whose sizes depend on the mapping technique:

```
  |  Tag  |  Index (Line Number)  |  Block Offset  |
  
  Tag:          Identifies WHICH memory block occupies this cache line
  Index:        Selects WHICH cache line to look in  (direct/set-associative)
  Block Offset: Byte position within the cache line
```

---

## 3. Direct Mapping

**Direct mapping** assigns each memory block to exactly **one fixed cache line**. There's no choice — each block has a predetermined home in the cache.

**Analogy:** Assigned parking spaces. If you have parking space #5, you must always park in #5, regardless of whether other spaces are empty.

### How It Works

$$\text{Cache Line} = \text{Memory Block Number} \mod \text{Number of Cache Lines}$$

```
  Example: 4 cache lines (0–3), memory blocks 0–15

  Memory Block    Cache Line (Block % 4)
       0       →       0
       1       →       1
       2       →       2
       3       →       3
       4       →       0    ← competes with block 0!
       5       →       1    ← competes with block 1!
       6       →       2
       7       →       3
       8       →       0    ← also competes with 0 and 4!
       ...
      12       →       0    ← blocks 0, 4, 8, 12 all fight for line 0
```

To distinguish which of the competing blocks currently occupies a cache line, a **tag** is stored alongside the data:

$$\text{Tag} = \lfloor \text{Memory Block Number} / \text{Number of Cache Lines} \rfloor$$

Each cache line contains:
```
  | Valid Bit | Tag | Data (one cache line of bytes) |
```

### Conflict Misses (The Main Problem)

If a program alternates between blocks 0 and 4, they constantly evict each other from cache line 0 — even though lines 1, 2, 3 may be empty. This is a **conflict miss**.

```
  Access sequence: 0 → 4 → 0 → 4 → 0 → 4 → ...
  
  Cache line 0:
  After access 0:  [Block 0]  ← MISS, load block 0
  After access 4:  [Block 4]  ← MISS, block 4 evicts block 0
  After access 0:  [Block 0]  ← MISS, block 0 evicts block 4
  After access 4:  [Block 4]  ← MISS again...
  
  100% miss rate — despite cache not being full!
```

### Advantages of Direct Mapping

| Advantage | Reason |
|-----------|--------|
| Very fast lookup | One calculation → one cache line to check |
| Simple hardware | No comparison circuits across multiple lines |
| Predictable access time | Always the same number of steps |
| Low cost | Minimal logic, small chip area |

### Disadvantages of Direct Mapping

| Disadvantage | Reason |
|-------------|--------|
| Conflict misses | Multiple blocks competing for same line |
| Poor cache utilization | Cache may be empty except for one busy line |
| Performance varies by access pattern | Unlucky mappings cause thrashing |

---

## 4. Associative (Fully Associative) Mapping

**Fully associative mapping** lets any memory block go into any cache line — complete freedom of placement. No fixed formula.

**Analogy:** A parking lot with no assigned spaces. Park wherever there's room. Finding your car later requires checking every spot (but you can look at all spots simultaneously).

### How It Works

- When a block needs to be cached, it goes into **any free line**
- When cache is full, a **replacement policy** (LRU, FIFO, LFU, etc.) evicts a block to make room
- To find a block, the system must **compare the requested address against all tags simultaneously** using special hardware called **CAM (Content Addressable Memory)** or **associative memory**

```
  Access sequence: 0 → 4 → 8 → 12 → 0 → 4 (same example, 4 cache lines)

  Step 1: Access block 0  → MISS, load into line 0
          Cache: [B0][ ][ ][ ]

  Step 2: Access block 4  → MISS, load into line 1
          Cache: [B0][B4][ ][ ]

  Step 3: Access block 8  → MISS, load into line 2
          Cache: [B0][B4][B8][ ]

  Step 4: Access block 12 → MISS, load into line 3
          Cache: [B0][B4][B8][B12]

  Step 5: Access block 0  → HIT! (still in line 0)
  Step 6: Access block 4  → HIT! (still in line 1)

  Blocks 0, 4, 8, 12 coexist peacefully — no conflict!
```

The **tag** in associative mapping stores the **complete** block number (not a partial address) since any block can be anywhere.

```
  Lookup process:
  CPU requests block 7
  ↓
  CAM checks ALL cache lines in parallel:
    Line 0: tag = 3? No
    Line 1: tag = 7? YES → return data from Line 1
    Line 2: tag = 1? No
    ...
  ↓
  Result in one clock cycle despite checking all lines
```

### Advantages of Associative Mapping

| Advantage | Reason |
|-----------|--------|
| No conflict misses | Any block can go anywhere |
| Maximum flexibility | All cache lines usable for any data |
| Higher hit rate | Blocks only evicted when cache is truly full |
| Works well with any access pattern | No unlucky address collisions |

### Disadvantages of Associative Mapping

| Disadvantage | Reason |
|-------------|--------|
| Expensive hardware | Need CAM comparison circuitry for every line |
| Higher power consumption | Parallel comparators active on every lookup |
| Larger chip area | Each line needs a full-width tag comparator |
| Not practical for large caches | Cost grows linearly with cache size |

---

## 5. Direct vs Associative — Comparison

| Aspect | Direct Mapping | Fully Associative |
|--------|---------------|-----------------|
| **Placement Rule** | Fixed: `Block % Cache Lines` | Flexible: any free line |
| **Search Method** | Direct calculation → one line | Parallel search of all lines (CAM) |
| **Hardware Complexity** | Simple and cheap | Complex and expensive |
| **Access Speed** | Very fast (arithmetic only) | Fast but needs CAM |
| **Conflict Misses** | Common — blocks compete | None — any block goes anywhere |
| **Cache Utilization** | Can be poor (empty lines unused) | Excellent |
| **Hit Rate** | Lower (conflicts hurt) | Higher (only capacity misses) |
| **Tag Size** | Smaller (partial address) | Larger (full block address) |
| **Best Use Case** | Cost/power-sensitive, simple | High-performance, small cache |

### Types of Cache Misses

```
  Compulsory Miss (Cold Miss):
    First time accessing a block — must load it regardless of mapping.
    Unavoidable.

  Capacity Miss:
    Cache is full and block was evicted due to size constraints.
    Only solvable by making cache larger.

  Conflict Miss (Collision Miss):
    Two blocks mapped to same line cause mutual eviction,
    even when other lines are empty.
    → Solved by associative mapping or larger caches.
    → Only occurs in direct-mapped and set-associative caches.
```

---

## 6. Real-World Analogy

Imagine organizing books borrowed from a large library (main memory) on your small home bookshelf (cache).

**Direct mapping — assigned shelf sections:**
```
  History books → always go on shelf 1
  Science books → always go on shelf 2
  Fiction       → always go on shelf 3

  You want 3 history books simultaneously.
  Only one fits on shelf 1.
  The others can't go on shelf 2 (even if empty) — they don't belong there.
  → Conflict! Shelf 1 is thrashing while shelf 2 sits empty.
```

**Associative mapping — any book anywhere:**
```
  Any book can go on any shelf.
  You use all space efficiently.
  Finding a book requires checking every shelf...
  ...but you can glance at all shelves simultaneously (CAM).
  → No conflicts, but you need good "eyes" to scan all shelves at once.
```

---

## 7. Practical Considerations

### What Modern CPUs Actually Use: Set-Associative Mapping

Pure direct or fully associative caches are rare in modern CPUs. The standard is **set-associative mapping** — a hybrid:

```
  Set-Associative Cache:

  Cache is divided into "sets", each containing N lines ("N-way associative").
  A memory block maps to a FIXED SET (like direct mapping),
  but can go into ANY LINE within that set (like associative mapping).

  2-way set-associative with 4 sets:
  Set 0: [Line 0] [Line 1]   ← 2 slots for competing blocks
  Set 1: [Line 2] [Line 3]
  Set 2: [Line 4] [Line 5]
  Set 3: [Line 6] [Line 7]

  Block 0 → goes to Set 0, either Line 0 or Line 1
  Block 4 → goes to Set 0, either Line 0 or Line 1 (2 slots = fewer conflicts!)
  Block 8 → goes to Set 0, either Line 0 or Line 1
```

| Cache Type | "Ways" (N) | Behavior |
|-----------|-----------|---------|
| Direct-mapped | 1-way | Each block has exactly 1 possible slot |
| 2-way set-associative | 2-way | Each block has 2 possible slots in its set |
| 4-way set-associative | 4-way | 4 possible slots — fewer conflicts |
| Fully associative | N-way (N = all lines) | Any block in any slot |

**Typical modern CPU cache configs:**
- L1 cache: 4-way or 8-way set-associative (~32–64 KB)
- L2 cache: 8-way set-associative (~256 KB – 1 MB)
- L3 cache: 16-way or 32-way set-associative (~8–64 MB)

### Where Fully Associative IS Used

**TLBs (Translation Lookaside Buffers):** These are small (32–128 entries) and performance-critical. The fully associative overhead is justifiable because the TLB is tiny and any TLB miss is extremely expensive.

### Impact on Program Performance

```
  // Bad for direct-mapped cache (classic conflict pattern):
  int A[1024], B[1024];  // if A and B map to same cache lines
  for (int i = 0; i < 1024; i++)
      sum += A[i] + B[i];   // A[i] evicts B[i] and vice versa — 100% miss rate!

  // Fix: pad arrays to shift their alignment
  int A[1024], pad[8], B[1024];  // offset B by 8 ints = 32 bytes
  // Now A[i] and B[i] map to DIFFERENT cache lines → conflict eliminated
```

Understanding cache mapping helps you write **cache-friendly** code — especially in scientific computing, game engines, and database systems where memory access patterns dominate performance.

---

## 8. Key Takeaways

- **Cache** is a small, fast memory buffer between CPU and RAM; **cache mapping** decides which RAM block occupies which cache slot
- **Cache hit** = data found in cache (fast); **cache miss** = must fetch from RAM (slow)
- **Direct mapping:** each block has exactly one allowed cache line (`block % num_lines`); simple and fast but suffers **conflict misses** when multiple hot blocks share a line
- **Fully associative mapping:** any block can go in any cache line; no conflict misses but requires expensive CAM hardware for parallel tag search
- **Three miss types:** compulsory (first access), capacity (cache full), conflict (only in direct/set-associative when blocks collide)
- **Tag** identifies which memory block a cache line currently holds; direct mapping uses a smaller partial tag; associative mapping needs a full-address tag
- **Modern CPUs use set-associative mapping** (N-way) as a practical compromise — divides cache into sets (fixed by address) with N slots per set (flexible placement within the set)
- **L1/L2 caches:** typically 4-way to 8-way set-associative; **TLBs:** fully associative (they're tiny, so hardware cost is justified)
- **Performance implication:** two arrays at addresses that map to the same cache lines will constantly evict each other (thrash); padding/alignment changes can fix this
- **Hit rate** is what matters most: fully associative maximizes it but costs too much for large caches; set-associative gives near-optimal hit rates at affordable hardware cost
