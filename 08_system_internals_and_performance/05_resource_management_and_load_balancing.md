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

## 10. Code Examples

> Working code that demonstrates resource management and load balancing in practice.

### C++ — Simple Version

Simulate a load balancer with three strategies: Round Robin, Least Connections, and Random — shows which server handles each request and how load distributes.

```cpp
// Simulate a load balancer with 3 classic strategies.
// Round Robin: cycle through servers in order (simple, fair).
// Least Connections: always pick the server with fewest active requests (best for uneven load).
// Random: pick a server at random (simple, works at scale).

#include <iostream>
#include <string>
#include <vector>
#include <algorithm>
#include <random>
#include <iomanip>

// ── Server ────────────────────────────────────────────────────────────────────
struct Server {
    std::string name;
    int active = 0;    // currently active connections
    int total  = 0;    // total requests served (for reporting)
};

// ── Load Balancer ─────────────────────────────────────────────────────────────
class LoadBalancer {
    std::vector<Server> servers;
    int rr_index = 0;                    // round-robin counter
    std::mt19937 rng{42};               // fixed seed for reproducible output

public:
    void add_server(const std::string& name) {
        servers.push_back({name});
    }

    // ── Round Robin: cycle in order ──────────────────────────────────────────
    Server* round_robin() {
        Server* s = &servers[rr_index % servers.size()];
        rr_index++;
        return s;
    }

    // ── Least Connections: pick server with fewest active connections ─────────
    Server* least_connections() {
        return &*std::min_element(servers.begin(), servers.end(),
            [](const Server& a, const Server& b){ return a.active < b.active; });
    }

    // ── Random: pick any server uniformly at random ───────────────────────────
    Server* random_pick() {
        std::uniform_int_distribution<int> dist(0, (int)servers.size() - 1);
        return &servers[dist(rng)];
    }

    // Route a request using the chosen strategy
    void route(const std::string& req, const std::string& strategy) {
        Server* s = nullptr;
        if      (strategy == "round_robin")       s = round_robin();
        else if (strategy == "least_connections") s = least_connections();
        else if (strategy == "random")            s = random_pick();
        else { std::cout << "Unknown strategy\n"; return; }

        s->active++;
        s->total++;
        std::cout << "  [" << std::setw(18) << strategy << "]  "
                  << std::setw(25) << req << " → " << s->name
                  << "  (active=" << s->active << ")\n";

        // Simulate some requests completing
        if (s->active > 2) s->active--;
    }

    void report() {
        std::cout << "  Summary:";
        for (const auto& s : servers)
            std::cout << "  " << s.name << "(total=" << s.total << " active=" << s.active << ")";
        std::cout << "\n";
    }
};

int main() {
    std::cout << "=== Load Balancer Simulation ===\n\n";

    std::vector<std::string> requests = {
        "GET /home", "GET /api/user", "POST /login",
        "GET /images/logo", "GET /api/products",
        "POST /checkout", "GET /about"
    };

    for (const std::string& strategy : {"round_robin", "least_connections", "random"}) {
        std::cout << "--- " << strategy << " ---\n";
        LoadBalancer lb;
        lb.add_server("srv-A");
        lb.add_server("srv-B");
        lb.add_server("srv-C");
        for (const auto& req : requests) lb.route(req, strategy);
        lb.report();
        std::cout << "\n";
    }

    return 0;
}
```

### C++ — Medium / LeetCode Style

Resource pool scheduler (assign tasks to nodes based on CPU/memory availability) plus LeetCode #621 Task Scheduler (minimum time to finish tasks with a per-type cooling period).

```cpp
// Part 1: Resource pool — assign tasks to servers using First Fit bin packing.
//         Each server has CPU and memory limits; reject tasks that don't fit anywhere.
// Part 2: LeetCode #621 Task Scheduler — greedy max-heap approach.
//         Given tasks [A,A,A,B,B,B] and cooldown n, find minimum total execution time.

#include <iostream>
#include <string>
#include <vector>
#include <queue>
#include <unordered_map>
#include <algorithm>
#include <iomanip>

// ── Task ──────────────────────────────────────────────────────────────────────
struct Task {
    std::string name;
    int cpu_pct;    // % of one CPU core required
    int mem_mb;     // MB of RAM required
};

// ── Server node ───────────────────────────────────────────────────────────────
struct Node {
    std::string              name;
    int                      cpu_total, mem_total;
    int                      cpu_used = 0, mem_used = 0;
    std::vector<std::string> running_tasks;

    bool can_fit(const Task& t) const {
        return (cpu_used + t.cpu_pct <= cpu_total) &&
               (mem_used + t.mem_mb  <= mem_total);
    }
    void assign(const Task& t) {
        cpu_used += t.cpu_pct;
        mem_used += t.mem_mb;
        running_tasks.push_back(t.name);
    }
};

// ── Resource Pool Scheduler (First Fit) ──────────────────────────────────────
void schedule_pool(std::vector<Node>& nodes, const std::vector<Task>& tasks) {
    std::cout << "=== Resource Pool — First Fit Scheduling ===\n\n";
    for (const auto& task : tasks) {
        std::cout << "Task \"" << task.name << "\" (CPU=" << task.cpu_pct
                  << "% MEM=" << task.mem_mb << "MB)\n";
        bool placed = false;
        for (auto& node : nodes) {
            if (node.can_fit(task)) {
                node.assign(task);
                std::cout << "  → " << node.name
                          << "  CPU " << node.cpu_used << "/" << node.cpu_total << "%"
                          << "  MEM " << node.mem_used << "/" << node.mem_total << "MB\n";
                placed = true;
                break;
            }
        }
        if (!placed) std::cout << "  → REJECTED: no node has enough free resources!\n";
    }
    std::cout << "\n  Cluster status:\n";
    for (const auto& n : nodes) {
        std::cout << "  " << std::setw(10) << n.name
                  << " CPU=" << n.cpu_used << "/" << n.cpu_total << "%"
                  << "  MEM=" << n.mem_used << "/" << n.mem_total << "MB"
                  << "  tasks=[";
        for (size_t i = 0; i < n.running_tasks.size(); ++i) {
            std::cout << n.running_tasks[i];
            if (i + 1 < n.running_tasks.size()) std::cout << ",";
        }
        std::cout << "]\n";
    }
}

// ── LeetCode #621: Task Scheduler ────────────────────────────────────────────
// Greedy: in each window of n+1 slots, pick the most-frequent remaining tasks.
// If we run out of tasks before filling the window, idle slots fill the gap.
int task_scheduler(std::vector<char>& tasks, int n) {
    std::unordered_map<char, int> freq;
    for (char t : tasks) freq[t]++;

    std::priority_queue<int> pq;  // max-heap of remaining counts
    for (auto& [ch, cnt] : freq) pq.push(cnt);

    int time = 0;
    while (!pq.empty()) {
        std::vector<int> window;
        for (int i = 0; i <= n && !pq.empty(); ++i) {
            window.push_back(pq.top() - 1);
            pq.pop();
            time++;
        }
        // Idle slots if fewer than n+1 tasks available (only matters for last window)
        if (!pq.empty()) time += (n + 1 - (int)window.size());
        for (int cnt : window) if (cnt > 0) pq.push(cnt);
    }
    return time;
}

int main() {
    // ── Resource pool ─────────────────────────────────────────────────────────
    std::vector<Node> nodes = {
        {"node-1", 100, 4096},
        {"node-2", 100, 4096},
        {"node-3", 100, 4096},
    };
    std::vector<Task> tasks = {
        {"web-server",   30, 512 },
        {"db-primary",   60, 2048},
        {"cache",        20, 1024},
        {"analytics",    80, 3000},   // large memory requirement
        {"log-ingester", 15, 256 },
        {"ml-training",  95, 4000},   // too large for any single node
    };
    schedule_pool(nodes, tasks);

    // ── LeetCode #621 ─────────────────────────────────────────────────────────
    std::cout << "\n=== LeetCode #621: Task Scheduler ===\n\n";

    struct Case { std::vector<char> tasks; int n; std::string note; };
    std::vector<Case> cases = {
        {{'A','A','A','B','B','B'}, 2, "A→B→idle→A→B→idle→A→B"},
        {{'A','A','A','B','B','B'}, 0, "AAABBB (no cooldown)"},
        {{'A','A','A','A','A','A','B','C','D','E','F','G'}, 2, "A dominates"},
    };
    for (auto& c : cases) {
        std::string s(c.tasks.begin(), c.tasks.end());
        std::sort(s.begin(), s.end());
        std::cout << "  tasks=" << s << "  n=" << c.n
                  << "  min_time=" << task_scheduler(c.tasks, c.n)
                  << "  (" << c.note << ")\n";
    }
    return 0;
}
```

### Python — Simple Version

Simulate a load balancer with Round Robin, Least Connections, and Random strategies — shows how each request is routed and how load distributes across three servers.

```python
# Simulate a load balancer with 3 classic strategies.
# Round Robin: cycle through servers in fixed order — simple, equal distribution.
# Least Connections: always send to the least-loaded server — better for uneven work.
# Random: pick a server randomly — simple, effective at scale.

import random

class Server:
    def __init__(self, name: str):
        self.name   = name
        self.active = 0   # currently active connections
        self.total  = 0   # total requests served

    def __repr__(self):
        return f"{self.name}(active={self.active}, total={self.total})"

class LoadBalancer:
    def __init__(self, servers: list[Server]):
        self.servers  = servers
        self._rr_idx  = 0

    def round_robin(self) -> Server:
        """Cycle through servers in fixed order — O(1), simple, fair."""
        s = self.servers[self._rr_idx % len(self.servers)]
        self._rr_idx += 1
        return s

    def least_connections(self) -> Server:
        """Pick the server with fewest active connections — best for uneven load."""
        return min(self.servers, key=lambda s: s.active)

    def random_pick(self) -> Server:
        """Pick a random server — simple, works well when servers are homogeneous."""
        return random.choice(self.servers)

    def route(self, request: str, strategy: str) -> Server:
        strategy_map = {
            "round_robin":       self.round_robin,
            "least_connections": self.least_connections,
            "random":            self.random_pick,
        }
        if strategy not in strategy_map:
            raise ValueError(f"Unknown strategy: {strategy}")

        s = strategy_map[strategy]()
        s.active += 1
        s.total  += 1

        print(f"  [{strategy:18s}]  {request:25s} → {s.name}  (active={s.active})")

        # Simulate some requests completing
        if s.active > 2:
            s.active -= 1

        return s

    def report(self):
        print("  Summary:", " | ".join(str(s) for s in self.servers))


if __name__ == "__main__":
    random.seed(42)
    requests = [
        "GET /home", "GET /api/user", "POST /login",
        "GET /images/logo", "GET /api/products",
        "POST /checkout", "GET /about",
    ]

    print("=== Load Balancer Simulation ===\n")

    for strategy in ["round_robin", "least_connections", "random"]:
        print(f"--- {strategy.replace('_', ' ').title()} ---")
        servers = [Server("srv-A"), Server("srv-B"), Server("srv-C")]
        lb = LoadBalancer(servers)
        for req in requests:
            lb.route(req, strategy)
        lb.report()
        print()
```

### Python — Medium Level

Resource pool with CPU/memory limits (First Fit bin packing) plus LeetCode #621 Task Scheduler solved with a greedy max-heap approach.

```python
# Part 1: Resource pool — assign tasks to cluster nodes using First Fit.
#         Nodes have CPU % and memory MB limits; tasks are rejected if nothing fits.
# Part 2: LeetCode #621 Task Scheduler — find minimum time to finish all tasks
#         given a cooldown n between tasks of the same type (greedy max-heap).

import heapq
from collections import Counter
from dataclasses import dataclass, field

# ── Resource Pool ─────────────────────────────────────────────────────────────
@dataclass
class Task:
    name:    str
    cpu_pct: int   # % of one CPU core
    mem_mb:  int   # MB of RAM

@dataclass
class Node:
    name:       str
    cpu_total:  int
    mem_total:  int
    cpu_used:   int  = 0
    mem_used:   int  = 0
    tasks:      list = field(default_factory=list)

    @property
    def cpu_free(self): return self.cpu_total - self.cpu_used
    @property
    def mem_free(self): return self.mem_total - self.mem_used

    def can_fit(self, t: Task) -> bool:
        return t.cpu_pct <= self.cpu_free and t.mem_mb <= self.mem_free

    def assign(self, t: Task):
        self.cpu_used += t.cpu_pct
        self.mem_used += t.mem_mb
        self.tasks.append(t.name)

def schedule_pool(nodes: list[Node], tasks: list[Task]):
    print("=== Resource Pool — First Fit Scheduling ===\n")
    for task in tasks:
        print(f"  Task '{task.name}' (CPU={task.cpu_pct}% MEM={task.mem_mb}MB)")
        placed = False
        for node in nodes:
            if node.can_fit(task):
                node.assign(task)
                bar = lambda used, total: "█" * (used * 10 // total) + "░" * (10 - used * 10 // total)
                print(f"    → {node.name}  CPU [{bar(node.cpu_used, node.cpu_total)}] {node.cpu_used}/{node.cpu_total}%"
                      f"  MEM [{bar(node.mem_used, node.mem_total)}] {node.mem_used}/{node.mem_total}MB")
                placed = True
                break
        if not placed:
            print(f"    → REJECTED: no node has enough free resources!")

    print("\n  Cluster status:")
    for node in nodes:
        print(f"    {node.name:<10}  CPU={node.cpu_used}/{node.cpu_total}%"
              f"  MEM={node.mem_used}/{node.mem_total}MB  tasks={node.tasks}")


# ── LeetCode #621: Task Scheduler ─────────────────────────────────────────────
# Greedy insight: in each window of n+1 slots, always schedule the most frequent
# remaining task type. If fewer task types remain than slots, fill with idle time.
# Time O(N log k) where N = total tasks, k = distinct task types.
def task_scheduler(tasks: list[str], n: int) -> int:
    freq = Counter(tasks)
    # Python heapq is min-heap; negate counts for max-heap behavior
    heap = [-cnt for cnt in freq.values()]
    heapq.heapify(heap)

    time = 0
    while heap:
        window = []
        # Fill one cooldown window (n+1 slots)
        for _ in range(n + 1):
            if heap:
                window.append(heapq.heappop(heap))

        time += n + 1   # full window length

        # Re-add tasks that still have remaining count
        for cnt in window:
            if cnt + 1 < 0:   # cnt is negative; cnt+1 means one fewer remaining
                heapq.heappush(heap, cnt + 1)

        # Last window may be incomplete — remove trailing idle slots
        if not heap:
            time -= (n + 1 - len(window))

    return time


if __name__ == "__main__":
    # ── Resource pool ─────────────────────────────────────────────────────────
    nodes = [
        Node("node-1", cpu_total=100, mem_total=4096),
        Node("node-2", cpu_total=100, mem_total=4096),
        Node("node-3", cpu_total=100, mem_total=4096),
    ]
    tasks = [
        Task("web-server",   cpu_pct=30, mem_mb=512 ),
        Task("db-primary",   cpu_pct=60, mem_mb=2048),
        Task("cache",        cpu_pct=20, mem_mb=1024),
        Task("analytics",    cpu_pct=80, mem_mb=3000),   # large memory
        Task("log-ingester", cpu_pct=15, mem_mb=256 ),
        Task("ml-training",  cpu_pct=95, mem_mb=4000),   # too large for any node
    ]
    schedule_pool(nodes, tasks)

    # ── Task Scheduler ────────────────────────────────────────────────────────
    print("\n=== LeetCode #621: Task Scheduler ===\n")
    cases = [
        (list("AAABBB"),    2, "A→B→idle→A→B→idle→A→B = 8"),
        (list("AAABBB"),    0, "AAABBB = 6 (no cooldown)"),
        (list("AAAAABCDE"), 2, "A dominates: 13"),
    ]
    for tasks_in, n, note in cases:
        result = task_scheduler(tasks_in, n)
        print(f"  tasks={''.join(sorted(tasks_in))}  n={n}  → min_time={result}  ({note})")
```

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
