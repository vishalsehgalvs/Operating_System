# Process States in OS: Process Concept Explained

> **One-line summary:**
> A process is a program actively running in memory. It moves through five states — **New → Ready → Running → Waiting → Terminated** — managed by the OS to maximize CPU efficiency and multitasking.

---

## Table of Contents

1. [What is a Process?](#1-what-is-a-process)
2. [Components of a Process](#2-components-of-a-process)
3. [Process vs Program: Quick Recap](#3-process-vs-program-quick-recap)
4. [The Five Process States](#4-the-five-process-states)
5. [State Transitions](#5-state-transitions)
6. [State Transition Diagram](#6-state-transition-diagram)
7. [Practical Example: Text Editor Process](#7-practical-example-text-editor-process)
8. [Why Process States Matter](#8-why-process-states-matter)
9. [Common Misconceptions](#9-common-misconceptions)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What is a Process?

A **process** is a program in execution — the active, living instance of a program loaded into memory with all its resources.

> Like a **recipe vs actually cooking**. The recipe book on your shelf is the program — just instructions, doing nothing. When you start following it with real ingredients and utensils, that's the process — the recipe has come to life.

When you launch an application, the OS:

1. Loads the program from disk into memory
2. Allocates CPU time, memory space, and other resources
3. Creates a **process** to manage execution

---

## 2. Components of a Process

| Component           | What it holds                                                       | Analogy                          |
| ------------------- | ------------------------------------------------------------------- | -------------------------------- |
| **Code (Text)**     | The actual instructions to be executed                              | The recipe steps                 |
| **Program Counter** | Tracks which instruction executes next                              | Your finger pointing at the step |
| **Stack**           | Temporary data — function params, local variables, return addresses | Your notepad of in-progress work |
| **Data Section**    | Global variables                                                    | Constants you reference often    |
| **Heap**            | Dynamically allocated memory during runtime                         | Extra bowls grabbed as needed    |

```
┌────────────────────────────┐  ← High address
│          Stack             │  ← Grows downward (function calls, local vars)
│            ↓               │
│                            │
│            ↑               │
│           Heap             │  ← Grows upward (dynamic allocations)
├────────────────────────────┤
│        Data Section        │  ← Global/static variables
├────────────────────────────┤
│     Code (Text Section)    │  ← Program instructions
└────────────────────────────┘  ← Low address
```

---

## 3. Process vs Program: Quick Recap

| Aspect             | Program                 | Process                     |
| ------------------ | ----------------------- | --------------------------- |
| Nature             | Passive entity          | Active entity               |
| Location           | Stored on disk          | Loaded in memory            |
| Lifespan           | Permanent until deleted | Temporary during execution  |
| Resources          | None allocated          | CPU, memory, I/O allocated  |
| Multiple instances | Single copy (file)      | Multiple processes possible |

> Opening three browser windows = **one program**, **three processes** — each with its own memory space and execution state.

---

## 4. The Five Process States

A process moves through five states during its lifetime. The OS manages every transition to ensure efficient CPU use and smooth multitasking.

---

### State 1: New

The process has just been **created**. The OS is setting up memory, allocating resources, and preparing the execution environment. The process is not yet eligible to run.

> Like a **flight scheduled but not yet cleared for takeoff** — it exists in the system, but isn't active on the runway yet.

---

### State 2: Ready

The process is **fully prepared to execute** and is waiting in the **ready queue** for the CPU to be assigned to it.

> Like **students with their hands raised** — all prepared to answer, but only one gets called at a time.

- Multiple processes can be in Ready simultaneously
- The **CPU scheduler** picks which one runs next based on the scheduling algorithm

---

### State 3: Running

The CPU scheduler has **assigned the CPU** to this process — it is now **actively executing instructions**.

> Like the student who was called on — they are now speaking.

- On a single-core CPU: only **one process** can be Running at any instant
- On multi-core: one process per core can run simultaneously
- A running process continues until: it completes, needs I/O, or is interrupted

---

### State 4: Waiting (Blocked)

The process is **waiting for an external event** — disk I/O, user input, network response — and cannot continue until it arrives.

> Like **waiting for food delivery** — you can't eat until it arrives, so you do nothing in the meantime.

- The process **voluntarily gives up the CPU**
- CPU is freed for other ready processes
- Once the event occurs, it moves back to **Ready** (not directly to Running)

---

### State 5: Terminated (Exit)

The process has **finished execution** (or was killed). The OS hasn't yet fully cleaned up its resources.

> Like **checking out from a hotel** — you've decided to leave, but the hotel still needs to process checkout, clean the room, and update records before it's truly done.

- After cleanup: memory freed, files closed, process removed from OS tables

---

## 5. State Transitions

Every transition has a specific trigger:

| Transition               | Trigger                                                            |
| ------------------------ | ------------------------------------------------------------------ |
| **New → Ready**          | OS finishes process creation and admits it to the ready queue      |
| **Ready → Running**      | CPU scheduler selects this process and assigns it the CPU          |
| **Running → Waiting**    | Process requests I/O or waits for an event (voluntary)             |
| **Waiting → Ready**      | The awaited I/O or event completes — process is eligible again     |
| **Running → Ready**      | Preemption — time slice expires or higher-priority process arrives |
| **Running → Terminated** | Process completes normally or is killed due to error               |

> Key rule: **Waiting → Ready**, never Waiting → Running directly. The process must re-enter the queue and wait for the scheduler again.

---

## 6. State Transition Diagram

```
        ┌─────────────────────────────────────────────────────────┐
        │                                                         │
        ▼                                                         │
     ┌─────┐   admitted    ┌───────┐  scheduler   ┌─────────┐   │ preempted
     │ NEW │ ────────────► │ READY │ ────────────► │ RUNNING │ ──┘
     └─────┘               └───────┘  dispatch     └─────────┘
                               ▲                       │   │
                               │  I/O or event         │   │ exit
                               │  completes            │   ▼
                               │                   ┌─────────┐    ┌────────────┐
                               └────────────────── │ WAITING │    │ TERMINATED │
                                                   └─────────┘    └────────────┘
                                                 (I/O or event wait)
```

```mermaid
stateDiagram-v2
    [*] --> New
    New --> Ready : admitted
    Ready --> Running : scheduler dispatch
    Running --> Ready : preempted / time slice expires
    Running --> Waiting : I/O or event wait
    Waiting --> Ready : I/O or event completes
    Running --> Terminated : exit / completion
    Terminated --> [*]
```

---

## 7. Practical Example: Text Editor Process

Tracing a single session of opening and using a text editor:

| Step                          | State               | What's happening                                |
| ----------------------------- | ------------------- | ----------------------------------------------- |
| Double-click text editor icon | **New**             | OS creates process, allocates memory            |
| Setup complete                | **Ready**           | Waiting in queue for CPU                        |
| Scheduler picks it            | **Running**         | App window appears on screen                    |
| Click "Open File"             | **Waiting**         | Process requests disk I/O — waits for file data |
| File data arrives from disk   | **Ready**           | Back in queue, waiting for CPU                  |
| Scheduler picks it again      | **Running**         | File contents displayed in editor               |
| You type text                 | **Running**         | Process executes continuously as you edit       |
| Click "Save"                  | **Waiting**         | Writes to disk — waits for I/O to complete      |
| Save finishes                 | **Ready → Running** | Shows "Save successful" message                 |
| You close the editor          | **Terminated**      | Cleanup begins — memory freed, files closed     |

---

## 8. Why Process States Matter

### Efficient Resource Utilization

When a process enters Waiting (e.g., reading from disk), the CPU would otherwise sit idle. Instead, the OS runs another Ready process — just like a chef who starts another dish while one bakes in the oven.

### Multitasking Capability

Rapid transitions between Running and Ready create the **illusion of simultaneous execution** on single-core CPUs. This is how you listen to music, browse, and edit a document "at the same time."

### System Responsiveness

Preemption (Running → Ready) lets high-priority processes cut in line, preventing one misbehaving or CPU-heavy program from freezing the entire system.

---

## 9. Common Misconceptions

| Misconception                        | Reality                                                                                                       |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| Waiting = Ready                      | Ready means "can run right now if given CPU." Waiting means "cannot run even with CPU — blocked on an event." |
| Only the running process has a state | Every process in the system is always in exactly one state simultaneously                                     |
| States are physical memory locations | States are logical labels — the process's code/data stays in the same memory regardless of state              |

---

## 10. Key Takeaways

- A **process** = program actively running in memory with CPU, memory, and I/O resources allocated.
- Every process has five components: code section, program counter, stack, data section, heap.
- **Five states**: New → Ready → Running → Waiting → Terminated.
- **New**: being created. **Ready**: waiting for CPU. **Running**: using CPU. **Waiting**: blocked on I/O/event. **Terminated**: done, being cleaned up.
- Waiting → Ready (never directly to Running) — must re-queue.
- Running → Ready happens via **preemption** (time slice expiry or higher-priority process).
- Process states enable: CPU efficiency, multitasking, and system responsiveness.
- A process skips Waiting only if it never needs I/O — but always passes through New and Terminated.
