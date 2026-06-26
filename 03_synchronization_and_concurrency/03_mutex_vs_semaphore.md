# Mutex vs Semaphore

> A mutex is a binary ownership lock — only the process that locked it can unlock it; a semaphore is a general-purpose integer counter that any process can signal, making it suitable for both mutual exclusion and resource counting.

---

## Table of Contents

1. [What Is a Mutex?](#1-what-is-a-mutex)
2. [How Mutex Works](#2-how-mutex-works)
3. [Mutex Ownership](#3-mutex-ownership)
4. [What Are Semaphores?](#4-what-are-semaphores)
5. [Semaphore Operations](#5-semaphore-operations)
6. [Types of Semaphores](#6-types-of-semaphores)
7. [Mutex vs Semaphore — Comparison](#7-mutex-vs-semaphore--comparison)
8. [Practical Example Using Semaphores](#8-practical-example-using-semaphores)
9. [Common Pitfalls and Best Practices](#9-common-pitfalls-and-best-practices)
10. [When to Use Which Tool](#10-when-to-use-which-tool)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What Is a Mutex?

A **mutex** (short for "mutual exclusion") is a locking mechanism that allows only **one process or thread** to access a critical section at a time.

**Bathroom key analogy:** The mutex is the key to a single-person bathroom. Whoever has the key can enter. Everyone else waits outside. When you're done, you hand the key back.

- Mutex has exactly **two states**: locked (1) or unlocked (0)
- A process must **acquire** the mutex before entering its critical section
- It must **release** the mutex when it exits

```
  Mutex = 1 (unlocked)

  Process P1 wants in:
  ┌──────────────────────────────────────────────┐
  │  mutex_lock()  → Mutex = 0 (locked)          │
  │  [Critical Section — only P1 is here]        │
  │  mutex_unlock() → Mutex = 1 (unlocked)       │
  └──────────────────────────────────────────────┘

  While P1 is inside:
  P2 calls mutex_lock() → BLOCKED (waits in queue)
  P3 calls mutex_lock() → BLOCKED (waits in queue)

  After P1 calls mutex_unlock():
  One of P2 or P3 wakes up and acquires the mutex
```

---

## 2. How Mutex Works

The mutex provides two primary operations: **lock** and **unlock**.

```c
// Process trying to access shared resource
mutex_lock(&my_mutex);    // Acquire the lock

// ── CRITICAL SECTION ──────────────────────────
shared_counter = shared_counter + 1;
// Only one process can execute this at a time
// ─────────────────────────────────────────────

mutex_unlock(&my_mutex);  // Release the lock
```

**What happens internally:**

```
  mutex_lock():
    Is mutex unlocked (= 1)?
      YES → set to 0 → process enters critical section
      NO  → block process → add to waiting queue

  mutex_unlock():
    Set mutex to 1
    Is there a process in the waiting queue?
      YES → wake it up → it acquires the lock
      NO  → mutex stays unlocked (= 1)
```

Even if ten processes try to increment `shared_counter` at the same time, the mutex guarantees they each do it one at a time.

---

## 3. Mutex Ownership

**Key rule:** Only the process that **locked** the mutex can **unlock** it.

> Like borrowing a library book — you checked it out, so you're responsible for returning it. Someone else can't return it on your behalf.

**Why ownership matters:**

- Prevents other processes from accidentally releasing a lock they don't own
- Ensures the process holding the lock actually **completes** its critical section before anyone else enters
- Helps detect programming errors (OS can warn if the wrong process tries to unlock)

**Edge case — process crash while holding mutex:**
If a process dies while holding the mutex, the OS must detect this and release the orphaned lock. Otherwise, all waiting processes wait forever (deadlock).

---

## 4. What Are Semaphores?

A **semaphore** is a more generalized synchronization tool that uses an **integer variable** to control access to shared resources. Unlike mutex, a semaphore can allow **multiple processes** to access resources simultaneously.

**Parking lot analogy:** A semaphore is like a parking lot with a space counter. If spaces are available, cars can enter. When full, new cars wait at the gate. When one car leaves, the counter goes up and the next waiting car can enter.

- Introduced by Dutch computer scientist **Edsger Dijkstra**
- The integer value represents **how many processes can currently access** the resource
- Any process can signal (increment) a semaphore — no ownership restriction

---

## 5. Semaphore Operations

Semaphores support exactly two atomic operations:

| Operation  | Also called | Effect                                             |
| ---------- | ----------- | -------------------------------------------------- |
| **wait**   | P, down     | Decrement the value; block if value < 0            |
| **signal** | V, up       | Increment the value; wake a waiting process if any |

Both operations are **guaranteed atomic** — they execute completely without interruption, which is essential for correctness in concurrent systems.

```c
// Wait operation (P or down)
wait(S) {
    S = S - 1;
    if (S < 0) {
        // Block this process, add to waiting queue
    }
}

// Signal operation (V or up)
signal(S) {
    S = S + 1;
    if (S <= 0) {
        // Wake up one waiting process from queue
    }
}
```

**Tip:** When `S` goes negative, the absolute value of `S` tells you **how many processes are waiting**. For example, `S = -2` means 2 processes are blocked.

---

## 6. Types of Semaphores

### Binary Semaphore

A binary semaphore can only have values **0 or 1** — it behaves very similarly to a mutex.

- Initialized to **1** → first process can enter immediately
- `wait()` sets it to 0 → critical section is locked
- `signal()` sets it back to 1 → next process can enter

```c
// Initialize binary semaphore to 1
binary_semaphore S = 1;

wait(S);                    // S becomes 0 → enter critical section

// ── CRITICAL SECTION ──────
write_to_shared_file(data);
// ──────────────────────────

signal(S);                  // S becomes 1 → next process can enter
```

**Difference from mutex:** Any process can call `signal()` on a binary semaphore — there's no ownership rule. Useful for **signaling between different processes**.

---

### Counting Semaphore

A counting semaphore can hold **any non-negative integer**. It manages access to resources that have **multiple identical instances**.

- Initialized to **N** → up to N processes can access the resource simultaneously
- Each `wait()` decrements the count (one more resource in use)
- Each `signal()` increments the count (one resource freed)

```c
// Initialize counting semaphore to 3 (3 printers available)
counting_semaphore printers = 3;

wait(printers);              // One printer now in use (printers = 2)

// ── CRITICAL SECTION ──────
print_document(my_document);
// ──────────────────────────

signal(printers);            // Printer released (printers = 3 again)
```

**Example with 4 processes and 3 printers:**

```
  Initial: printers = 3

  P1 calls wait() → printers = 2  (P1 printing)
  P2 calls wait() → printers = 1  (P2 printing)
  P3 calls wait() → printers = 0  (P3 printing)
  P4 calls wait() → printers = -1 (P4 BLOCKED — waiting)

  P1 finishes, calls signal() → printers = 0, P4 is unblocked
  P4 starts printing
```

---

## 7. Mutex vs Semaphore — Comparison

| Aspect          | Mutex                                | Semaphore                                    |
| --------------- | ------------------------------------ | -------------------------------------------- |
| Purpose         | Mutual exclusion only                | Mutual exclusion AND signaling               |
| Value           | Binary (locked = 0 / unlocked = 1)   | Any non-negative integer                     |
| Ownership       | Only the locker can unlock it        | Any process can call signal                  |
| Use Case        | Protecting a single critical section | Managing resource pools or process signaling |
| Mechanism       | Locking                              | Signaling + counting                         |
| Error detection | Better (OS can detect wrong owner)   | Weaker (any process can signal)              |

**When to choose:**

- **Mutex** → protecting shared data where the same process locks and unlocks (most common case)
- **Binary Semaphore** → signaling between processes (one process signals, a different one waits)
- **Counting Semaphore** → managing pools of identical resources (N database connections, N printers)

Binary semaphores can technically replace mutex in many cases, but mutex provides clearer semantics and better error detection through ownership enforcement.

---

## 8. Practical Example Using Semaphores

**Problem:** 8 processes want to use 5 identical disk drives.

```c
// System initialization
counting_semaphore disk_drives = 5;

// Any process wanting a drive:
wait(disk_drives);    // Get a drive (decrements counter)

use_disk_drive();     // Do work

signal(disk_drives);  // Return the drive (increments counter)
```

**What happens when all 5 drives are busy:**

```
  Initial: disk_drives = 5

  P1 calls wait() → disk_drives = 4
  P2 calls wait() → disk_drives = 3
  P3 calls wait() → disk_drives = 2
  P4 calls wait() → disk_drives = 1
  P5 calls wait() → disk_drives = 0

  P6 calls wait() → disk_drives = -1 → P6 BLOCKED
  P7 calls wait() → disk_drives = -2 → P7 BLOCKED
  P8 calls wait() → disk_drives = -3 → P8 BLOCKED

  P1 finishes, calls signal() → disk_drives = -2 → P6 unblocked
  P2 finishes, calls signal() → disk_drives = -1 → P7 unblocked
  P3 finishes, calls signal() → disk_drives = 0  → P8 unblocked
```

Processes don't need to know **which** specific drive they get — just that one is available. The semaphore handles the counting.

---

## 9. Common Pitfalls and Best Practices

### Forgetting to release

Every `lock` must have a matching `unlock`. Every `wait` must have a matching `signal`. Forgetting to release leaves resources permanently locked — other processes wait forever.

```c
// BAD — mutex never released if exception occurs mid-way
mutex_lock(&m);
risky_operation();   // What if this throws?
mutex_unlock(&m);

// GOOD — use try/finally or RAII patterns (in C++/Java/Python)
mutex_lock(&m);
try {
    risky_operation();
} finally {
    mutex_unlock(&m);
}
```

### Avoiding Deadlocks

A deadlock happens when processes wait for each other in a circle:

```
  P1 holds Mutex_A, wants Mutex_B
  P2 holds Mutex_B, wants Mutex_A
  → Both wait forever (deadlock)
```

**Fix:** Always acquire multiple locks in the **same order** across all processes:

```
  Rule: Every process must acquire Mutex_A before Mutex_B
  → P2 now tries to get A first → P1 finishes → no deadlock
```

**Alternative:** Use timeouts — if a lock isn't acquired within a time limit, release any held locks and retry.

### Getting initialization right

| Scenario                      | Initial Value | Why                                   |
| ----------------------------- | ------------- | ------------------------------------- |
| Binary semaphore (mutex-like) | 1             | First process can enter immediately   |
| Binary semaphore (signaling)  | 0             | Waiting process blocks until signaled |
| Counting semaphore            | N             | N = number of available resources     |

Setting a mutual-exclusion semaphore to 0 by mistake means **no process can ever enter** — the first `wait()` blocks immediately.

---

## 10. When to Use Which Tool

| Scenario                                                | Use                    |
| ------------------------------------------------------- | ---------------------- |
| Protecting a shared data structure (one at a time)      | **Mutex**              |
| One process signals another that an event occurred      | **Binary Semaphore**   |
| Managing N identical database connections               | **Counting Semaphore** |
| Limiting how many threads run a heavy operation at once | **Counting Semaphore** |
| Producer-consumer — tracking items in a buffer          | **Counting Semaphore** |
| Simple critical section with strict ownership semantics | **Mutex**              |

---

## 11. Key Takeaways

- A **mutex** is a binary lock with **ownership** — only the locking process can unlock it; best for protecting single critical sections
- A **semaphore** is an integer counter with two operations: `wait` (decrement) and `signal` (increment); any process can signal
- **Binary semaphore** (values 0/1) ≈ mutex but without ownership — useful for signaling between different processes
- **Counting semaphore** (values 0 to N) — manages pools of N identical resources
- Both tools guarantee **atomicity** of their operations — no race conditions within the lock/signal mechanism itself
- When `S < 0`, the absolute value of `S` = number of blocked processes waiting
- Always pair every `wait` with a `signal`; always pair every `lock` with an `unlock`
- To avoid deadlocks with multiple locks: **always acquire locks in the same order** across all processes
- Binary semaphores can substitute mutex functionally, but mutex gives stronger safety guarantees through ownership rules
