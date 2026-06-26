# Preemptive vs Non-Preemptive CPU Scheduling

> CPU scheduling decides which process gets the CPU next — non-preemptive lets a process run until it's done, while preemptive lets the OS forcibly take the CPU away and give it to another process.

---

## Table of Contents

1. [What is CPU Scheduling?](#1-what-is-cpu-scheduling)
2. [Non-Preemptive Scheduling](#2-non-preemptive-scheduling)
3. [Preemptive Scheduling](#3-preemptive-scheduling)
4. [Side-by-Side Comparison](#4-side-by-side-comparison)
5. [Advantages and Disadvantages](#5-advantages-and-disadvantages)
6. [Real-World Analogy](#6-real-world-analogy)
7. [Which Do Modern Operating Systems Use?](#7-which-do-modern-operating-systems-use)
8. [Impact on Process States](#8-impact-on-process-states)
9. [Context Switching Impact](#9-context-switching-impact)
10. [When to Choose Which Type](#10-when-to-choose-which-type)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What is CPU Scheduling?

CPU scheduling is the process of **deciding which process in the ready queue gets to execute on the CPU**. The scheduler must answer:

- Which process runs next?
- When should a running process be taken off the CPU?

This decision-making approach defines whether scheduling is **preemptive** or **non-preemptive**.

**Teacher analogy:** A teacher managing students who want to ask questions can either:

- Let each student finish their entire question before the next one speaks (non-preemptive)
- Interrupt students mid-question to give others a chance (preemptive)

---

## 2. Non-Preemptive Scheduling

### Definition

Once the CPU is allocated to a process, **that process keeps the CPU until it voluntarily releases it** — either by completing or entering a waiting state (like waiting for I/O).

The OS **cannot forcibly take away the CPU** from a running process.

**Theater analogy:** Once a movie starts in a single-screen theater, it plays until the end before the next movie begins.

### Key Characteristics

- Process runs until **completion or until it blocks** for I/O
- No forced interruption by the scheduler
- Simple to implement and understand
- Lower scheduling overhead
- Can lead to **poor response times** for interactive processes

### When is CPU Released?

Only in two situations:

1. Process **completes** and terminates
2. Process **voluntarily waits** (requests I/O, waits for user input)

The scheduler cannot interrupt even if a higher-priority process arrives.

### Simple Example

```
Processes:  P1 (10ms)   P2 (5ms)   P3 (8ms)
Arrival:    All arrive at t=0

Timeline (Non-Preemptive, P1 starts first):
  0         10        15       23
  ├── P1 ────┼── P2 ──┼── P3 ──┤
             ↑         ↑
       P1 finishes   P2 finishes
       P2 starts     P3 starts

No interruptions — each runs to completion before next begins.
```

---

## 3. Preemptive Scheduling

### Definition

The OS **can interrupt a running process** and allocate the CPU to another process. The scheduler actively decides when to switch based on priority, time quantum, or other factors.

**Kitchen analogy:** A chef switches between preparing multiple orders — doesn't finish one dish completely before starting another, multitasks to serve everyone efficiently.

### Key Characteristics

- Processes **can be interrupted** during execution
- Higher scheduling overhead due to frequent context switches
- **Better response time** for interactive processes
- Requires mechanisms to protect data consistency
- More complex to implement

### When Does Preemption Occur?

1. A **higher priority process** arrives in the ready queue
2. A process **uses up its time slice** (time quantum) — e.g., Round Robin
3. The scheduler decides another waiting process needs a fair turn

The OS has control and can **forcibly take the CPU away** from a running process.

### Simple Example

```
Processes:  P1 (10ms, priority 3)   P2 (5ms, priority 1)   P3 (8ms, priority 2)
           (1 = highest priority)

Timeline (Preemptive priority scheduling):
  0    2         7              15       23
  ├─P1─┼─── P2 ──┼───── P3 ─────┼── P1 ──┤
       ↑          ↑               ↑
   P2 arrives,  P2 done,        P3 done,
   preempts P1  P3 gets CPU     P1 resumes

P1 runs 2ms → preempted by P2 → P2 completes → P3 completes → P1 finishes remaining 8ms
```

---

## 4. Side-by-Side Comparison

| Aspect            | Non-Preemptive                          | Preemptive                            |
| ----------------- | --------------------------------------- | ------------------------------------- |
| CPU Control       | Process keeps CPU until done or blocked | OS can take CPU away anytime          |
| Interruption      | No forced interruption                  | Can interrupt running processes       |
| Context Switching | Less frequent                           | More frequent                         |
| Overhead          | Lower                                   | Higher                                |
| Response Time     | Can be poor for waiting processes       | Generally better and more predictable |
| Complexity        | Simple to implement                     | More complex                          |
| Fairness          | Less fair (long processes block others) | More fair to all processes            |
| Starvation Risk   | Higher (short processes may wait long)  | Lower (can ensure all get CPU time)   |
| Use Cases         | Batch systems, embedded systems         | Interactive systems, multitasking OS  |
| Data Consistency  | Easier to maintain                      | Requires synchronization mechanisms   |

---

## 5. Advantages and Disadvantages

### Non-Preemptive

**Advantages:**

- Simple and easy to implement
- Lower overhead — fewer context switches
- Predictable execution for batch processing
- No need for complex synchronization
- Works well for long-running computational tasks

**Disadvantages:**

- Poor response time for short or interactive processes
- Can cause **starvation** if long processes keep arriving
- Not suitable for time-sharing systems
- Cannot handle urgent tasks that arrive later
- Less fair CPU allocation

### Preemptive

**Advantages:**

- Better response time for interactive applications
- Fairer CPU allocation among processes
- Can handle **high-priority urgent tasks immediately**
- Reduces waiting time for most processes
- Essential for modern multitasking operating systems

**Disadvantages:**

- Higher overhead from frequent context switches
- More complex implementation
- Requires mechanisms to protect shared data
- Potential for **race conditions** if not handled properly
- Increases system complexity overall

---

## 6. Real-World Analogy

**Bank customer service desk:**

```
  NON-PREEMPTIVE                      PREEMPTIVE
  ┌──────────────────────────┐        ┌──────────────────────────┐
  │ Rep helps Customer A     │        │ Rep helps Customer A     │
  │ completely (30 min)      │        │ Customer B arrives with  │
  │ before helping anyone    │        │ a quick question →       │
  │ else, even if Customer B │        │ Rep pauses A, helps B    │
  │ only needs 1 minute      │        │ (2 min), returns to A    │
  └──────────────────────────┘        └──────────────────────────┘
  Simple and orderly, but              More effort to track where
  frustrating for quick requests       things left off, but fairer
```

---

## 7. Which Do Modern Operating Systems Use?

**Most modern operating systems (Windows, Linux, macOS) use preemptive scheduling.**

Why: Users expect responsive systems that handle multiple tasks simultaneously. When you're typing in a word processor, downloading files, and playing music at the same time, you want all apps to respond quickly. Preemptive scheduling ensures no single process monopolizes the CPU.

**Exception:** Some embedded and real-time systems may use non-preemptive scheduling for specific tasks where **predictability and simplicity** are more important than responsiveness.

---

## 8. Impact on Process States

Scheduling type directly affects how processes transition between states:

```
  NON-PREEMPTIVE state transitions:
  New → Ready → Running → Waiting (voluntary)
                        → Terminated (voluntary)
  Running NEVER goes back to Ready (no forced interruption)

  PREEMPTIVE state transitions:
  New → Ready → Running → Waiting (voluntary I/O)
                        → Terminated (completion)
                        → Ready ← (forced preemption!)
                               ↑
                    This transition only exists in preemptive!
```

In **preemptive** scheduling, a running process can be moved **back to the ready state** by the scheduler. This is the defining difference.

---

## 9. Context Switching Impact

| Factor                    | Non-Preemptive                  | Preemptive                                         |
| ------------------------- | ------------------------------- | -------------------------------------------------- |
| How often switches happen | Only at completion or I/O block | Frequently — at each time slice or priority change |
| Context switch overhead   | Low                             | High                                               |
| Cache efficiency          | Better (process runs longer)    | Worse (frequent switches flush cache)              |
| Responsiveness            | Poor for interactive tasks      | Good for interactive tasks                         |

The trade-off: **more context switches = more overhead but better responsiveness**.

Modern OS designers accept this overhead because user experience requires it.

---

## 10. When to Choose Which Type

### Choose Non-Preemptive when:

- Building **batch processing systems** (jobs run to completion without user interaction)
- System has **simple, predictable workloads**
- Minimal overhead and simple implementation are priorities
- **Embedded systems** with limited resources and predictable tasks

### Choose Preemptive when:

- Building **interactive, multi-user systems**
- **Response time** is critical for user experience
- Need to handle processes with **varying priorities**
- System must be **fair** to all running processes
- Developing a **general-purpose operating system**

---

## 11. Key Takeaways

- **Non-preemptive:** Process keeps CPU until done or blocked — simpler, lower overhead, but poor responsiveness
- **Preemptive:** OS can forcibly take CPU away — complex, higher overhead, but responsive and fair
- The key difference: in preemptive, a running process can transition **back to the Ready state** (forced by OS); in non-preemptive, it cannot
- **Modern OS use preemptive scheduling** — users expect all apps to respond even when many are running
- Non-preemptive scheduling can cause **starvation** — a short process waiting behind a very long one
- Preemptive scheduling needs **synchronization mechanisms** to prevent race conditions on shared data
- Both have their place — the right choice depends on system requirements (interactivity vs. simplicity)
- These concepts are the foundation for understanding **specific CPU scheduling algorithms** (FCFS, SJF, Round Robin, etc.)
