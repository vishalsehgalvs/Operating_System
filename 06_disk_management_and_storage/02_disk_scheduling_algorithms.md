# Disk Scheduling Algorithms: FCFS, SSTF, SCAN, C-SCAN, LOOK, C-LOOK

> Disk scheduling algorithms decide the ORDER to serve pending I/O requests so the disk head moves as efficiently as possible — FCFS is fair but slow, SSTF is fast but can starve, SCAN sweeps like an elevator for balance, and C-LOOK is the most practical modern choice.

---

## Table of Contents

1. [Setup: Common Example Queue](#1-setup-common-example-queue)
2. [FCFS — First-Come, First-Served](#2-fcfs--first-come-first-served)
3. [SSTF — Shortest Seek Time First](#3-sstf--shortest-seek-time-first)
4. [SCAN — Elevator Algorithm](#4-scan--elevator-algorithm)
5. [C-SCAN — Circular SCAN](#5-c-scan--circular-scan)
6. [LOOK Algorithm](#6-look-algorithm)
7. [C-LOOK Algorithm](#7-c-look-algorithm)
8. [Full Comparison](#8-full-comparison)
9. [Choosing the Right Algorithm](#9-choosing-the-right-algorithm)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Setup: Common Example Queue

All algorithms below use the same scenario so results can be compared directly:

```
  Disk size:          200 tracks (0 to 199)
  Current head:       50
  Direction (initial): moving toward higher tracks (right / toward 199)

  Pending request queue: 95, 180, 34, 119, 11, 123
```

**Goal:** Minimize total head movement (total tracks traversed).

---

## 2. FCFS — First-Come, First-Served

**Rule:** Serve requests in the exact order they arrived — no reordering.

```
  Queue arrival order: 95, 180, 34, 119, 11, 123

  Head moves:
  50 → 95  │ distance: 45
  95 → 180 │ distance: 85
  180 → 34 │ distance: 146  ← huge jump back!
  34 → 119 │ distance: 85
  119 → 11 │ distance: 108  ← another big jump back!
  11 → 123 │ distance: 112
  ────────────────────────
  Total head movement: 581 tracks
```

**Head path visualization:**

```
  0   11  34  50  95  119 123 180 199
  │    │   │   │   │   │   │   │
       ←───←───CURRENT                  (start at 50)
            →───────→                   (to 95)
                    →───────────→       (to 180)
       ←───────────────────────         (to 34)
                →───────────────        (to 119)
       ←────────────────                (to 11)
                    ────────────→       (to 123)
```

### Properties

| Property            | Value                                |
| ------------------- | ------------------------------------ |
| Total head movement | 581 tracks (worst here)              |
| Starvation          | None — every request served in order |
| Fairness            | Perfect                              |
| Efficiency          | Poor                                 |

**When to use:** Low-traffic systems, or when request ordering has strict priority (guaranteed order of service matters more than performance).

---

## 3. SSTF — Shortest Seek Time First

**Rule:** Always pick the request CLOSEST to the current head position — greedy nearest-neighbor.

```
  From 50: Closest in queue = 34 (distance 16) ← pick it
  From 34: Remaining {95, 180, 119, 11, 123}. Closest = 11 (distance 23) ← pick it
  From 11: Remaining {95, 180, 119, 123}. Closest = 95 (distance 84) ← pick it
  From 95: Remaining {180, 119, 123}. Closest = 119 (distance 24) ← pick it
  From 119: Remaining {180, 123}. Closest = 123 (distance 4) ← pick it
  From 123: Remaining {180}. Only 180 left ← pick it

  Head path: 50 → 34 → 11 → 95 → 119 → 123 → 180
  Distances:    16 + 23 + 84 + 24 +  4  +  57  = 208 tracks
```

**Head path visualization:**

```
  0   11  34  50  95  119 123        180 199
       ←───←───│                         (50→34→11)
               ────────────────────→     (11→95→119→123→180)
```

### Starvation Example

```
  Suppose head is at track 50 and requests keep arriving at tracks 45–55.
  SSTF always picks them (nearest) before getting to track 180.

  Track 180 request: waits indefinitely → STARVATION!

  This is the key weakness of SSTF.
```

| Property            | Value                             |
| ------------------- | --------------------------------- |
| Total head movement | 208 tracks                        |
| Starvation          | Yes — distant requests may starve |
| Fairness            | Poor                              |
| Efficiency          | Very good throughput              |

---

## 4. SCAN — Elevator Algorithm

**Rule:** Head moves in ONE direction, serves all requests it encounters. When it reaches the disk END, it reverses and serves requests in the other direction.

```
  Head at 50, moving toward 0 (left)

  Going left (serving requests on the way):
  50 → 34  │ distance: 16
  34 → 11  │ distance: 23
  11 → 0   │ distance: 11  (reaches disk end at 0, reverses)

  Now going right (serving remaining requests):
  0 → 95   │ distance: 95
  95 → 119 │ distance: 24
  119 → 123│ distance: 4
  123 → 180│ distance: 57
  ────────────────────────
  Total: 16+23+11+95+24+4+57 = 230 tracks

  Service order: 34, 11, [reversal at 0], 95, 119, 123, 180
```

**Elevator analogy:**

```
  You're in an elevator on floor 5 (track 50).
  Requests at floors: 3, 1, 9, 11, 11.3, 18

  Elevator goes DOWN: stops at 3, 1, then hits LOBBY (floor 0)
  Reverses → goes UP: stops at 9, 11, 11.3, 18

  Every request gets served. No floor waits forever.
```

| Property            | Value                                                     |
| ------------------- | --------------------------------------------------------- |
| Total head movement | 230 tracks                                                |
| Starvation          | No — all requests served eventually                       |
| Edge bias           | Yes — requests near disk edges get slightly shorter waits |
| Wasteful travel     | Travels to disk end even if no requests there             |

---

## 5. C-SCAN — Circular SCAN

**Rule:** Like SCAN, but when the head reaches the END, it **jumps back to the beginning** without serving any requests on the return. Only services requests in ONE direction.

```
  Head at 50, moving toward 199 (right)

  Going right (serving):
  50 → 95  │ distance: 45
  95 → 119 │ distance: 24
  119 → 123│ distance: 4
  123 → 180│ distance: 57
  180 → 199│ distance: 19  (reaches end, NO service on return)

  Jump back to beginning:
  199 → 0  │ (not counted as service movement, but physically happens)

  Now going right again (serving):
  0 → 11   │ distance: 11
  11 → 34  │ distance: 23
  ────────────────────────
  Total: 45+24+4+57+19+11+23 = 183 tracks
  (The 199→0 jump: typically counted separately or as 199 additional tracks)
```

**Benefit over SCAN:**

```
  SCAN: requests at track 5 might wait for the head to travel all the way to 199,
        reverse, then come back — that's a LONG wait.

  C-SCAN: after reaching 199, head ALWAYS jumps back to 0 and sweeps right again.
          Maximum wait = time for ONE complete sweep from 0 to 199.
          All requests have MORE UNIFORM wait times.
```

| Property             | Value                                                    |
| -------------------- | -------------------------------------------------------- |
| Total head movement  | 183 tracks (+ 199 for return jump)                       |
| Starvation           | No                                                       |
| Wait time uniformity | Better than SCAN — all locations treated equally         |
| Wasteful travel      | Travels to disk end AND 199→0 jump even with no requests |

---

## 6. LOOK Algorithm

**Rule:** Like SCAN, but the head only goes as far as the **LAST REQUEST** in each direction — it doesn't travel to the physical disk boundary.

```
  Head at 50, moving toward 0 (left)

  Last request in left direction = 11 (not 0)

  Going left (to last request):
  50 → 34  │ distance: 16
  34 → 11  │ distance: 23
  Reverse at 11 (NOT at 0!)

  Going right:
  11 → 95  │ distance: 84
  95 → 119 │ distance: 24
  119 → 123│ distance: 4
  123 → 180│ distance: 57
  ────────────────────────
  Total: 16+23+84+24+4+57 = 208 tracks

  SCAN total was 230. LOOK saves 22 tracks (not going 11→0→11 unnecessarily).
```

**Key difference from SCAN:** No wasted trip to disk boundaries.

---

## 7. C-LOOK Algorithm

**Rule:** Like C-SCAN, but after reaching the last request in the current direction, jumps back to the **FIRST PENDING REQUEST** (not to track 0).

```
  Head at 50, moving toward 199 (right)

  Going right (to last request in this direction = 180):
  50 → 95  │ distance: 45
  95 → 119 │ distance: 24
  119 → 123│ distance: 4
  123 → 180│ distance: 57

  Jump back to FIRST pending request (= 11, not 0):
  180 → 11 │ distance: 169  (jump, not service)

  Going right from 11:
  11 → 34  │ distance: 23
  ────────────────────────
  Total service movement: 45+24+4+57+23 = 153 tracks  (+169 for the jump)
```

**Why C-LOOK is used in practice:**

- No wasted travel to physical disk ends
- Uniform wait times (like C-SCAN)
- Most efficient real-world balance

---

## 8. Full Comparison

All algorithms on same queue (head=50, requests: 95, 180, 34, 119, 11, 123):

| Algorithm  | Total Head Movement | Starvation | Fairness | Notes                                        |
| ---------- | ------------------- | ---------- | -------- | -------------------------------------------- |
| **FCFS**   | **581 tracks**      | No         | Perfect  | Simple; poor perf                            |
| **SSTF**   | **208 tracks**      | **Yes**    | Poor     | Best throughput; unfair                      |
| **SCAN**   | **230 tracks**      | No         | Good     | Travels to disk ends                         |
| **C-SCAN** | **183 tracks**      | No         | Best     | Uniform waits; end travel                    |
| **LOOK**   | **208 tracks**      | No         | Good     | Like SCAN, no end travel                     |
| **C-LOOK** | **153+ tracks**     | No         | Best     | Like C-SCAN, no end travel; practical choice |

```
  Performance ranking (seek efficiency):
  C-LOOK ≈ C-SCAN > SSTF ≈ LOOK > SCAN >> FCFS

  Fairness ranking:
  FCFS > C-SCAN ≈ C-LOOK > SCAN ≈ LOOK > SSTF

  Modern real-world choice: C-LOOK (best balance)
```

---

## 9. Choosing the Right Algorithm

| Situation                                     | Recommended Algorithm | Why                                           |
| --------------------------------------------- | --------------------- | --------------------------------------------- |
| Mixed workload, general server                | C-LOOK or LOOK        | Balanced performance + no starvation          |
| Maximum throughput, starvation acceptable     | SSTF                  | Greedy minimum seek distance                  |
| Strict fairness (all requests equal priority) | FCFS                  | FIFO guarantees                               |
| Real-time workloads (bounded response time)   | Deadline scheduler    | Combines SSTF + timeout to prevent starvation |
| Database servers with heavy random I/O        | C-LOOK                | Uniform service, no starvation                |

**Modern OS schedulers:**

| OS                  | HDD Scheduler                            | SSD/NVMe Scheduler  |
| ------------------- | ---------------------------------------- | ------------------- |
| Linux (kernel 5.0+) | BFQ (Budget Fair Queuing) or MQ-Deadline | None or MQ-Deadline |
| Windows             | I/O Scheduler (internal)                 | NVMe-native queuing |
| macOS               | IOKit (internal)                         | NVMe-native queuing |

Linux's **BFQ (Budget Fair Queuing)** provides both low latency and fairness for interactive workloads — it's essentially a C-LOOK variant with additional fairness mechanisms. For SSDs, Linux often sets the scheduler to "none" since the NVMe controller handles its own native command queuing (NCQ) optimally.

---

## 9. Code Examples

> Working code that demonstrates all 5 disk scheduling algorithms in practice.

### C++ — Simple Version

FCFS disk scheduling with an ASCII head-movement trace showing the path of the disk head.

```cpp
// FCFS Disk Scheduling with ASCII Head Movement Trace
// Serves requests in the exact order they arrived — simplest algorithm.
// Shows a visual trace of where the disk head travels.

#include <iostream>
#include <vector>
#include <cmath>

int main() {
    int initial_head = 53;
    std::vector<int> requests = {98, 183, 37, 122, 14, 124, 65, 67};

    std::cout << "FCFS Disk Scheduling\n";
    std::cout << "Initial head: " << initial_head << "\n";
    std::cout << "Requests: ";
    for (int r : requests) std::cout << r << " ";
    std::cout << "\n\n";

    // ASCII ruler: cylinder 0..199 mapped to 40-char width
    const int SCALE = 200, WIDTH = 40;
    auto col = [&](int cyl) { return (int)((double)cyl / SCALE * WIDTH); };

    std::cout << "Head movement trace (0 to 199):\n";
    std::cout << "0" << std::string(WIDTH - 2, ' ') << "199\n";
    std::cout << "|" << std::string(WIDTH - 2, '-') << "|\n";

    // Print starting position
    std::string start_row(WIDTH, ' ');
    start_row[col(initial_head)] = 'H';
    std::cout << start_row << "  <- start (" << initial_head << ")\n";

    int total_seek = 0;
    int current = initial_head;

    for (int req : requests) {
        int distance = std::abs(req - current);
        total_seek += distance;

        // Draw movement range with '=' and destination with '*'
        std::string row(WIDTH, ' ');
        int from_col = col(current), to_col = col(req);
        if (from_col > to_col) std::swap(from_col, to_col);
        for (int i = from_col; i <= to_col; i++) row[i] = '=';
        row[col(req)] = '*';

        std::cout << row << "  " << current << " -> " << req
                  << "  (+" << distance << ")\n";
        current = req;
    }

    std::cout << "\nTotal head movement: " << total_seek << " cylinders\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style

All 5 disk scheduling algorithms (FCFS, SSTF, SCAN, C-SCAN, LOOK, C-LOOK) with total movement comparison.

```cpp
// All 5 Disk Scheduling Algorithms — Comparison
// Input: initial head position + list of cylinder requests
// Output: service order and total cylinders traveled for each algorithm

#include <iostream>
#include <vector>
#include <algorithm>
#include <cmath>

// ── FCFS: serve in arrival order ────────────────────────────────────────────
int fcfs(int head, std::vector<int> reqs) {
    int total = 0, cur = head;
    std::cout << "  FCFS  : " << cur;
    for (int r : reqs) {
        total += std::abs(r - cur); cur = r;
        std::cout << " -> " << r;
    }
    std::cout << "\n";
    return total;
}

// ── SSTF: always pick the nearest unserved request ──────────────────────────
int sstf(int head, std::vector<int> reqs) {
    std::vector<bool> served(reqs.size(), false);
    int total = 0, cur = head;
    std::cout << "  SSTF  : " << cur;
    for (size_t i = 0; i < reqs.size(); i++) {
        int nearest = -1, min_dist = INT_MAX;
        for (size_t j = 0; j < reqs.size(); j++) {
            if (!served[j]) {
                int d = std::abs(reqs[j] - cur);
                if (d < min_dist) { min_dist = d; nearest = (int)j; }
            }
        }
        served[nearest] = true;
        total += min_dist; cur = reqs[nearest];
        std::cout << " -> " << cur;
    }
    std::cout << "\n";
    return total;
}

// ── SCAN: sweep right to max_cyl, then sweep left ───────────────────────────
int scan(int head, std::vector<int> reqs, int max_cyl) {
    std::vector<int> left, right;
    for (int r : reqs) (r < head ? left : right).push_back(r);
    std::sort(left.rbegin(), left.rend());    // serve largest-first on way back
    std::sort(right.begin(), right.end());    // serve smallest-first going right

    int total = 0, cur = head;
    std::cout << "  SCAN  : " << cur;
    for (int r : right) { total += std::abs(r - cur); cur = r; std::cout << " -> " << r; }
    // Travel to disk boundary before reversing
    total += std::abs(max_cyl - cur); cur = max_cyl;
    std::cout << " -> " << max_cyl << "(end)";
    for (int r : left)  { total += std::abs(r - cur); cur = r; std::cout << " -> " << r; }
    std::cout << "\n";
    return total;
}

// ── C-SCAN: sweep right to end, jump to 0, sweep right again ────────────────
int cscan(int head, std::vector<int> reqs, int max_cyl) {
    std::vector<int> left, right;
    for (int r : reqs) (r < head ? left : right).push_back(r);
    std::sort(left.begin(),  left.end());   // serve after the jump (ascending)
    std::sort(right.begin(), right.end());

    int total = 0, cur = head;
    std::cout << "  C-SCAN: " << cur;
    for (int r : right) { total += std::abs(r - cur); cur = r; std::cout << " -> " << r; }
    total += std::abs(max_cyl - cur); cur = max_cyl;
    std::cout << " -> " << max_cyl << "(end)";
    total += max_cyl; cur = 0;   // jump from max_cyl back to 0
    std::cout << " -> 0(jump)";
    for (int r : left)  { total += r - cur; cur = r; std::cout << " -> " << r; }
    std::cout << "\n";
    return total;
}

// ── LOOK: like SCAN but reverses at last request, not disk boundary ──────────
int look(int head, std::vector<int> reqs) {
    std::vector<int> left, right;
    for (int r : reqs) (r < head ? left : right).push_back(r);
    std::sort(left.rbegin(), left.rend());
    std::sort(right.begin(), right.end());

    int total = 0, cur = head;
    std::cout << "  LOOK  : " << cur;
    for (int r : right) { total += std::abs(r - cur); cur = r; std::cout << " -> " << r; }
    for (int r : left)  { total += std::abs(r - cur); cur = r; std::cout << " -> " << r; }
    std::cout << "\n";
    return total;
}

// ── C-LOOK: like C-SCAN but jumps to first pending, not cylinder 0 ───────────
int clook(int head, std::vector<int> reqs) {
    std::vector<int> left, right;
    for (int r : reqs) (r < head ? left : right).push_back(r);
    std::sort(left.begin(),  left.end());
    std::sort(right.begin(), right.end());

    int total = 0, cur = head;
    std::cout << "  C-LOOK: " << cur;
    for (int r : right) { total += std::abs(r - cur); cur = r; std::cout << " -> " << r; }
    if (!left.empty()) {
        total += std::abs(cur - left.front()); cur = left.front();
        std::cout << " -> " << cur << "(jump)";
        for (size_t i = 1; i < left.size(); i++) {
            total += std::abs(left[i] - cur); cur = left[i];
            std::cout << " -> " << cur;
        }
    }
    std::cout << "\n";
    return total;
}

int main() {
    int head = 53;
    std::vector<int> requests = {98, 183, 37, 122, 14, 124, 65, 67};
    int MAX_CYL = 199;

    std::cout << "Disk Scheduling Algorithms Comparison\n";
    std::cout << "Head=" << head << "  Requests: ";
    for (int r : requests) std::cout << r << " ";
    std::cout << "\n" << std::string(65, '=') << "\n";

    std::cout << "Service order:\n";
    int m1 = fcfs (head, requests);
    int m2 = sstf (head, requests);
    int m3 = scan (head, requests, MAX_CYL);
    int m4 = cscan(head, requests, MAX_CYL);
    int m5 = look (head, requests);
    int m6 = clook(head, requests);

    std::cout << "\nTotal head movement (cylinders):\n";
    std::cout << std::string(35, '-') << "\n";
    printf("  FCFS   : %4d\n", m1);
    printf("  SSTF   : %4d\n", m2);
    printf("  SCAN   : %4d  (includes travel to boundary)\n", m3);
    printf("  C-SCAN : %4d  (includes jump from end to 0)\n", m4);
    printf("  LOOK   : %4d\n", m5);
    printf("  C-LOOK : %4d  (includes jump to first pending)\n", m6);
    return 0;
}
```

### Python — Simple Version

SSTF implementation step-by-step, showing which request is chosen at each move.

```python
# SSTF (Shortest Seek Time First) Disk Scheduling — Step-by-Step
# At each step, pick the unserved request closest to the current head position.
# Classic greedy algorithm: minimizes immediate seek but can starve far requests.

def sstf(initial_head, requests):
    """
    SSTF disk scheduling.
    Returns: total head movement (cylinders).
    """
    remaining = list(requests)    # requests not yet served
    current   = initial_head
    total     = 0
    order     = [current]

    print(f"SSTF Disk Scheduling")
    print(f"Initial head : {initial_head}")
    print(f"Requests     : {requests}\n")
    print(f"{'Step':<5} {'From':>5} {'To':>5} {'Dist':>6}  Remaining")
    print("-" * 55)

    step = 1
    while remaining:
        # Greedy choice: nearest cylinder to current head
        nearest  = min(remaining, key=lambda r: abs(r - current))
        distance = abs(nearest - current)

        print(f"{step:<5} {current:>5} {nearest:>5} {distance:>6}  {remaining}")

        total += distance
        remaining.remove(nearest)
        current = nearest
        order.append(current)
        step += 1

    print("-" * 55)
    print(f"Service order  : {order}")
    print(f"Total movement : {total} cylinders")
    return total

# Classic OS textbook example
sstf(initial_head=53,
     requests=[98, 183, 37, 122, 14, 124, 65, 67])
```

### Python — Medium Level

All 5 algorithms implemented and compared, with an ASCII bar chart of total head movement.

```python
# All 5 Disk Scheduling Algorithms — Python Implementation + Comparison
# Each function returns (total_movement, service_order).

def fcfs(head, requests):
    total, cur, order = 0, head, [head]
    for r in requests:
        total += abs(r - cur); cur = r; order.append(cur)
    return total, order

def sstf(head, requests):
    remaining, total, cur, order = list(requests), 0, head, [head]
    while remaining:
        nearest = min(remaining, key=lambda r: abs(r - cur))
        total += abs(nearest - cur); cur = nearest
        remaining.remove(cur); order.append(cur)
    return total, order

def scan(head, requests, max_cyl=199):
    left  = sorted([r for r in requests if r <  head], reverse=True)
    right = sorted([r for r in requests if r >= head])
    total, cur, order = 0, head, [head]
    for r in right: total += abs(r - cur); cur = r; order.append(cur)
    if right:        total += abs(max_cyl - cur); cur = max_cyl; order.append(cur)
    for r in left:  total += abs(r - cur); cur = r; order.append(cur)
    return total, order

def cscan(head, requests, max_cyl=199):
    left  = sorted([r for r in requests if r <  head])
    right = sorted([r for r in requests if r >= head])
    total, cur, order = 0, head, [head]
    for r in right:
        total += abs(r - cur); cur = r; order.append(cur)
    total += abs(max_cyl - cur); cur = max_cyl; order.append(cur)
    total += max_cyl;            cur = 0;       order.append(0)   # jump to 0
    for r in left:
        total += abs(r - cur); cur = r; order.append(cur)
    return total, order

def look(head, requests):
    left  = sorted([r for r in requests if r <  head], reverse=True)
    right = sorted([r for r in requests if r >= head])
    total, cur, order = 0, head, [head]
    for r in right: total += abs(r - cur); cur = r; order.append(cur)
    for r in left:  total += abs(r - cur); cur = r; order.append(cur)
    return total, order

def clook(head, requests):
    left  = sorted([r for r in requests if r <  head])
    right = sorted([r for r in requests if r >= head])
    total, cur, order = 0, head, [head]
    for r in right: total += abs(r - cur); cur = r; order.append(cur)
    if left:
        total += abs(cur - left[0]); cur = left[0]; order.append(cur)  # jump
        for r in left[1:]:
            total += abs(r - cur); cur = r; order.append(cur)
    return total, order

# ── Run comparison ────────────────────────────────────────────────────────────
head     = 53
requests = [98, 183, 37, 122, 14, 124, 65, 67]

algorithms = [
    ("FCFS",   fcfs(head, requests)),
    ("SSTF",   sstf(head, requests)),
    ("SCAN",   scan(head, requests)),
    ("C-SCAN", cscan(head, requests)),
    ("LOOK",   look(head, requests)),
    ("C-LOOK", clook(head, requests)),
]

print(f"Disk Scheduling | Head={head} | Requests={requests}")
print("=" * 65)
print(f"{'Algorithm':<10} {'Movement':>10}  Bar (each block = 10 cylinders)")
print("-" * 65)

for name, (total, order) in algorithms:
    bar = "#" * (total // 10)
    print(f"{name:<10} {total:>7} cyl  {bar}")

print("\nDetailed service order:")
for name, (total, order) in algorithms:
    print(f"  {name:<8}: {' -> '.join(map(str, order))}")
```

---

## 10. Key Takeaways

- **FCFS**: serve in arrival order — perfect fairness, zero optimization, 581 tracks (worst)
- **SSTF**: always pick nearest request — best greedy seek performance (208 tracks), but **can starve** distant requests
- **SCAN** (elevator): sweep direction → hit disk end → reverse → sweep back; no starvation, slight edge bias, wastes travel to boundaries (230 tracks)
- **C-SCAN**: sweep one direction → hit end → **jump to beginning** → sweep again; more **uniform wait times** than SCAN (183 tracks + jump)
- **LOOK**: like SCAN but reverses at last request, not disk boundary — saves unnecessary travel (208 tracks)
- **C-LOOK**: like C-SCAN but jumps to first pending request, not track 0 — best practical balance of performance + uniformity (153 tracks + jump)
- **Starvation**: only SSTF can starve requests; all directional algorithms (SCAN/LOOK variants) are starvation-free
- **Modern OS schedulers** (Linux BFQ, MQ-Deadline) are sophisticated C-LOOK variants with additional fairness and deadline mechanisms
- **For SSDs**: seek time doesn't exist → disk scheduling is about write coalescing and wear leveling, not head movement; often use "none" scheduler
- No algorithm is universally best — choice depends on workload: throughput-first (SSTF) vs fairness-first (SCAN/LOOK) vs balanced (C-LOOK)
