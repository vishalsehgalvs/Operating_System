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

## 6. Code Examples

> Working code that demonstrates FIFO, LRU, and Optimal page replacement in practice.

### C++ — Simple Version

Implement all three algorithms; run on reference string `7,0,1,2,0,3,0,4,2,3,0,3,2` with 3 frames.

```cpp
// Page Replacement: FIFO, LRU, Optimal
// Compile: g++ -std=c++17 page_replace.cpp -o page_replace

#include <iostream>
#include <vector>
#include <deque>
#include <set>
#include <algorithm>
#include <climits>
using namespace std;

// ===== FIFO =====
// Evict the page that has been in memory the LONGEST (oldest arrival time)
int fifo(const vector<int>& refs, int frames) {
    deque<int> q;     // front = oldest (evict next); back = newest
    set<int>   inRAM;
    int faults = 0;
    for (int page : refs) {
        if (inRAM.count(page)) continue;   // hit
        faults++;
        if ((int)q.size() == frames) {
            inRAM.erase(q.front());
            q.pop_front();                 // evict oldest
        }
        q.push_back(page);
        inRAM.insert(page);
    }
    return faults;
}

// ===== LRU =====
// Evict the page that was LEAST RECENTLY USED (idle the longest)
int lru(const vector<int>& refs, int frames) {
    deque<int> q;     // front = LRU (evict next); back = MRU
    set<int>   inRAM;
    int faults = 0;
    for (int page : refs) {
        if (inRAM.count(page)) {
            // Hit: move this page to MRU position (back of deque)
            q.erase(find(q.begin(), q.end(), page));
            q.push_back(page);
            continue;
        }
        faults++;
        if ((int)q.size() == frames) {
            inRAM.erase(q.front());
            q.pop_front();                 // evict LRU page
        }
        q.push_back(page);
        inRAM.insert(page);
    }
    return faults;
}

// ===== OPTIMAL =====
// Evict the page whose NEXT USE is farthest in the future (requires future knowledge)
int optimal(const vector<int>& refs, int frames) {
    vector<int> inRAM;
    int faults = 0;
    for (int i = 0; i < (int)refs.size(); i++) {
        int page = refs[i];
        if (find(inRAM.begin(), inRAM.end(), page) != inRAM.end()) continue;
        faults++;
        if ((int)inRAM.size() < frames) {
            inRAM.push_back(page);
            continue;
        }
        // Find which current page in RAM has the farthest next use
        int evictIdx = 0, farthest = -1;
        for (int j = 0; j < (int)inRAM.size(); j++) {
            int nextUse = INT_MAX;   // assume "never used again" = infinity
            for (int k = i + 1; k < (int)refs.size(); k++)
                if (refs[k] == inRAM[j]) { nextUse = k; break; }
            if (nextUse > farthest) { farthest = nextUse; evictIdx = j; }
        }
        inRAM[evictIdx] = page;
    }
    return faults;
}

int main() {
    // Classic reference string from OS textbooks
    vector<int> refs   = {7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2};
    int         frames = 3;

    cout << "Reference string: ";
    for (int p : refs) cout << p << " ";
    cout << "\nFrames: " << frames << "\n\n";

    cout << "FIFO    page faults: " << fifo(refs, frames)    << "\n";
    cout << "LRU     page faults: " << lru(refs, frames)     << "\n";
    cout << "Optimal page faults: " << optimal(refs, frames) << "\n";
    // FIFO=10, LRU=9, Optimal=7
    return 0;
}
```

### C++ — Medium / LeetCode Style

**LeetCode #146 — LRU Cache** (the O(1) implementation with `unordered_map` + doubly linked list) combined with page fault counting.

```cpp
// LRU Cache — O(1) get/put using unordered_map + doubly linked list
// This is the exact data structure used for efficient LRU page replacement.
// LeetCode #146: https://leetcode.com/problems/lru-cache/
// Compile: g++ -std=c++17 lru_cache.cpp -o lru_cache

#include <iostream>
#include <unordered_map>
#include <list>
#include <vector>
using namespace std;

// O(1) LRU via doubly linked list + hash map:
//   list: front = MRU, back = LRU; each node holds a page number
//   map:  page -> iterator into the list (for O(1) removal from any position)
class LRUPageReplacer {
    int capacity;
    list<int>                         order;  // front=MRU, back=LRU
    unordered_map<int, list<int>::iterator> pos;    // page -> list position
    int faults = 0;

public:
    explicit LRUPageReplacer(int cap) : capacity(cap) {}

    // Access a page; returns true if it was already in RAM (hit), false if fault
    bool access(int page) {
        if (pos.count(page)) {
            // Hit: move this page to the front (MRU)
            order.erase(pos[page]);
            order.push_front(page);
            pos[page] = order.begin();
            return true;
        }
        // Page fault
        faults++;
        if ((int)order.size() == capacity) {
            // Evict the LRU page (back of list)
            int lruPage = order.back();
            order.pop_back();
            pos.erase(lruPage);
        }
        order.push_front(page);
        pos[page] = order.begin();
        return false;
    }

    int getFaults() const { return faults; }

    void printRAM() const {
        cout << "  RAM (MRU->LRU): ";
        for (int p : order) cout << p << " ";
        cout << "\n";
    }
};

// Count page faults for a reference string using the O(1) LRU cache
int lruPageFaults(const vector<int>& refs, int frames) {
    LRUPageReplacer lru(frames);
    for (int p : refs) lru.access(p);
    return lru.getFaults();
}

int main() {
    vector<int> refs   = {7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2};
    int         frames = 3;

    cout << "=== LRU Cache (O(1) — LeetCode #146 style) ===\n";
    cout << "Reference string: ";
    for (int p : refs) cout << p << " ";
    cout << "\nFrames: " << frames << "\n\n";

    LRUPageReplacer lru(frames);
    for (int page : refs) {
        bool hit = lru.access(page);
        cout << "Page " << page << ": " << (hit ? "HIT " : "MISS");
        lru.printRAM();
    }

    cout << "\nTotal page faults: " << lru.getFaults() << "\n";  // 9
    return 0;
}
```

### Python — Simple Version

Implement FIFO, LRU, and Optimal; print a comparison table for the classic reference string.

```python
# Page Replacement: FIFO, LRU, Optimal
# Run: python page_replace.py

from collections import deque


def fifo(refs: list[int], frames: int) -> int:
    """Evict the page that arrived first (oldest)."""
    queue  = deque()   # left=oldest, right=newest
    in_ram = set()
    faults = 0

    for page in refs:
        if page in in_ram:
            continue   # hit
        faults += 1
        if len(queue) == frames:
            in_ram.discard(queue.popleft())   # evict oldest
        queue.append(page)
        in_ram.add(page)
    return faults


def lru(refs: list[int], frames: int) -> int:
    """Evict the page that was least recently used."""
    order  = []        # left=LRU, right=MRU
    in_ram = set()
    faults = 0

    for page in refs:
        if page in in_ram:
            order.remove(page)
            order.append(page)   # move to MRU position
            continue
        faults += 1
        if len(order) == frames:
            evicted = order.pop(0)   # remove LRU (leftmost)
            in_ram.discard(evicted)
        order.append(page)
        in_ram.add(page)
    return faults


def optimal(refs: list[int], frames: int) -> int:
    """Evict the page whose next use is farthest in the future."""
    in_ram = []
    faults = 0

    for i, page in enumerate(refs):
        if page in in_ram:
            continue
        faults += 1
        if len(in_ram) < frames:
            in_ram.append(page)
            continue
        # Find page in RAM with the farthest next use
        def next_use(p):
            for k in range(i + 1, len(refs)):
                if refs[k] == p:
                    return k
            return float('inf')  # never used again -> evict this one

        victim = max(in_ram, key=next_use)
        in_ram[in_ram.index(victim)] = page

    return faults


def main():
    refs   = [7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2]
    frames = 3

    print(f"Reference string : {refs}")
    print(f"Frames           : {frames}\n")

    fifo_f = fifo(refs, frames)
    lru_f  = lru(refs, frames)
    opt_f  = optimal(refs, frames)

    print(f"FIFO    page faults : {fifo_f}")
    print(f"LRU     page faults : {lru_f}")
    print(f"Optimal page faults : {opt_f}")
    print(f"\nOptimal is the theoretical minimum: {opt_f} faults")
    print(f"LRU overhead over Optimal: {lru_f - opt_f} extra fault(s)")
    # FIFO=10, LRU=9, Optimal=7


main()
```

### Python — Medium Level

**LeetCode #146 — LRU Cache** implemented with `OrderedDict` (O(1) moves); use it to count page faults.

```python
# LRU Cache — O(1) implementation using OrderedDict (Python's built-in)
# Equivalent to the doubly linked list + hash map solution in LeetCode #146.
# Run: python lru_cache.py

from collections import OrderedDict


class LRUPageReplacer:
    """
    O(1) LRU page replacement using OrderedDict.
    OrderedDict maintains insertion order; move_to_end() is O(1).
    Evict the item at the front (last=False = least recently used).
    """

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache    = OrderedDict()   # page -> dummy value; LRU is first key
        self.faults   = 0

    def access(self, page: int) -> bool:
        """Access a page. Returns True if hit, False if page fault."""
        if page in self.cache:
            self.cache.move_to_end(page)   # mark as most recently used (O(1))
            return True  # hit

        # Page fault
        self.faults += 1
        if len(self.cache) == self.capacity:
            self.cache.popitem(last=False)  # evict LRU (first item, O(1))
        self.cache[page] = None
        return False

    def ram_state(self) -> list[int]:
        """Return pages from LRU to MRU."""
        return list(self.cache.keys())


def main():
    refs   = [7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2]
    frames = 3

    replacer = LRUPageReplacer(frames)

    print(f"=== LRU Cache (O(1), OrderedDict) ===")
    print(f"Reference string : {refs}")
    print(f"Frames           : {frames}\n")
    print(f"{'Page':<6} {'Result':<8} {'RAM (LRU -> MRU)'}")
    print("-" * 40)

    for page in refs:
        hit    = replacer.access(page)
        result = "HIT" if hit else f"FAULT #{replacer.faults}"
        print(f"  {page:<4}  {result:<10} {replacer.ram_state()}")

    print(f"\nTotal page faults: {replacer.faults}")   # 9
    print("(Same result as simple LRU, but O(1) per access instead of O(n))")


main()
```

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
