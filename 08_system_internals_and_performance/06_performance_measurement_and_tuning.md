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
