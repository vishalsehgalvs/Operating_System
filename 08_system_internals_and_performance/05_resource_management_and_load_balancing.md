# OS Resource Management and CPU Load Balancing

> **Resource management** is how the OS controls and allocates CPU, memory, I/O, and storage to processes fairly and efficiently; **load balancing** extends this to multicore/multiprocessor systems by distributing processes across CPUs so no single core is overwhelmed while others sit idle — together these determine how well your system handles multiple concurrent tasks.

---

## Table of Contents

1. [What is Resource Management?](#1-what-is-resource-management)
2. [Key Resources and Goals](#2-key-resources-and-goals)
3. [Resource Allocation Strategies](#3-resource-allocation-strategies)
4. [Resource Allocation Graph](#4-resource-allocation-graph)
5. [What is Load Balancing?](#5-what-is-load-balancing)
6. [Load Balancing Approaches](#6-load-balancing-approaches)
7. [Processor Affinity](#7-processor-affinity)
8. [Load Balancing Metrics](#8-load-balancing-metrics)
9. [Challenges in Resource Management](#9-challenges-in-resource-management)
10. [Monitoring and Tuning](#10-monitoring-and-tuning)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What is Resource Management?

**Resource management** is the process by which the OS controls and allocates system resources — CPU time, memory, disk, I/O devices — to processes and threads so they can make progress without conflicting with each other.

**Analogy:** A shared printer in an office. Multiple people send print jobs at the same time. The OS (office manager) must decide the order, allocate printer time fairly, and ensure no job waits forever. Without management, everyone would fight over the printer and nothing would print correctly.

```
  Without resource management:
  Process A: grabs all CPU time — others starve
  Process B: allocates all RAM — others crash with OOM
  Process C: holds a file open indefinitely — others can't access it

  With resource management:
  OS scheduler: CPU shared using time slices across all processes
  OS memory manager: each process gets its virtual address space
  OS file system: locking and permissions on file access
  → Everything runs in parallel, fairly
```

---

## 2. Key Resources and Goals

### Resources Managed by the OS

| Resource          | What OS Controls                                            |
| ----------------- | ----------------------------------------------------------- |
| **CPU**           | Which process runs on which core, for how long (scheduling) |
| **Memory (RAM)**  | Allocation of virtual address spaces, page tables, swap     |
| **I/O Devices**   | Access ordering for disks, keyboards, mice, NICs            |
| **Files/Storage** | Disk space allocation, file locks, permissions              |

### Goals of Resource Management

| Goal               | Meaning                                               |
| ------------------ | ----------------------------------------------------- |
| **Fairness**       | Every process gets a reasonable share of resources    |
| **Efficiency**     | Resources shouldn't sit idle when work is waiting     |
| **Responsiveness** | Interactive apps (browser, terminal) respond quickly  |
| **Throughput**     | Maximize tasks completed per unit of time             |
| **No starvation**  | No process waits indefinitely for a resource it needs |

These goals often **conflict.** For example:

- Prioritizing responsiveness (interactive apps get CPU first) can reduce throughput
- Maximizing throughput (run big batch jobs without interruption) hurts responsiveness
- → OS must make trade-offs based on system type (desktop, server, real-time)

---

## 3. Resource Allocation Strategies

### Static Allocation

Resources are assigned to a process at **creation time** and stay fixed until the process terminates.

```
  Process created → assigned 100 MB RAM and 2 CPU cores → keeps them forever

  Pros: Simple, predictable, zero runtime overhead
  Cons: Inflexible — what if process only needs 10 MB most of the time?
        Resources sit idle, can't be used by others
```

### Dynamic Allocation

Resources are adjusted **at runtime** based on current needs and system state.

```
  Process starts with 10 MB → working set grows → OS allocates more RAM
  Process blocked on I/O → CPU time given to other processes
  Process priority raised → gets more CPU slices

  Pros: Adaptive, efficient — resources used only when actually needed
  Cons: More complex OS logic; some overhead from constant tracking
```

| Strategy    | Advantages                        | Disadvantages                    |
| ----------- | --------------------------------- | -------------------------------- |
| **Static**  | Simple, predictable, low overhead | Inflexible, may waste resources  |
| **Dynamic** | Adaptive, efficient resource use  | Complex, higher runtime overhead |

Most modern OSes use **dynamic allocation** for CPU and memory, with some static elements (e.g., hard memory limits per container/VM).

---

## 4. Resource Allocation Graph

The **Resource Allocation Graph (RAG)** is a visual tool the OS uses to track which processes hold which resources and which are waiting — used to detect deadlocks.

```
  Nodes:
    ○ = Process (P1, P2, ...)
    □ = Resource type (R1, R2, ...)
    • = Individual instance of resource (inside the rectangle)

  Edges:
    □──►○  =  Allocation edge  (resource IS held by process)
    ○──►□  =  Request edge     (process IS WAITING for resource)

  Example — no deadlock:

    P1 ──request──► R1 ◄──holds── P2

  Example — deadlock cycle:

    P1 ──holds──► R1
    R1 ──holds──► P2
    P2 ──holds──► R2
    R2 ──holds──► P1   ← circular! → deadlock
```

If the graph contains a cycle AND each resource has only one instance → deadlock is guaranteed. The OS can detect or prevent deadlocks by analyzing this graph (Banker's Algorithm uses a related structure).

---

## 5. What is Load Balancing?

**Load balancing** is the technique of distributing workload evenly across multiple CPUs/cores so no single processor is overwhelmed while others are idle.

**Analogy:** A grocery store with 10 checkout lanes but only 1 cashier working while the other 9 are empty. Customers queue for a long time despite available capacity. Load balancing opens more lanes when queues grow — ensuring all capacity is used.

```
  Without load balancing (bad):

  CPU 0: [P1][P2][P3][P4][P5]  ← 100% busy, queue building
  CPU 1: [idle]
  CPU 2: [idle]
  CPU 3: [idle]
  → Users experience slowdowns despite 75% of CPU sitting unused

  With load balancing (good):

  CPU 0: [P1][P5]
  CPU 1: [P2]
  CPU 2: [P3]
  CPU 3: [P4]
  → Each CPU at ~25% load, work finishes 4x faster
```

### Types of Load Balancing

| Type                   | How It Works                                                                |
| ---------------------- | --------------------------------------------------------------------------- |
| **Static**             | Assign processes to CPUs at creation time based on fixed rules              |
| **Dynamic**            | Continuously monitor CPU loads and migrate processes at runtime             |
| **Processor Affinity** | Keep process on same CPU for cache locality (may sacrifice perfect balance) |

---

## 6. Load Balancing Approaches

### Push Migration

A dedicated kernel task periodically checks load on each CPU. If imbalance is found, it **pushes** processes from the overloaded CPU to a less busy one.

```
  Scheduler thread runs every N ms:

  Check loads:  CPU0=85%  CPU1=10%  CPU2=12%  CPU3=80%
  Decision:     Move 2 processes from CPU0 → CPU1
                Move 1 process  from CPU3 → CPU2
  Result:       CPU0=45%  CPU1=40%  CPU2=42%  CPU3=40%  (balanced!)
```

**Analogy:** A manager watching employees and reassigning tasks when someone is overwhelmed. The system actively monitors and redistributes.

### Pull Migration

An **idle or lightly loaded CPU** proactively pulls processes from the queue of a busy CPU. Reactive rather than proactive.

```
  CPU1 finishes its last task:
  CPU1: "I'm idle — let me check other CPUs' queues..."
  CPU1: "CPU0 has 5 processes queued — I'll take 2."
  CPU1: pulls 2 processes from CPU0's ready queue
```

**Analogy:** A waiter who finishes serving their tables and asks a busy colleague: "Can I help with any of your tables?"

### Symmetric Multiprocessing (SMP)

All CPUs share a **single global ready queue**. Any CPU that becomes idle simply picks the next process from the queue. Load balance is achieved automatically — no explicit migration needed.

```
  Global Ready Queue: [P1][P2][P3][P4][P5][P6]
                              ↑         ↑
                           CPU0 picks  CPU1 picks

  Each CPU grabs work as soon as it becomes free.
  Queue length itself determines distribution.
```

| Approach           | How It Works                                       | Best For                               |
| ------------------ | -------------------------------------------------- | -------------------------------------- |
| **Push Migration** | System actively moves tasks from busy to idle CPUs | Predictable workloads                  |
| **Pull Migration** | Idle CPUs steal tasks from busy ones               | Dynamic variable workloads             |
| **SMP**            | All CPUs share one global queue                    | General-purpose multiprocessor systems |

Linux uses a combination: per-CPU run queues (for cache efficiency) + periodic load balancing using push/pull migration.

---

## 7. Processor Affinity

**Processor affinity** is the tendency of the scheduler to keep a process running on the **same CPU** it was previously on, rather than migrating it.

**Why?** When a process runs on a CPU, that CPU's cache fills with the process's working data. Migrating the process to another CPU means:

- New CPU cache is **cold** (empty)
- Must re-fetch all data from RAM (slow)
- Cache warmup takes time

```
  Process P on CPU0 for 50ms:
  CPU0 cache: [P's code][P's data][P's stack]  ← hot cache, fast!

  If P migrated to CPU1:
  CPU1 cache: [empty]
  → All memory accesses go to RAM (100x slower than cache)
  → Significant performance penalty until cache warms up

  Affinity keeps P on CPU0 to preserve cache contents.
```

### Soft vs Hard Affinity

| Type              | Behavior                                                                     |
| ----------------- | ---------------------------------------------------------------------------- |
| **Soft Affinity** | OS _prefers_ to keep process on same CPU but will move it for load balancing |
| **Hard Affinity** | Process is _locked_ to specific CPU(s) — OS will never violate this          |

```c
// Setting hard CPU affinity in Linux (restrict process to CPUs 0 and 2 only)
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(0, &cpuset);  // allow CPU 0
CPU_SET(2, &cpuset);  // allow CPU 2
sched_setaffinity(pid, sizeof(cpuset), &cpuset);
```

**Use case for hard affinity:**

- Real-time systems: ensure critical process always runs on a dedicated core
- NUMA (Non-Uniform Memory Access) systems: keep process on CPU closest to its RAM
- Isolation: security-sensitive process pinned to a specific core

---

## 8. Load Balancing Metrics

The OS needs measurements to decide when and how to rebalance:

### CPU Utilization

Percentage of time a CPU is busy (not idle).

```
  CPU0: 90% utilization → overloaded, high-priority target for migration
  CPU1: 15% utilization → underloaded, good destination for migrated tasks
```

⚠️ High utilization ≠ good performance if the CPU is busy with low-priority background tasks while important interactive tasks wait.

### Queue Length

Number of processes in a CPU's ready queue waiting to run.

```
  CPU0: 8 processes queued → seriously overloaded
  CPU1: 1 process queued  → good candidate to receive migrated work
```

Queue length is often more actionable than utilization for triggering migration.

### Response Time

How quickly the system responds to user input. A well-balanced system has **consistent, low** response times across all CPUs.

| Metric              | What It Measures                 | Target                                 |
| ------------------- | -------------------------------- | -------------------------------------- |
| **CPU Utilization** | % time CPU is busy (not idle)    | 70–80% (leaves headroom for bursts)    |
| **Queue Length**    | Processes waiting in ready queue | Low and evenly distributed across CPUs |
| **Response Time**   | Time from request to response    | Consistent and minimal                 |

---

## 9. Challenges in Resource Management

### Resource Contention

Multiple processes need the same resource at the same time. Poor handling causes bottlenecks.

```
  Disk contention example:
  10 processes all issue disk writes simultaneously
  → Disk can handle ~1 operation at a time
  → 9 processes block waiting → throughput collapses

  Solution: I/O scheduler (CFQ, deadline, NOOP) orders disk requests
             to minimize seek time and serve all processes fairly
```

### Priority Inversion

A high-priority process is blocked waiting for a resource held by a low-priority process (which itself gets preempted by medium-priority processes).

```
  P_high wants mutex M
  P_low holds mutex M
  P_medium preempts P_low (doesn't need M)

  Result: P_high blocked indefinitely by P_medium that doesn't even interact with M

  Fix: Priority inheritance — P_low temporarily gets P_high's priority
       until it releases M, so P_medium can't preempt it
```

### Migration Overhead

Moving a process between CPUs isn't free:

- Save current process context (registers, stack pointer)
- Update scheduler data structures on both CPUs
- Cold cache on destination CPU → slower execution until warmed up
- Cache coherence traffic between CPUs

```
  If migration cost > performance gain from better balance → don't migrate.

  OS uses a "migration threshold":
  Only migrate if load difference > some percentage AND
  process has been running long enough to justify cache warmup cost.
```

---

## 10. Monitoring and Tuning

### Common Monitoring Tools

| Tool                | Platform | Shows                                                   |
| ------------------- | -------- | ------------------------------------------------------- |
| `top` / `htop`      | Linux    | Per-process CPU %, memory, nice value, per-CPU bars     |
| `vmstat`            | Linux    | CPU usage, memory, swap, I/O, context switches          |
| `iostat`            | Linux    | Per-disk I/O statistics, utilization, wait times        |
| `mpstat`            | Linux    | Per-CPU utilization (great for NUMA/multicore analysis) |
| Task Manager        | Windows  | CPU/memory/disk per process; per-core CPU usage         |
| Performance Monitor | Windows  | Detailed counters, historical graphs                    |

### Key Tunable Parameters

| Parameter                       | Effect                                                             |
| ------------------------------- | ------------------------------------------------------------------ |
| **CPU time slice / quantum**    | Longer = more throughput; shorter = more responsiveness            |
| **Nice value** (`-20` to `+19`) | Adjusts process priority in Linux scheduler                        |
| **`/proc/sys/kernel/sched_*`**  | Linux kernel scheduler tuning (migration cost, balancing interval) |
| **Swappiness**                  | Aggressiveness of swapping memory pages to disk                    |
| **I/O scheduler**               | CFQ (fairness), deadline (low latency), noop (SSDs)                |
| **cgroups / cpusets**           | Hard limits on CPU/memory per group of processes (containers)      |

---

## 11. Key Takeaways

- **Resource management** = OS controls CPU time, memory, I/O, storage; goals are fairness, efficiency, responsiveness, throughput, no starvation
- **Static allocation** assigns resources at process creation (simple, inflexible); **dynamic allocation** adjusts at runtime (efficient, more complex)
- **Resource Allocation Graph** tracks which process holds/waits for which resources — cycles indicate deadlocks
- **Load balancing** distributes processes across multiple CPUs so no core is over/underloaded
- **Push migration:** system detects imbalance and moves processes from busy → idle CPUs (proactive)
- **Pull migration:** idle CPUs steal work from busy CPUs' queues (reactive)
- **SMP (Symmetric Multiprocessing):** all CPUs share one global ready queue — balance is automatic
- **Processor affinity:** keep process on same CPU for cache locality; soft affinity is the OS preference, hard affinity is an enforced lock
- **Load metrics:** CPU utilization (% busy), queue length (waiting processes), response time — all used together to decide when to rebalance
- **Resource contention** bottlenecks when multiple processes fight for the same resource; solved by scheduling, locking, and queuing
- **Priority inversion** blocks high-priority work due to low-priority resource holders; solved by priority inheritance
- **Migration overhead** (cold cache, context save) means too much migration can hurt more than it helps — threshold-based migration is used
- **Monitoring tools** (`htop`, `vmstat`, `mpstat`) let you observe resource usage; **tuning parameters** (nice values, cgroups, I/O scheduler) let you optimize for your workload
