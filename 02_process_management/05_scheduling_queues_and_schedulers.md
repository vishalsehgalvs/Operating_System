# Scheduling Queues & Schedulers

> Operating systems use queues to organize processes by their current state and schedulers to decide which process gets CPU time next — together they keep all your running programs moving forward efficiently.

---

## Table of Contents

1. [What Are Scheduling Queues?](#1-what-are-scheduling-queues)
2. [Job Queue](#2-job-queue)
3. [Ready Queue](#3-ready-queue)
4. [Device Queues](#4-device-queues)
5. [Process Movement Between Queues](#5-process-movement-between-queues)
6. [What Are Schedulers?](#6-what-are-schedulers)
7. [Long-Term Scheduler (Job Scheduler)](#7-long-term-scheduler-job-scheduler)
8. [Short-Term Scheduler (CPU Scheduler)](#8-short-term-scheduler-cpu-scheduler)
9. [Medium-Term Scheduler](#9-medium-term-scheduler)
10. [Comparison of Schedulers](#10-comparison-of-schedulers)
11. [How Schedulers Work Together](#11-how-schedulers-work-together)
12. [CPU-Bound vs I/O-Bound Processes](#12-cpu-bound-vs-io-bound-processes)
13. [Queue Implementation](#13-queue-implementation)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. What Are Scheduling Queues?

Scheduling queues are **data structures that hold processes in different states**, waiting for system resources. They are organized waiting lines where processes stand in order based on their current needs and status.

When a process is created or changes state, it **moves between different queues**. The OS uses these queues to track:

- Which processes are ready to run
- Which are waiting for I/O operations
- Which are currently executing

**Restaurant analogy:** Customers (processes) move through different waiting areas — the waiting list, the table (running), and the queue for the restroom (I/O wait) — before leaving.

There are **three main types of queues**:

```
  ┌──────────────────────────────────────────────────────────────┐
  │                        JOB QUEUE                            │
  │         (all processes in the system — master list)         │
  │                                                              │
  │   ┌─────────────────────────────────────────────────────┐   │
  │   │                   READY QUEUE                       │   │
  │   │    (in memory, waiting for CPU time)                │   │
  │   └─────────────────────────────────────────────────────┘   │
  │                                                              │
  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
  │   │ Disk Queue  │  │ Print Queue │  │ Net Queue   │  ...   │
  │   │ (waiting    │  │ (waiting    │  │ (waiting    │        │
  │   │  for disk)  │  │  for print) │  │  for net)   │        │
  │   └─────────────┘  └─────────────┘  └─────────────┘        │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. Job Queue

The job queue contains **all processes that exist in the system**, regardless of their current state. When you launch an application, the process enters the job queue first.

- Tracks every process from **creation to termination**
- Most comprehensive queue — includes running, waiting, and ready processes
- Like the master guest list at a restaurant — everyone who has walked in, whether waiting, eating, or paying

---

## 3. Ready Queue

The ready queue holds processes that are **loaded in main memory and ready to execute** — they just need CPU time.

- A process enters here when it's in the **Ready state**
- The CPU scheduler picks processes from this queue
- Like customers already seated who placed their order — just waiting for the waiter

```
  READY QUEUE (Linked List of PCBs)

  [PCB: Process A] → [PCB: Process B] → [PCB: Process C] → NULL
        ↑                                                      ↑
      front                                                  rear

  CPU Scheduler picks from front → process gets CPU time
```

---

## 4. Device Queues

Device queues (also called **waiting queues**) contain processes waiting for a specific I/O device. Each device has its own queue.

- Process needs disk → goes to **disk queue**
- Process needs printer → goes to **print queue**
- When I/O completes → process moves back to **ready queue**
- Like customers who stepped out to use the restroom — they return to their table when done

```
  Process requests I/O
         │
         ▼
  ┌─────────────────┐     I/O completes
  │  Device Queue   │ ──────────────────► Ready Queue
  │  (e.g., disk)   │
  └─────────────────┘
```

---

## 5. Process Movement Between Queues

Processes move between queues based on state transitions and resource needs.

### Typical Process Flow

| Event                | Queue Transition               | Reason                       |
| -------------------- | ------------------------------ | ---------------------------- |
| Process Created      | → Job Queue → Ready Queue      | Process is ready to execute  |
| CPU Allocated        | Ready Queue → Running          | Scheduler selects process    |
| I/O Request          | Running → Device Queue         | Waiting for I/O operation    |
| I/O Completed        | Device Queue → Ready Queue     | Ready to resume execution    |
| Process Terminates   | Exits all queues               | Execution complete           |
| Swapped Out (memory) | Ready Queue → Disk (suspended) | Medium-term scheduler action |
| Swapped In           | Disk → Ready Queue             | Memory available again       |

### Full Process Journey Diagram

```
  New Process
      │
      ▼
  ┌──────────┐     admitted      ┌─────────────┐    CPU given    ┌─────────┐
  │ Job Queue│ ────────────────► │ Ready Queue │ ──────────────► │ Running │
  └──────────┘                   └─────────────┘                 └────┬────┘
                                        ▲                             │
                                        │   I/O done                  │ I/O request
                                        │                             ▼
                                        │                      ┌─────────────┐
                                        └──────────────────────│ Device Queue│
                                                               └─────────────┘
                                                                      │
                                                               (I/O completes)
                                                               goes back to ready
```

---

## 6. What Are Schedulers?

Schedulers are **OS components that decide which process runs next**. They select processes from queues and allocate system resources to them.

Like managers who decide which customer gets served next — ensuring fairness and efficient resource use.

There are **three types of schedulers**, each operating at a different time scale:

```
  ┌──────────────────────────────────────────────────────────────┐
  │                LONG-TERM SCHEDULER                           │
  │         Runs infrequently — controls who enters memory       │
  ├──────────────────────────────────────────────────────────────┤
  │                MEDIUM-TERM SCHEDULER                         │
  │         Runs occasionally — swaps processes in/out           │
  ├──────────────────────────────────────────────────────────────┤
  │                SHORT-TERM SCHEDULER                          │
  │         Runs every few ms — picks who gets CPU now           │
  └──────────────────────────────────────────────────────────────┘
```

---

## 7. Long-Term Scheduler (Job Scheduler)

The long-term scheduler decides **which processes get admitted into the ready queue** from the job queue. It controls the **degree of multiprogramming** — how many processes are in memory at once.

- Runs **infrequently** (every few minutes, or when a process terminates)
- Carefully balances **CPU-bound** and **I/O-bound** processes in memory
- Like the restaurant host deciding which waiting customers get seated based on table availability

**Key responsibility:** If too many processes are admitted, memory gets full. If too few, CPU sits idle.

---

## 8. Short-Term Scheduler (CPU Scheduler)

The short-term scheduler selects **which process from the ready queue gets the CPU next**. It runs very frequently — potentially every few milliseconds.

- Must be **extremely fast** — it runs constantly
- Uses algorithms based on priority, fairness, and efficiency
- Like the waiter deciding which table to serve next among all seated customers

**Key responsibility:** Every time the CPU becomes free (or a time slice expires), this scheduler picks the next process.

---

## 9. Medium-Term Scheduler

The medium-term scheduler **temporarily removes processes from memory to disk** (swap out), then **brings them back later** (swap in). This helps manage memory when the system is overloaded.

- Runs at a **moderate frequency**
- Also called the **Swapper**
- Like a restaurant manager asking waiting customers to step outside temporarily during peak hours, then inviting them back when space opens up

```
  Memory full → Swap out (move process to disk)
                     │
                     ▼
               [ Process on Disk ]
                     │
  Memory free  ← Swap in (bring process back)
```

---

## 10. Comparison of Schedulers

| Aspect              | Long-Term                      | Short-Term                      | Medium-Term                     |
| ------------------- | ------------------------------ | ------------------------------- | ------------------------------- |
| Also Known As       | Job Scheduler                  | CPU Scheduler                   | Swapper                         |
| Execution Frequency | Infrequent                     | Very Frequent                   | Moderate                        |
| Speed Required      | Can be slow                    | Must be very fast               | Moderate speed                  |
| Primary Function    | Admit processes to ready queue | Allocate CPU to ready processes | Swap processes in/out of memory |
| Controls            | Degree of multiprogramming     | CPU utilization                 | Memory management               |
| Presence            | Not in all systems             | Present in all systems          | Optional (swap-based systems)   |

---

## 11. How Schedulers Work Together

All three schedulers collaborate across different time scales:

```
  NEW PROCESS
       │
       ▼
  ┌─────────────────────────────────┐
  │      LONG-TERM SCHEDULER        │  ← Decides: admit or wait?
  │  (Gatekeeper — runs rarely)     │
  └──────────────┬──────────────────┘
                 │ admits process
                 ▼
          [ READY QUEUE ]
                 │
                 ▼
  ┌─────────────────────────────────┐
  │      SHORT-TERM SCHEDULER       │  ← Decides: who gets CPU now?
  │  (Every few milliseconds)       │
  └──────────────┬──────────────────┘
                 │ gives CPU
                 ▼
            [ RUNNING ]
                 │
       (if memory pressure)
                 ▼
  ┌─────────────────────────────────┐
  │      MEDIUM-TERM SCHEDULER      │  ← Decides: swap out to free memory?
  │  (Occasional — manages memory)  │
  └─────────────────────────────────┘
```

**Practical example:** You open Chrome, VLC, and Notepad simultaneously:

1. **Long-term** admits all three into memory → they enter the ready queue
2. **Short-term** rapidly switches between them every few ms → all appear to run at once
3. You open 20 more apps → memory is full → **medium-term** swaps Notepad to disk → you click Notepad → it swaps back in

---

## 12. CPU-Bound vs I/O-Bound Processes

Processes are classified by how they use resources — this helps schedulers make better decisions.

### CPU-Bound Processes

- Spend most time **performing computations**
- Rarely request I/O — keep the CPU busy for long stretches
- Examples: video encoding, scientific calculations, compression
- Benefit from **longer CPU time slices**

### I/O-Bound Processes

- Spend most time **waiting for I/O** operations
- Use CPU in **short bursts** between I/O requests
- Examples: text editors, database queries, web browsing, file copying
- Frequently move between **ready queue** and **device queues**

| Characteristic   | CPU-Bound                    | I/O-Bound                   |
| ---------------- | ---------------------------- | --------------------------- |
| Primary Resource | CPU time                     | I/O devices                 |
| Typical Behavior | Long CPU bursts              | Short CPU bursts            |
| Queue Residence  | Mostly in ready queue        | Frequently in device queues |
| Examples         | Video rendering, compression | Text editing, web browsing  |

**Why balance matters:** Long-term scheduler tries to maintain a **mix of both types** — CPU-bound processes keep the CPU busy while I/O-bound processes wait for devices, and vice versa.

---

## 13. Queue Implementation

Queues in the OS are typically implemented as **linked lists of PCB pointers**. This allows efficient insertion and deletion as processes move between states.

```c
// Simplified structure of a process queue node
struct QueueNode {
    PCB* process;      // Pointer to Process Control Block
    QueueNode* next;   // Pointer to next node
};

// Ready queue structure
struct ReadyQueue {
    QueueNode* front;  // First process in queue (next to get CPU)
    QueueNode* rear;   // Last process in queue (most recently added)
    int count;         // Number of processes waiting
};

// Adding a process to the ready queue
void enqueue(ReadyQueue* queue, PCB* process) {
    QueueNode* newNode = createNode(process);

    if (queue->rear == NULL) {
        // Queue is empty
        queue->front = queue->rear = newNode;
    } else {
        // Add to end of queue
        queue->rear->next = newNode;
        queue->rear = newNode;
    }
    queue->count++;
}
```

Real OS implementations are more complex — they include priority ordering, synchronization locks, and optimized data structures for fast insertion/removal.

---

## 13. Code Examples

> Working code that demonstrates job queue → ready queue → CPU → I/O queue → ready queue flow in practice.

### C++ — Simple Version

Three-queue model — processes flow from Job Queue through Ready Queue, run on CPU, then wait in I/O Queue.

```cpp
#include <iostream>
#include <queue>
#include <string>
#include <vector>
using namespace std;

struct Process {
    int    pid;
    string name;
    int    burst;      // CPU work remaining
    bool   needsIO;    // will it request I/O before finishing?
    string state;
};

int main() {
    // --- JOB QUEUE: processes waiting to be loaded into memory ---
    queue<Process> jobQueue;
    jobQueue.push({1, "Chrome",   5, true,  "NEW"});
    jobQueue.push({2, "VSCode",   3, false, "NEW"});
    jobQueue.push({3, "Terminal", 4, true,  "NEW"});

    cout << "=== Job Queue -> Ready Queue (Long-Term Scheduler) ===\n";
    queue<Process> readyQueue;  // processes in memory waiting for CPU
    while (!jobQueue.empty()) {
        Process p = jobQueue.front(); jobQueue.pop();
        p.state = "READY";
        readyQueue.push(p);
        cout << "Loaded PID " << p.pid << " ('" << p.name << "') into memory -> READY\n";
    }

    queue<Process> ioWaitQueue; // processes blocked on I/O
    int clock = 0;
    const int TIME_QUANTUM = 2;

    cout << "\n=== Ready Queue -> CPU -> I/O (Short-Term Scheduler) ===\n";

    while (!readyQueue.empty() || !ioWaitQueue.empty()) {
        // --- CPU: dispatch next ready process ---
        if (!readyQueue.empty()) {
            Process p = readyQueue.front(); readyQueue.pop();
            p.state = "RUNNING";
            int runFor = min(TIME_QUANTUM, p.burst);
            cout << "t=" << clock << "  PID " << p.pid
                 << " RUNNING (" << runFor << " units)\n";
            clock   += runFor;
            p.burst -= runFor;

            if (p.burst == 0) {
                p.state = "TERMINATED";
                cout << "t=" << clock << "  PID " << p.pid << " TERMINATED\n";
            } else if (p.needsIO) {
                // Needs I/O before it can continue
                p.state = "WAITING";
                p.needsIO = false;  // won't request I/O again
                ioWaitQueue.push(p);
                cout << "t=" << clock << "  PID " << p.pid
                     << " -> I/O WAIT queue\n";
            } else {
                // Time quantum expired, preempted back to ready
                p.state = "READY";
                readyQueue.push(p);
            }
        }

        // --- I/O completion: move one waiting process back to ready ---
        if (!ioWaitQueue.empty()) {
            Process p = ioWaitQueue.front(); ioWaitQueue.pop();
            p.state = "READY";
            readyQueue.push(p);
            cout << "t=" << clock << "  PID " << p.pid
                 << " I/O done -> READY queue\n";
        }
    }

    cout << "\nAll processes done at t=" << clock << "\n";
    return 0;
}
// Compile: g++ -std=c++17 queues.cpp -o queues
```

### C++ — Medium / LeetCode Style

Full scheduler with all three scheduler roles (long-term, short-term, medium-term swap).

```cpp
#include <iostream>
#include <queue>
#include <vector>
#include <string>
#include <algorithm>
using namespace std;

// Scheduler with three queues + medium-term swapping
// Time: O(total_burst), Space: O(n)

struct Process {
    int    pid;
    string name;
    int    burst;
    int    remaining;
    bool   needsIO;
    string state;
    int    arrivalTime;
    int    finishTime = 0;

    Process(int id, string n, int b, bool io, int at = 0)
        : pid(id), name(n), burst(b), remaining(b),
          needsIO(io), state("NEW"), arrivalTime(at) {}
};

class Scheduler {
    queue<Process*> jobQueue;    // NEW: waiting to be admitted
    queue<Process*> readyQueue;  // READY: waiting for CPU
    queue<Process*> ioQueue;     // WAITING: blocked on I/O
    vector<Process*> suspended;  // swapped to disk (medium-term)
    int clock = 0;
    const int QUANTUM     = 2;
    const int MEMORY_LIMIT = 2;  // max processes in memory at once
    int inMemory = 0;

public:
    void admit(Process* p) {
        // Long-term scheduler: let in if memory available, else suspend
        if (inMemory < MEMORY_LIMIT) {
            p->state = "READY";
            readyQueue.push(p);
            inMemory++;
            cout << "[LT] PID " << p->pid << " admitted to memory -> READY\n";
        } else {
            p->state = "SUSPENDED";
            suspended.push_back(p);
            cout << "[LT] PID " << p->pid
                 << " swapped to disk (memory full) -> SUSPENDED\n";
        }
    }

    void runOneCycle() {
        // Short-term: dispatch next ready process
        if (!readyQueue.empty()) {
            Process* p = readyQueue.front(); readyQueue.pop();
            p->state = "RUNNING";
            int runFor = min(QUANTUM, p->remaining);
            cout << "t=" << clock << "  PID " << p->pid
                 << " RUNNING (" << runFor << " units)\n";
            clock        += runFor;
            p->remaining -= runFor;

            if (p->remaining == 0) {
                p->state     = "TERMINATED";
                p->finishTime = clock;
                inMemory--;
                cout << "t=" << clock << "  PID " << p->pid << " TERMINATED\n";
                // Medium-term: swap in a suspended process
                if (!suspended.empty()) {
                    Process* s = suspended.back(); suspended.pop_back();
                    admit(s);
                }
            } else if (p->needsIO) {
                p->state = "WAITING"; p->needsIO = false;
                ioQueue.push(p);
            } else {
                p->state = "READY";
                readyQueue.push(p);
            }
        }
        // Simulate I/O completion
        if (!ioQueue.empty()) {
            Process* p = ioQueue.front(); ioQueue.pop();
            p->state = "READY";
            readyQueue.push(p);
            cout << "t=" << clock << "  PID " << p->pid << " I/O done -> READY\n";
        }
    }

    bool hasWork() {
        return !readyQueue.empty() || !ioQueue.empty();
    }
};

int main() {
    vector<Process> procs = {
        {1, "Chrome",   5, true},
        {2, "VSCode",   3, false},
        {3, "Terminal", 4, true},
        {4, "Slack",    2, false},
    };

    Scheduler sched;
    cout << "=== Admitting processes (Long-Term Scheduler) ===\n";
    for (auto& p : procs) sched.admit(&p);

    cout << "\n=== Running (Short + Medium Term) ===\n";
    while (sched.hasWork()) sched.runOneCycle();

    cout << "\nAll done.\n";
    return 0;
}
// Compile: g++ -std=c++17 full_scheduler.cpp -o full_scheduler
```

### Python — Simple Version

Process lifecycle through job queue → ready queue → CPU → I/O queue → ready queue.

```python
# Scheduling queues simulation
from collections import deque

class Process:
    def __init__(self, pid, name, burst, needs_io=False):
        self.pid      = pid
        self.name     = name
        self.burst    = burst
        self.needs_io = needs_io
        self.state    = "NEW"


# Three queues matching OS theory
job_queue  = deque([   # Long-term: waiting to enter memory
    Process(1, "Chrome",   5, needs_io=True),
    Process(2, "VSCode",   3, needs_io=False),
    Process(3, "Terminal", 4, needs_io=True),
])
ready_queue = deque()  # Short-term: in memory, waiting for CPU
io_queue    = deque()  # Device queue: blocked on I/O

# --- Long-Term Scheduler: admit all from job queue ---
print("=== Long-Term Scheduler: Job Queue -> Ready Queue ===")
while job_queue:
    p = job_queue.popleft()
    p.state = "READY"
    ready_queue.append(p)
    print(f"  PID {p.pid} ({p.name}) admitted -> READY")

TIME_QUANTUM = 2
clock = 0

# --- Short-Term Scheduler: dispatch from ready queue ---
print("\n=== Short-Term Scheduler ===")
while ready_queue or io_queue:
    if ready_queue:
        p = ready_queue.popleft()
        p.state  = "RUNNING"
        run_for  = min(TIME_QUANTUM, p.burst)
        print(f"t={clock:>3}  PID {p.pid} RUNNING ({run_for} units)")
        clock    += run_for
        p.burst  -= run_for

        if p.burst == 0:
            p.state = "TERMINATED"
            print(f"t={clock:>3}  PID {p.pid} TERMINATED")
        elif p.needs_io:
            p.state    = "WAITING"
            p.needs_io = False
            io_queue.append(p)
            print(f"t={clock:>3}  PID {p.pid} -> I/O WAIT")
        else:
            p.state = "READY"
            ready_queue.append(p)

    # Simulate I/O completing
    if io_queue:
        p = io_queue.popleft()
        p.state = "READY"
        ready_queue.append(p)
        print(f"t={clock:>3}  PID {p.pid} I/O done -> READY")

print(f"\nAll done at t={clock}")
```

### Python — Medium Level

Scheduler with all three tiers — long-term admission, short-term dispatch, medium-term swap — plus per-process stats.

```python
from collections import deque
from dataclasses import dataclass, field

@dataclass
class Process:
    pid:       int
    name:      str
    burst:     int
    needs_io:  bool = False
    state:     str  = "NEW"
    remaining: int  = field(init=False)
    finish_t:  int  = 0
    wait_t:    int  = 0

    def __post_init__(self):
        self.remaining = self.burst


class SchedulerSim:
    """Three-level scheduler simulation.
    Time: O(total_burst), Space: O(n)
    """
    MEMORY_LIMIT = 2  # max concurrent processes in memory

    def __init__(self):
        self.ready    : deque[Process] = deque()
        self.io_wait  : deque[Process] = deque()
        self.suspended: list[Process]  = []
        self.in_memory = 0
        self.clock     = 0

    def admit(self, p: Process) -> None:
        """Long-term: load into memory, or suspend to disk if full."""
        if self.in_memory < self.MEMORY_LIMIT:
            p.state = "READY"
            self.ready.append(p)
            self.in_memory += 1
            print(f"[LT] PID {p.pid} ({p.name}) -> READY (in memory)")
        else:
            p.state = "SUSPENDED"
            self.suspended.append(p)
            print(f"[LT] PID {p.pid} ({p.name}) -> SUSPENDED (disk)")

    def step(self, quantum: int = 2) -> None:
        """Short-term: dispatch, run one quantum, handle I/O."""
        if not self.ready:
            return

        p = self.ready.popleft()
        p.state   = "RUNNING"
        run_for   = min(quantum, p.remaining)
        print(f"t={self.clock:>3}  PID {p.pid} RUNNING ({run_for} units)")
        self.clock    += run_for
        p.remaining   -= run_for

        if p.remaining == 0:
            p.state   = "TERMINATED"
            p.finish_t = self.clock
            self.in_memory -= 1
            print(f"t={self.clock:>3}  PID {p.pid} TERMINATED  TAT={p.finish_t}")
            # Medium-term: swap in a suspended process
            if self.suspended:
                self.admit(self.suspended.pop())
        elif p.needs_io:
            p.state = "WAITING"; p.needs_io = False
            self.io_wait.append(p)
        else:
            p.state = "READY"
            self.ready.append(p)

        # Simulate I/O completion
        if self.io_wait:
            done = self.io_wait.popleft()
            done.state = "READY"
            self.ready.append(done)
            print(f"t={self.clock:>3}  PID {done.pid} I/O done -> READY")

    def has_work(self) -> bool:
        return bool(self.ready or self.io_wait)


if __name__ == "__main__":
    jobs = [
        Process(1, "Chrome",   5, needs_io=True),
        Process(2, "VSCode",   3, needs_io=False),
        Process(3, "Terminal", 4, needs_io=True),
        Process(4, "Slack",    2, needs_io=False),
    ]

    sim = SchedulerSim()
    print("=== Admitting processes ===")
    for j in jobs:
        sim.admit(j)

    print("\n=== Running ===")
    while sim.has_work():
        sim.step(quantum=2)
```

---

## 14. Key Takeaways

- **Three queues:** Job queue (all processes), ready queue (waiting for CPU), device queues (waiting for I/O)
- **Processes move** between queues based on state — created → ready → running → I/O wait → ready → terminated
- **Three schedulers** work at different time scales and handle different concerns:
  - Long-term: who enters memory (rarely runs)
  - Short-term: who gets CPU now (runs every few ms)
  - Medium-term: who gets swapped to disk when memory is tight (optional)
- **Short-term scheduler must be fast** — it runs constantly; a slow scheduler wastes CPU time on scheduling instead of running processes
- **Balance CPU-bound and I/O-bound** processes to keep both CPU and I/O devices busy
- Queues are implemented as **linked lists of PCB pointers** for efficient insertion and removal
- These concepts directly lead into **CPU scheduling algorithms** — the rules the short-term scheduler uses to pick processes
