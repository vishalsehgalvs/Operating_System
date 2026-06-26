# Threads in Operating Systems

> A thread is the smallest unit of execution inside a process — multiple threads share the same memory and resources but each runs independently, letting programs do many things at once without the cost of creating separate processes.

---

## Table of Contents

1. [What Are Threads?](#1-what-are-threads)
2. [Key Characteristics of Threads](#2-key-characteristics-of-threads)
3. [Process vs Thread](#3-process-vs-thread)
4. [Why Use Threads?](#4-why-use-threads)
5. [Components of a Thread](#5-components-of-a-thread)
6. [Single-Threaded vs Multithreaded Processes](#6-single-threaded-vs-multithreaded-processes)
7. [Types of Threads](#7-types-of-threads)
8. [Thread Lifecycle](#8-thread-lifecycle)
9. [Real-World Examples](#9-real-world-examples)
10. [Practical Example: File Downloads](#10-practical-example-file-downloads)
11. [Benefits and Challenges](#11-benefits-and-challenges)
12. [When to Use Threads](#12-when-to-use-threads)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. What Are Threads?

A **thread** is the smallest unit of execution within a process. While a process is a running program with its own memory and resources, a thread is a lightweight component that runs **inside** that process.

**Restaurant analogy:** The process is the entire restaurant operation. Each waiter is a thread — they all work within the same restaurant (process), share the same kitchen and supplies, but each handles different tables independently.

```
  PROCESS (Restaurant)
  ┌──────────────────────────────────────────────┐
  │  Shared: Code, Data, Files, Heap memory      │
  │                                              │
  │  Thread 1      Thread 2      Thread 3        │
  │  (Waiter A)    (Waiter B)    (Waiter C)      │
  │  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
  │  │Own Stack│  │Own Stack│  │Own Stack│      │
  │  │Own PC   │  │Own PC   │  │Own PC   │      │
  │  │Own Regs │  │Own Regs │  │Own Regs │      │
  │  └─────────┘  └─────────┘  └─────────┘      │
  └──────────────────────────────────────────────┘
```

---

## 2. Key Characteristics of Threads

- **Lightweight** — creating and managing threads is much cheaper than creating processes
- **Shared address space** — all threads in a process share the same memory (code, data, heap, open files)
- **Independent execution state** — each thread has its own stack, program counter, and registers
- **Fast communication** — threads communicate via shared memory, no IPC needed
- **Fast context switching** — switching between threads of the same process is cheaper than switching processes

---

## 3. Process vs Thread

| Aspect            | Process                                 | Thread                                |
| ----------------- | --------------------------------------- | ------------------------------------- |
| Definition        | Independent program in execution        | Lightweight unit within a process     |
| Memory Space      | Separate address space                  | Shares process address space          |
| Creation Time     | Slower (more overhead)                  | Faster (less overhead)                |
| Communication     | Requires IPC mechanisms                 | Direct shared memory access           |
| Context Switching | Expensive (save entire process state)   | Cheaper (less state to save)          |
| Resource Sharing  | Minimal (isolated)                      | High (shares code, data, files)       |
| Crash Impact      | One process crash doesn't affect others | Thread crash can kill entire process  |
| Termination       | Terminates independently                | Terminating process kills all threads |

**Key distinction:** A process has its own house. Threads are the people living in it — they share the house but each has their own bedroom (stack).

---

## 4. Why Use Threads?

### Improved Responsiveness

A thread can handle user input while another does background work. Without threads, a word processor would **freeze** while auto-saving. With threads, you keep typing while saving happens silently.

### Resource Sharing

Threads naturally share memory — like roommates sharing a kitchen. No need for complex communication to pass data between them.

### Economy

- Thread creation is cheaper than process creation
- Context switching between threads of the same process is faster (no address space switch needed)

### Utilization of Multiprocessor Systems

On multi-core CPUs, different threads can run **truly in parallel** on different cores simultaneously — a single app can fully exploit the hardware.

```
  Single-core (concurrent):        Multi-core (parallel):
  Core 1: T1 T2 T1 T2 T1...       Core 1: T1 T1 T1 T1...
                                   Core 2: T2 T2 T2 T2...
  (rapid switching)                (truly simultaneous)
```

---

## 5. Components of a Thread

Each thread maintains its own **private state** while sharing the process's resources:

### Private to each thread:

| Component       | Purpose                                                                                    |
| --------------- | ------------------------------------------------------------------------------------------ |
| Thread ID       | Unique identifier for this thread within the process                                       |
| Program Counter | Points to the next instruction this thread will execute                                    |
| Register Set    | Stores current working variables and temporary data                                        |
| Stack           | Local variables, function parameters, return addresses — grows/shrinks with function calls |

### Shared by all threads in the process:

| Shared Resource | What it contains                    |
| --------------- | ----------------------------------- |
| Code section    | The program's compiled instructions |
| Data section    | Global and static variables         |
| Heap            | Dynamically allocated memory        |
| Open files      | File descriptors, network sockets   |

---

## 6. Single-Threaded vs Multithreaded Processes

### Single-Threaded (old/simple approach)

One thread handles everything — UI, computation, I/O. If one thing blocks, the whole program freezes.

```
// Single-threaded calculator
start → wait for input → do long calculation → (UI FROZEN here) → show result
```

### Multithreaded (modern approach)

Multiple threads handle different concerns simultaneously.

```
// Multithreaded calculator
UI Thread:           handle input → update display → handle input → ...
Calculation Thread:  wait → perform_long_calc() → send result to UI → wait
                     (UI stays responsive the whole time)
```

```
  SINGLE-THREADED:                 MULTITHREADED:
  ┌────┐                           ┌────┐  ┌────┐  ┌────┐
  │Main│                           │ T1 │  │ T2 │  │ T3 │
  │    │ → sequential              │    │  │    │  │    │
  │    │   one task                │    │  │    │  │    │
  │    │   at a time               │ UI │  │Save│  │Net │
  └────┘                           └────┘  └────┘  └────┘
                                   concurrent / parallel
```

---

## 7. Types of Threads

### User-Level Threads

- Managed entirely by a **user-space library** — kernel is unaware
- Fast to create and switch (no system calls needed)
- **Problem:** If one thread blocks on I/O, the **entire process blocks** because the kernel sees only one thread

### Kernel-Level Threads

- Managed directly by the **OS kernel**
- Can run in parallel on multiple cores — kernel schedules them independently
- If one thread blocks, others in the same process **keep running**
- **Trade-off:** Slower operations because each thread action requires a system call

| Aspect               | User-Level Threads          | Kernel-Level Threads                |
| -------------------- | --------------------------- | ----------------------------------- |
| Who manages them     | User-space library          | OS kernel                           |
| Creation speed       | Fast (no system call)       | Slower (system call needed)         |
| Multiprocessor use   | Poor (kernel sees 1 thread) | Full (kernel schedules each thread) |
| If one thread blocks | Whole process blocks        | Only that thread blocks             |

---

## 8. Thread Lifecycle

Threads go through the same basic states as processes:

```
              create()
  ┌─────────────────────────┐
  │           NEW           │
  └────────────┬────────────┘
               │ start()
               ▼
  ┌─────────────────────────┐
  │          READY          │◄──── unblocked / preempted
  └────────────┬────────────┘
               │ scheduler picks it
               ▼
  ┌─────────────────────────┐
  │         RUNNING         │────► BLOCKED/WAITING
  └────────────┬────────────┘      (waiting for I/O, lock, or event)
               │ task complete
               ▼
  ┌─────────────────────────┐
  │        TERMINATED       │
  └─────────────────────────┘
```

| State      | Description                                           |
| ---------- | ----------------------------------------------------- |
| New        | Thread created but not yet started                    |
| Ready      | Waiting for the scheduler to assign it to a processor |
| Running    | Actively executing instructions on a processor        |
| Blocked    | Waiting for an event (I/O, lock, another thread)      |
| Terminated | Finished execution, resources released                |

---

## 9. Real-World Examples

### Web Browser

| Thread         | Job                                          |
| -------------- | -------------------------------------------- |
| Render thread  | Draws page layout and graphics               |
| Network thread | Loads images and resources from the internet |
| JS thread      | Executes JavaScript code on the page         |
| UI thread      | Lets you scroll, click links, switch tabs    |

→ Without threads, loading one large image would freeze your entire browser.

### Word Processor (e.g., Microsoft Word)

| Thread             | Job                                         |
| ------------------ | ------------------------------------------- |
| Input thread       | Handles keyboard input, displays characters |
| Spell-check thread | Underlines mistakes in the background       |
| Auto-save thread   | Periodically saves the document silently    |

### Video Game

| Thread         | Job                                                |
| -------------- | -------------------------------------------------- |
| Game logic     | AI calculations for computer-controlled characters |
| Render thread  | Draws the game world at 60 fps                     |
| Audio thread   | Music and sound effects                            |
| Network thread | Multiplayer communication                          |

---

## 10. Practical Example: File Downloads

### Without Threads — Sequential (30 seconds total)

```
start
download_file("file1.pdf")   // 10 seconds
download_file("file2.pdf")   // 10 seconds  (file1 must finish first)
download_file("file3.pdf")   // 10 seconds  (file2 must finish first)
// Total: 30 seconds
```

### With Threads — Parallel (~10 seconds total)

```
start
thread1 = create_thread(download_file, "file1.pdf")
thread2 = create_thread(download_file, "file2.pdf")
thread3 = create_thread(download_file, "file3.pdf")

start(thread1)   // all start at the same time
start(thread2)
start(thread3)

wait_for(thread1, thread2, thread3)  // wait for the slowest one
// Total: ~10 seconds (time of slowest download)
```

```
  Time →    0        5        10
  Thread1:  ├─── file1.pdf ───┤
  Thread2:  ├─── file2.pdf ───┤
  Thread3:  ├─── file3.pdf ───┤
                              ↑
                         All done at 10s
```

---

## 11. Benefits and Challenges

### Benefits

| Benefit                 | Why it matters                        |
| ----------------------- | ------------------------------------- |
| Better performance      | Parallel execution on multi-core CPUs |
| Improved responsiveness | Background tasks don't freeze the UI  |
| Economical              | Less overhead than processes          |
| Easy communication      | Shared memory — no IPC needed         |

### Challenges

| Challenge              | What it means                                                               |
| ---------------------- | --------------------------------------------------------------------------- |
| Race conditions        | Two threads modify the same data at the same time → unpredictable results   |
| Harder debugging       | Bugs appear sporadically depending on thread timing — hard to reproduce     |
| Synchronization needed | Developers must coordinate access to shared data (locks, semaphores)        |
| Complexity             | Multithreaded code is harder to write and reason about than sequential code |

---

## 12. When to Use Threads

### Good use cases:

- Multiple independent tasks that can run concurrently (server handling many clients)
- Keeping UI responsive while background work happens (long computation, file I/O, network)
- Compute-intensive tasks on multi-core systems (video encoding, scientific simulations)
- Processing different parts of a large dataset in parallel

### When NOT to use threads:

- Simple sequential programs where tasks must happen in strict order
- Tasks too small/fast — thread creation overhead outweighs the benefit
- Complex data dependencies where most operations need exclusive access to shared resources

---

## 13. Key Takeaways

- A **thread** is the smallest unit of execution — lives inside a process, shares its memory
- Threads are **lightweight**: faster to create, cheaper context switches, no address space switch needed
- **Private per thread:** Stack, Program Counter, Register set, Thread ID
- **Shared by all threads:** Code, Data, Heap, Open files
- **Why threads?** Responsiveness + resource sharing + economy + multiprocessor utilization
- **User-level threads:** Fast but block the whole process on I/O; **Kernel-level threads:** Slower but truly parallel
- Thread **lifecycle:** New → Ready → Running → Blocked → Terminated (same pattern as processes)
- Main **challenge:** Shared memory means race conditions — requires careful synchronization
- Real apps (browsers, games, editors) all use multiple threads to stay fast and responsive
- Next topic: **User vs Kernel Threads** — how threads are implemented and mapped to hardware
