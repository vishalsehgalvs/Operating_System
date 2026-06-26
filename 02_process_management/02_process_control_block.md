# Process Control Block (PCB): OS Process Tracking Guide

> **One-line summary:**
> A **PCB** is a data structure the OS maintains for every process — it's the process's complete identity card, holding everything the OS needs to pause, resume, and manage that process.

---

## Table of Contents

1. [What is a PCB?](#1-what-is-a-pcb)
2. [Why Do We Need a PCB?](#2-why-do-we-need-a-pcb)
3. [Components of a PCB](#3-components-of-a-pcb)
4. [PCB Structure at a Glance](#4-pcb-structure-at-a-glance)
5. [How PCB Works in Practice (Context Switch Walkthrough)](#5-how-pcb-works-in-practice-context-switch-walkthrough)
6. [PCB vs Process](#6-pcb-vs-process)
7. [Where is the PCB Stored?](#7-where-is-the-pcb-stored)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What is a PCB?

A **Process Control Block (PCB)** is a data structure the OS creates for every process. It stores all the information the OS needs to track, pause, and resume that process.

> Like **reading multiple books at once** — every time you switch books, you need a bookmark to remember where you left off. The PCB is that bookmark, plus your notes on everything about where you were.

- Created when a process is created (New state)
- Deleted when the process terminates
- One PCB per process, always

---

## 2. Why Do We Need a PCB?

Processes constantly move between states: Running → Waiting → Ready → Running. Every time the CPU switches from one process to another, it must:

1. **Save** the current process's exact state
2. **Load** the next process's saved state

Without a PCB, the OS has no idea where each process was when it last ran — execution would be corrupted.

> The PCB is a **snapshot** of the process at any point in time, allowing the OS to freeze and unfreeze processes seamlessly.

---

## 3. Components of a PCB

### Process Identification Information

| Field                 | Description                                                        |
| --------------------- | ------------------------------------------------------------------ |
| **Process ID (PID)**  | Unique number for this process — no two active processes share one |
| **Parent PID (PPID)** | PID of the process that created this one                           |
| User ID               | Which user owns this process                                       |

> PID is like a student ID — uniquely identifies you in the system.

---

### Process State Information

The **current state** of the process: New, Ready, Running, Waiting, or Terminated.

The OS checks this to decide what action to take next — only Ready processes can be selected for execution.

---

### Program Counter

The **address of the next instruction** to be executed.

> Like a bookmark pointing to the exact line in the recipe you need to do next.

When the OS switches away from a process, it saves the program counter in the PCB. When the process resumes, that value is loaded back — no instruction is skipped or repeated.

---

### CPU Registers

The CPU uses registers for all active computation. Since different processes share the same physical registers, their values must be saved when switching.

Saved registers include:

- Accumulators
- Index registers
- Stack pointers
- General-purpose registers

When the process resumes, these values are restored exactly — the process continues as if nothing happened.

---

### CPU Scheduling Information

Data the scheduler uses to decide which process runs next:

| Field             | Description                                         |
| ----------------- | --------------------------------------------------- |
| Priority          | Higher priority = scheduled before lower priority   |
| Queue pointers    | Which scheduling queue this process is currently in |
| Scheduling params | Time slice remaining, wait time, etc.               |

---

### Memory Management Information

Tracks what memory is allocated to this process:

| Field               | Description                                             |
| ------------------- | ------------------------------------------------------- |
| Base register       | Starting address of the process in memory               |
| Limit register      | How much memory the process can access                  |
| Page/segment tables | Maps the process's virtual addresses to physical memory |

This prevents processes from accessing each other's memory — core to security and stability.

---

### Accounting Information

Tracks resource usage for performance analysis and billing in multi-user systems:

| Field          | Description                         |
| -------------- | ----------------------------------- |
| CPU time used  | Total CPU time consumed so far      |
| Time limits    | Max allowed CPU time                |
| Account number | For billing in shared/cloud systems |

> This is the data your Task Manager displays for CPU % and memory usage.

---

### I/O Status Information

Lists all I/O resources currently held by the process:

| Field              | Description                             |
| ------------------ | --------------------------------------- |
| Allocated devices  | I/O devices assigned to this process    |
| Open files         | List of file descriptors currently open |
| I/O request status | Status of pending read/write operations |

When the process terminates, the OS uses this to release all I/O resources cleanly.

---

## 4. PCB Structure at a Glance

```
┌────────────────────────────────────────┐
│         PROCESS CONTROL BLOCK          │
├────────────────────────────────────────┤
│  Process ID (PID): 1234                │
│  Parent PID (PPID): 1001               │
├────────────────────────────────────────┤
│  Process State: Ready                  │
├────────────────────────────────────────┤
│  Program Counter: 0x4005C8             │
├────────────────────────────────────────┤
│  CPU Registers:                        │
│    AX=10, BX=20, SP=0xFF00, ...        │
├────────────────────────────────────────┤
│  Priority: 5                           │
│  Queue pointer: ready_queue[3]         │
├────────────────────────────────────────┤
│  Memory: Base=0x1000, Limit=0x5000     │
├────────────────────────────────────────┤
│  CPU Time Used: 250ms                  │
├────────────────────────────────────────┤
│  Open Files: [file1.txt, log.txt]      │
│  Allocated Devices: [keyboard]         │
└────────────────────────────────────────┘
```

| Component       | Example Value             |
| --------------- | ------------------------- |
| Process ID      | 1234                      |
| Process State   | Ready                     |
| Program Counter | 0x4005C8                  |
| CPU Registers   | AX=10, BX=20              |
| Priority        | 5                         |
| Memory limits   | Base=0x1000, Limit=0x5000 |
| Open files      | [file1.txt, file2.log]    |
| CPU time used   | 250ms                     |

---

## 5. How PCB Works in Practice (Context Switch Walkthrough)

**Scenario**: CPU is running Process A (text editor). OS decides to switch to Process B (music player).

```
CPU running Process A
        │
        ▼
┌─────────────────────────────┐
│ Step 1: Save Process A      │
│  - Save PC → PCB_A          │
│  - Save registers → PCB_A   │
│  - Update state = "Ready"   │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ Step 2: Select Process B    │
│  - Scheduler picks B        │
│  - Retrieve PCB_B           │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ Step 3: Load Process B      │
│  - Load PC from PCB_B       │
│  - Restore registers from B │
│  - Update state = "Running" │
└─────────────────────────────┘
        │
        ▼
CPU resumes Process B from saved instruction
```

This entire sequence takes **milliseconds** — the user perceives both programs as running at the same time.

> The time spent in this switch is **pure overhead** — no useful work is done during it. That's why PCBs are designed to be compact and stored in fast-access memory.

---

## 6. PCB vs Process

| Aspect     | Process                              | PCB                                           |
| ---------- | ------------------------------------ | --------------------------------------------- |
| Definition | A program actively running in memory | Data structure storing info about the process |
| Nature     | Active — performs tasks              | Passive — stores metadata                     |
| Contains   | Code, data, stack, heap              | PID, state, PC, registers, memory info        |
| Purpose    | Execute instructions                 | Help the OS manage the process                |
| Lifetime   | Creation → Termination               | Same as the process it describes              |

> If the process is a **student**, the PCB is the **school's academic record** for that student — maintained by administration, not the student themselves.

---

## 7. Where is the PCB Stored?

- PCBs live in **kernel memory** — a protected region user processes cannot directly access
- The OS maintains a **Process Table**: a table or linked list of all active PCBs
- When the OS needs process info, it looks up the PCB in the Process Table
- Stored in **fast-access memory** since PCBs are read/written during every context switch

```
Kernel Memory
┌─────────────────────────────────────┐
│           Process Table             │
│  ┌─────────┐ ┌─────────┐ ┌───────┐ │
│  │ PCB_101 │ │ PCB_102 │ │ PCB_n │ │
│  └─────────┘ └─────────┘ └───────┘ │
└─────────────────────────────────────┘
         ↑ Only the OS can access this
```

User programs **cannot** read or write PCBs directly — this is a core security guarantee of every OS.

---

## 8. Key Takeaways

- A **PCB** is the OS's complete data record for a process — created on start, deleted on termination.
- It holds: PID, process state, program counter, CPU registers, scheduling info, memory limits, accounting data, and I/O status.
- Without PCBs, the OS cannot pause and resume processes — **context switching would be impossible**.
- PCBs are stored in **protected kernel memory** — user programs cannot access them directly.
- The OS keeps all PCBs in a **Process Table** for fast lookup.
- The time spent reading/writing PCBs during a context switch is pure overhead — PCBs are kept compact to minimize this.
- Analogy: PCB = the process's **identity card + bookmark + resource ledger**, all in one.
