# Advanced CPU Scheduling Algorithms: HRRN to MLQ

> Four advanced scheduling algorithms — HRRN fixes SJF's starvation problem, Round Robin gives every process fair time slices, Priority Scheduling serves critical processes first, and MLQ organizes processes into separate queues each with its own rules.

---

## Table of Contents

1. [Highest Response Ratio Next (HRRN)](#1-highest-response-ratio-next-hrrn)
2. [Round Robin Scheduling](#2-round-robin-scheduling)
3. [Priority-Based Scheduling](#3-priority-based-scheduling)
4. [Multilevel Queue Scheduling (MLQ)](#4-multilevel-queue-scheduling-mlq)
5. [Comparison of All Four Algorithms](#5-comparison-of-all-four-algorithms)
6. [Key Takeaways](#6-key-takeaways)

---

## 1. Highest Response Ratio Next (HRRN)

### What It Is

HRRN is a **non-preemptive** algorithm designed to fix SJF's starvation problem. It picks the next process using a **response ratio** that combines both waiting time and burst time — so processes that have been waiting a long time automatically get higher priority over time.

**Restaurant analogy:** Serve customers based not just on how quick their order is to prepare, but also on how long they've been waiting. No one waits forever.

### The Formula

$$\text{Response Ratio} = \frac{\text{Waiting Time} + \text{Burst Time}}{\text{Burst Time}}$$

- A fresh process with no waiting time has ratio = 1.0
- As waiting time grows, ratio grows → process climbs the priority ladder
- **Longest-waiting processes eventually overtake short ones** → no starvation

### HRRN Example

| Process | Burst Time |
| ------- | ---------- |
| P1      | 6          |
| P2      | 3          |
| P3      | 8          |
| P4      | 4          |

All arrive at t=0. Initial ratio for all = (0 + BT) / BT = **1.0** → pick shortest (P2) as tiebreaker.

**Step 1: Execute P2** (t=0 → t=3). Recalculate ratios with waiting=3:

| Process | Ratio = (3 + BT) / BT            |
| ------- | -------------------------------- |
| P1      | (3 + 6) / 6 = **1.50**           |
| P3      | (3 + 8) / 8 = **1.375**          |
| P4      | (3 + 4) / 4 = **1.75** ← highest |

**Step 2: Execute P4** (t=3 → t=7). Recalculate with waiting=7:

| Process | Ratio = (7 + BT) / BT            |
| ------- | -------------------------------- |
| P1      | (7 + 6) / 6 = **2.17** ← highest |
| P3      | (7 + 8) / 8 = **1.875**          |

**Step 3: Execute P1** (t=7 → t=13).  
**Step 4: Execute P3** (t=13 → t=21).

**Execution order: P2 → P4 → P1 → P3**

Notice: P1 (longer job) got ahead of P3 because it had been waiting longer — fairness maintained.

### Pros and Cons

| Pros                                                   | Cons                                             |
| ------------------------------------------------------ | ------------------------------------------------ |
| No starvation — waiting time increases ratio over time | Non-preemptive — can't interrupt running process |
| Balances short job efficiency with fairness            | Requires burst time knowledge in advance         |
| Better than SJF for mixed workloads                    | More calculation per scheduling decision         |

---

## 2. Round Robin Scheduling

### What It Is

Round Robin is a **preemptive** algorithm built for **time-sharing systems**. Every process gets a small fixed unit of CPU time called a **time quantum** (or time slice), typically 10–100 ms. When the quantum expires, the process moves to the back of the ready queue.

**Carousel analogy:** Each child gets exactly 2 minutes on the ride, then the next child goes. Everyone gets a fair turn.

### How It Works

```
  Ready Queue (FIFO):
  [P1] [P2] [P3] [P4]
    │
    ▼  Run for 1 quantum
  CPU
    │
    ├── If process COMPLETES within quantum → exits
    └── If quantum EXPIRES → goes to BACK of queue → next process gets CPU
```

### Round Robin Example

Processes (all arrive at t=0), **time quantum = 4**:

| Process | Burst Time |
| ------- | ---------- |
| P1      | 10         |
| P2      | 4          |
| P3      | 6          |
| P4      | 3          |

```
Step-by-step execution:
  t=0-4:   P1 runs (remaining: 10-4 = 6)
  t=4-8:   P2 runs (remaining: 4-4 = 0) ✓ DONE
  t=8-12:  P3 runs (remaining: 6-4 = 2)
  t=12-15: P4 runs (remaining: 3-3 = 0) ✓ DONE  (only 3 left, finishes early)
  t=15-19: P1 runs (remaining: 6-4 = 2)
  t=19-21: P3 runs (remaining: 2-2 = 0) ✓ DONE
  t=21-23: P1 runs (remaining: 2-2 = 0) ✓ DONE

Gantt Chart:
  0    4    8    12  15   19  21  23
  │ P1 │ P2 │ P3 │P4 │ P1 │P3 │P1 │
```

| Process | Completion | Turnaround (CT−AT) | Waiting (TAT−BT) |
| ------- | ---------- | ------------------ | ---------------- |
| P1      | 23         | 23 − 0 = **23**    | 23 − 10 = **13** |
| P2      | 8          | 8 − 0 = **8**      | 8 − 4 = **4**    |
| P3      | 21         | 21 − 0 = **21**    | 21 − 6 = **15**  |
| P4      | 15         | 15 − 0 = **15**    | 15 − 3 = **12**  |

**Average Waiting Time = (13 + 4 + 15 + 12) / 4 = 11 units**

### Choosing the Right Time Quantum

| Quantum Size          | Effect                                                                       |
| --------------------- | ---------------------------------------------------------------------------- |
| Too small (e.g., 1ms) | Excessive context switches — system wastes time switching instead of running |
| Too large (e.g., ∞)   | Degenerates into FCFS — loses time-sharing benefit                           |
| Just right            | 80% of processes complete within one quantum                                 |

```
  Quantum too small:
  P1[1ms] switch P2[1ms] switch P3[1ms] switch...
  ← lots of overhead, little useful work →

  Quantum too large:
  P1[runs forever]... = FCFS
```

**Typical values in real systems: 10–100 milliseconds**

### Pros and Cons

| Pros                                       | Cons                                        |
| ------------------------------------------ | ------------------------------------------- |
| No starvation — every process gets a turn  | Higher context switch overhead              |
| Good response time for interactive systems | Average waiting time can be high            |
| Simple and fair                            | Performance depends heavily on quantum size |

---

## 3. Priority-Based Scheduling

### What It Is

Each process is assigned a **priority number**. The CPU is given to the process with the **highest priority** (typically: smaller number = higher priority).

**Emergency room analogy:** Heart attack patient is treated before someone with a minor cut — severity (priority), not arrival order, determines service order.

### Priority Assignment

- **Internal:** Set by OS based on memory needs, time limits, I/O vs CPU usage
- **External:** Set by user/admin based on process importance, department, billing

### Preemptive vs Non-Preemptive Priority

| Type           | Behavior                                                          |
| -------------- | ----------------------------------------------------------------- |
| Non-preemptive | Running process finishes even if higher-priority process arrives  |
| Preemptive     | Running process is interrupted if higher-priority process arrives |

### Priority Scheduling Example (Non-Preemptive)

Lower number = higher priority:

| Process | Burst | Priority | Arrival |
| ------- | ----- | -------- | ------- |
| P1      | 6     | 2        | 0       |
| P2      | 3     | 1        | 0       |
| P3      | 8     | 3        | 0       |
| P4      | 4     | 2        | 0       |

```
  t=0:  P2 has highest priority (1) → P2 runs (completes at t=3)
  t=3:  P1 and P4 both have priority 2 → tiebreak by arrival → P1 runs (completes at t=9)
  t=9:  P4 runs (completes at t=13)
  t=13: P3 runs (completes at t=21)

  Execution order: P2 → P1 → P4 → P3
```

### Starvation and Aging

**Problem:** Low-priority processes may **wait indefinitely** if high-priority processes keep arriving.

**Solution — Aging:** Gradually increase the priority of processes that have been waiting a long time.

```
  Example aging rule: increase priority by 1 every 10 minutes of waiting

  P5 starts at priority 10:
  After 10 min → priority 9
  After 20 min → priority 8
  ...
  After 90 min → priority 1 (highest) → finally gets CPU

  No process waits forever.
```

### Pros and Cons

| Pros                                     | Cons                                                  |
| ---------------------------------------- | ----------------------------------------------------- |
| Critical processes execute first         | Starvation for low-priority processes (without aging) |
| Flexible — internal or external priority | Priority inversion problems in preemptive mode        |
| Can model real-world urgency well        | Requires careful priority assignment                  |

---

## 4. Multilevel Queue Scheduling (MLQ)

### What It Is

MLQ **divides the ready queue into multiple separate queues** based on process type. Each queue has its own scheduling algorithm. Processes are **permanently assigned** to one queue.

**Airport analogy:** First Class, Business, Economy — separate check-in lines, each moves by different rules, but all eventually board.

### Typical MLQ Structure

```
  ┌─────────────────────────────────────┐  ← Highest Priority
  │  Queue 1: System Processes          │  (Preemptive Priority)
  ├─────────────────────────────────────┤
  │  Queue 2: Interactive Processes     │  (Round Robin)
  ├─────────────────────────────────────┤
  │  Queue 3: Batch Processes           │  (FCFS)
  └─────────────────────────────────────┘  ← Lowest Priority

  Rule: Higher-priority queue MUST be empty before lower queue gets CPU.
```

Each queue can use a **different algorithm** suited to its process type.

### MLQ Example

Two queues:

- **Foreground (Interactive):** Round Robin, quantum = 4 — gets 80% CPU time
- **Background (Batch):** FCFS — gets 20% CPU time

| Process | Type       | Burst |
| ------- | ---------- | ----- |
| P1      | Foreground | 8     |
| P2      | Background | 10    |
| P3      | Foreground | 6     |

```
  Foreground (Round Robin, q=4):
  t=0-4:   P1 runs (remaining: 4)
  t=4-8:   P3 runs (remaining: 2)
  t=8-12:  P1 runs (remaining: 0) ✓ DONE
  t=12-14: P3 runs (remaining: 0) ✓ DONE

  Background (FCFS — starts only after foreground is empty):
  t=14-24: P2 runs ✓ DONE
```

### Scheduling Between Queues

| Method         | Description                                               |
| -------------- | --------------------------------------------------------- |
| Fixed Priority | Higher queues always preempt lower ones (starvation risk) |
| Time Slice     | Each queue gets a fixed % of CPU time (e.g., 80/20 split) |

### Pros and Cons

| Pros                                           | Cons                                                      |
| ---------------------------------------------- | --------------------------------------------------------- |
| Different process types get optimal scheduling | Processes **cannot move between queues** — inflexible     |
| System processes always get priority           | Low-priority queues can starve if higher queues stay busy |
| Modular — each queue is independent            | Classification at start may not match behavior later      |

> This inflexibility led to **Multilevel Feedback Queue (MLFQ)** — allows processes to move between queues based on actual behavior. If a process uses too much CPU it moves to a lower queue; if it waits too long it moves up.

---

## 5. Comparison of All Four Algorithms

| Algorithm   | Preemptive? | Starvation?             | Best Use Case                         |
| ----------- | ----------- | ----------------------- | ------------------------------------- |
| HRRN        | No          | No                      | Batch systems needing fairness        |
| Round Robin | Yes         | No                      | Time-sharing, interactive systems     |
| Priority    | Both        | Yes (without aging)     | Real-time systems, critical processes |
| MLQ         | Yes         | Possible (lower queues) | Systems with distinct process types   |

### Quick Decision Guide

```
  Need fairness + no starvation + non-preemptive?  → HRRN
  Need equal turns for interactive users?           → Round Robin
  Some processes are genuinely more important?      → Priority (with aging)
  Have distinct process categories?                 → MLQ
  Need processes to move between categories?        → MLFQ (next step)
```

---

## 6. Key Takeaways

- **HRRN** = SJF + fairness. Response Ratio = (Wait + Burst) / Burst. Longer-waiting processes grow their ratio → no starvation. Non-preemptive.
- **Round Robin** = fairness through time slicing. Every process gets a quantum, then goes to the back. Quantum size is critical — too small wastes time on context switches, too large becomes FCFS.
- **Priority Scheduling** = urgent processes first. Can be preemptive or non-preemptive. **Starvation risk** — solve with **aging** (increase priority of long-waiting processes).
- **MLQ** = separate queues for different process types, each with its own algorithm. Processes cannot move between queues — fixed assignment. Can starve lower queues.
- MLFQ (Multilevel Feedback Queue) improves MLQ by allowing processes to move between queues based on behavior — the algorithm used in most real modern operating systems.
- All advanced algorithms are trade-offs: no single algorithm is best for every situation — the right choice depends on system type (batch vs interactive vs real-time) and what you're optimizing for (throughput, response time, or fairness).
