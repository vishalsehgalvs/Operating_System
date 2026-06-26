# Critical Section Problem in OS

> The critical section problem asks how to design a protocol so that when one process accesses shared data, no other process can do the same simultaneously — any correct solution must guarantee mutual exclusion, progress, and bounded waiting.

---

## Table of Contents

1. [What Is a Critical Section?](#1-what-is-a-critical-section)
2. [Example of Shared Resource Access](#2-example-of-shared-resource-access)
3. [Structure of a Process](#3-structure-of-a-process)
4. [Three Requirements for a Valid Solution](#4-three-requirements-for-a-valid-solution)
5. [Why the Problem Occurs](#5-why-the-problem-occurs)
6. [Race Condition Walkthrough](#6-race-condition-walkthrough)
7. [Approaches to Solving the Problem](#7-approaches-to-solving-the-problem)
8. [Real-World Analogy](#8-real-world-analogy)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What Is a Critical Section?

A **critical section** is a piece of code in a process that accesses **shared resources** — variables, files, hardware devices, or memory that other processes also use.

**Rule:** When one process is executing its critical section, **no other process** should be allowed to execute its own critical section that uses the **same shared resource**.

**Bathroom analogy:** The bathroom (critical section) can only hold one person at a time. Lock the door when you enter, unlock it when you leave.

```
  PROCESS                     SHARED RESOURCE
  ┌─────────────────┐
  │ Remainder Code  │         No access to shared resource
  ├─────────────────┤
  │  Entry Section  │ ──────► Check if resource is free; request access
  ├─────────────────┤
  │ CRITICAL SECTION│ ──────► Access shared resource (only 1 process at a time!)
  ├─────────────────┤
  │  Exit Section   │ ──────► Signal that resource is released
  ├─────────────────┤
  │ Remainder Code  │         No access to shared resource
  └─────────────────┘
```

---

## 2. Example of Shared Resource Access

Two processes both want to increment a shared counter:

```c
// Shared variable
int counter = 5;

// Process P1
counter = counter + 1;  // expects counter = 6 after this

// Process P2
counter = counter + 1;  // expects counter = 7 after both run
```

**Expected result:** 7  
**What can happen without synchronization:** Both processes read `5`, both compute `6`, both write `6` → final value is **6, not 7**.

This is a **race condition** — the result depends on unpredictable execution timing.

---

## 3. Structure of a Process

Every process that accesses shared resources follows this four-part structure:

```
do {
    // ── ENTRY SECTION ──────────────────────────────────
    // Request permission to enter the critical section
    // If another process is inside → wait here

    // ── CRITICAL SECTION ───────────────────────────────
    // Access the shared resource
    // Only ONE process should be here at a time!

    // ── EXIT SECTION ───────────────────────────────────
    // Signal that you're done with the critical section
    // Release any locks or flags

    // ── REMAINDER SECTION ──────────────────────────────
    // The rest of the process code (no shared resource access)

} while (true);
```

The challenge: implement the **entry and exit sections correctly** so processes never interfere inside the critical section.

Without this, processes can corrupt shared data → wrong results, crashes, or security vulnerabilities.

---

## 4. Three Requirements for a Valid Solution

Any correct solution to the critical section problem **must** satisfy all three:

### 1. Mutual Exclusion

If Process P1 is executing in its critical section, **no other process** can execute in its critical section that uses the same shared resource.

> "One person in the bathroom" rule — if someone is inside, the door is locked.

### 2. Progress

If no process is currently in its critical section and some processes **want** to enter:

- Only processes **not in their remainder section** can participate in deciding who enters next
- This decision **cannot be postponed indefinitely**

> If the bathroom is empty and someone wants to use it, don't keep the door locked for no reason. Someone must get in.

### 3. Bounded Waiting

There must be a **finite limit** on how many times other processes can enter their critical sections after a process has requested entry and **before that request is granted**.

> No one waits in line forever — starvation is not allowed.

### Requirements Summary

| Requirement      | What It Guarantees                                       | Analogy                              |
| ---------------- | -------------------------------------------------------- | ------------------------------------ |
| Mutual Exclusion | Only 1 process in critical section at a time             | One person in the bathroom           |
| Progress         | If critical section is free, entry isn't delayed forever | Empty room → someone gets in quickly |
| Bounded Waiting  | Every requesting process eventually gets its turn        | No one waits in line forever         |

---

## 5. Why the Problem Occurs

The critical section problem arises from **concurrent execution + shared resources**.

On a **single-core CPU:**

- OS rapidly switches between processes (context switching)
- A switch can happen between any two machine instructions
- A process can be interrupted mid-way through a counter increment

On **multi-core systems:**

- Processes can truly run in parallel on different cores
- Race conditions can occur from simultaneous execution, not just switching

```
  HIGH-LEVEL CODE:   counter++        (looks like 1 operation)

  MACHINE CODE:      1. LOAD  counter → register
                     2. ADD   register, 1
                     3. STORE register → counter

  Context switch can happen between steps 1 and 2, or 2 and 3!
```

---

## 6. Race Condition Walkthrough

```c
// Initial value
counter = 5;

// Process P1 steps:
// 1. Read counter (gets 5)
// 2. Increment it (calculates 6)
// 3. Write it back (counter = 6)

// Process P2 steps:
// 1. Read counter (gets 5)
// 2. Increment it (calculates 6)
// 3. Write it back (counter = 6)
```

**Step-by-step interleaved execution:**

| Step | P1 Action              | P2 Action              | counter |
| ---- | ---------------------- | ---------------------- | ------- |
| 1    | Reads counter → temp=5 | —                      | 5       |
| 2    | —                      | Reads counter → temp=5 | 5       |
| 3    | temp = 5+1 = 6         | —                      | 5       |
| 4    | —                      | temp = 5+1 = 6         | 5       |
| 5    | Writes counter = 6     | —                      | **6**   |
| 6    | —                      | Writes counter = 6     | **6**   |

**Result: 6 instead of 7.** P2's write overwrote P1's write because they both read the same initial value.

> Same as two people editing the same Google Doc simultaneously — one person's changes get overwritten by the other.

---

## 7. Approaches to Solving the Problem

Three categories of solutions exist, each building on the previous:

### Software Solutions

- Pure algorithmic logic, no special hardware needed
- Examples: **Peterson's Solution**, **Dekker's Algorithm**
- Use shared flags and a `turn` variable to coordinate who enters next
- Complex and not always efficient on modern hardware (compilers/CPUs may reorder instructions)

**Peterson's Solution (2 processes):**

```c
// Shared variables
bool flag[2] = {false, false};
int turn;

// Process Pi wants to enter critical section:
flag[i] = true;      // "I want to enter"
turn = j;            // "You go first if you want"
while (flag[j] && turn == j);  // Wait if other process wants in and it's their turn

// --- CRITICAL SECTION ---

flag[i] = false;     // "I'm done"
```

### Hardware Solutions

Modern CPUs provide **atomic instructions** — they complete as a single uninterruptible operation, even on multicore systems:

- `test_and_set` — reads a value and sets it to true atomically
- `compare_and_swap` — updates a value only if it matches expected, atomically

These form the foundation for all higher-level sync primitives (locks, mutexes).

### OS Primitives (High-Level Tools)

Built on top of hardware support — easier and safer for programmers:

- **Mutex (lock):** binary lock — locked/unlocked
- **Semaphore:** counting lock — controls access for N processes
- **Monitor:** structured sync built into programming languages

These are covered in the next topics.

---

## 8. Real-World Analogy

**Shared office printer:**

```
  CRITICAL SECTION = Using the printer
  ENTRY SECTION    = Checking if printer is free + reserving it
  EXIT SECTION     = Finishing print job + signaling printer is free

  Without sync:
  Person A sends job → pages 1-3 printed
  Person B sends job simultaneously → pages from B interleave with A's pages
  Both documents are garbled (race condition on the printer)

  With sync (mutual exclusion):
  Person A acquires the printer lock
  Person B waits at the entry section
  Person A finishes → releases lock
  Person B enters → prints cleanly
```

---

## 8. Code Examples

> Working code that demonstrates Peterson's Algorithm — the classic software solution to the critical section problem.

### C++ — Simple Version

Conceptual simulation of Peterson's Algorithm showing entry/exit logic for two processes.

```cpp
#include <iostream>
#include <string>

// Peterson's Algorithm state (shared between two processes)
bool flag[2] = {false, false};  // flag[i] = true means process i wants to enter
int  turn    = 0;               // which process gets to enter if both want in

// Entry section for process `id` (0 or 1)
void entry_section(int id) {
    int other = 1 - id;     // the other process (0→1, 1→0)
    flag[id] = true;        // "I want to enter the critical section"
    turn     = other;       // "But you can go first — I'm polite"

    // Spin-wait: only enter if the other is NOT interested, OR it is my turn
    while (flag[other] && turn == other) {
        // busy waiting — in practice this wastes CPU; real OS uses sleep/wake
    }
    // CRITICAL SECTION starts here
}

// Exit section for process `id`
void exit_section(int id) {
    flag[id] = false;   // "I'm done — no longer interested in the critical section"
}

int main() {
    // --- Case 1: No contention (only P0 wants to enter) ---
    std::cout << "=== No Contention ===\n";
    entry_section(0);
    std::cout << "P0: inside CRITICAL SECTION\n";
    exit_section(0);
    std::cout << "P0: exited\n\n";

    // --- Case 2: P0 inside, P1 tries to enter → P1 must block ---
    std::cout << "=== Contention: P0 inside, P1 tries to enter ===\n";
    entry_section(0);
    std::cout << "P0: inside CRITICAL SECTION\n";

    // P1 performs its entry protocol
    flag[1] = true;  // P1 wants in
    turn = 0;        // P1 says "P0 goes first"
    bool p1_blocked = (flag[0] && turn == 0);
    std::cout << "P1: blocked = " << std::boolalpha << p1_blocked
              << " (mutual exclusion holds!)\n";

    exit_section(0);  // P0 leaves
    std::cout << "P0: exited — P1 may now enter (flag[0] = "
              << std::boolalpha << flag[0] << ")\n";
    exit_section(1);

    return 0;
}
```

### C++ — Medium / LeetCode Style

Peterson's Algorithm with real `std::thread` — proves correctness under actual concurrency.

```cpp
#include <iostream>
#include <thread>
#include <atomic>

// Use atomics so the compiler doesn't reorder loads/stores
std::atomic<bool> flag0{false}, flag1{false};
std::atomic<int>  turn{0};
int shared_counter = 0;  // protected by Peterson's protocol

void process0(int iterations) {
    for (int i = 0; i < iterations; i++) {
        // ENTRY SECTION (Peterson for process 0)
        flag0.store(true, std::memory_order_seq_cst);
        turn.store(1,     std::memory_order_seq_cst);
        while (flag1.load(std::memory_order_seq_cst) &&
               turn.load(std::memory_order_seq_cst) == 1) { /* spin */ }

        // CRITICAL SECTION
        shared_counter++;

        // EXIT SECTION
        flag0.store(false, std::memory_order_seq_cst);
    }
}

void process1(int iterations) {
    for (int i = 0; i < iterations; i++) {
        // ENTRY SECTION (Peterson for process 1)
        flag1.store(true, std::memory_order_seq_cst);
        turn.store(0,     std::memory_order_seq_cst);
        while (flag0.load(std::memory_order_seq_cst) &&
               turn.load(std::memory_order_seq_cst) == 0) { /* spin */ }

        // CRITICAL SECTION
        shared_counter++;

        // EXIT SECTION
        flag1.store(false, std::memory_order_seq_cst);
    }
}

int main() {
    const int N = 50000;
    std::thread t0(process0, N);
    std::thread t1(process1, N);
    t0.join(); t1.join();
    // Peterson guarantees mutual exclusion → result is always exactly 100000
    std::cout << "Expected: " << 2*N << ", Got: " << shared_counter << "\n";
    return 0;
}
```

### Python — Simple Version

Single-threaded walkthrough of Peterson's Algorithm that shows the entry/exit protocol step by step.

```python
# Peterson's Algorithm — demonstration of the protocol logic
# (single-threaded simulation; shows the key checks at each step)

flag = [False, False]  # flag[i] = process i wants to enter CS
turn = 0               # "polite" turn variable — whose turn it is when both want in

def entry_section(pid):
    """Entry protocol for process pid (0 or 1)."""
    other = 1 - pid
    flag[pid] = True    # "I want to enter"
    global turn
    turn = other        # "You go first" (politeness — prevents livelock)

    # Wait while: other wants in AND it's the other's turn
    while flag[other] and turn == other:
        pass  # spin (busy wait)

def exit_section(pid):
    """Exit protocol for process pid."""
    flag[pid] = False   # "I'm done — release my interest"

def critical_section(pid):
    """Simulated critical section work."""
    print(f"  P{pid} → INSIDE critical section")

def run(pid):
    entry_section(pid)
    critical_section(pid)
    exit_section(pid)
    print(f"  P{pid} → exited critical section")

# Test 1: P0 alone — no contention
print("=== Test 1: P0 alone ===")
run(0)

# Test 2: P1 alone — no contention
print("\n=== Test 2: P1 alone ===")
run(1)

# Test 3: Contention — P0 inside, manually check P1's wait condition
print("\n=== Test 3: Contention demo ===")
flag[0] = True   # P0 wants in (or is inside)
turn = 1         # P0 set turn = P1 (politely)
flag[1] = True   # P1 also wants in
turn = 0         # P1 set turn = P0 (politely)
p1_must_wait = flag[0] and turn == 0
print(f"  P0 is in CS, P1 checks: flag[0]={flag[0]}, turn={turn}")
print(f"  P1 must wait: {p1_must_wait}  ← mutual exclusion holds!")
flag[0] = False  # P0 exits
print("  P0 exited — P1 can now enter")
```

### Python — Medium Level

Uses `threading.Thread` to demonstrate Peterson's Algorithm under real concurrency.

```python
import threading

flag = [False, False]
turn = 0
shared_counter = 0

def process(pid, iterations):
    global flag, turn, shared_counter
    other = 1 - pid
    for _ in range(iterations):
        # ENTRY
        flag[pid] = True
        turn = other
        while flag[other] and turn == other:
            pass  # spin

        # CRITICAL SECTION
        shared_counter += 1

        # EXIT
        flag[pid] = False

N = 20000
t0 = threading.Thread(target=process, args=(0, N))
t1 = threading.Thread(target=process, args=(1, N))
t0.start(); t1.start()
t0.join(); t1.join()
print(f"Peterson's — Expected: {2*N}, Got: {shared_counter}")
# Note: CPython's GIL provides some protection; for a true hardware-level demo
# the C++ version with seq_cst atomics is more rigorous.
```

---

## 9. Key Takeaways

- A **critical section** is code that accesses shared resources — only one process should execute it at a time
- The **critical section problem** asks: how do we design entry/exit protocols that prevent conflicts?
- Every process has 4 parts: **Entry → Critical Section → Exit → Remainder**
- **Three non-negotiable requirements:** mutual exclusion + progress + bounded waiting
- The problem exists because `counter++` is actually 3 machine instructions, and context switches can happen between any of them
- **Solutions come in three levels:**
  1. Software (Peterson's algorithm — flags + turn variable)
  2. Hardware (atomic instructions: `test_and_set`, `compare_and_swap`)
  3. OS primitives (mutex, semaphore, monitor) — the practical tools
- Critical section ≠ deadlock — deadlock is when processes wait for each other in a cycle; this is about preventing simultaneous shared resource access
