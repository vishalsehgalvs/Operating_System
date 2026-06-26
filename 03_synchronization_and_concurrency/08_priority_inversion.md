# Priority Inversion in OS

> Priority inversion happens when a high-priority process is forced to wait for a low-priority process to release a shared resource — made worse when medium-priority processes preempt the lock holder, causing the highest-priority task to run last; it is fixed by temporarily boosting the lock holder's priority (Priority Inheritance or Priority Ceiling Protocol).

---

## Table of Contents

1. [What Is Priority Inversion?](#1-what-is-priority-inversion)
2. [Step-by-Step Scenario](#2-step-by-step-scenario)
3. [Why It Is Dangerous](#3-why-it-is-dangerous)
4. [Famous Example: Mars Pathfinder (1997)](#4-famous-example-mars-pathfinder-1997)
5. [Causes of Priority Inversion](#5-causes-of-priority-inversion)
6. [Solution 1 — Priority Inheritance Protocol (PIP)](#6-solution-1--priority-inheritance-protocol-pip)
7. [Solution 2 — Priority Ceiling Protocol (PCP)](#7-solution-2--priority-ceiling-protocol-pcp)
8. [Comparison of Solutions](#8-comparison-of-solutions)
9. [Semaphore Code Example](#9-semaphore-code-example)
10. [Priority Inversion vs Starvation vs Deadlock](#10-priority-inversion-vs-starvation-vs-deadlock)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What Is Priority Inversion?

**Priority inversion** is a scheduling anomaly where a **high-priority process is forced to wait** behind a lower-priority process because the lower-priority process holds a resource (mutex/semaphore) that the high-priority process needs.

**VIP payment machine analogy:**

```
  VIP customer (P_high) arrives at coffee shop
  Wants to pay → payment machine is IN USE
  Regular customer (P_low) is using the machine
  Another customer (P_medium) cuts in line while P_low uses machine

  VIP must wait for:
    1. P_medium to finish (doesn't even use the machine!)
    2. P_low to finish and release the machine

  The HIGHEST priority customer runs LAST → priority inversion!
```

In normal priority scheduling:

```
  Expected:  P_high → P_medium → P_low
  Actual:    P_low (holds lock) ... P_medium (preempts) ... P_low resumes ... P_high

  P_high effectively runs after P_medium — priorities are INVERTED
```

---

## 2. Step-by-Step Scenario

**Three processes, one shared resource R:**

| Process | Priority           | Resource R      |
| ------- | ------------------ | --------------- |
| P1      | Low (1 = lowest)   | Holds lock on R |
| P2      | Medium (2)         | Does NOT need R |
| P3      | High (3 = highest) | Needs lock on R |

**Timeline without any fix:**

```
  TIME ──────────────────────────────────────────────────────────────►

  t=0:  P1 starts, acquires lock on R
        ─────[P1 holds R]──────────────────────────────────────────►

  t=1:  P3 arrives, tries to acquire R → R is locked → P3 BLOCKS
        P3: WAITING ──────────────────────────────────────────────►

  t=2:  P2 arrives, priority > P1 → P2 PREEMPTS P1
        P1 is paused (still holds lock!)
        ─────────────────────[P2 runs (no R needed)]──────────────►

  t=4:  P2 finishes → P1 resumes (now highest ready priority)
        ──────────────────────────────[P1 resumes, releases R]────►

  t=5:  P3 finally acquires R and runs
        ────────────────────────────────────────────[P3 runs]─────►

  Execution order: P1 → P2 → P1 → P3
  P3 (highest priority!) runs LAST — after P2 (medium priority)!
```

```mermaid
sequenceDiagram
    participant P3 as P3 (High)
    participant P2 as P2 (Medium)
    participant P1 as P1 (Low)
    participant R as Resource R

    P1->>R: acquire lock ✓
    P3->>R: acquire lock → BLOCKED (held by P1)
    Note over P3: P3 waits...
    P2->>P1: preempts P1 (P2 priority > P1)
    Note over P1: P1 paused, still holds lock!
    P2->>P2: runs to completion (doesn't need R)
    Note over P1: P1 resumes
    P1->>R: release lock
    P3->>R: acquire lock ✓
    Note over P3: P3 finally runs — LAST!
```

---

## 3. Why It Is Dangerous

In normal systems, a delayed high-priority task is merely an inconvenience. In **real-time systems**, it is catastrophic:

```
  Real-Time System Examples:
  ──────────────────────────────────────────────────────────────────
  Anti-lock Brake System (ABS):
    High-priority: Brake control (must respond in <5ms)
    Low-priority:  Sensor logging (holds shared buffer)
    Medium-priority: Diagnostics (preempts sensor logging)
    → Priority inversion: brake response delayed → accident

  Airbag Controller:
    High-priority: Airbag deployment (must fire within milliseconds)
    Low-priority:  Data recording (holds memory lock)
    → Priority inversion: airbag fires too late

  Spacecraft:
    High-priority: Communication system (see Mars Pathfinder)
    → System resets, mission at risk
```

**Two failure modes:**

1. **Bounded inversion** — P_high waits only until P_low releases the lock (acceptable)
2. **Unbounded inversion** — P_medium keeps preempting P_low, delaying P_high indefinitely (dangerous)

---

## 4. Famous Example: Mars Pathfinder (1997)

```
  MARS PATHFINDER — Priority Inversion in Space

  OS: VxWorks (real-time OS)
  Problem discovered: July 1997, days after landing

  Processes involved:
    High:   ASI/MET communication bus task (must run frequently)
    Low:    Meteorological data gathering (held shared semaphore)
    Medium: Several background tasks (kept preempting low task)

  What happened:
    1. Low-priority met task acquired shared semaphore
    2. High-priority bus task tried to acquire same semaphore → BLOCKED
    3. Medium-priority tasks preempted the met task repeatedly
    4. Bus task missed its deadline → watchdog timer detected it
    5. Watchdog reset the entire system
    6. Repeated resets → mission threatened

  Fix applied remotely from Earth:
    Engineers enabled the "priority inheritance" flag
    in VxWorks semaphore creation
    → Problem resolved with a one-line config change!
```

This incident made priority inversion famous and is now a standard case study in real-time OS design.

---

## 5. Causes of Priority Inversion

Priority inversion requires all of these conditions to occur together:

```
  1. Priority-based preemptive scheduling
         (the scheduler can interrupt a lower-priority task)

  2. Shared resource protected by a lock (mutex/semaphore)
         (only one process can hold it at a time)

  3. A LOW-priority process acquires the resource first

  4. A HIGH-priority process requests the same resource
         (and blocks, waiting for the lock)

  5. A MEDIUM-priority process arrives and preempts the low-priority process
         (which still holds the lock!)

  Result: HIGH waits for MEDIUM, even though MEDIUM doesn't use the resource
```

**Without condition 5** (no medium-priority process), the inversion is **bounded** — P_high waits only as long as P_low's critical section, which is usually short and acceptable.

**With condition 5**, the inversion is **unbounded** — P_high waits for all medium-priority processes to finish, which can be arbitrarily long.

---

## 6. Solution 1 — Priority Inheritance Protocol (PIP)

**Idea:** When a high-priority process blocks on a resource, the process currently holding that resource **inherits the high priority temporarily**.

```
  Rule: If P_high blocks waiting for a lock held by P_low,
        then P_low's effective priority = max(P_low.priority, P_high.priority)
        until P_low releases the lock.
```

**Same scenario WITH Priority Inheritance:**

```
  t=0:  P1 (priority 1) acquires lock on R

  t=1:  P3 (priority 3) tries to acquire R → BLOCKED
        *** P1 INHERITS priority 3 from P3 ***
        P1's effective priority = 3 (was 1)

  t=2:  P2 (priority 2) arrives
        P2 tries to preempt P1... but P1 now has priority 3!
        P2 (priority 2) CANNOT preempt P1 (priority 3) → P2 waits

  t=3:  P1 finishes critical section, releases lock
        P1 returns to original priority 1

  t=4:  P3 acquires lock, runs (highest ready priority)

  t=5:  P2 runs

  Execution order: P1 (boosted) → P3 → P2  ✓  Correct!
  P3 runs before P2, as intended.
```

```
  Priority Timeline:

  P1: ─[prio=1]─[BOOST to prio=3]─────────[release, back to prio=1]─►
  P2: ─────────────────────────────[wait]──[runs]─────────────────────►
  P3: ─────────[blocks on R]───────────────[runs after P1 releases]───►
                     ▲
                     P1 inherits P3's priority here
```

**Limitation:** With multiple locks and multiple processes, priority can chain through several hops (transitive inheritance). Implementation is complex.

---

## 7. Solution 2 — Priority Ceiling Protocol (PCP)

**Idea:** Each resource is pre-assigned a **priority ceiling** = the highest priority of any process that might ever lock it. When any process locks the resource, it **immediately inherits the ceiling priority** — even before any high-priority process blocks.

```
  Setup:
  Resource R → ceiling priority = 3  (P3 is the highest-priority user of R)

  Rule: When any process acquires R, its effective priority = max(own, ceiling)
```

**Same scenario WITH Priority Ceiling:**

```
  t=0:  P1 (priority 1) acquires lock on R
        *** P1 IMMEDIATELY gets priority 3 (ceiling of R) ***

  t=1:  P3 tries to acquire R → BLOCKED (P1 holds it at priority 3)
        P2 (priority 2) cannot preempt P1 (now priority 3)

  t=2:  P1 finishes critical section, releases lock
        P1 returns to original priority 1

  t=3:  P3 acquires lock, runs

  t=4:  P2 runs

  Execution order: P1 (at ceiling) → P3 → P2  ✓  Correct!
  Bonus: No blocking on P3's part when it finally gets the lock
```

**Extra benefit:** PCP can prevent **deadlocks** — because a process can only acquire a resource if its priority is strictly higher than the ceiling of all currently locked resources, preventing circular lock acquisition.

**Limitation:** You must know in advance which processes will use which resources, so the ceiling can be pre-assigned.

---

## 8. Comparison of Solutions

| Aspect                     | Priority Inheritance (PIP)        | Priority Ceiling (PCP)                |
| -------------------------- | --------------------------------- | ------------------------------------- |
| When priority changes      | When high-priority process BLOCKS | Immediately upon resource acquisition |
| Knowledge required upfront | None — purely reactive            | Must know resource usage patterns     |
| Blocking time for P_high   | Can be longer (waits for P_low)   | Shorter and more predictable          |
| Implementation complexity  | Moderate (handle chains)          | Higher (pre-assign ceilings)          |
| Prevents deadlock?         | No                                | Yes (in some formulations)            |
| CPU overhead               | Lower                             | Slightly higher                       |
| Best for                   | General-purpose systems           | Hard real-time, safety-critical       |
| Real-world example         | Mars Pathfinder fix (VxWorks)     | RTOS, automotive, aerospace           |

---

## 9. Semaphore Code Example

```c
// Shared resource protected by semaphore
semaphore mutex = 1;

// ── WITHOUT priority inheritance ─────────────────────────────────
// P1 (Low priority = 1)
void P1() {
    wait(mutex);              // Acquires lock
    perform_long_operation(); // Critical section
    signal(mutex);            // Releases lock
}

// P2 (Medium priority = 2) — doesn't need mutex
void P2() {
    perform_computation();    // Preempts P1 while P1 holds mutex
}

// P3 (High priority = 3)
void P3() {
    wait(mutex);   // BLOCKS — held by P1
    urgent_task(); // Critical section
    signal(mutex); // Releases lock
}

// Timeline WITHOUT inheritance:
//   P1 locks → P3 blocks → P2 preempts P1 → P2 runs →
//   P1 resumes → P1 unlocks → P3 runs
//   P3 waited for BOTH P1 and P2!

// ── WITH priority inheritance ─────────────────────────────────────
// Same code, but OS applies PIP automatically:
//   P1 locks → P3 blocks → OS boosts P1 to priority 3 →
//   P2 cannot preempt → P1 unlocks → P3 runs → P2 runs
//   P3 waited only for P1's critical section!
```

**In VxWorks (the Mars Pathfinder OS):**

```c
// Creating a mutex WITH priority inheritance
SEM_ID mutex = semMCreate(SEM_Q_PRIORITY | SEM_INVERSION_SAFE, SEM_FULL);
//                                              ^^^^^^^^^^^^^^^^^^^
//                        This flag enables priority inheritance!
// The Pathfinder fix was enabling this one flag.
```

**In POSIX (Linux/Unix):**

```c
pthread_mutex_t mutex;
pthread_mutexattr_t attr;

pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT); // PIP
// or:
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_PROTECT); // PCP (with ceiling)
pthread_mutexattr_setprioceiling(&attr, ceiling_priority);

pthread_mutex_init(&mutex, &attr);
```

---

## 10. Priority Inversion vs Starvation vs Deadlock

| Aspect             | Priority Inversion                     | Starvation                          | Deadlock                           |
| ------------------ | -------------------------------------- | ----------------------------------- | ---------------------------------- |
| Who is affected    | High-priority process                  | Low-priority process                | Multiple processes (all blocked)   |
| Cause              | Low-priority holds resource high needs | Scheduler always picks someone else | Circular resource dependency       |
| Process state      | BLOCKED (waiting for lock)             | READY (never selected)              | BLOCKED (waiting for each other)   |
| Duration           | Temporary (ends when lock is released) | Can be indefinite                   | Permanent (without intervention)   |
| Solution           | Priority Inheritance / Ceiling         | Aging                               | Prevention / Avoidance / Detection |
| Can self-resolve?  | Yes, if medium tasks eventually finish | No                                  | No                                 |
| Resource involved? | Yes (shared resource)                  | No (just CPU time)                  | Yes (circular hold-and-wait)       |

```
  PRIORITY INVERSION:   P_high BLOCKED ──waiting for lock──► P_low (holds lock)
  STARVATION:           P_low  READY   ──never selected by──► scheduler
  DEADLOCK:             P1 BLOCKED ──waiting for R2──► P2 BLOCKED ──waiting for R1──► P1
```

---

## 10. Code Examples

> Working code that demonstrates priority inversion and Priority Inheritance Protocol (PIP) in practice.

### C++ — Simple Version

Step-by-step simulation showing what goes wrong without PIP, then how PIP prevents the inversion.

```cpp
#include <iostream>
#include <string>

enum class State { RUNNING, READY, BLOCKED, DONE };

struct Task {
    std::string name;
    int         priority;     // lower = higher urgency (1 = highest)
    State       state;
    bool        holds_lock;
};

void simulate_without_pip() {
    std::cout << "=== WITHOUT Priority Inheritance ===\n\n";

    Task low  = {"Low-cam",    3, State::RUNNING, false};
    Task med  = {"Med-telemetry", 2, State::READY,   false};
    Task high = {"High-nav",   1, State::READY,   false};

    // t1: Low acquires shared lock
    low.holds_lock = true;
    std::cout << "t1: " << low.name << " acquires lock (priority=" << low.priority << ")\n";

    // t2: High becomes ready, tries lock → blocked immediately (Low holds it)
    high.state = State::BLOCKED;
    low.state  = State::READY;  // High preempts Low (higher priority)
    std::cout << "t2: " << high.name << " tries lock → BLOCKED (Low holds it)\n";

    // t3: Medium (priority 2) preempts Low (priority 3) — Medium is NOT blocked
    med.state = State::RUNNING;
    std::cout << "t3: " << med.name << " preempts Low (Medium priority > Low priority)\n";
    std::cout << "    *** INVERSION: High (pri=" << high.priority << ") waits while "
              << "Medium (pri=" << med.priority << ") runs! ***\n";

    // t4: Medium finishes
    med.state = State::DONE;
    std::cout << "t4: " << med.name << " finishes\n";

    // t5: Low resumes, completes CS, releases lock
    low.state = State::RUNNING; low.holds_lock = false;
    std::cout << "t5: " << low.name << " releases lock\n";

    // t6: High finally gets lock
    high.state = State::RUNNING; high.holds_lock = true;
    std::cout << "t6: " << high.name << " gets lock — delayed by Medium!\n\n";
}

void simulate_with_pip() {
    std::cout << "=== WITH Priority Inheritance Protocol ===\n\n";

    Task low  = {"Low-cam",    3, State::RUNNING, false};
    Task med  = {"Med-telemetry", 2, State::READY,   false};
    Task high = {"High-nav",   1, State::READY,   false};

    int low_effective_priority = low.priority;  // will be temporarily changed

    // t1: Low acquires lock
    low.holds_lock = true;
    std::cout << "t1: " << low.name << " acquires lock (priority=" << low.priority << ")\n";

    // t2: High blocks → Low INHERITS High's priority
    high.state = State::BLOCKED;
    low_effective_priority = high.priority;   // Low now runs at priority 1 temporarily!
    std::cout << "t2: " << high.name << " blocked → Low inherits priority "
              << low_effective_priority << " (PIP boost)\n";

    // t3: Medium tries to preempt — but Low's inherited priority (1) > Medium's (2)
    std::cout << "t3: " << med.name << " wants CPU → CANNOT preempt Low "
              << "(Low's effective priority " << low_effective_priority
              << " > Med's " << med.priority << ")\n";

    // t4: Low finishes CS, reverts priority, releases lock
    low.holds_lock = false;
    low_effective_priority = low.priority;  // reverts to 3
    std::cout << "t4: " << low.name << " releases lock, priority reverts to "
              << low.priority << "\n";

    // t5: High gets lock immediately
    high.state = State::RUNNING; high.holds_lock = true;
    std::cout << "t5: " << high.name << " gets lock IMMEDIATELY — no inversion!\n\n";
    std::cout << "Medium runs only after High finishes — priority order preserved.\n";
}

int main() {
    simulate_without_pip();
    simulate_with_pip();
    return 0;
}
```

### C++ — Medium / LeetCode Style

Priority inheritance mutex using `std::condition_variable` — lock holder's priority is boosted when a higher-priority thread blocks.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <chrono>
#include <string>
#include <atomic>

// Simplified Priority Inheritance Mutex
// Tracks who holds the lock and what priority the highest blocked waiter has
struct PIMutex {
    std::mutex              internal;
    std::condition_variable cv;
    bool                    locked = false;
    std::string             holder;
    std::atomic<int>        min_waiter_priority{999};  // 999 = no waiter

    void lock(const std::string& name, int priority) {
        std::unique_lock<std::mutex> lk(internal);
        while (locked) {
            // Priority Inheritance: tell the holder our priority so it can inherit
            if (priority < min_waiter_priority) {
                min_waiter_priority = priority;
                std::cout << "  [PIP] " << name << " blocked — "
                          << holder << " inherits priority " << priority << "\n";
            }
            cv.wait(lk);
        }
        locked = true;
        holder = name;
        min_waiter_priority = 999;
        std::cout << "  [Lock] " << name << " acquired lock\n";
    }

    void unlock(const std::string& name) {
        std::unique_lock<std::mutex> lk(internal);
        locked = false;
        holder = "";
        std::cout << "  [Lock] " << name << " released lock\n";
        cv.notify_all();
    }
};

PIMutex pi_mtx;

void low_task() {
    pi_mtx.lock("Low", 3);
    std::cout << "[Low ] In critical section\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(150));
    pi_mtx.unlock("Low");
    std::cout << "[Low ] Done\n";
}

void high_task() {
    std::this_thread::sleep_for(std::chrono::milliseconds(40));  // start after Low
    std::cout << "[High] Requesting lock...\n";
    pi_mtx.lock("High", 1);
    std::cout << "[High] Got lock — running!\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(30));
    pi_mtx.unlock("High");
    std::cout << "[High] Done\n";
}

void medium_task() {
    std::this_thread::sleep_for(std::chrono::milliseconds(70));  // starts after High blocks
    std::cout << "[Med ] Running (no lock needed)\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(80));
    std::cout << "[Med ] Done\n";
}

int main() {
    std::cout << "--- Priority Inheritance Protocol ---\n";
    std::thread tl(low_task), th(high_task), tm(medium_task);
    tl.join(); th.join(); tm.join();
    std::cout << "\nHigh got the lock as soon as Low released it — Med did not delay it.\n";
    return 0;
}
```

### Python — Simple Version

Annotated walkthrough of both scenarios — clearly shows what changes with PIP.

```python
class Task:
    def __init__(self, name, priority):
        self.name       = name
        self.priority   = priority  # lower = higher urgency
        self.state      = "READY"
        self.holds_lock = False

    def __repr__(self):
        return f"{self.name}(pri={self.priority}, {self.state})"


def without_pip():
    print("=== WITHOUT Priority Inheritance ===")
    low  = Task("Low-cam",      3)
    med  = Task("Med-telemetry",2)
    high = Task("High-nav",     1)

    # t1: Low acquires lock
    low.state = "RUNNING"; low.holds_lock = True
    print(f"t1: {low.name} acquires lock")

    # t2: High becomes ready, preempts Low, tries lock → blocked
    low.state = "READY"; high.state = "BLOCKED"
    print(f"t2: {high.name} tries lock → BLOCKED (Low holds it)")

    # t3: Med preempts Low (Med priority 2 > Low priority 3)
    med.state = "RUNNING"
    print(f"t3: {med.name} preempts Low!")
    print(f"    *** INVERSION: High (pri={high.priority}) "
          f"waits while Med (pri={med.priority}) runs ***")

    # t4-t5: Med finishes, Low resumes, releases lock
    med.state = "DONE"
    low.state = "RUNNING"; low.holds_lock = False
    print(f"t4: {med.name} done. t5: {low.name} releases lock")

    # t6: High finally runs
    high.state = "RUNNING"; high.holds_lock = True
    print(f"t6: {high.name} gets lock\n"
          f"    Total delay = Med time + remaining Low time\n")


def with_pip():
    print("=== WITH Priority Inheritance Protocol (PIP) ===")
    low  = Task("Low-cam",      3)
    med  = Task("Med-telemetry",2)
    high = Task("High-nav",     1)

    inherited = low.priority  # tracks Low's effective priority

    # t1: Low acquires lock
    low.state = "RUNNING"; low.holds_lock = True
    print(f"t1: {low.name} acquires lock (priority={low.priority})")

    # t2: High blocks → Low INHERITS High's priority
    high.state = "BLOCKED"
    inherited = high.priority  # Low now has effective priority 1
    print(f"t2: {high.name} blocked → {low.name} inherits priority {inherited}")

    # t3: Med tries to preempt — CANNOT (Low's inherited priority > Med's)
    print(f"t3: {med.name} wants to preempt → BLOCKED "
          f"(Low effective priority {inherited} > Med {med.priority})")

    # t4: Low finishes, reverts priority, releases lock
    low.holds_lock = False; inherited = low.priority  # reverts to 3
    print(f"t4: {low.name} releases lock, priority reverts to {low.priority}")

    # t5: High gets lock immediately
    high.state = "RUNNING"; high.holds_lock = True
    print(f"t5: {high.name} gets lock IMMEDIATELY")
    print("    Med runs only after High — priority order preserved!\n")


without_pip()
with_pip()
```

### Python — Medium Level

`threading`-based simulation with a `PriorityMutex` that boosts the holder's priority on contention.

```python
import threading
import time

class PriorityMutex:
    """Mutex with Priority Inheritance: when a higher-priority thread blocks,
    the holder's effective priority is raised to prevent medium tasks from preempting."""

    def __init__(self, name):
        self.name      = name
        self._lock     = threading.Lock()
        self._cv       = threading.Condition(self._lock)
        self._holder   = None
        self._inherited_priority = None  # highest-priority waiter's value

    def acquire(self, task_name, priority):
        with self._cv:
            while self._holder is not None:
                if self._inherited_priority is None or priority < self._inherited_priority:
                    self._inherited_priority = priority
                    print(f"  [PIP] '{task_name}' blocked — "
                          f"'{self._holder}' inherits priority {priority}")
                self._cv.wait()
            self._holder = task_name
            self._inherited_priority = None
            print(f"  [Mutex] '{task_name}' acquired '{self.name}'")

    def release(self, task_name):
        with self._cv:
            self._holder = None
            self._inherited_priority = None
            print(f"  [Mutex] '{task_name}' released '{self.name}'")
            self._cv.notify_all()


shared = PriorityMutex("sensor_data")

def low_task():
    shared.acquire("Low", priority=3)
    print("[Low ] Critical section — processing sensor data")
    time.sleep(0.15)   # holds lock for a while
    shared.release("Low")
    print("[Low ] Done")

def high_task():
    time.sleep(0.03)   # start after Low has the lock
    print("[High] Need sensor data — requesting lock")
    shared.acquire("High", priority=1)
    print("[High] Got lock — running critical path!")
    time.sleep(0.02)
    shared.release("High")
    print("[High] Done")

def medium_task():
    time.sleep(0.06)   # start after High is blocked
    print("[Med ] Running (CPU-bound, no lock needed)")
    time.sleep(0.10)
    print("[Med ] Done")

print("--- Priority Inheritance Protocol Demo ---")
threads = [
    threading.Thread(target=low_task),
    threading.Thread(target=high_task),
    threading.Thread(target=medium_task),
]
for t in threads: t.start()
for t in threads: t.join()
print("\nObserve: High got the lock as soon as Low released it.")
print("Medium could not delay the critical High task.")
```

---

## 11. Key Takeaways

- **Priority inversion** = a high-priority process waits behind a low-priority one because the low-priority process holds a shared resource (mutex/semaphore) the high-priority one needs
- It becomes **unbounded** when medium-priority processes preempt the lock holder, delaying the high-priority process indefinitely
- The **Mars Pathfinder (1997)** incident is the most famous real-world example — fixed remotely by enabling one flag in VxWorks
- Root causes: preemptive scheduling + shared locks + different-priority processes
- **Priority Inheritance Protocol (PIP):** When P_high blocks on a lock, P_low inherits P_high's priority until it releases the lock — reactive, simple, no upfront knowledge needed
- **Priority Ceiling Protocol (PCP):** Lock is pre-assigned a ceiling priority; any process that acquires the lock immediately gets the ceiling priority — proactive, prevents deadlock, but needs upfront resource-usage knowledge
- PIP is sufficient for most general-purpose systems; PCP is preferred in hard real-time and safety-critical systems
- In POSIX: `PTHREAD_PRIO_INHERIT` enables PIP; `PTHREAD_PRIO_PROTECT` enables PCP
- Priority inversion is **different from starvation** (high vs low priority affected) and **different from deadlock** (temporary vs permanent block)
- Without these protocols, high-priority tasks in real-time systems can miss critical deadlines — potentially fatal in safety systems
