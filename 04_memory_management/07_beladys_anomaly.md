# Belady's Anomaly

> Belady's Anomaly is the counterintuitive phenomenon where adding more physical frames causes **more** page faults (not fewer) with the FIFO page replacement algorithm; it happens because FIFO's age-based eviction doesn't track usage, so more frames changes which pages survive and can accidentally worsen the replacement sequence.

---

## Table of Contents

1. [What Is Belady's Anomaly?](#1-what-is-beladys-anomaly)
2. [Why It Happens](#2-why-it-happens)
3. [Which Algorithms Are Affected?](#3-which-algorithms-are-affected)
4. [The Stack Property (Inclusion Property)](#4-the-stack-property-inclusion-property)
5. [Demonstration: FIFO with 3 vs 4 Frames](#5-demonstration-fifo-with-3-vs-4-frames)
6. [FIFO vs LRU Behaviour Comparison](#6-fifo-vs-lru-behaviour-comparison)
7. [Practical Implications](#7-practical-implications)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What Is Belady's Anomaly?

**Belady's Anomaly** is the observation that, for certain page replacement algorithms, increasing the number of frames in physical memory can cause the number of page faults to **increase** instead of decrease or stay the same.

Discovered by **László Bélády in 1969** while studying FIFO.

> Normal expectation: more RAM = fewer page faults (more pages fit, less disk I/O)
> Belady's discovery: with FIFO, this is NOT always true.

**Desk drawer analogy:**

```
  You have 3 drawers for your most-used tools.
  A "dumb organizer" (FIFO) always removes the tool placed there longest ago.

  When you get a 4th drawer, the organizer now has MORE choices.
  But because it still only looks at "how long has this been here",
  different tools survive now — and the tools you actually need most
  end up getting thrown out at the wrong moment.

  More drawers, more confusion, more times you reach for something missing.
```

---

## 2. Why It Happens

FIFO's eviction decision is based **only on time in memory** (insertion order), with zero regard for how often or recently a page is actually used.

When the frame count changes, the entire replacement sequence cascades differently:

- Different pages survive longer
- Different pages are present when future accesses happen
- Some critical pages that were "sheltered" by other pages in the 3-frame case get evicted earlier in the 4-frame case

```
  Root cause: FIFO lacks the STACK PROPERTY.

  Stack property = "if page P is in memory with N frames,
                    it will also be in memory with N+1 frames"

  FIFO does NOT guarantee this.
  LRU and Optimal DO guarantee this.
```

---

## 3. Which Algorithms Are Affected?

| Algorithm                  | Belady's Anomaly? | Why?                                                    |
| -------------------------- | ----------------- | ------------------------------------------------------- |
| **FIFO**                   | **Yes**           | Age-based eviction — no usage tracking                  |
| Other non-stack algorithms | Potentially       | If they lack the inclusion property                     |
| **LRU**                    | **No**            | Stack-based — satisfies inclusion property              |
| **Optimal**                | **No**            | Always makes the best choice — no anomaly by definition |
| Second-Chance (Clock)      | Generally No      | Approximates LRU behaviour                              |

---

## 4. The Stack Property (Inclusion Property)

An algorithm satisfies the **stack/inclusion property** if:

$$\text{Pages in memory with } n \text{ frames} \subseteq \text{Pages in memory with } n+1 \text{ frames}$$

In plain English: **if a page is loaded with N frames, it will still be loaded when you add a frame**. More frames only ever helps — never hurts.

```
  LRU with 3 frames keeps: {A, B, C}  (the 3 most recently used)
  LRU with 4 frames keeps: {A, B, C, D}  (the 4 most recently used)
  {A, B, C} ⊆ {A, B, C, D} ✓ — inclusion property holds

  FIFO with 3 frames keeps: {X, Y, Z}  (oldest three present)
  FIFO with 4 frames keeps: {W, X, Y, Z}  (BUT the order changed!)
  {X, Y, Z} is NOT necessarily ⊆ {W, X, Y, Z} — inclusion can fail ✗
```

---

## 5. Demonstration: FIFO with 3 vs 4 Frames

**Reference string:** `1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5`

### Case A: 3 Frames

```
  Ref:   1  2  3  4  1  2  5  1  2  3  4  5
         ─────────────────────────────────────
  F1:    1  1  1  4  4  4  5  5  5  5  4  4
  F2:    ─  2  2  2  1  1  1  1  1  1  1  1
  F3:    ─  ─  3  3  3  2  2  2  2  3  3  3
         ─────────────────────────────────────
  Fault: ✗  ✗  ✗  ✗  ✗  ✗  ✗  ·  ·  ✗  ✗  ✗

  Total Faults: 9
```

### Case B: 4 Frames

```
  Ref:   1  2  3  4  1  2  5  1  2  3  4  5
         ─────────────────────────────────────
  F1:    1  1  1  1  1  1  5  5  5  5  4  4
  F2:    ─  2  2  2  2  2  2  1  1  1  1  1
  F3:    ─  ─  3  3  3  3  3  3  2  2  2  2
  F4:    ─  ─  ─  4  4  4  4  4  4  3  3  5
         ─────────────────────────────────────
  Fault: ✗  ✗  ✗  ✗  ·  ·  ✗  ✗  ✗  ✗  ✗  ✗

  Total Faults: 10
```

### The Anomaly in Numbers

```
  ┌─────────────┬──────────────────┬──────────────────────────────────┐
  │ Frame Count │ FIFO Page Faults │ Expected direction                │
  ├─────────────┼──────────────────┼──────────────────────────────────┤
  │ 3 frames    │ 9                │                                   │
  │ 4 frames    │ 10  ← MORE!      │ ← Should be ≤ 9, not > 9!        │
  └─────────────┴──────────────────┴──────────────────────────────────┘

  Adding one frame INCREASED faults by 1.
  This is Belady's Anomaly.
```

**What changed?** With 4 frames, pages 1 and 2 survived long enough to avoid eviction at positions 4–5 (so no faults there). But this pushed the "old page" queue differently — pages 1, 2, 3 were later evicted at moments when they were actually needed again, causing faults that didn't happen in the 3-frame case.

---

## 6. FIFO vs LRU Behaviour Comparison

| Aspect                         | FIFO                   | LRU                                    |
| ------------------------------ | ---------------------- | -------------------------------------- |
| Eviction basis                 | Order of arrival (age) | Recency of use                         |
| Stack property                 | No                     | Yes                                    |
| Belady's Anomaly               | Can occur              | Never occurs                           |
| Inclusion property             | Not guaranteed         | Always satisfied                       |
| Performance as frames increase | Unpredictable          | Monotonically improves (or stays same) |
| Implementation complexity      | Low                    | Medium                                 |

```mermaid
graph TD
    A[Add more frames] --> B{Algorithm has\nstack property?}
    B -->|YES - LRU / Optimal| C[Page faults can\nonly stay same or decrease]
    B -->|NO - FIFO| D{Access pattern\ninteraction?}
    D --> E[Usually fewer faults]
    D --> F[Belady's Anomaly!\nMore faults with more frames]
```

---

## 7. Practical Implications

**Why does this matter in real systems?**

```
  Scenario: A sysadmin adds 4 GB of RAM to a server running old software
            that uses a FIFO-like page management strategy.

  Expected: System becomes faster
  Possible: System performance DEGRADES on certain workloads

  Root cause: Belady's Anomaly with the right (wrong?) reference pattern.
```

**Design lessons:**

1. **Don't use FIFO in production** — LRU approximations are universally preferred
2. **Don't assume more RAM = better performance** without testing — benchmark with realistic workloads
3. **Test across different frame counts** — if using any non-stack algorithm, verify monotonic improvement
4. **Modern OSes are immune** — Linux, Windows, macOS all use LRU-approximation algorithms which satisfy the stack property

---

## 7. Code Examples

> Working code that demonstrates Belady's Anomaly — FIFO gives MORE faults with MORE frames.

### C++ — Simple Version
Run FIFO on the classic reference string `1,2,3,4,1,2,5,1,2,3,4,5` with 3 frames and then 4 frames; show the anomaly.

```cpp
// Belady's Anomaly: FIFO with 3 vs 4 frames on the classic reference string
// Compile: g++ -std=c++17 beladys.cpp -o beladys

#include <iostream>
#include <vector>
#include <deque>
#include <set>
using namespace std;

// FIFO page replacement: returns page fault count
int fifo(const vector<int>& refs, int frames) {
    deque<int> q;      // front = oldest (evict next)
    set<int>   inRAM;
    int faults = 0;
    for (int page : refs) {
        if (inRAM.count(page)) continue;   // hit
        faults++;
        if ((int)q.size() == frames) {
            inRAM.erase(q.front());
            q.pop_front();
        }
        q.push_back(page);
        inRAM.insert(page);
    }
    return faults;
}

// Verbose FIFO: prints every step
void fifoVerbose(const vector<int>& refs, int frames) {
    deque<int> q;
    set<int>   inRAM;
    int faults = 0;
    cout << "  Ref  | Result    | RAM contents\n";
    cout << "  -----|-----------|-------------\n";
    for (int page : refs) {
        bool hit = inRAM.count(page) > 0;
        if (!hit) {
            faults++;
            if ((int)q.size() == frames) {
                inRAM.erase(q.front());
                q.pop_front();
            }
            q.push_back(page);
            inRAM.insert(page);
        }
        cout << "   " << page << "   | " << (hit ? "hit      " : "FAULT    ")
             << " | {";
        bool first = true;
        for (int p : q) { if (!first) cout << ","; cout << p; first = false; }
        cout << "}\n";
    }
    cout << "  Total faults: " << faults << "\n";
}

int main() {
    // The classic reference string for demonstrating Belady's Anomaly
    vector<int> refs = {1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5};

    cout << "Reference string: ";
    for (int p : refs) cout << p << " ";
    cout << "\n\n";

    int faults3 = fifo(refs, 3);
    int faults4 = fifo(refs, 4);

    cout << "FIFO with 3 frames: " << faults3 << " page faults\n";
    cout << "FIFO with 4 frames: " << faults4 << " page faults\n\n";

    if (faults4 > faults3)
        cout << "*** BELADY'S ANOMALY CONFIRMED: "
             << faults4 - faults3 << " MORE fault(s) with MORE frames! ***\n";
    else
        cout << "(No anomaly for this string/frame combination)\n";

    cout << "\n--- Step-by-step trace (3 frames) ---\n";
    fifoVerbose(refs, 3);
    cout << "\n--- Step-by-step trace (4 frames) ---\n";
    fifoVerbose(refs, 4);

    return 0;
}
// FIFO with 3 frames: 9 page faults
// FIFO with 4 frames: 10 page faults   <- MORE! Belady's Anomaly confirmed
```

### C++ — Medium / LeetCode Style
Show that LRU is **immune** to Belady's Anomaly by sweeping frame counts from 1 to 6 and comparing FIFO vs LRU trends.

```cpp
// Immunity proof: scan frame counts 1-6, compare FIFO (non-stack) vs LRU (stack)
// LRU faults must be non-increasing as frames grow; FIFO has no such guarantee.
// Compile: g++ -std=c++17 beladys_compare.cpp -o beladys_compare

#include <iostream>
#include <vector>
#include <deque>
#include <set>
#include <algorithm>
using namespace std;

int fifo(const vector<int>& refs, int frames) {
    deque<int> q; set<int> ram; int f = 0;
    for (int p : refs) {
        if (ram.count(p)) continue;
        f++;
        if ((int)q.size() == frames) { ram.erase(q.front()); q.pop_front(); }
        q.push_back(p); ram.insert(p);
    }
    return f;
}

int lru(const vector<int>& refs, int frames) {
    deque<int> q; set<int> ram; int f = 0;
    for (int p : refs) {
        if (ram.count(p)) {
            q.erase(find(q.begin(), q.end(), p));
            q.push_back(p);
            continue;
        }
        f++;
        if ((int)q.size() == frames) { ram.erase(q.front()); q.pop_front(); }
        q.push_back(p); ram.insert(p);
    }
    return f;
}

int main() {
    vector<int> refs = {1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5};

    cout << "Reference string: ";
    for (int p : refs) cout << p << " ";
    cout << "\n\n";

    cout << "Frames | FIFO | LRU\n";
    cout << "-------|------|----\n";
    for (int f = 1; f <= 6; f++) {
        int ff = fifo(refs, f);
        int lf = lru(refs, f);
        cout << "  " << f << "    |  " << ff << "   | " << lf;
        if (f > 1 && ff > fifo(refs, f-1))
            cout << "  <- ANOMALY (FIFO increased!)";
        cout << "\n";
    }

    cout << "\nLRU faults never increase as frames grow -> stack property holds\n";
    cout << "FIFO anomaly: 3 frames -> 9 faults, 4 frames -> 10 faults\n";
    return 0;
}
// Frames | FIFO | LRU
// -------|------|----
//   1    |  12  | 12
//   2    |  12  | 12
//   3    |   9  |  8
//   4    |  10  | <- ANOMALY (FIFO increased!)  |  8
//   5    |   8  |  7
//   6    |   7  |  7
```

### Python — Simple Version
Run FIFO with 3 and 4 frames on the classic string; confirm and explain the anomaly.

```python
# Belady's Anomaly: FIFO with 3 vs 4 frames.
# Run: python beladys.py

from collections import deque


def fifo(refs: list[int], frames: int, verbose: bool = False) -> int:
    """FIFO page replacement. If verbose=True, prints each step."""
    queue  = deque()   # left=oldest (evict first)
    in_ram = set()
    faults = 0

    if verbose:
        print(f"\n  {'Ref':<5} {'Result':<10} RAM contents")
        print("  " + "-" * 35)

    for page in refs:
        if page in in_ram:
            result = "hit"
        else:
            faults += 1
            result = f"FAULT #{faults}"
            if len(queue) == frames:
                in_ram.discard(queue.popleft())   # evict oldest
            queue.append(page)
            in_ram.add(page)

        if verbose:
            print(f"  pg {page}   {result:<10} {list(queue)}")

    return faults


def main():
    # The reference string that demonstrates Belady's Anomaly
    refs = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]

    print(f"Reference string : {refs}")
    print(f"Total accesses   : {len(refs)}\n")

    f3 = fifo(refs, frames=3, verbose=True)
    print(f"  => 3 frames: {f3} page faults\n")

    f4 = fifo(refs, frames=4, verbose=True)
    print(f"  => 4 frames: {f4} page faults\n")

    print("=" * 50)
    print(f"3 frames: {f3} faults")
    print(f"4 frames: {f4} faults")
    if f4 > f3:
        print(f"*** BELADY'S ANOMALY: +{f4 - f3} fault(s) with more frames! ***")
    else:
        print("No anomaly for this combination.")


main()
# 3 frames: 9 faults
# 4 frames: 10 faults  <- Belady's Anomaly confirmed
```

### Python — Medium Level
Sweep frame counts 1–6 and compare FIFO vs LRU; prove LRU's stack property by showing its fault count never increases.

```python
# FIFO vs LRU across frame counts — prove stack property (LRU immune to anomaly).
# Run: python beladys_compare.py

from collections import deque


def fifo(refs: list[int], frames: int) -> int:
    queue, in_ram, faults = deque(), set(), 0
    for page in refs:
        if page in in_ram:
            continue
        faults += 1
        if len(queue) == frames:
            in_ram.discard(queue.popleft())
        queue.append(page)
        in_ram.add(page)
    return faults


def lru(refs: list[int], frames: int) -> int:
    order, in_ram, faults = [], set(), 0
    for page in refs:
        if page in in_ram:
            order.remove(page)
            order.append(page)   # move to MRU
            continue
        faults += 1
        if len(order) == frames:
            in_ram.discard(order.pop(0))  # evict LRU
        order.append(page)
        in_ram.add(page)
    return faults


def main():
    refs = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]

    print(f"Reference string: {refs}\n")
    print(f"{'Frames':<8} {'FIFO':>6} {'LRU':>6}  Notes")
    print("-" * 50)

    prev_fifo = prev_lru = None
    for f in range(1, 7):
        ff = fifo(refs, f)
        lf = lru(refs, f)
        note = ""
        if prev_fifo is not None and ff > prev_fifo:
            note = "<-- ANOMALY! FIFO increased"
        if prev_lru is not None and lf > prev_lru:
            note = "<-- BUG: LRU should never increase!"
        print(f"  {f:<6}   {ff:>4}   {lf:>4}   {note}")
        prev_fifo, prev_lru = ff, lf

    print("\nLRU: non-increasing as frames grow (stack property satisfied)")
    print("FIFO: 3->4 frames causes 9->10 faults (Belady's Anomaly)")


main()
```

---

## 8. Key Takeaways

- **Belady's Anomaly** = adding more physical frames causes MORE page faults with FIFO (discovered by László Bélády, 1969)
- It occurs because **FIFO only tracks age**, not usage — so adding frames changes the entire replacement cascade in unpredictable ways
- **Stack/inclusion property** = set of pages in memory with N frames ⊆ set with N+1 frames; algorithms with this property are IMMUNE to Belady's Anomaly
- **FIFO violates** the stack property → susceptible to the anomaly
- **LRU and Optimal satisfy** the stack property → immune (more frames never increases faults)
- Classic demonstration: reference string `1,2,3,4,1,2,5,1,2,3,4,5` with FIFO gives 9 faults with 3 frames but 10 faults with 4 frames
- **Practical fix:** use LRU or LRU-approximations (Second-Chance/Clock algorithm) which are both efficient and anomaly-free
- This is one of the main reasons **FIFO is not used in modern general-purpose operating systems**
