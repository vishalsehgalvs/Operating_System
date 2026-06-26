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

## 7. Code Examples

> Working code that demonstrates PCB structure, creation, and context save/restore in practice.

### C++ — Simple Version

Define a PCB struct with all OS fields — create, display, and compare two PCBs.

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

// PCB — the OS's complete record for one process
struct PCB {
    // --- Process identity ---
    int    pid;         // unique process ID
    string name;        // human-readable name
    string state;       // NEW / READY / RUNNING / WAITING / TERMINATED

    // --- CPU state (saved during context switch) ---
    int    programCounter; // next instruction to execute
    int    regAX;          // simulated CPU register A
    int    regBX;          // simulated CPU register B

    // --- Scheduling info ---
    int    priority;       // lower number = higher priority
    int    cpuTimeUsed;    // total CPU ms consumed so far

    // --- Memory info ---
    int    baseAddress;    // start of process in memory
    int    sizeBytes;      // how much memory it owns

    // --- I/O ---
    string openFiles;      // comma-separated list of open files
};

// Create a new PCB with default CPU state
PCB createPCB(int pid, const string& name, int priority, int base, int size) {
    PCB pcb;
    pcb.pid           = pid;
    pcb.name          = name;
    pcb.state         = "NEW";
    pcb.programCounter = 0;    // starts at first instruction
    pcb.regAX         = 0;
    pcb.regBX         = 0;
    pcb.priority      = priority;
    pcb.cpuTimeUsed   = 0;
    pcb.baseAddress   = base;
    pcb.sizeBytes     = size;
    pcb.openFiles     = "none";
    return pcb;
}

void printPCB(const PCB& p) {
    cout << "=== PCB [PID " << p.pid << ": " << p.name << "] ===\n";
    cout << "  State    : " << p.state         << "\n";
    cout << "  PC       : " << p.programCounter << "\n";
    cout << "  Registers: AX=" << p.regAX << ", BX=" << p.regBX << "\n";
    cout << "  Priority : " << p.priority       << "\n";
    cout << "  CPU used : " << p.cpuTimeUsed    << " ms\n";
    cout << "  Memory   : base=" << p.baseAddress
         << ", size=" << p.sizeBytes << " bytes\n";
    cout << "  Files    : " << p.openFiles      << "\n\n";
}

int main() {
    // Create two processes
    PCB p1 = createPCB(101, "TextEditor", 5, 4096,  2048);
    PCB p2 = createPCB(102, "Browser",    3, 8192, 16384);

    // Simulate running p1 for a while
    p1.state          = "RUNNING";
    p1.programCounter = 42;
    p1.regAX          = 17;
    p1.cpuTimeUsed    = 10;
    p1.openFiles      = "notes.txt, config.cfg";

    p2.state = "READY";

    cout << "Process Table:\n\n";
    printPCB(p1);
    printPCB(p2);

    return 0;
}
// Compile: g++ -std=c++17 pcb.cpp -o pcb
```

### C++ — Medium / LeetCode Style

PCB manager with save and restore to simulate context switching between processes.

```cpp
#include <iostream>
#include <unordered_map>
#include <string>
using namespace std;

// CPU register snapshot — saved into PCB on context switch
struct CPUContext {
    int pc;    // program counter
    int ax;    // register A
    int bx;    // register B
    int sp;    // stack pointer
};

struct PCB {
    int        pid;
    string     name;
    string     state;
    CPUContext ctx;       // saved CPU state
    int        priority;
    int        cpuUsed;   // total ms on CPU
};

// Simulates the OS Process Table — maps PID -> PCB
// Space: O(n) where n = number of active processes
class ProcessTable {
    unordered_map<int, PCB> table;
public:
    void add(int pid, const string& name, int priority) {
        table[pid] = {pid, name, "NEW", {0,0,0,0}, priority, 0};
    }

    // Save current CPU registers into the process's PCB
    // Called when the running process is preempted
    void saveContext(int pid, int pc, int ax, int bx, int sp) {
        if (table.count(pid)) {
            table[pid].ctx  = {pc, ax, bx, sp};
            table[pid].state = "READY";  // no longer running
            cout << "[CTX SAVE]    PID " << pid << " -> PC=" << pc
                 << ", AX=" << ax << ", BX=" << bx << "\n";
        }
    }

    // Restore CPU registers from the process's PCB
    // Called when the process is dispatched next
    CPUContext restoreContext(int pid) {
        if (table.count(pid)) {
            table[pid].state = "RUNNING";
            CPUContext& ctx = table[pid].ctx;
            cout << "[CTX RESTORE] PID " << pid << " -> PC=" << ctx.pc
                 << ", AX=" << ctx.ax << ", BX=" << ctx.bx << "\n";
            return ctx;
        }
        return {};
    }

    void printAll() {
        cout << "\n--- Process Table ---\n";
        for (auto& [pid, p] : table)
            cout << "PID " << p.pid << " (" << p.name << ") state="
                 << p.state << " PC=" << p.ctx.pc << "\n";
    }
};

int main() {
    ProcessTable pt;
    pt.add(101, "TextEditor", 5);
    pt.add(102, "Browser",    3);

    cout << "--- Initial State ---\n";
    pt.printAll();

    cout << "\n--- PID 101 runs, then gets preempted ---\n";
    pt.restoreContext(101);           // dispatch PID 101
    pt.saveContext(101, 42, 17, 0, 8000);  // preempt: save its state

    cout << "\n--- PID 102 gets the CPU ---\n";
    pt.restoreContext(102);           // dispatch PID 102

    pt.printAll();
    return 0;
}
// Compile: g++ -std=c++17 pcb_manager.cpp -o pcb_manager
```

### Python — Simple Version

PCB as a dataclass — create two processes, simulate saving/restoring CPU state.

```python
# PCB — Process Control Block simulation

from dataclasses import dataclass, field

@dataclass
class PCB:
    # Identity
    pid:   int
    name:  str
    state: str = "NEW"

    # CPU state — saved here during a context switch
    program_counter: int = 0
    reg_ax:          int = 0
    reg_bx:          int = 0

    # Scheduling & memory
    priority:      int = 5
    cpu_time_used: int = 0
    base_address:  int = 0
    size_bytes:    int = 0

    # I/O
    open_files: list = field(default_factory=list)

    def display(self):
        print(f"=== PCB [PID {self.pid}: {self.name}] ===")
        print(f"  State      : {self.state}")
        print(f"  PC         : {self.program_counter}")
        print(f"  Registers  : AX={self.reg_ax}, BX={self.reg_bx}")
        print(f"  Priority   : {self.priority}")
        print(f"  CPU used   : {self.cpu_time_used} ms")
        print(f"  Memory     : base={self.base_address}, size={self.size_bytes}")
        print(f"  Open files : {self.open_files}\n")


# Create two processes
p1 = PCB(pid=101, name="TextEditor", priority=5,
         base_address=4096, size_bytes=2048)
p2 = PCB(pid=102, name="Browser",    priority=3,
         base_address=8192, size_bytes=16384)

# Simulate p1 running for a while
p1.state           = "RUNNING"
p1.program_counter = 42
p1.reg_ax          = 17
p1.cpu_time_used   = 10
p1.open_files      = ["notes.txt", "config.cfg"]
p2.state           = "READY"

print("Process Table:\n")
p1.display()
p2.display()
```

### Python — Medium Level

ProcessTable class with save/restore context — simulates what the OS does on every context switch.

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class CPUContext:
    """Snapshot of CPU registers saved into a PCB."""
    pc: int = 0   # program counter
    ax: int = 0   # register A
    bx: int = 0   # register B
    sp: int = 0   # stack pointer

@dataclass
class PCB:
    pid:      int
    name:     str
    state:    str = "NEW"
    ctx:      CPUContext = field(default_factory=CPUContext)
    priority: int = 5
    cpu_used: int = 0


class ProcessTable:
    """OS-level table that maps PID -> PCB.
    Space: O(n) where n = number of active processes
    """
    def __init__(self):
        self._table: dict[int, PCB] = {}

    def add(self, pid: int, name: str, priority: int = 5) -> None:
        self._table[pid] = PCB(pid=pid, name=name, priority=priority)

    def save_context(self, pid: int, pc: int, ax: int, bx: int, sp: int) -> None:
        """Save CPU state into PCB when process is preempted."""
        p = self._table[pid]
        p.ctx   = CPUContext(pc, ax, bx, sp)
        p.state = "READY"
        print(f"[SAVE]    PID {pid:>3} | PC={pc}, AX={ax}, BX={bx}")

    def restore_context(self, pid: int) -> CPUContext:
        """Load CPU state from PCB when process is dispatched."""
        p = self._table[pid]
        p.state = "RUNNING"
        print(f"[RESTORE] PID {pid:>3} | PC={p.ctx.pc}, AX={p.ctx.ax}, BX={p.ctx.bx}")
        return p.ctx

    def print_table(self) -> None:
        print("\n--- Process Table ---")
        for p in self._table.values():
            print(f"  PID {p.pid} ({p.name:12}) state={p.state:10} PC={p.ctx.pc}")


if __name__ == "__main__":
    pt = ProcessTable()
    pt.add(101, "TextEditor", priority=5)
    pt.add(102, "Browser",    priority=3)

    pt.print_table()

    print("\n--- Dispatch PID 101 ---")
    pt.restore_context(101)

    print("\n--- Preempt PID 101, dispatch PID 102 ---")
    pt.save_context(101, pc=42, ax=17, bx=0, sp=8000)
    pt.restore_context(102)

    pt.print_table()
```

---

## 8. Key Takeaways

- A **PCB** is the OS's complete data record for a process — created on start, deleted on termination.
- It holds: PID, process state, program counter, CPU registers, scheduling info, memory limits, accounting data, and I/O status.
- Without PCBs, the OS cannot pause and resume processes — **context switching would be impossible**.
- PCBs are stored in **protected kernel memory** — user programs cannot access them directly.
- The OS keeps all PCBs in a **Process Table** for fast lookup.
- The time spent reading/writing PCBs during a context switch is pure overhead — PCBs are kept compact to minimize this.
- Analogy: PCB = the process's **identity card + bookmark + resource ledger**, all in one.
