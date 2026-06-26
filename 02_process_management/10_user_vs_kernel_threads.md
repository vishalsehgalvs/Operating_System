# User vs Kernel Threads: Multithreading Models

> Threads can be managed in user space (fast but invisible to the OS) or kernel space (slower but fully parallel) — modern OSes use a one-to-one model where each thread maps directly to a kernel thread, giving the best balance of performance and parallelism.

---

## Table of Contents

1. [Understanding Thread Management](#1-understanding-thread-management)
2. [User-Level Threads](#2-user-level-threads)
3. [Kernel-Level Threads](#3-kernel-level-threads)
4. [Comparison: User-Level vs Kernel-Level](#4-comparison-user-level-vs-kernel-level)
5. [Hybrid Multithreading Models](#5-hybrid-multithreading-models)
6. [Practical Example: Web Server](#6-practical-example-web-server)
7. [Choosing the Right Model](#7-choosing-the-right-model)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Understanding Thread Management

Thread management involves: creating threads, scheduling them, switching between them, and terminating them. This management can happen at **two places**:

| Layer        | Who manages threads   | Analogy                                           |
| ------------ | --------------------- | ------------------------------------------------- |
| User space   | Thread library in app | Team members dividing work among themselves       |
| Kernel space | OS kernel directly    | Manager directly assigning tasks to each employee |

```
  ┌─────────────────────────────────────────────────────┐
  │                USER SPACE                           │
  │   Thread Library (Pthreads, Java Threads, etc.)     │
  │   Thread 1   Thread 2   Thread 3   Thread 4         │
  ├─────────────────────────────────────────────────────┤
  │              KERNEL SPACE                           │
  │   Kernel Thread(s) — what the OS actually sees      │
  └─────────────────────────────────────────────────────┘
```

The relationship between user-level threads and kernel threads defines the **threading model**.

---

## 2. User-Level Threads

### What They Are

User-level threads are managed entirely by a **thread library in user space**. The OS kernel has **no knowledge** they exist — it sees only a single process with a single thread.

**Group project analogy:** Team members divide tasks among themselves without telling the teacher. The teacher thinks only one student is working, but internally many collaborate and switch tasks.

### How They Work

1. A runtime library (e.g., POSIX Pthreads in user mode, Java Green Threads) handles all thread operations
2. The library maintains a **thread table in user space** tracking each thread's state, registers, and stack
3. The library runs **its own scheduler** to decide which thread runs next
4. Context switches happen **entirely in user space** — no mode switch needed

```
  USER SPACE
  ┌─────────────────────────────────────────────────┐
  │  Thread Library (scheduler + thread table)       │
  │                                                  │
  │  Thread A  Thread B  Thread C  Thread D          │
  │  [stack]   [stack]   [stack]   [stack]           │
  └──────────────────────┬──────────────────────────┘
                         │ (one single-threaded process)
  KERNEL SPACE           ▼
  ┌─────────────────────────────────────────────────┐
  │               Kernel Thread 1                   │
  └─────────────────────────────────────────────────┘
```

### Advantages

| Advantage                | Why                                             |
| ------------------------ | ----------------------------------------------- |
| Very fast creation       | No system call needed — only a few microseconds |
| Very fast context switch | No mode switch to kernel — all in user space    |
| Highly portable          | Same library works across Windows, Linux, macOS |

### Disadvantages

| Disadvantage                   | Why                                                                              |
| ------------------------------ | -------------------------------------------------------------------------------- |
| Cannot use multiple CPU cores  | Kernel sees only 1 thread → entire process on 1 core                             |
| One blocking thread blocks ALL | Kernel blocks the whole process on any blocking system call                      |
| Hard to preempt                | Library can't receive clock interrupts easily — threads must cooperatively yield |

```
  BLOCKING PROBLEM:

  Thread A → makes file read() system call
  Kernel: "Oh, process is blocked" → BLOCKS ENTIRE PROCESS
  Thread B, C, D: also stuck, even though they had no I/O
```

---

## 3. Kernel-Level Threads

### What They Are

Kernel-level threads are managed directly by the **OS kernel**. The kernel maintains a thread table and each thread is a **separate schedulable entity** known to the OS.

**Manager analogy:** A company manager directly assigns tasks to each employee, tracks their progress, and decides who works on what — complete visibility and control.

### How They Work

1. Creating a thread makes a **system call to the kernel**
2. Kernel creates a new entry in its thread table, allocates kernel resources
3. Kernel scheduler treats each thread as **independently schedulable**
4. Different threads from the same process can run on **different CPU cores simultaneously**

```
  USER SPACE
  ┌──────────────────────────────────────────────────┐
  │  Thread A    Thread B    Thread C    Thread D     │
  └────┬─────────────┬────────────┬──────────────────┘
       │             │            │
  KERNEL SPACE       │            │
  ┌────▼─────────────▼────────────▼──────────────────┐
  │  Kernel T1   Kernel T2   Kernel T3                │
  │     ↓             ↓            ↓                  │
  │  CPU Core 1  CPU Core 2  CPU Core 3               │
  └──────────────────────────────────────────────────┘
  TRUE PARALLEL EXECUTION
```

### Advantages

| Advantage                 | Why                                                        |
| ------------------------- | ---------------------------------------------------------- |
| True multiprocessing      | Kernel schedules threads on different cores simultaneously |
| Blocking is isolated      | One thread blocks → others in same process keep running    |
| Preemptive scheduling     | Kernel scheduler handles time slicing for threads          |
| Better system integration | Works with signals, kernel services, I/O directly          |

### Disadvantages

| Disadvantage                | Why                                                                       |
| --------------------------- | ------------------------------------------------------------------------- |
| Slower creation/destruction | Each operation requires a system call                                     |
| Slower context switch       | Requires entering kernel mode, saving kernel data structures              |
| Higher memory use           | Each thread needs kernel memory for TCB and kernel stack                  |
| Scalability limit           | OS may cap threads per process; thousands of threads strain kernel memory |

---

## 4. Comparison: User-Level vs Kernel-Level

| Aspect               | User-Level Threads               | Kernel-Level Threads                  |
| -------------------- | -------------------------------- | ------------------------------------- |
| Management           | Thread library in user space     | OS kernel                             |
| Kernel Awareness     | Kernel unaware of threads        | Kernel fully aware                    |
| Creation Speed       | Very fast (microseconds)         | Slower (system call overhead)         |
| Context Switch Speed | Very fast (no mode switch)       | Slower (requires kernel mode)         |
| Multicore Support    | Cannot use multiple cores        | Can use all available cores           |
| Blocking Behavior    | One blocking thread blocks all   | Only blocking thread pauses           |
| Scheduling           | Library-based, often cooperative | Kernel preemptive scheduling          |
| Portability          | Highly portable                  | OS-dependent                          |
| Memory Overhead      | Lower (only user space)          | Higher (kernel structures per thread) |

---

## 5. Hybrid Multithreading Models

Modern OSes often combine both approaches, mapping multiple user-level threads onto a number of kernel threads.

### Many-to-One Model

```
  User threads:   T1  T2  T3  T4  T5
                   \  |  /  |  /
                    \ | /   |/
  Kernel thread:    KT1 (only one)
```

- Many user threads → 1 kernel thread
- Pure user-level threading — fast but no multiprocessing
- If any thread blocks, **entire process blocks**
- Used in early systems (e.g., GNU Portable Threads) — **rarely used today**

### One-to-One Model

```
  User threads:   T1   T2   T3   T4
                   |    |    |    |
  Kernel threads: KT1  KT2  KT3  KT4
                   |    |    |    |
  CPU cores:     Core1 Core2 Core1 Core2  (scheduled by OS)
```

- Each user thread → 1 kernel thread
- True parallel execution across all cores
- One blocking thread doesn't affect others
- **Modern standard** — used by Windows, Linux (pthreads), macOS
- Trade-off: higher overhead per thread; OS may limit thread count

### Many-to-Many Model

```
  User threads:   T1  T2  T3  T4  T5  T6
                   \  |  / \  |  /
  Kernel threads:   KT1    KT2    KT3
                     |      |      |
  CPU cores:       Core1  Core2  Core3
```

- Many user threads → smaller/equal number of kernel threads
- Kernel thread count adjusts dynamically based on needs
- Best of both: fast user-level ops + multicore use + blocking isolation
- More complex to implement
- Used in Solaris; less common in modern Linux/Windows which prefer one-to-one

### Model Summary Table

| Model        | User:Kernel | Multicore? | Blocking Isolation | Complexity |
| ------------ | ----------- | ---------- | ------------------ | ---------- |
| Many-to-One  | N:1         | No         | No                 | Low        |
| One-to-One   | 1:1         | Yes        | Yes                | Medium     |
| Many-to-Many | M:N         | Yes        | Yes                | High       |

---

## 6. Practical Example: Web Server

A web server handles many client connections, each in a separate thread:

### With User-Level Threads

- Create thousands of threads quickly with minimal overhead
- Thread switching is extremely fast
- **BUT** — all run on one CPU core, and any blocking I/O stalls all requests
- Not suitable for I/O-heavy or multicore servers

### With Kernel-Level Threads (one-to-one)

- Threads run on different CPU cores simultaneously → genuine parallel request handling
- One thread blocking on disk I/O doesn't stall others
- **BUT** — creating thousands of threads strains kernel memory; typically use a **thread pool** instead

### With Hybrid (Many-to-Many)

- Thread pool of kernel threads (e.g., one or two per CPU core)
- Many lightweight user tasks mapped onto these kernel threads
- Efficient: no kernel explosion, no blocking issue, uses all cores

```
  Client requests: C1 C2 C3 C4 C5 C6 C7 C8 ...
                    \  |  / \  |  / \  |  /
  User tasks:        U1  U2   U3  U4   U5  U6
                      \  /     \  /     \  /
  Kernel threads:      KT1      KT2      KT3
                        |        |        |
  CPU cores:          Core1    Core2    Core3
```

---

## 7. Choosing the Right Model

| Need                                                    | Best Choice                                                 |
| ------------------------------------------------------- | ----------------------------------------------------------- |
| Very frequent thread creation/destruction + portability | User-level threads                                          |
| Full multicore utilization + blocking I/O safety        | Kernel-level threads (one-to-one)                           |
| Thousands of lightweight concurrent tasks               | Hybrid (M:N) — e.g., Go goroutines on top of kernel threads |
| Most modern desktop/server applications                 | Kernel-level (one-to-one) — default in Linux, Windows       |

**Real-world examples:**

- **POSIX Pthreads (Linux):** One-to-one kernel threads
- **Windows Threads:** One-to-one kernel threads
- **Go goroutines:** Many-to-many (M goroutines on N kernel threads, managed by Go runtime)
- **Java Virtual Threads (Java 21+):** Many-to-many (lightweight virtual threads on kernel threads)

---

## 7. Code Examples

> Working code that demonstrates Many-to-One, One-to-One, and Many-to-Many threading models in practice.

### C++ — Simple Version
Simulate Many-to-One (N user threads share 1 kernel thread) and One-to-One (each user thread has its own kernel thread).

```cpp
#include <iostream>
#include <vector>
#include <string>
using namespace std;

// --- Model 1: Many-to-One (N:1) ---
// All user threads are managed in user space.
// Only one kernel thread exists — OS sees only one thread.
// If any user thread blocks on I/O, ALL threads are blocked.

struct UserThread {
    int id;
    string task;
    bool blocked = false;  // true if waiting on I/O
};

void manyToOne(vector<UserThread> threads) {
    cout << "=== Many-to-One Model (N user threads -> 1 kernel thread) ===\n";
    cout << "Kernel thread: 1 (OS sees only this)\n\n";

    for (auto& ut : threads) {
        if (ut.blocked) {
            cout << "  UserThread " << ut.id << " blocked on I/O\n";
            cout << "  *** ALL threads blocked! Kernel thread can't run others ***\n";
            break;
        }
        cout << "  UserThread " << ut.id << " running: " << ut.task << "\n";
        // Note: only runs one at a time — no true parallelism possible
    }
}

// --- Model 2: One-to-One (1:1) ---
// Each user thread maps to its own kernel thread.
// OS manages each thread independently.
// One blocking thread doesn't block others.
// True parallelism on multi-core CPUs.

struct KernelThread {
    int id;
    int userThreadId;  // which user thread it serves
};

void oneToOne(vector<UserThread> threads) {
    cout << "\n=== One-to-One Model (each user thread -> own kernel thread) ===\n";
    cout << "Kernel threads: " << threads.size() << " (one per user thread)\n\n";

    vector<KernelThread> kthreads;
    for (auto& ut : threads)
        kthreads.push_back({ut.id, ut.id});

    for (int i = 0; i < (int)threads.size(); i++) {
        if (threads[i].blocked) {
            cout << "  KThread " << kthreads[i].id
                 << " blocked (UserThread " << threads[i].id << " on I/O)\n";
            cout << "  -> Other kernel threads keep running! No global block.\n";
        } else {
            cout << "  KThread " << kthreads[i].id
                 << " running: " << threads[i].task << "\n";
        }
    }
}

int main() {
    vector<UserThread> threads = {
        {1, "handle request A", false},
        {2, "read from disk",   true},   // this one blocks
        {3, "compute hash",     false},
    };

    manyToOne(threads);
    oneToOne(threads);
    return 0;
}
// Compile: g++ -std=c++17 thread_models.cpp -o thread_models
```

### C++ — Medium / LeetCode Style
Simulate all three models (Many-to-One, One-to-One, Many-to-Many) with a scheduler showing how user threads map to kernel threads.

```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <string>
using namespace std;

// Thread model simulation: how user threads map to kernel threads
// Time: O(n * tasks), Space: O(n + k)

struct UserThread {
    int    id;
    string task;
    bool   isBlocked;   // simulates I/O wait
    int    mappedKThread = -1;  // which kernel thread serves it (-1 = none)
};

struct KernelThread {
    int  id;
    bool busy = false;
    int  servingUT = -1;  // user thread currently on this KT
};

// --- Many-to-One: all UTs share 1 KT ---
void manyToOne(vector<UserThread> uts) {
    cout << "=== Many-to-One (N:" << uts.size() << " -> 1:1 KT) ===\n";
    KernelThread kt{0};
    for (auto& ut : uts) {
        if (ut.isBlocked) {
            cout << "  UT" << ut.id << " blocked -> KT0 blocked -> ALL UTs blocked!\n";
            return;  // entire process stalls
        }
        cout << "  UT" << ut.id << " running on KT0: " << ut.task << "\n";
        // Sequential — no parallelism
    }
}

// --- One-to-One: each UT gets its own KT ---
void oneToOne(vector<UserThread> uts) {
    cout << "\n=== One-to-One (1:1) === [" << uts.size() << " kernel threads] ===\n";
    vector<KernelThread> kts(uts.size());
    for (int i = 0; i < (int)kts.size(); i++) kts[i].id = i;

    for (int i = 0; i < (int)uts.size(); i++) {
        uts[i].mappedKThread = i;
        if (uts[i].isBlocked)
            cout << "  UT" << uts[i].id << " blocked on KT" << i
                 << " -> other KTs unaffected\n";
        else
            cout << "  UT" << uts[i].id << " runs on KT" << i
                 << ": " << uts[i].task << "\n";
    }
}

// --- Many-to-Many: M user threads on N kernel threads (M > N) ---
// The runtime multiplexes UTs onto available KTs
// If a UT blocks, its KT picks up another UT
void manyToMany(vector<UserThread> uts, int numKThreads) {
    cout << "\n=== Many-to-Many (" << uts.size() << " UTs on "
         << numKThreads << " KTs) ===\n";

    vector<KernelThread> kts(numKThreads);
    for (int i = 0; i < numKThreads; i++) kts[i].id = i;

    queue<int> freeKTs;
    for (int i = 0; i < numKThreads; i++) freeKTs.push(i);

    for (auto& ut : uts) {
        if (freeKTs.empty()) {
            cout << "  UT" << ut.id << " waiting (all KTs busy)...\n";
            freeKTs.push(0);  // simulate one KT becoming free
        }
        int kt = freeKTs.front(); freeKTs.pop();
        if (ut.isBlocked) {
            cout << "  UT" << ut.id << " -> KT" << kt
                 << " blocked, runtime reschedules KT" << kt << " to next UT\n";
            freeKTs.push(kt);  // KT freed immediately (runtime handles block)
        } else {
            cout << "  UT" << ut.id << " runs on KT" << kt
                 << ": " << ut.task << "\n";
        }
    }
}

int main() {
    vector<UserThread> uts = {
        {1, "handle request A", false},
        {2, "read from disk",   true },   // blocks
        {3, "compute hash",     false},
        {4, "render frame",     false},
    };

    manyToOne(uts);
    oneToOne(uts);
    manyToMany(uts, 2);  // 4 user threads, only 2 kernel threads
    return 0;
}
// Compile: g++ -std=c++17 thread_models_all.cpp -o thread_models_all
```

### Python — Simple Version
Mock UserThread and KernelThread classes — show how Many-to-One vs One-to-One behave when one thread blocks.

```python
# User-level vs Kernel-level thread model simulation

class UserThread:
    def __init__(self, uid, task, blocks=False):
        self.uid    = uid
        self.task   = task
        self.blocks = blocks   # simulates I/O blocking

class KernelThread:
    def __init__(self, kid):
        self.kid    = kid
        self.busy   = False


# --- Many-to-One: ALL user threads share ONE kernel thread ---
def many_to_one(user_threads: list[UserThread]) -> None:
    print("=== Many-to-One (N user threads -> 1 kernel thread) ===")
    kt = KernelThread(0)

    for ut in user_threads:
        if ut.blocks:
            print(f"  UT{ut.uid} blocked on I/O -> KT0 blocked")
            print("  *** Whole process stalled! No other UT can run. ***")
            return
        print(f"  UT{ut.uid} runs on KT0: {ut.task}")


# --- One-to-One: each user thread has its own kernel thread ---
def one_to_one(user_threads: list[UserThread]) -> None:
    print("\n=== One-to-One (each user thread -> own kernel thread) ===")
    kthreads = [KernelThread(i) for i in range(len(user_threads))]

    for i, ut in enumerate(user_threads):
        kt = kthreads[i]
        if ut.blocks:
            print(f"  UT{ut.uid} blocked on KT{kt.kid} -> other KTs keep running!")
        else:
            print(f"  UT{ut.uid} runs on KT{kt.kid}: {ut.task}")


# Demo
threads = [
    UserThread(1, "handle request A", blocks=False),
    UserThread(2, "read from disk",   blocks=True),   # this one blocks
    UserThread(3, "compute hash",     blocks=False),
]

many_to_one(threads)
one_to_one(threads)
```

### Python — Medium Level
Simulate all three models with a Many-to-Many runtime that reschedules kernel threads when a user thread blocks.

```python
from collections import deque
from dataclasses import dataclass, field

@dataclass
class UserThread:
    uid:    int
    task:   str
    blocks: bool = False
    mapped_kt: int = -1

@dataclass
class KernelThread:
    kid:  int
    busy: bool = False


def many_to_one(uts: list[UserThread]) -> None:
    """All UTs share one KT. One block = everyone blocks."""
    print("=== Many-to-One ===")
    for ut in uts:
        if ut.blocks:
            print(f"  UT{ut.uid} I/O block -> KT0 blocked -> ALL UTs stalled!")
            return
        print(f"  UT{ut.uid} -> KT0 : {ut.task}")


def one_to_one(uts: list[UserThread]) -> None:
    """Each UT gets its own KT. One block doesn't affect others."""
    print("\n=== One-to-One ===")
    kts = [KernelThread(i) for i in range(len(uts))]
    for i, ut in enumerate(uts):
        ut.mapped_kt = i
        if ut.blocks:
            print(f"  UT{ut.uid} -> KT{i} blocked  (others unaffected)")
        else:
            print(f"  UT{ut.uid} -> KT{i} running : {ut.task}")


def many_to_many(uts: list[UserThread], num_kts: int) -> None:
    """M user threads multiplexed onto N kernel threads (M > N).
    When a UT blocks, its KT is reassigned to another UT.
    Time: O(n), Space: O(n + num_kts)
    """
    print(f"\n=== Many-to-Many ({len(uts)} UTs on {num_kts} KTs) ===")
    free_kts: deque[int] = deque(range(num_kts))  # available kernel threads

    for ut in uts:
        if not free_kts:
            print(f"  UT{ut.uid} waiting (all {num_kts} KTs busy)...")
            free_kts.append(0)  # simulate one completing

        kt_id = free_kts.popleft()

        if ut.blocks:
            # Runtime detects block, reschedules this KT to next ready UT
            print(f"  UT{ut.uid} -> KT{kt_id} blocked -> runtime recycles KT{kt_id}")
            free_kts.appendleft(kt_id)  # KT freed immediately
        else:
            print(f"  UT{ut.uid} -> KT{kt_id} running : {ut.task}")
            # KT stays busy until UT finishes; simplified: free it after


if __name__ == "__main__":
    threads = [
        UserThread(1, "handle request A"),
        UserThread(2, "read from disk",   blocks=True),
        UserThread(3, "compute hash"),
        UserThread(4, "render frame"),
    ]

    many_to_one(threads)
    one_to_one(threads)
    many_to_many(threads, num_kts=2)   # 4 UTs on only 2 KTs
```

---

## 8. Key Takeaways

- **User-level threads:** Managed by library in user space — kernel blind to them. Fast creation and switching, but can't use multiple cores and one blocking call freezes all threads
- **Kernel-level threads:** Managed by OS — kernel fully aware. Slower creation/switching, but true multicore parallelism and isolated blocking
- **Three models:**
  - Many-to-One (N:1): Fast, no parallelism — legacy
  - One-to-One (1:1): True parallel, slight overhead — **modern standard**
  - Many-to-Many (M:N): Best of both, complex — used in Go, Java virtual threads
- **Why one-to-one dominates:** Kernel thread overhead has dropped; simplicity outweighs the cost for most apps
- **Why goroutines/virtual threads use M:N:** Millions of lightweight tasks can't each have a kernel thread — the M:N model gives concurrency without kernel explosion
- This topic completes **Section 2: Process Management** — Section 3 is Synchronization and Concurrency, which builds directly on threads
