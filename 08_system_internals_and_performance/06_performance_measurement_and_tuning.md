# OS Performance Measurement and Tuning

> **Performance measurement** is collecting data on how efficiently the OS uses CPU, memory, disk, and network; **tuning** is making targeted changes to fix the bottlenecks you find — the workflow is always: measure baseline → identify the single worst bottleneck → make one change → measure again → repeat.

---

## Table of Contents

1. [What is Performance Measurement?](#1-what-is-performance-measurement)
2. [Key Performance Metrics](#2-key-performance-metrics)
3. [CPU Performance Metrics](#3-cpu-performance-metrics)
4. [Memory Performance Metrics](#4-memory-performance-metrics)
5. [Disk Performance Metrics](#5-disk-performance-metrics)
6. [Performance Monitoring Tools](#6-performance-monitoring-tools)
7. [Identifying Bottlenecks](#7-identifying-bottlenecks)
8. [Performance Tuning Techniques](#8-performance-tuning-techniques)
9. [Benchmarking Best Practices](#9-benchmarking-best-practices)
10. [The Tuning Workflow](#10-the-tuning-workflow)
11. [Real-World Example](#11-real-world-example)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What is Performance Measurement?

**Performance measurement** is the process of collecting and analyzing data about how efficiently an OS uses its resources. It answers questions like:

- Is my CPU being utilized properly, or is it idle while processes wait for disk?
- Are processes waiting too long for CPU time?
- Is memory pressure causing excessive paging?

**Analogy:** A doctor checking vital signs before diagnosing a patient. You can't prescribe treatment without knowing what's wrong. Tuning without measurement is like fixing a car blindfolded — changes might make things worse, and you'll never know why.

```
  Measurement → Bottleneck Identification → Targeted Tuning → Measure Again
      ↑                                                              |
      └──────────────────────────────────────────────────────────────┘
                        (iterate until goals met)
```

---

## 2. Key Performance Metrics

| Metric                     | What It Measures                        | Why It Matters                   |
| -------------------------- | --------------------------------------- | -------------------------------- |
| **Throughput**             | Processes/tasks completed per unit time | Overall system productivity      |
| **Turnaround Time**        | Submission → completion time            | User-perceived total job speed   |
| **Waiting Time**           | Time in ready queue before CPU          | Scheduling efficiency            |
| **Response Time**          | Request → first response                | Critical for interactive systems |
| **CPU Utilization**        | % time CPU is actively busy             | CPU efficiency                   |
| **Memory Utilization**     | % RAM in use                            | Memory management effectiveness  |
| **Page Fault Rate**        | Page faults per memory accesses         | Memory pressure indicator        |
| **Disk Throughput / IOPS** | MB/s or ops/second on storage           | I/O subsystem capacity           |
| **I/O Wait**               | % time CPU idle waiting for I/O         | Disk bottleneck indicator        |

**Key insight — metrics conflict:**

- Maximizing throughput (long time slices, batch jobs) hurts response time
- Maximizing responsiveness (short time slices, preemption) reduces throughput
- → OS must trade off based on system type (desktop → responsiveness; server → throughput)

---

## 3. CPU Performance Metrics

### CPU Utilization Formula

$$\text{CPU Utilization} = \frac{\text{Busy Time}}{\text{Total Time}} \times 100$$

**Example:**

```
Total observation time: 60 seconds
CPU idle time:          15 seconds
CPU busy time:          45 seconds

CPU Utilization = (45 / 60) × 100 = 75%
```

| Utilization Range  | Interpretation                                                  |
| ------------------ | --------------------------------------------------------------- |
| < 50%              | Underutilized — may have too much CPU or workload is mostly I/O |
| 70–80%             | Healthy — good balance of utilization with room for bursts      |
| > 90% consistently | Bottleneck — CPU may be the limiting factor                     |
| 100% constantly    | Overloaded — requests queuing, response time degrading          |

### CPU-Bound vs I/O-Bound Processes

```
  CPU-bound process:
  ████████████████████████████████████  (spends most time computing)
  Example: video encoding, compression, scientific simulations
  → Increasing CPU speed / adding cores directly helps

  I/O-bound process:
  ██░░░░░░░░░░░░░██░░░░░░░░░░░░░██
  (short CPU bursts, long I/O waits)
  Example: database query reading from disk, web server handling requests
  → Increasing CPU barely helps; need faster storage or more RAM
```

Identifying your workload type is the first step in knowing where to focus tuning.

---

## 4. Memory Performance Metrics

### Page Fault Rate

$$\text{Page Fault Rate} = \frac{\text{Page Faults}}{\text{Total Memory Accesses}} \times 100$$

**Example:**

```
Total memory accesses: 10,000
Page faults:           500

Page Fault Rate = (500 / 10,000) × 100 = 5%
```

| Page Fault Rate            | Interpretation                                                |
| -------------------------- | ------------------------------------------------------------- |
| < 5%                       | Acceptable — working set fits comfortably in RAM              |
| 5–10%                      | Warning — working set approaching RAM limit                   |
| > 10%                      | Critical — may be heading toward thrashing                    |
| Very high + disk I/O spike | Thrashing — OS spends more time paging than running processes |

### Memory Utilization

- High RAM usage alone isn't bad — Linux uses free RAM as disk cache (shows as "used" but is released on demand)
- **Warning signal:** Check swap usage instead. Heavy swapping = genuine memory shortage

```
  Linux memory breakdown (free -h):

  Total: 16 GB
  Used:  14 GB   ← don't panic! Most may be disk cache
  Buff/Cache: 10 GB  ← this is releasable; not really "used"
  Available:  8 GB   ← what actually matters for new processes
  Swap used:  2 GB   ← THIS is the concern — processes are being paged to disk
```

---

## 5. Disk Performance Metrics

| Metric                | What It Measures                                                           |
| --------------------- | -------------------------------------------------------------------------- |
| **Throughput (MB/s)** | Data volume transferred per second — important for sequential reads/writes |
| **IOPS**              | Random read/write operations per second — important for databases          |
| **Average Seek Time** | Time to position disk head to right track (HDDs only)                      |
| **I/O Wait %**        | Fraction of time CPU is idle waiting for disk operations                   |
| **Disk Queue Length** | Pending I/O operations — high value = disk overloaded                      |

```
  Typical values:
  HDD:          100–200 IOPS,   ~100 MB/s sequential, seek ~5ms
  SATA SSD:   ~50,000 IOPS,   ~500 MB/s sequential,   no seek delay
  NVMe SSD:  ~500,000 IOPS,  ~3,500 MB/s sequential,  no seek delay

  A database doing random reads on HDD vs NVMe can be 1000x faster on NVMe.
```

---

## 6. Performance Monitoring Tools

### Linux / Unix Tools

```bash
# Real-time CPU, memory, process view
top          # basic
htop         # interactive, color, per-CPU bars

# Virtual memory statistics (CPU, memory, swap, I/O, context switches)
vmstat 1 5   # update every 1 second, 5 times

# Per-CPU utilization (great for multicore analysis)
mpstat -P ALL 1

# Disk I/O statistics
iostat -x 1           # extended stats every 1 second

# Process memory consumers
ps aux --sort=-%mem | head -10

# Real-time disk I/O per process
iotop

# Network statistics
netstat -s
ss -s
```

### Windows Tools

| Tool                                   | Purpose                                               |
| -------------------------------------- | ----------------------------------------------------- |
| **Task Manager**                       | Quick overview of CPU/memory/disk/network per process |
| **Resource Monitor**                   | Real-time graphs; per-process breakdown               |
| **Performance Monitor**                | Detailed counters, historical graphs, custom alerts   |
| **Process Explorer**                   | Sysinternals: deep per-process details                |
| **WPA (Windows Performance Analyzer)** | Advanced ETW-based profiling                          |

---

## 7. Identifying Bottlenecks

A **bottleneck** is the single resource limiting overall system performance. Like a narrow pipe segment restricting flow — widening other sections won't help until you fix the narrowest point.

```
  System pipeline:
  CPU → Memory → Disk → Network

  Finding the bottleneck:
  Check CPU utilization:  40%   ← NOT the bottleneck
  Check memory pressure:  low   ← NOT the bottleneck
  Check disk I/O wait:    85%   ← BOTTLENECK!
  → Focus tuning on storage; adding more CPU cores won't help
```

### CPU Bottleneck Signs

- CPU utilization > 90% consistently
- High CPU ready queue (many processes waiting for CPU)
- Low I/O wait (CPU is the slow part, not disk)

### Memory Bottleneck Signs

- High page fault rate (> 10%)
- Active swap usage
- Processes being killed by OOM killer
- `vmstat` shows `si`/`so` (swap in/out) > 0 regularly

### Disk Bottleneck Signs

- High `%iowait` in `top`/`vmstat`
- Disk utilization near 100% (`iostat -x`)
- Long disk queue lengths
- Low CPU utilization despite slow system (CPU waiting for disk)

---

## 8. Performance Tuning Techniques

### CPU Tuning

```bash
# Lower priority of background process (nice value -20 to +19; lower = higher priority)
nice -n 10 ./background_batch_job    # runs with lower priority

# Raise priority of important process
nice -n -5 ./important_service       # higher priority (requires sudo)

# Change priority of already running process (PID 1234)
renice -n 10 -p 1234

# Pin process to specific CPUs (reduce cache thrashing in NUMA systems)
taskset -c 0,1 ./my_application      # run only on CPUs 0 and 1

# Change scheduling policy to real-time
chrt -f 50 ./realtime_process        # SCHED_FIFO, priority 50
```

**Scheduling classes:**

| Class                                | Use Case                               | Priority |
| ------------------------------------ | -------------------------------------- | -------- |
| Real-time (`SCHED_FIFO`, `SCHED_RR`) | Time-critical (audio, control systems) | Highest  |
| Interactive                          | Desktop apps, web servers              | High     |
| Normal (SCHED_OTHER)                 | General-purpose processes              | Normal   |
| Batch                                | Background jobs, compilations          | Low      |
| Idle                                 | Only runs when nothing else needs CPU  | Lowest   |

### Memory Tuning

```bash
# Check current swappiness (0 = avoid swap; 100 = swap aggressively)
cat /proc/sys/vm/swappiness           # default: 60

# Set swappiness lower — keep more data in RAM (good for servers with enough RAM)
sudo sysctl vm.swappiness=10

# Make permanent across reboots
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf

# Tune transparent huge pages (can help databases, scientific workloads)
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

**Rule of thumb for swappiness:**

- Desktop with plenty of RAM: `vm.swappiness=10` (prefer RAM, minimal swap)
- Server under memory pressure: `vm.swappiness=60` (allow more swapping)
- Database server: `vm.swappiness=1` (databases manage their own cache; OS swapping hurts them)

### Disk Tuning

```bash
# Check current I/O scheduler
cat /sys/block/sda/queue/scheduler
# Output: [mq-deadline] kyber bfq none

# Change I/O scheduler
echo deadline | sudo tee /sys/block/sda/queue/scheduler   # good for databases
echo noop     | sudo tee /sys/block/sda/queue/scheduler   # good for SSDs
echo cfq      | sudo tee /sys/block/sda/queue/scheduler   # fair for desktop
```

| I/O Scheduler                     | Best For                     | Why                                                     |
| --------------------------------- | ---------------------------- | ------------------------------------------------------- |
| **noop / none**                   | SSDs, NVMe                   | SSDs don't need seek optimization; no ordering overhead |
| **deadline**                      | Databases, latency-sensitive | Bounds maximum wait time for each request               |
| **CFQ (Completely Fair Queuing)** | Desktop multi-user systems   | Fair time slices per process                            |
| **BFQ**                           | Interactive desktop          | Low latency for interactive I/O                         |

---

## 9. Benchmarking Best Practices

### Establishing a Baseline

```
  Before ANY tuning:
  1. Record: CPU utilization, memory usage, page fault rate, disk IOPS, response time
  2. Run the same workload multiple times
  3. Average the results (remove outliers)
  4. Save as "baseline"

  After tuning:
  Run identical workload → compare to baseline → quantify improvement
  If no improvement (or worse): REVERT the change
```

### Rules for Valid Benchmarks

| Rule                           | Why It Matters                                                                   |
| ------------------------------ | -------------------------------------------------------------------------------- |
| **Change one thing at a time** | Multiple simultaneous changes make it impossible to know what helped             |
| **Run long enough**            | Short runs miss steady-state behavior; run until system stabilizes               |
| **Isolate the system**         | Other processes add noise; close unnecessary apps during benchmarking            |
| **Warm up caches**             | First run is slower (cold cache); run several times and use steady-state numbers |
| **Simulate real workload**     | Synthetic benchmarks may not reflect actual use patterns                         |

### Common Benchmarking Mistakes

```
  ✗ Testing with other background processes consuming resources
  ✗ Running tests for only a few seconds
  ✗ Comparing first-run results (cold cache) to later runs (warm cache)
  ✗ Changing scheduler AND swappiness AND RAID config simultaneously
  ✗ Benchmarking on a VM (hypervisor adds variability)
```

---

## 10. The Tuning Workflow

```
  Step 1: Measure baseline
          → Collect CPU, memory, disk metrics under normal load

  Step 2: Identify bottleneck
          → Which resource is limiting? (CPU/memory/disk/network)

  Step 3: Research solutions
          → What can be tuned for this type of bottleneck?

  Step 4: Make ONE change
          → Adjust a single parameter

  Step 5: Measure again
          → Run same workload, collect same metrics

  Step 6: Compare to baseline
          → Improved? Keep the change.
          → Same or worse? REVERT the change.

  Step 7: Document
          → Record what you changed and the outcome

  Step 8: Iterate
          → Go back to Step 2 (new bottleneck may have emerged)
```

**When to stop tuning:**

- System meets performance requirements (SLA, user satisfaction)
- Remaining gains require hardware upgrades (more RAM, faster SSD)
- Further tuning costs more engineering time than hardware would

---

## 11. Real-World Example

**Scenario:** Web server with slow response times during peak hours.

```
  Initial Investigation (vmstat, iostat, top):
  ─────────────────────────────────────────────
  CPU Utilization:   40%    ← not the bottleneck
  Memory:            65%    ← fine
  Disk Utilization:  95%    ← BOTTLENECK!
  Avg I/O Wait:      300ms  ← terrible
  Disk Queue Length: 15     ← very high
  Response Time:     2500ms ← unacceptable

  Diagnosis: Database making many random reads → disk I/O bottleneck

  Tuning Actions (one at a time):
  1. Change I/O scheduler: CFQ → deadline
  2. Increase database buffer pool: 1 GB → 4 GB (more hot data cached in RAM)
  3. Move hot database tables to SSD
  4. Enable write combining (batch small writes together)

  After Tuning:
  ─────────────────────────────────────────────
  CPU Utilization:   55%    ← went UP (CPU no longer waiting for disk)
  Memory:            80%    ← went up (larger buffer pool)
  Disk Utilization:  60%    ← much better
  Avg I/O Wait:      50ms   ← 6x improvement
  Disk Queue Length: 3      ← healthy
  Response Time:     400ms  ← 84% improvement!
```

**Key lesson:** CPU utilization actually _increased_ after tuning — this is expected and good. Before tuning, CPU was idle 60% of the time waiting for disk. After tuning, CPU is doing more useful work because disk no longer bottlenecks it.

---

## 11. Code Examples

> Working code that demonstrates performance measurement and tuning in practice.

### C++ — Simple Version

Use `std::chrono` to time and compare operations — array vs linked list sequential access, binary search vs linear search, string copy vs move — shows how to profile code before optimizing.

```cpp
// Measure and compare the performance of different operations using std::chrono.
// Key rule of performance tuning: MEASURE FIRST, then optimize.
// Covers: array vs linked list, sorted vs unsorted search, copy vs move semantics.

#include <iostream>
#include <vector>
#include <list>
#include <algorithm>
#include <chrono>
#include <numeric>
#include <iomanip>
#include <string>

// ── Timing helper ─────────────────────────────────────────────────────────────
// Runs fn() reps times and returns the average elapsed time in microseconds.
template<typename Fn>
double time_us(Fn fn, int reps = 5) {
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < reps; ++i) fn();
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::micro>(end - start).count() / reps;
}

void print_row(const std::string& label, double us) {
    std::cout << "  " << std::left << std::setw(45) << label
              << std::right << std::setw(10) << std::fixed << std::setprecision(1)
              << us << " µs\n";
}

int main() {
    std::cout << "=== Performance Measurement ===\n\n";
    const int N = 100'000;

    // ── 1. Array vs Linked List sequential access ─────────────────────────────
    // vector: elements are contiguous in memory → CPU cache loads 16 ints per cache line
    // list:   each node is a separate heap allocation → every element = cache miss
    std::cout << "1. Sequential sum over N=" << N << " elements:\n";

    std::vector<int> arr(N); std::iota(arr.begin(), arr.end(), 0);
    std::list<int>   lst(arr.begin(), arr.end());

    double t_arr = time_us([&]{
        volatile long s = 0; for (int x : arr) s += x;
    }, 20);
    double t_lst = time_us([&]{
        volatile long s = 0; for (int x : lst) s += x;
    }, 20);

    print_row("vector<int> sum  (cache-friendly, contiguous)", t_arr);
    print_row("list<int> sum    (cache-unfriendly, scattered)", t_lst);
    std::cout << "  → list is " << std::setprecision(0) << t_lst / t_arr
              << "x slower (each node = separate cache miss)\n\n";

    // ── 2. Binary search (sorted) vs linear search (unsorted) ─────────────────
    // Binary search: O(log N) = 17 comparisons for N=100000
    // Linear search: O(N) = up to 100000 comparisons
    std::cout << "2. Search (N=" << N << "):\n";

    std::vector<int> sorted_v = arr;   // already sorted 0..N-1
    std::vector<int> unsorted_v = arr;
    std::shuffle(unsorted_v.begin(), unsorted_v.end(), std::mt19937{42});

    double t_bin = time_us([&]{
        volatile bool f = std::binary_search(sorted_v.begin(), sorted_v.end(), N/2);
    }, 10000);
    double t_lin = time_us([&]{
        volatile bool f = (std::find(unsorted_v.begin(), unsorted_v.end(), N/2) != unsorted_v.end());
    }, 10000);

    print_row("binary_search  (sorted,   O(log N))", t_bin);
    print_row("linear search  (unsorted, O(N))",     t_lin);
    std::cout << "  → linear is " << std::setprecision(0) << t_lin / t_bin
              << "x slower (O(N) vs O(log N))\n\n";

    // ── 3. String copy vs move semantics ──────────────────────────────────────
    // copy: allocates new heap buffer + copies all bytes → O(N)
    // move: just swaps the internal pointer → O(1) — no allocation, no copy
    std::cout << "3. String copy vs move (10 KB string):\n";
    std::string big(10'000, 'X');

    double t_copy = time_us([&]{
        std::string c = big;         // deep copy: new allocation + memcpy
        volatile char x = c[0];
    }, 50000);
    double t_move = time_us([&]{
        std::string tmp = big;
        std::string m = std::move(tmp);   // O(1): swap internal pointers only
        volatile char x = m[0];
    }, 50000);

    print_row("std::string copy  (new alloc + memcpy, O(N))", t_copy);
    print_row("std::move         (pointer swap only, O(1))",   t_move);
    std::cout << "  → copy is " << std::setprecision(1) << t_copy / t_move
              << "x slower than move\n\n";

    std::cout << "Key takeaway: always measure before optimizing — results may surprise you!\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style

Function call profiler that tracks call counts and durations per function, plus a cache-friendly vs cache-unfriendly matrix traversal benchmark that demonstrates why memory access patterns matter.

```cpp
// Part 1: Lightweight function profiler — wraps any callable, records timing per function name.
//         Reports sorted by total time (hottest functions first) — like gprof or perf.
// Part 2: Cache access pattern benchmark — row-major vs column-major matrix traversal.
//         Demonstrates why memory layout matters as much as algorithm complexity.

#include <iostream>
#include <string>
#include <unordered_map>
#include <chrono>
#include <functional>
#include <vector>
#include <numeric>
#include <iomanip>
#include <algorithm>

// ── PART 1: Function Profiler ─────────────────────────────────────────────────

struct Stats {
    long long calls    = 0;
    double    total_us = 0.0;
    double    min_us   = 1e18;
    double    max_us   = 0.0;
    double    avg_us() const { return calls ? total_us / calls : 0.0; }
};

class Profiler {
    std::unordered_map<std::string, Stats> data;

public:
    // Profile any callable — call this in place of the direct function call
    template<typename Fn>
    auto measure(const std::string& name, Fn fn) {
        auto t0 = std::chrono::high_resolution_clock::now();
        auto result = fn();
        double us = std::chrono::duration<double, std::micro>(
            std::chrono::high_resolution_clock::now() - t0).count();

        auto& s = data[name];
        s.calls++;
        s.total_us += us;
        s.min_us    = std::min(s.min_us, us);
        s.max_us    = std::max(s.max_us, us);
        return result;
    }

    void report() {
        std::cout << "\n=== Profiler Report (sorted by total time) ===\n";
        std::cout << std::left  << std::setw(28) << "Function"
                  << std::right << std::setw(8)  << "Calls"
                  << std::setw(12) << "Total µs"
                  << std::setw(10) << "Avg µs"
                  << std::setw(9)  << "Min µs"
                  << std::setw(9)  << "Max µs" << "\n";
        std::cout << std::string(78, '-') << "\n";

        std::vector<std::pair<std::string, Stats>> sorted(data.begin(), data.end());
        std::sort(sorted.begin(), sorted.end(),
            [](const auto& a, const auto& b){ return a.second.total_us > b.second.total_us; });

        for (const auto& [name, s] : sorted) {
            std::cout << std::left  << std::setw(28) << name
                      << std::right << std::setw(8)  << s.calls
                      << std::setw(12) << std::fixed << std::setprecision(1) << s.total_us
                      << std::setw(10) << s.avg_us()
                      << std::setw(9)  << s.min_us
                      << std::setw(9)  << s.max_us << "\n";
        }
    }
};

// ── PART 2: Cache-Friendly vs Cache-Unfriendly Matrix Traversal ───────────────
// A C++ 2D array is stored in ROW-MAJOR order: arr[0][0], arr[0][1], ..., arr[0][N-1], arr[1][0]...
// Row-major access: reads consecutive memory locations → hits cache line each time → fast
// Column-major access: jumps N elements between reads → every access is a fresh cache line → slow

static const int N = 1000;

double sum_row_major(int mat[N][N]) {
    double s = 0;
    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            s += mat[i][j];   // consecutive memory → cache hit every 16 elements
    return s;
}

double sum_col_major(int mat[N][N]) {
    double s = 0;
    for (int j = 0; j < N; ++j)
        for (int i = 0; i < N; ++i)
            s += mat[i][j];   // jumps 1000 ints = 4000 bytes between reads → cache miss
    return s;
}

int main() {
    // ── Profiler demo ─────────────────────────────────────────────────────────
    Profiler prof;
    std::vector<int> data(1000);
    std::iota(data.begin(), data.end(), 0);

    for (int i = 0; i < 1000; ++i) {
        prof.measure("linear_search", [&]{
            return (int)(std::find(data.begin(), data.end(), 500) != data.end());
        });
        prof.measure("sort_copy", [&]{
            auto v = data; std::sort(v.begin(), v.end()); return (int)v.size();
        });
    }
    for (int i = 0; i < 5000; ++i) {
        prof.measure("binary_search", [&]{
            return (int)std::binary_search(data.begin(), data.end(), 500);
        });
    }
    prof.report();

    // ── Cache benchmark ────────────────────────────────────────────────────────
    static int mat[N][N];
    for (int i = 0; i < N; ++i) for (int j = 0; j < N; ++j) mat[i][j] = i*N+j;

    auto time_fn = [](auto fn, int reps=5) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile double x = 0;
        for (int i = 0; i < reps; ++i) x += fn();
        return std::chrono::duration<double, std::micro>(
            std::chrono::high_resolution_clock::now() - t0).count() / reps;
    };

    double t_row = time_fn([&]{ return sum_row_major(mat); });
    double t_col = time_fn([&]{ return sum_col_major(mat); });

    std::cout << "\n=== Cache Access Patterns (" << N << "x" << N << " matrix) ===\n";
    std::cout << std::fixed << std::setprecision(0);
    std::cout << "  Row-major (cache-friendly):    " << t_row << " µs\n";
    std::cout << "  Column-major (cache-hostile):  " << t_col << " µs\n";
    std::cout << "  Column-major is " << std::setprecision(1) << t_col / t_row << "x slower\n";
    std::cout << "  (Both compute the same sum — difference is purely memory access pattern)\n";
    return 0;
}
```

### Python — Simple Version

Measure and compare common Python operation pairs using `time.perf_counter()` — list vs tuple, dict lookup vs linear search, `+=` vs `join`, for-loop vs list comprehension.

```python
# Measure and compare the performance of common Python operations.
# Use time.perf_counter() for high-resolution wall-clock timing.
# Rule: MEASURE FIRST before assuming what's slow.

import time

def measure(label: str, fn, reps: int = 100_000) -> float:
    """Run fn() reps times and return the average time in microseconds."""
    start = time.perf_counter()
    for _ in range(reps):
        fn()
    elapsed_us = (time.perf_counter() - start) * 1_000_000 / reps
    print(f"  {label:<50s} {elapsed_us:8.3f} µs")
    return elapsed_us

if __name__ == "__main__":
    print("=== Python Performance Measurement ===\n")
    N = 1_000

    # ── 1. List vs Tuple indexed access ──────────────────────────────────────
    # Tuples are immutable; Python stores them more compactly and accesses them faster.
    print("1. Indexed access:")
    lst   = list(range(N))
    tup   = tuple(range(N))
    t_lst = measure("list[500]   (mutable)",                lambda: lst[500])
    t_tup = measure("tuple[500]  (immutable, no overhead)", lambda: tup[500])
    print(f"    → tuple is {t_lst/t_tup:.1f}x faster for indexing\n")

    # ── 2. Dict lookup vs list linear search ──────────────────────────────────
    # Dict: hash table → O(1) average.  List: linear scan → O(N).
    print("2. Membership test (N=1000):")
    d    = {i: i for i in range(N)}
    t_d  = measure("500 in dict  (hash lookup, O(1))",        lambda: 500 in d)
    t_l  = measure("500 in list  (linear scan, O(N))",        lambda: 500 in lst)
    print(f"    → dict is {t_l/t_d:.0f}x faster for membership test\n")

    # ── 3. String concatenation: += vs join ───────────────────────────────────
    # Each += creates a NEW string object (Python strings are immutable) → O(n²) total.
    # join() builds once from a list → O(n).
    print("3. String building (100 words):")
    words = ["hello"] * 100

    def concat_plus():
        s = ""
        for w in words: s += w   # creates new string each iteration
        return s

    def concat_join():
        return "".join(words)    # allocates once, copies once

    t_plus = measure("str += in loop  (O(n²) allocations)",  concat_plus, reps=10_000)
    t_join = measure("''.join(words)  (O(n), single alloc)", concat_join, reps=10_000)
    print(f"    → join is {t_plus/t_join:.1f}x faster\n")

    # ── 4. List comprehension vs explicit for-loop ────────────────────────────
    # Comprehensions run the loop body in optimized C bytecode (LOAD_FAST, LIST_APPEND).
    # Explicit append() loops go through the Python bytecode interpreter for each call.
    print("4. List building (N=1000 elements):")
    data = list(range(N))

    t_comp = measure("[x*2 for x in data]     (comprehension)", lambda: [x*2 for x in data], reps=10_000)

    def loop_version():
        result = []
        for x in data: result.append(x * 2)
        return result

    t_loop = measure("for x in data: result.append(x*2)", loop_version, reps=10_000)
    print(f"    → comprehension is {t_loop/t_comp:.1f}x faster\n")

    print("  Key insight: always MEASURE before optimizing — intuition is often wrong!")
```

### Python — Medium Level

Profiler decorator that tracks call count and timing per function plus a cache-friendly vs cache-unfriendly matrix access benchmark — the two most practical performance engineering tools.

```python
# Part 1: @profile decorator — measures every call to a decorated function.
#         Report shows hottest functions (most total time) and most-called functions.
# Part 2: Cache-friendly vs cache-unfriendly matrix access — row-major vs column-major.
#         Shows why memory access ORDER matters as much as algorithm complexity.

import time
import functools
from collections import defaultdict
from dataclasses import dataclass, field

# ── PART 1: Profiler ──────────────────────────────────────────────────────────

@dataclass
class Stats:
    calls:    int   = 0
    total_us: float = 0.0
    min_us:   float = float("inf")
    max_us:   float = 0.0

    @property
    def avg_us(self): return self.total_us / self.calls if self.calls else 0.0

# Global registry: function name → Stats
_registry: dict[str, Stats] = defaultdict(Stats)

def profile(fn=None, *, name=None):
    """
    Decorator: @profile  or  @profile(name='label')
    Every call is timed and accumulated into _registry.
    """
    if fn is None:                        # called as @profile(name='...')
        return lambda f: profile(f, name=name)

    label = name or fn.__qualname__

    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        result = fn(*args, **kwargs)
        us = (time.perf_counter() - t0) * 1_000_000

        s = _registry[label]
        s.calls    += 1
        s.total_us += us
        s.min_us    = min(s.min_us, us)
        s.max_us    = max(s.max_us, us)
        return result

    return wrapper

def print_profile_report():
    """Print profiler stats sorted by total time (hottest first)."""
    print("\n=== Profiler Report ===")
    print(f"  {'Function':<32} {'Calls':>7} {'Total µs':>10} {'Avg µs':>9} {'Min µs':>8} {'Max µs':>8}")
    print("  " + "─" * 78)
    for name, s in sorted(_registry.items(), key=lambda kv: kv[1].total_us, reverse=True):
        print(f"  {name:<32} {s.calls:>7} {s.total_us:>10.1f} {s.avg_us:>9.2f} {s.min_us:>8.2f} {s.max_us:>8.2f}")


# ── Profiled functions ─────────────────────────────────────────────────────────
@profile
def linear_search(lst: list, target: int) -> bool:
    return target in lst                     # O(N) scan

@profile
def binary_search(lst: list, target: int) -> bool:
    lo, hi = 0, len(lst) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if   lst[mid] == target: return True
        elif lst[mid] < target:  lo = mid + 1
        else:                    hi = mid - 1
    return False

@profile
def sort_copy(lst: list) -> list:
    return sorted(lst)                       # O(N log N)

@profile(name="string_join")
def build_string(words: list) -> str:
    return "".join(words)                    # O(N)


# ── PART 2: Cache-Friendly vs Cache-Unfriendly Matrix Access ──────────────────
# Python stores list-of-lists as a list of row objects.
# Row-major: accesses each row sequentially — relatively cache-friendly.
# Col-major: jumps between row objects — each mat[i][j] dereferences a different object.

@profile(name="sum_row_major")
def sum_row_major(mat: list[list[int]]) -> int:
    """Access row-by-row (inner loop over columns — sequential per row)."""
    total = 0
    for row in mat:
        for val in row:
            total += val
    return total

@profile(name="sum_col_major")
def sum_col_major(mat: list[list[int]]) -> int:
    """Access column-by-column (inner loop over rows — jumps between row objects)."""
    total = 0
    n = len(mat)
    for j in range(n):
        for i in range(n):
            total += mat[i][j]   # each mat[i] is a separate Python list object
    return total


if __name__ == "__main__":
    import random

    N = 1_000
    data    = list(range(N))
    sorted_ = sorted(data)
    random.shuffle(data)
    words   = ["performance"] * 200

    print("=== Running profiled workloads ===")
    for _ in range(500):
        linear_search(data, N // 2)
        binary_search(sorted_, N // 2)
    for _ in range(100):
        sort_copy(data[:])
        build_string(words)

    # ── Cache benchmark ────────────────────────────────────────────────────────
    print("Running cache access pattern benchmark...")
    SIZE = 200
    mat  = [[i * SIZE + j for j in range(SIZE)] for i in range(SIZE)]
    for _ in range(30):
        sum_row_major(mat)
        sum_col_major(mat)

    print_profile_report()

    # Summary insights
    row_s = _registry["sum_row_major"]
    col_s = _registry["sum_col_major"]
    ratio = col_s.avg_us / row_s.avg_us if row_s.avg_us else 0
    print(f"\n  Cache insight: column-major is {ratio:.1f}x slower than row-major")
    print(f"  (both compute the same sum — only the access ORDER differs)")

    hottest     = max(_registry.items(), key=lambda kv: kv[1].total_us)[0]
    most_called = max(_registry.items(), key=lambda kv: kv[1].calls)[0]
    print(f"\n  Hottest function (most CPU time): {hottest}")
    print(f"  Most-called function:             {most_called}")
    print(f"\n  → Fix the hottest function first for maximum performance gain!")
```

---

## 12. Key Takeaways

- **Performance measurement** = collecting metrics on CPU, memory, disk, network; needed before any tuning
- **Core metrics:** throughput (tasks/sec), response time (time to first response), CPU utilization (% busy), page fault rate, IOPS/disk throughput
- **Healthy CPU utilization:** 70–80% — high enough to be productive, low enough to handle bursts
- **Page fault rate > 10%** signals memory pressure; heavy swap activity means thrashing risk
- **Bottleneck rule:** fix the single most constrained resource first — improving anything else won't help until that bottleneck is cleared
- **CPU bottleneck:** utilization > 90%, high ready queue; **Memory:** heavy paging/swap; **Disk:** high I/O wait, disk utilization near 100%
- **Linux tuning tools:** `top`/`htop`, `vmstat`, `iostat`, `mpstat`, `iotop`; **Windows:** Task Manager, Resource Monitor, Performance Monitor
- **CPU tuning:** `nice`/`renice` for priorities; `taskset` for CPU affinity; scheduling classes (real-time, interactive, batch)
- **Memory tuning:** `vm.swappiness` (low = keep in RAM; for database servers use 1); increase RAM for memory-bound workloads
- **Disk tuning:** choose right I/O scheduler (`noop`/`none` for SSD; `deadline` for databases; `CFQ` for desktop fairness)
- **Benchmarking rules:** establish baseline first; change one thing at a time; run long enough; warm up caches; isolate the system
- **Tuning workflow:** measure → identify bottleneck → one change → measure → compare → document → repeat
- **When to stop:** system meets requirements; hardware upgrade is more cost-effective than further software tuning
