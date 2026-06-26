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
