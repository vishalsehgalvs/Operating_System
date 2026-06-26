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

| Advantage               | Reason                                       |
| ----------------------- | -------------------------------------------- |
| Very fast lookup        | One calculation → one cache line to check    |
| Simple hardware         | No comparison circuits across multiple lines |
| Predictable access time | Always the same number of steps              |
| Low cost                | Minimal logic, small chip area               |

### Disadvantages of Direct Mapping

| Disadvantage                         | Reason                                      |
| ------------------------------------ | ------------------------------------------- |
| Conflict misses                      | Multiple blocks competing for same line     |
| Poor cache utilization               | Cache may be empty except for one busy line |
| Performance varies by access pattern | Unlucky mappings cause thrashing            |

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

| Advantage                          | Reason                                       |
| ---------------------------------- | -------------------------------------------- |
| No conflict misses                 | Any block can go anywhere                    |
| Maximum flexibility                | All cache lines usable for any data          |
| Higher hit rate                    | Blocks only evicted when cache is truly full |
| Works well with any access pattern | No unlucky address collisions                |

### Disadvantages of Associative Mapping

| Disadvantage                   | Reason                                       |
| ------------------------------ | -------------------------------------------- |
| Expensive hardware             | Need CAM comparison circuitry for every line |
| Higher power consumption       | Parallel comparators active on every lookup  |
| Larger chip area               | Each line needs a full-width tag comparator  |
| Not practical for large caches | Cost grows linearly with cache size          |

---

## 5. Direct vs Associative — Comparison

| Aspect                  | Direct Mapping                   | Fully Associative                  |
| ----------------------- | -------------------------------- | ---------------------------------- |
| **Placement Rule**      | Fixed: `Block % Cache Lines`     | Flexible: any free line            |
| **Search Method**       | Direct calculation → one line    | Parallel search of all lines (CAM) |
| **Hardware Complexity** | Simple and cheap                 | Complex and expensive              |
| **Access Speed**        | Very fast (arithmetic only)      | Fast but needs CAM                 |
| **Conflict Misses**     | Common — blocks compete          | None — any block goes anywhere     |
| **Cache Utilization**   | Can be poor (empty lines unused) | Excellent                          |
| **Hit Rate**            | Lower (conflicts hurt)           | Higher (only capacity misses)      |
| **Tag Size**            | Smaller (partial address)        | Larger (full block address)        |
| **Best Use Case**       | Cost/power-sensitive, simple     | High-performance, small cache      |

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

| Cache Type            | "Ways" (N)            | Behavior                                   |
| --------------------- | --------------------- | ------------------------------------------ |
| Direct-mapped         | 1-way                 | Each block has exactly 1 possible slot     |
| 2-way set-associative | 2-way                 | Each block has 2 possible slots in its set |
| 4-way set-associative | 4-way                 | 4 possible slots — fewer conflicts         |
| Fully associative     | N-way (N = all lines) | Any block in any slot                      |

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

## 8. Code Examples

> Working code that demonstrates cache mapping — direct mapped, set-associative, and TLB — in practice.

### C++ — Simple Version

Simulate a direct mapped cache: compute slot index from block address, detect hits and misses.

```cpp
#include <iostream>
#include <vector>

// Direct Mapped Cache Simulator
// A cache is a small, fast memory. Each memory block maps to exactly ONE cache slot.
// Formula: cache_index = block_address % num_cache_lines
// If the tag in that slot matches, it's a HIT. Otherwise it's a MISS.

struct CacheLine {
    bool valid = false;   // Is there data stored here?
    int  tag   = -1;      // Which memory block is stored here?
};

class DirectMappedCache {
    int cacheSize;                  // Number of lines in the cache
    std::vector<CacheLine> lines;   // The actual cache storage
    int hits = 0, misses = 0;

public:
    DirectMappedCache(int size) : cacheSize(size), lines(size) {}

    bool access(int blockAddress) {
        int index = blockAddress % cacheSize;   // Which cache slot?
        int tag   = blockAddress / cacheSize;   // Which specific block?

        if (lines[index].valid && lines[index].tag == tag) {
            hits++;
            std::cout << "  Block " << blockAddress << " -> slot " << index << " -> HIT\n";
            return true;   // Cache hit!
        } else {
            misses++;
            // Load the block into cache (evicts whatever was there)
            lines[index].valid = true;
            lines[index].tag   = tag;
            std::cout << "  Block " << blockAddress << " -> slot " << index << " -> MISS (loaded)\n";
            return false;  // Cache miss
        }
    }

    void printStats() {
        int total = hits + misses;
        std::cout << "\nTotal: " << total
                  << " | Hits: " << hits
                  << " | Misses: " << misses
                  << " | Hit rate: " << (100.0 * hits / total) << "%\n";
    }
};

int main() {
    // Cache with 4 lines. Memory has 16 blocks.
    DirectMappedCache cache(4);

    // Blocks 0 and 4 BOTH map to slot 0 — they thrash each other (conflict miss!)
    std::vector<int> accesses = {0, 1, 2, 3, 0, 4, 0, 1, 2, 3};

    std::cout << "Direct Mapped Cache Simulation (4 lines)\n";
    std::cout << "=========================================\n";
    for (int block : accesses) {
        cache.access(block);
    }
    cache.printStats();

    return 0;
}
```

### C++ — Medium / LeetCode Style

N-way set-associative cache with LRU replacement, plus a TLB (Translation Lookaside Buffer) simulation.

```cpp
#include <iostream>
#include <vector>
#include <list>

// N-Way Set-Associative Cache with LRU replacement
// Address layout: [ tag | set_index | block_offset ]
// Within each set: LRU eviction when all N ways are occupied.

struct CacheLine {
    bool valid = false;
    int tag = -1;
};

class SetAssociativeCache {
    int numSets, numWays;
    // Each set is a list of ways — front = most recently used (LRU at back)
    std::vector<std::list<CacheLine>> sets;
    int hits = 0, misses = 0;

public:
    SetAssociativeCache(int sets_, int ways)
        : numSets(sets_), numWays(ways), sets(sets_) {}

    bool access(int blockAddr) {
        int setIdx = blockAddr % numSets;   // Which set?
        int tag    = blockAddr / numSets;   // Tag identifies the block
        auto& set  = sets[setIdx];

        // Search all ways in this set for a tag match
        for (auto it = set.begin(); it != set.end(); ++it) {
            if (it->valid && it->tag == tag) {
                // HIT — move to front (most recently used)
                set.splice(set.begin(), set, it);
                hits++;
                std::cout << "  Block " << blockAddr << " -> set " << setIdx << " -> HIT\n";
                return true;
            }
        }

        // MISS — load block; evict LRU (back of list) if set is full
        misses++;
        if ((int)set.size() >= numWays) {
            set.pop_back();
        }
        set.push_front({true, tag});
        std::cout << "  Block " << blockAddr << " -> set " << setIdx << " -> MISS\n";
        return false;
    }

    void printStats() {
        int total = hits + misses;
        std::cout << "Hits: " << hits << " | Misses: " << misses
                  << " | Hit rate: " << (100.0 * hits / total) << "%\n";
    }
};

// TLB — Translation Lookaside Buffer (fully associative, LRU)
// Maps virtual page number (VPN) -> physical frame number (PFN)
class TLB {
    int capacity;
    std::list<std::pair<int,int>> entries;   // (vpn, pfn) — LRU at back
    int hits = 0, misses = 0;

public:
    TLB(int cap) : capacity(cap) {}

    int translate(int vpn, int pfn_from_page_table) {
        for (auto it = entries.begin(); it != entries.end(); ++it) {
            if (it->first == vpn) {
                entries.splice(entries.begin(), entries, it);
                hits++;
                std::cout << "  VPN " << vpn << " -> TLB HIT  -> PFN " << it->second << "\n";
                return it->second;
            }
        }
        // TLB miss — walk page table (simulated by pfn_from_page_table)
        misses++;
        if ((int)entries.size() >= capacity) entries.pop_back();
        entries.push_front({vpn, pfn_from_page_table});
        std::cout << "  VPN " << vpn << " -> TLB MISS -> PFN " << pfn_from_page_table << " (loaded)\n";
        return pfn_from_page_table;
    }

    void printStats() {
        int total = hits + misses;
        std::cout << "TLB hits: " << hits << " | misses: " << misses
                  << " | Hit rate: " << (100.0 * hits / total) << "%\n";
    }
};

int main() {
    std::cout << "=== 2-Way Set-Associative Cache (4 sets) ===\n";
    SetAssociativeCache cache(4, 2);
    for (int b : {0, 1, 4, 0, 8, 1, 4, 0, 5}) cache.access(b);
    cache.printStats();

    std::cout << "\n=== TLB (4-entry, fully associative, LRU) ===\n";
    TLB tlb(4);
    // (vpn, pfn) pairs — simulate page table lookups on TLB miss
    for (auto [vpn, pfn] : std::vector<std::pair<int,int>>{
            {0,10},{1,20},{2,30},{3,40},{0,10},{4,50},{1,20},{0,10}})
        tlb.translate(vpn, pfn);
    tlb.printStats();

    return 0;
}
```

### Python — Simple Version

Direct mapped cache simulation with hit/miss tracking and statistics.

```python
# Direct Mapped Cache Simulation
# Each memory block maps to exactly one cache slot: slot = block % cache_size
# If the block stored in that slot matches what we want -> HIT, else -> MISS

class DirectMappedCache:
    def __init__(self, size):
        self.size = size
        # Each slot: {'valid': bool, 'tag': int or None}
        self.slots = [{'valid': False, 'tag': None} for _ in range(size)]
        self.hits = 0
        self.misses = 0

    def access(self, block_address):
        slot_index = block_address % self.size     # Which cache slot?
        tag = block_address // self.size            # Tag identifies the exact block

        slot = self.slots[slot_index]

        if slot['valid'] and slot['tag'] == tag:
            # Cache hit — data is already in the cache
            self.hits += 1
            print(f"  Block {block_address:2d} -> slot {slot_index} -> HIT")
        else:
            # Cache miss — load block into cache (evicts whatever was there)
            self.misses += 1
            slot['valid'] = True
            slot['tag'] = tag
            print(f"  Block {block_address:2d} -> slot {slot_index} -> MISS (tag={tag})")

    def stats(self):
        total = self.hits + self.misses
        print(f"\nTotal: {total} | Hits: {self.hits} | Misses: {self.misses} "
              f"| Hit rate: {100 * self.hits / total:.1f}%")


# Demo: 4-slot cache, 16 blocks of memory
print("Direct Mapped Cache (4 slots)")
print("==============================")
cache = DirectMappedCache(size=4)

# Blocks 0 and 4 BOTH map to slot 0 -> they thrash each other (conflict misses)
access_sequence = [0, 1, 2, 3, 0, 4, 0, 1, 2, 3]
print(f"Access sequence: {access_sequence}\n")
for block in access_sequence:
    cache.access(block)

cache.stats()
```

### Python — Medium Level

Set-associative cache with LRU (using `OrderedDict`) plus a fully associative TLB.

```python
from collections import OrderedDict

# N-Way Set-Associative Cache with LRU replacement
# Address -> set_index (modulo), tag (integer division)
# Within each set: LRU eviction when all ways are occupied.

class SetAssociativeCache:
    def __init__(self, num_sets, num_ways):
        self.num_sets = num_sets
        self.num_ways = num_ways
        # Each set is an OrderedDict {tag: True} — last item = most recently used
        self.sets = [OrderedDict() for _ in range(num_sets)]
        self.hits = self.misses = 0

    def access(self, block_addr):
        set_idx   = block_addr % self.num_sets
        tag       = block_addr // self.num_sets
        cache_set = self.sets[set_idx]

        if tag in cache_set:
            # HIT — promote to most-recently-used position
            cache_set.move_to_end(tag)
            self.hits += 1
            print(f"  Block {block_addr:2d} -> set {set_idx} -> HIT")
        else:
            # MISS — evict LRU (first item) if set is full
            self.misses += 1
            if len(cache_set) >= self.num_ways:
                cache_set.popitem(last=False)    # Evict oldest
            cache_set[tag] = True                # Insert as newest
            print(f"  Block {block_addr:2d} -> set {set_idx} -> MISS")

    def stats(self):
        total = self.hits + self.misses
        print(f"Hits: {self.hits} | Misses: {self.misses} | "
              f"Hit rate: {100 * self.hits / total:.1f}%")


# TLB — fully associative with LRU eviction
# Maps virtual page number (VPN) -> physical frame number (PFN)
class TLB:
    def __init__(self, capacity):
        self.capacity = capacity
        self.entries = OrderedDict()   # {vpn: pfn}
        self.hits = self.misses = 0

    def translate(self, vpn, pfn_from_page_table):
        if vpn in self.entries:
            self.entries.move_to_end(vpn)
            self.hits += 1
            print(f"  VPN {vpn} -> TLB HIT  -> PFN {self.entries[vpn]}")
        else:
            self.misses += 1
            if len(self.entries) >= self.capacity:
                self.entries.popitem(last=False)      # Evict LRU
            self.entries[vpn] = pfn_from_page_table
            print(f"  VPN {vpn} -> TLB MISS -> PFN {pfn_from_page_table} (page table walk)")
        return self.entries[vpn]

    def stats(self):
        total = self.hits + self.misses
        print(f"TLB: Hits={self.hits} Misses={self.misses} "
              f"Hit rate={100 * self.hits / total:.1f}%")


print("=== 2-Way Set-Associative Cache (4 sets) ===")
cache = SetAssociativeCache(num_sets=4, num_ways=2)
for b in [0, 1, 4, 0, 8, 1, 4, 0, 5]:
    cache.access(b)
cache.stats()

print("\n=== TLB (4 entries, fully associative, LRU) ===")
tlb = TLB(capacity=4)
for vpn, pfn in [(0,10),(1,20),(2,30),(3,40),(0,10),(4,50),(1,20),(0,10)]:
    tlb.translate(vpn, pfn)
tlb.stats()
```

---

## 9. Key Takeaways

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
