# Context Switching in Operating Systems: How It Works

> **One-line summary:**
> **Context switching** is when the OS saves the full state of the currently running process into its PCB, then loads the saved state of the next process — enabling multitasking on a single CPU.

---

## Table of Contents

1. [What is Context Switching?](#1-what-is-context-switching)
2. [Why is Context Switching Necessary?](#2-why-is-context-switching-necessary)
3. [When Does Context Switching Occur?](#3-when-does-context-switching-occur)
4. [What Happens During a Context Switch?](#4-what-happens-during-a-context-switch)
5. [Step-by-Step Example](#5-step-by-step-example)
6. [Context Switching Overhead](#6-context-switching-overhead)
7. [Context Switching vs Mode Switching](#7-context-switching-vs-mode-switching)
8. [Reducing Overhead](#8-reducing-overhead)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is Context Switching?

**Context switching** is the mechanism where the CPU stops executing one process and starts executing another — while preserving the first process's exact state so it can resume later without any loss.

The **"context"** = all information the CPU needs to resume a process:

- Register values
- Program counter
- Process state
- Memory management info

All of this is stored in the process's **PCB** (Process Control Block).

> Like a **chef cooking multiple dishes** — they work on one dish, pause it at the right moment, switch to another, and return to the first one exactly where they left off. The PCB is their set of notes on each dish.

Without context switching, a CPU could only run **one program at a time**.

---

## 2. Why is Context Switching Necessary?

| Reason             | What it enables                                |
| ------------------ | ---------------------------------------------- |
| **Multitasking**   | Multiple apps run "simultaneously" on one CPU  |
| **Responsiveness** | No single process monopolizes the CPU          |
| **Resource use**   | CPU stays busy while one process waits for I/O |
| **Time-sharing**   | Every process gets fair access to CPU time     |

> Rapid context switching creates the **illusion of parallel execution** — the CPU actually runs only one process at any instant, but switches so fast users can't perceive the gaps.

---

## 3. When Does Context Switching Occur?

Context switching is triggered by specific events, not randomly:

| Trigger                             | What happens                                                           |
| ----------------------------------- | ---------------------------------------------------------------------- |
| **Time slice expires**              | Process used up its CPU quantum → scheduler switches to next process   |
| **Process waits for I/O**           | Process enters Waiting state → CPU switches to another Ready process   |
| **Higher-priority process arrives** | A higher-priority process becomes ready → current process is preempted |
| **Process voluntarily yields**      | Process calls `yield()` or blocks → OS switches to next ready process  |

---

## 4. What Happens During a Context Switch?

A context switch has two phases: **save** the current process, **load** the next process.

### Phase 1: Save the Current Process Context

The OS writes the following from CPU into **PCB of the current process**:

| What's saved           | Why it matters                                    |
| ---------------------- | ------------------------------------------------- |
| Program Counter (PC)   | Which instruction to execute next when it resumes |
| CPU registers          | All intermediate computation values in flight     |
| Process state          | Updated to "Ready" or "Waiting"                   |
| Memory management info | Page tables, memory limits                        |
| I/O status             | Open files, pending I/O requests                  |

### Phase 2: Load the Next Process Context

The OS reads from the **PCB of the next process** and loads it into the CPU:

- Program counter is restored → CPU jumps to where the process last stopped
- All registers are restored → computation resumes from the exact same values
- Process state updated to "Running"
- CPU begins executing the next process seamlessly

```
┌──────────────────────────────────────────────────────────┐
│                    CONTEXT SWITCH                        │
│                                                          │
│  CPU running Process A                                   │
│         ↓  [interrupt / time slice expired]              │
│  OS saves PC, registers, state → PCB_A                   │
│         ↓                                                │
│  OS loads PC, registers, state ← PCB_B                   │
│         ↓                                                │
│  CPU running Process B                                   │
└──────────────────────────────────────────────────────────┘
```

> During the switch itself, the CPU does **zero useful work** — this is pure overhead.

---

## 5. Step-by-Step Example

**Scenario**: Process A (text editor) and Process B (music player) sharing one CPU.

| Step | Action            | Detail                                                    |
| ---- | ----------------- | --------------------------------------------------------- |
| 1    | Process A running | Text editor executes on CPU                               |
| 2    | Timer interrupt   | Time slice for A expires                                  |
| 3    | Save context of A | PC, registers, state saved to PCB_A; state → Ready        |
| 4    | Load context of B | PC, registers, state restored from PCB_B; state → Running |
| 5    | Process B running | Music player now executes on CPU                          |
| 6    | Timer interrupt   | Time slice for B expires                                  |
| 7    | Save context of B | PC, registers, state saved to PCB_B; state → Ready        |
| 8    | Load context of A | PC, registers, state restored from PCB_A; state → Running |
| 9    | Process A running | Text editor resumes exactly where it stopped              |

```mermaid
sequenceDiagram
    participant CPU
    participant OS
    participant PCB_A
    participant PCB_B

    CPU->>OS: Timer interrupt (A's time slice expired)
    OS->>PCB_A: Save PC, registers, state
    OS->>PCB_B: Load PC, registers, state
    OS->>CPU: Resume Process B
    CPU->>OS: Timer interrupt (B's time slice expired)
    OS->>PCB_B: Save PC, registers, state
    OS->>PCB_A: Load PC, registers, state
    OS->>CPU: Resume Process A
```

---

## 6. Context Switching Overhead

Context switching is **pure overhead** — no user work gets done during the switch. This has three cost components:

| Overhead type      | What it means                                                                           |
| ------------------ | --------------------------------------------------------------------------------------- |
| **Time overhead**  | CPU cycles spent saving and restoring context                                           |
| **Cache overhead** | CPU cache gets "polluted" — new process accesses different memory, causing cache misses |
| **TLB overhead**   | Translation Lookaside Buffer (address translation cache) may need flushing              |

**Factors that make context switching more expensive:**

- More CPU registers to save/restore
- Complex memory management (large page tables)
- Processes with large memory footprints
- High switch frequency

**Implication**: Too many context switches = high CPU usage, low actual throughput, sluggish system.

---

## 7. Context Switching vs Mode Switching

These are often confused — they are **different things**:

| Aspect       | Context Switching                      | Mode Switching                         |
| ------------ | -------------------------------------- | -------------------------------------- |
| Definition   | Switching from one process to another  | Switching between user and kernel mode |
| What changes | Entire process context (full PCB swap) | Execution privilege level only         |
| Frequency    | Less frequent                          | More frequent (every system call)      |
| Overhead     | Higher                                 | Lower                                  |
| Example      | Browser → text editor                  | App makes a `read()` system call       |

> Mode switching happens **inside** a process (e.g., when it calls the OS). Context switching happens **between** processes.

---

## 8. Reducing Overhead

OS designers use several techniques to minimize context switch cost:

| Technique                | How it helps                                                                                                                    |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| **Hardware support**     | Some CPUs have multi-register banks or special save/restore instructions                                                        |
| **Efficient scheduling** | Smart time slice length balances responsiveness vs. switch frequency                                                            |
| **Thread use**           | Switching between threads (same process) is much cheaper than between processes — they share memory, only registers need saving |
| **Longer time slices**   | Fewer switches, but reduced responsiveness — OS finds the right balance                                                         |

> Threads within the same process share memory — a **thread context switch** only needs to save/restore registers and stack pointer, not page tables. Much faster.

---

## 9. Key Takeaways

- **Context switching** = OS saves current process state to PCB, loads next process state from PCB, CPU resumes new process.
- The **"context"** = program counter + CPU registers + process state + memory info + I/O status.
- Triggers: time slice expiry, I/O wait, higher-priority process arriving.
- During the switch itself, **no useful work is done** — it is pure overhead.
- Overhead has three parts: time, cache invalidation, TLB flush.
- **Context switch ≠ mode switch**: mode switch changes privilege level within the same process; context switch changes the active process entirely.
- Threads reduce overhead — thread switches are cheaper than process switches because they share memory.
- Too frequent switching → CPU busy but not productive → system feels slow.
