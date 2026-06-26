# Page Replacement Algorithms: FIFO, LRU, and Optimal

> When all physical frames are occupied and a new page must be loaded, the OS must evict a victim page — which one to choose is decided by the page replacement algorithm; FIFO evicts the oldest page, LRU evicts the least recently used, and Optimal (theoretical) evicts the page whose next use is farthest in the future.

---

## Table of Contents

1. [Why Page Replacement Matters](#1-why-page-replacement-matters)
2. [FIFO — First In, First Out](#2-fifo--first-in-first-out)
3. [LRU — Least Recently Used](#3-lru--least-recently-used)
4. [Optimal — Belady's Optimal Algorithm](#4-optimal--beladys-optimal-algorithm)
5. [Side-by-Side Comparison](#5-side-by-side-comparison)
6. [LRU Approximations Used in Real OSes](#6-lru-approximations-used-in-real-oses)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Why Page Replacement Matters

When a **page fault** occurs and all physical frames are full, the OS cannot just load the new page — it must first **evict (replace) an existing page** to make room.

```
  Physical RAM (3 frames, all occupied):
  ┌────────┬────────┬────────┐
  │ Page 7 │ Page 2 │ Page 5 │
  └────────┴────────┴────────┘

  Page fault for Page 9 → which one do we kick out?

  This is the page replacement decision.
  Bad choices → more page faults → slower system
  Good choices → fewer page faults → faster system
```

**The goal:** minimize total page faults for a given reference string (the sequence of page accesses).

**Setup used for all examples:**

```
  Reference string: 7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2
  Number of frames: 3
```

---

## 2. FIFO — First In, First Out

### How It Works

Maintain a **queue** of pages in the order they were loaded. When eviction is needed, remove the **oldest page** (head of the queue).

> Like a coffee shop queue: the first person in line is the first to leave.

### Step-by-Step Trace

| Request | Frames (oldest→newest) | Page Fault? | Evicted    |
| ------- | ---------------------- | ----------- | ---------- |
| 7       | [7, –, –]              | ✗ Yes       | —          |
| 0       | [7, 0, –]              | ✗ Yes       | —          |
| 1       | [7, 0, 1]              | ✗ Yes       | —          |
| 2       | [2, 0, 1]              | ✗ Yes       | 7 (oldest) |
| 0       | [2, 0, 1]              | ✓ No        | —          |
| 3       | [2, 3, 1]              | ✗ Yes       | 0 (oldest) |
| 0       | [2, 3, 0]              | ✗ Yes       | 1 (oldest) |
| 4       | [4, 3, 0]              | ✗ Yes       | 2 (oldest) |
| 2       | [4, 2, 0]              | ✗ Yes       | 3 (oldest) |
| 3       | [4, 2, 3]              | ✗ Yes       | 0 (oldest) |
| 0       | [0, 2, 3]              | ✗ Yes       | 4 (oldest) |
| 3       | [0, 2, 3]              | ✓ No        | —          |
| 2       | [0, 2, 3]              | ✓ No        | —          |

**Total page faults: 11** (out of 13 accesses)

Note: FIFO evicted page 0 multiple times even though it was frequently accessed — it doesn't consider usage!

### Pros and Cons

| Pros                  | Cons                                                      |
| --------------------- | --------------------------------------------------------- |
| Simple — just a queue | Ignores usage frequency                                   |
| Minimal overhead      | Can evict heavily-used pages                              |
| Easy to implement     | Suffers from Belady's Anomaly (more frames → more faults) |

---

## 3. LRU — Least Recently Used

### How It Works

Track when each page was **last accessed**. When eviction is needed, remove the page that **hasn't been used for the longest time**.

> Based on **temporal locality** — if a page was used recently, it will likely be used again soon. Evict the one that's been idle the longest.

### Step-by-Step Trace

| Request | Frames (LRU→MRU)   | Page Fault? | Evicted                                |
| ------- | ------------------ | ----------- | -------------------------------------- |
| 7       | [7, –, –]          | ✗ Yes       | —                                      |
| 0       | [7, 0, –]          | ✗ Yes       | —                                      |
| 1       | [7, 0, 1]          | ✗ Yes       | —                                      |
| 2       | [2, 0, 1]          | ✗ Yes       | 7 (LRU)                                |
| 0       | [1, 0, 2] (0 used) | ✓ No        | —                                      |
| 3       | [0, 2, 3]          | ✗ Yes       | 1 (LRU)                                |
| 0       | [2, 3, 0]          | ✓ No        | —                                      |
| 4       | [3, 0, 4]          | ✗ Yes       | 2 (LRU)                                |
| 2       | [0, 4, 2]          | ✗ Yes       | 3 (LRU)                                |
| 3       | [0, 2, 3]          | ✗ Yes       | 4 (LRU)                                |
| 0       | [2, 3, 0]          | ✗ Yes       | 4 → 0 already… wait, evict LRU=4 above |
| 3       | [2, 0, 3]... ✓ No  | ✓ No        | —                                      |
| 2       | [0, 3, 2]          | ✓ No        | —                                      |

**Total page faults: 10** (1 fewer than FIFO — smarter eviction)

### Implementation Approaches

**Counter/Timestamp:**

```c
// Associate a timestamp with each page
page_table[p].last_used = current_time

// On fault: scan all pages, evict the one with the oldest timestamp
// O(n) scan per eviction — accurate but slow
```

**Stack (most practical):**

```
  Keep pages in a stack ordered by recency.
  On any page access: move that page to the TOP of the stack.
  On eviction: remove the page at the BOTTOM (least recently used).

  Example after accessing: 7, 0, 1, 2, 0
  Stack (bottom=LRU, top=MRU): [1, 2, 0]
  Evict: 1 (bottom)
```

### Pros and Cons

| Pros                                  | Cons                                              |
| ------------------------------------- | ------------------------------------------------- |
| Exploits temporal locality            | More complex than FIFO                            |
| Generally fewer faults than FIFO      | Must update metadata on every access              |
| Does NOT suffer from Belady's Anomaly | Full LRU needs hardware counters or stack updates |

---

## 4. Optimal — Belady's Optimal Algorithm

### How It Works

**Look into the future.** When eviction is needed, replace the page whose **next use is farthest ahead** in the reference string. If a page is never used again, evict it first.

> Like having a crystal ball — perfect decisions, zero regrets.

### Step-by-Step Trace

| Request | Frames    | Page Fault? | Evicted | Why                      |
| ------- | --------- | ----------- | ------- | ------------------------ |
| 7       | [7, –, –] | ✗ Yes       | —       | —                        |
| 0       | [7, 0, –] | ✗ Yes       | —       | —                        |
| 1       | [7, 0, 1] | ✗ Yes       | —       | —                        |
| 2       | [2, 0, 1] | ✗ Yes       | 7       | 7 never appears again    |
| 0       | [2, 0, 1] | ✓ No        | —       | —                        |
| 3       | [2, 0, 3] | ✗ Yes       | 1       | 1's next use is farthest |
| 0       | [2, 0, 3] | ✓ No        | —       | —                        |
| 4       | [4, 0, 3] | ✗ Yes       | 2       | 2's next use at pos 8    |
| 2       | [4, 0, 2] | ✗ Yes       | 3       | 3's next use at pos 9    |
| 3       | [4, 3, 2] | ✗ Yes       | 0       | 0's next use at pos 10   |
| 0       | [0, 3, 2] | ✗ Yes       | 4       | 4 never appears again    |
| 3       | [0, 3, 2] | ✓ No        | —       | —                        |
| 2       | [0, 3, 2] | ✓ No        | —       | —                        |

**Total page faults: 9** — the theoretical minimum for this input.

### Why Study It If It's Not Implementable?

Optimal serves as a **benchmark / lower bound**. System designers run it on recorded traces and compare:

```
  FIFO:    11 faults
  LRU:     10 faults
  Optimal:  9 faults  ← theoretical best

  LRU is 90% of the way to optimal — a great practical choice.
  FIFO wastes 2 extra faults vs optimal — significant at scale.
```

---

## 5. Side-by-Side Comparison

| Criteria              | FIFO               | LRU                         | Optimal           |
| --------------------- | ------------------ | --------------------------- | ----------------- |
| Eviction rule         | Oldest loaded page | Least recently accessed     | Farthest next use |
| Page faults (example) | 11                 | 10                          | 9                 |
| Implementable?        | Yes                | Yes (with overhead)         | No (needs future) |
| Implementation cost   | Very low (queue)   | Medium (timestamps/stack)   | N/A               |
| Belady's Anomaly?     | Yes (FIFO only)    | No                          | No                |
| Basis                 | Time in memory     | Temporal locality           | Clairvoyance      |
| Used in modern OSes?  | Rarely             | Yes (or LRU approximations) | Benchmark only    |

---

## 6. LRU Approximations Used in Real OSes

Exact LRU is expensive because every memory access must update a counter or stack. Real OSes use approximations:

### Reference Bit (Clock Algorithm / Second-Chance)

```
  Each page has a reference bit (R), initially 0.
  Hardware sets R=1 whenever a page is accessed.

  On eviction — scan pages in circular order (like a clock hand):
    If R=1 → give it a "second chance": set R=0, move to next page
    If R=0 → evict this page (hasn't been used since last scan)

  Pages that were recently used get their bit reset before eviction,
  giving them a second chance.

  This approximates LRU cheaply — O(1) eviction amortized.
```

### Clock Algorithm Visual

```
         ┌───────────────────────────────────┐
  Clock  │ Page A (R=1) → Page B (R=0) evict│
  hand──►│ Page C (R=1) → Page D (R=1)      │
         └───────────────────────────────────┘

  Page B: R=0 → evicted (clock hand selects it)
  Pages A, C, D: R=1 → reset to 0, skipped this round
```

**Linux** uses an enhanced version of the clock algorithm. **Windows** uses a **working set** approach that also considers LRU order within a process's allocated frames.

---

## 7. Key Takeaways

- Page replacement is needed when **all frames are occupied** and a new page must be loaded
- **FIFO** — evict the page that has been in memory the longest; simple, but can evict frequently-used pages; suffers from Belady's Anomaly
- **LRU** — evict the page that has not been accessed for the longest time; uses temporal locality; no Belady's Anomaly; most commonly used in practice
- **Optimal** — evict the page with the farthest next use; minimum possible faults; requires future knowledge → only usable as a benchmark
- For this reference string (3 frames): FIFO=11, LRU=10, Optimal=9
- **Belady's Anomaly** = counterintuitive behavior where FIFO causes _more_ faults with _more_ frames (LRU and Optimal are immune — next topic explains this in detail)
- Real OSes use **LRU approximations** (clock/second-chance algorithm with reference bits) because exact LRU is too expensive to maintain on every memory access
- Writing programs that access memory in **spatial and temporal locality patterns** naturally reduces page faults regardless of which algorithm the OS uses
