# CPU Scheduling Algorithms: FCFS, SJF & SRTF

> Three foundational CPU scheduling algorithms — FCFS serves processes in arrival order, SJF picks the shortest job next, and SRTF preempts the current process if a shorter one arrives — each making a different trade-off between simplicity and efficiency.

---

## Table of Contents

1. [What Is CPU Scheduling?](#1-what-is-cpu-scheduling)
2. [Important Scheduling Metrics](#2-important-scheduling-metrics)
3. [First Come First Serve (FCFS)](#3-first-come-first-serve-fcfs)
4. [Shortest Job First (SJF)](#4-shortest-job-first-sjf)
5. [Shortest Remaining Time First (SRTF)](#5-shortest-remaining-time-first-srtf)
6. [Comparison: FCFS vs SJF vs SRTF](#6-comparison-fcfs-vs-sjf-vs-srtf)
7. [Practical Considerations](#7-practical-considerations)
8. [When to Use Which Algorithm](#8-when-to-use-which-algorithm)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What Is CPU Scheduling?

CPU scheduling is the process of **selecting which process from the ready queue gets the CPU next**. The scheduler uses a specific algorithm to make this decision.

Different algorithms have different goals:

- Minimize waiting time
- Maximize CPU utilization
- Ensure fairness

**Restaurant analogy:** The strategy a manager uses to decide which table to serve next affects customer satisfaction, wait times, and overall efficiency.

---

## 2. Important Scheduling Metrics

These metrics are used to evaluate and compare scheduling algorithms:

| Metric          | Definition                               | Formula                                    |
| --------------- | ---------------------------------------- | ------------------------------------------ |
| Arrival Time    | When process enters the ready queue      | Given                                      |
| Burst Time      | Total CPU time needed to complete        | Given (also called execution/service time) |
| Completion Time | Exact moment process finishes            | Measured from time 0                       |
| Turnaround Time | Total time from arrival to completion    | Completion Time − Arrival Time             |
| Waiting Time    | Time spent waiting in the ready queue    | Turnaround Time − Burst Time               |
| Response Time   | Time from arrival until first CPU access | For non-preemptive: = Waiting Time         |

```
  TIMELINE for a single process:

  Arrival          First gets CPU        Completes
     │                   │                  │
     ▼                   ▼                  ▼
  ───┼───────────────────┼──────────────────┼───►
     │←── Waiting Time ──│                  │
     │                   │←── Burst Time ──►│
     │←────────── Turnaround Time ─────────►│
```

---

## 3. First Come First Serve (FCFS)

### What It Is

The **simplest scheduling algorithm** — the process that arrives first gets executed first, like a queue at a ticket counter.

- **Non-preemptive** — once started, runs to completion
- Processes are maintained in a FIFO queue

### How It Works

1. CPU becomes available
2. Pick the process at the **front of the queue** (earliest arrival)
3. Run it to completion
4. Repeat

### Example 1 — All arrive at time 0

| Process | Arrival | Burst |
| ------- | ------- | ----- |
| P1      | 0       | 24    |
| P2      | 0       | 3     |
| P3      | 0       | 3     |

```
Gantt Chart (FCFS, order: P1 → P2 → P3):
  0                   24  27  30
  ├──────── P1 ────────┤P2 │P3 │
```

| Process | Completion | Turnaround (CT−AT) | Waiting (TAT−BT) |
| ------- | ---------- | ------------------ | ---------------- |
| P1      | 24         | 24 − 0 = **24**    | 24 − 24 = **0**  |
| P2      | 27         | 27 − 0 = **27**    | 27 − 3 = **24**  |
| P3      | 30         | 30 − 0 = **30**    | 30 − 3 = **27**  |

**Average Waiting Time = (0 + 24 + 27) / 3 = 17 ms**

### Convoy Effect

P2 and P3 (each needing only 3ms) wait 24ms and 27ms because P1 ran first. This is called the **convoy effect** — short processes get stuck behind a long one.

> Like standing behind someone with a full shopping cart when you only have one item.

### Pros and Cons

| Pros                 | Cons                                      |
| -------------------- | ----------------------------------------- |
| Simple to implement  | Convoy effect — poor average waiting time |
| Fair (arrival order) | Not suitable for time-sharing systems     |
| No starvation        | Long processes block shorter ones         |

---

## 4. Shortest Job First (SJF)

### What It Is

Selects the process with the **smallest burst time** first. Aims to minimize average waiting time.

- **Non-preemptive** — once started, runs to completion even if a shorter job arrives
- Optimal for minimizing average waiting time when all processes arrive at time 0

### How It Works

1. CPU becomes available
2. Look at **all processes in the ready queue**
3. Pick the one with **shortest burst time**
4. Run it to completion

### Example 1 — All arrive at time 0 (same processes as FCFS)

| Process | Arrival | Burst |
| ------- | ------- | ----- |
| P1      | 0       | 24    |
| P2      | 0       | 3     |
| P3      | 0       | 3     |

```
Gantt Chart (SJF: shortest burst first — P2=P3=3, then P1=24):
  0   3   6                    30
  │P2 │P3 │──────── P1 ────────│
```

| Process | Completion | Turnaround      | Waiting         |
| ------- | ---------- | --------------- | --------------- |
| P2      | 3          | 3 − 0 = **3**   | 3 − 3 = **0**   |
| P3      | 6          | 6 − 0 = **6**   | 6 − 3 = **3**   |
| P1      | 30         | 30 − 0 = **30** | 30 − 24 = **6** |

**Average Waiting Time = (0 + 3 + 6) / 3 = 3 ms** — much better than FCFS's 17ms!

### Example 2 — Different arrival times

| Process | Arrival | Burst |
| ------- | ------- | ----- |
| P1      | 0       | 8     |
| P2      | 1       | 4     |
| P3      | 2       | 9     |
| P4      | 3       | 5     |

```
Step-by-step (non-preemptive — P1 already running when others arrive):
  t=0:  Only P1 available → P1 starts (burst 8)
  t=1:  P2 arrives, but P1 keeps running (non-preemptive)
  t=2:  P3 arrives, P1 keeps running
  t=3:  P4 arrives, P1 keeps running
  t=8:  P1 completes. Ready queue: P2(4), P3(9), P4(5)
        SJF picks P2 (shortest burst = 4)
  t=12: P2 completes. Ready queue: P3(9), P4(5)
        SJF picks P4 (burst = 5)
  t=17: P4 completes. Only P3 left
  t=26: P3 completes

Gantt Chart:
  0        8    12     17         26
  ├── P1 ──┤ P2 ├─ P4 ─┤─── P3 ───┤
```

| Process | Completion | Turnaround (CT−AT) | Waiting (TAT−BT) |
| ------- | ---------- | ------------------ | ---------------- |
| P1      | 8          | 8 − 0 = **8**      | 8 − 8 = **0**    |
| P2      | 12         | 12 − 1 = **11**    | 11 − 4 = **7**   |
| P3      | 26         | 26 − 2 = **24**    | 24 − 9 = **15**  |
| P4      | 17         | 17 − 3 = **14**    | 14 − 5 = **9**   |

**Average Waiting Time = (0 + 7 + 15 + 9) / 4 = 7.75 ms**

### Pros and Cons

| Pros                                          | Cons                                               |
| --------------------------------------------- | -------------------------------------------------- |
| Minimum average waiting time (optimal at t=0) | Hard to predict burst time in advance              |
| Better than FCFS for mixed workloads          | Long processes can **starve**                      |
|                                               | Not preemptive — can't react to new short arrivals |

---

## 5. Shortest Remaining Time First (SRTF)

### What It Is

The **preemptive version of SJF**. At any moment, if a new process arrives with a **shorter remaining time** than the currently running process, the CPU is preempted and given to the new process.

- **Preemptive** — running process can be interrupted
- Compares new arrival's burst time vs. current process's **remaining** time
- Gives the best average waiting time of the three

### How It Works

1. New process arrives → compare its burst time to current process's remaining time
2. If new process is shorter → **preempt** current process, give CPU to new one
3. Preempted process goes back to ready queue
4. Always keep running the process with the least remaining time

### Example — Same as SJF Example 2

| Process | Arrival | Burst |
| ------- | ------- | ----- |
| P1      | 0       | 8     |
| P2      | 1       | 4     |
| P3      | 2       | 9     |
| P4      | 3       | 5     |

```
Step-by-step (preemptive):
  t=0:  P1 starts (remaining = 8)
  t=1:  P2 arrives (burst=4) < P1 remaining (7) → PREEMPT P1, start P2
  t=2:  P3 arrives (burst=9) > P2 remaining (3) → P2 continues
  t=3:  P4 arrives (burst=5) > P2 remaining (2) → P2 continues
  t=5:  P2 completes. Ready: P1(7), P3(9), P4(5) → P4 starts (shortest)
  t=10: P4 completes. Ready: P1(7), P3(9) → P1 resumes (shorter)
  t=17: P1 completes. Only P3 left
  t=26: P3 completes

Gantt Chart:
  0 1      5       10       17        26
  │P1│ P2  │─ P4 ──│── P1 ──│─── P3 ──│
```

**Calculating waiting time for SRTF:**

- Waiting Time = Turnaround Time − Burst Time (but account for preemptions)

| Process | Completion | Turnaround (CT−AT) | Waiting (TAT−BT) |
| ------- | ---------- | ------------------ | ---------------- |
| P1      | 17         | 17 − 0 = **17**    | 17 − 8 = **9**   |
| P2      | 5          | 5 − 1 = **4**      | 4 − 4 = **0**    |
| P3      | 26         | 26 − 2 = **24**    | 24 − 9 = **15**  |
| P4      | 10         | 10 − 3 = **7**     | 7 − 5 = **2**    |

**Average Waiting Time = (9 + 0 + 15 + 2) / 4 = 6.5 ms** — better than SJF's 7.75ms!

### Pros and Cons

| Pros                                          | Cons                                       |
| --------------------------------------------- | ------------------------------------------ |
| Best average waiting time                     | Hard to predict remaining burst time       |
| Responsive — reacts to short jobs immediately | High context switching overhead            |
|                                               | Long processes can **starve indefinitely** |
|                                               | Most complex of the three                  |

---

## 6. Comparison: FCFS vs SJF vs SRTF

| Feature              | FCFS           | SJF             | SRTF            |
| -------------------- | -------------- | --------------- | --------------- |
| Preemption           | Non-preemptive | Non-preemptive  | Preemptive      |
| Selection Criteria   | Arrival time   | Burst time      | Remaining time  |
| Average Waiting Time | High           | Low             | Lowest          |
| Complexity           | Simple         | Moderate        | Complex         |
| Context Switches     | Minimal        | Minimal         | Frequent        |
| Starvation Possible? | No             | Yes (long jobs) | Yes (long jobs) |
| Convoy Effect        | Yes            | Reduced         | Minimal         |
| Overhead             | Low            | Low             | High            |

### Results on the same 4-process example

| Algorithm | Avg Waiting Time             |
| --------- | ---------------------------- |
| FCFS      | — (not shown, would be high) |
| SJF       | 7.75 ms                      |
| SRTF      | 6.5 ms                       |

---

## 7. Practical Considerations

### The Burst Time Problem

In real operating systems, **knowing exact burst time in advance is nearly impossible**. Systems solve this with prediction:

**Exponential Averaging Formula:**

$$\text{predicted\_next} = \alpha \times \text{actual\_previous} + (1 - \alpha) \times \text{predicted\_previous}$$

- $\alpha$ (alpha) is a weighting factor between 0 and 1
- Higher $\alpha$ = more weight on recent behavior
- This lets SJF/SRTF work in practice — predictions aren't perfect, but they're usually good enough

```
  Example with α = 0.5:
  Previous burst = 10ms, predicted was 8ms
  New prediction = 0.5 × 10 + 0.5 × 8 = 9ms
```

### Starvation Problem

Both SJF and SRTF can **starve long processes** — if short processes keep arriving, the long process never gets CPU. The fix is **aging** — gradually increase the priority of processes that have been waiting a long time.

---

## 8. When to Use Which Algorithm

| Use This | When...                                                                 |
| -------- | ----------------------------------------------------------------------- |
| **FCFS** | Batch systems, fairness > efficiency, similar burst times, simple needs |
| **SJF**  | Good burst estimates available, want minimum avg wait, batch processing |
| **SRTF** | Interactive systems, response time matters, context switch cost is ok   |

> In practice, most OS use more sophisticated algorithms (Round Robin, Priority, MLFQ) that **combine ideas** from these basic ones. Understanding FCFS/SJF/SRTF is foundational to understanding all of them.

---

## 9. Key Takeaways

- **Scheduling metrics:** Arrival time, Burst time, Completion time, Turnaround time (CT−AT), Waiting time (TAT−BT)
- **FCFS:** Simplest — first in, first out. Non-preemptive. Suffers from **convoy effect**. No starvation but poor average wait time
- **SJF:** Picks shortest burst first. Non-preemptive. Optimal average wait when all arrive at t=0. **Starvation possible** for long jobs
- **SRTF:** Preemptive SJF — preempts current process if a shorter one arrives. **Best average waiting time** but most complex and highest overhead
- **Burst time prediction** is needed in practice — systems use exponential averaging on historical data
- All three are non-practical alone — they lead to **starvation** without additional mechanisms like aging
- These are the building blocks for more advanced algorithms (Round Robin, Priority Scheduling, HRRN, MLQ)
