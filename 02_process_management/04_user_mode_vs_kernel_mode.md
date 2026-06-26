# User Mode vs Kernel Mode

> Two privilege levels built into the CPU that control what programs can do — User Mode restricts apps to safe operations, Kernel Mode gives the OS full control over hardware and memory.

---

## Table of Contents

1. [What Are User Mode and Kernel Mode?](#1-what-are-user-mode-and-kernel-mode)
2. [User Mode](#2-user-mode)
3. [Kernel Mode](#3-kernel-mode)
4. [Why Do We Need Two Modes?](#4-why-do-we-need-two-modes)
5. [How Mode Switching Works](#5-how-mode-switching-works)
6. [Practical Example: Reading a File](#6-practical-example-reading-a-file)
7. [Triggers for Mode Switching](#7-triggers-for-mode-switching)
8. [User Mode vs Kernel Mode Comparison](#8-user-mode-vs-kernel-mode-comparison)
9. [Real-World Analogy](#9-real-world-analogy)
10. [Performance Considerations](#10-performance-considerations)
11. [Common Misconceptions](#11-common-misconceptions)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What Are User Mode and Kernel Mode?

User Mode and Kernel Mode are **two distinct privilege levels** that the CPU operates in. They control what instructions a process can execute and what system resources it can access.

The OS uses these modes to protect itself and hardware from potentially harmful operations by user applications.

```
  ┌─────────────────────────────────────────────────────┐
  │                   USER SPACE                        │
  │   Browser | Text Editor | Game | Your App           │
  │          (restricted — cannot touch hardware)       │
  ├─────────────────────────────────────────────────────┤
  │              [ SYSTEM CALL BOUNDARY ]               │
  ├─────────────────────────────────────────────────────┤
  │                  KERNEL SPACE                       │
  │   OS Kernel | Device Drivers | System Services      │
  │        (privileged — full hardware access)          │
  └─────────────────────────────────────────────────────┘
```

**Hotel analogy:** Guests (user programs) can use their rooms and common areas, but only staff (kernel) can access boiler rooms, electrical panels, and security systems.

---

## 2. User Mode

User Mode is the **restricted execution environment** where most applications run.

**Key characteristics:**

- Programs have **limited access** to system resources
- Cannot directly interact with hardware
- Cannot read/write memory of other processes
- If a program **crashes**, only that program is affected — OS and other apps keep running

**Who runs here:** Your browser, text editor, games, terminal apps — anything you launch as a user.

```
  USER MODE
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │   Browser    │  │  Text Editor │  │    Game      │
  │  (Process A) │  │  (Process B) │  │  (Process C) │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                 │                 │
         │    cannot directly touch hardware │
         └─────────────────┴─────────────────┘
                           │
                    System Calls ↓
```

---

## 3. Kernel Mode

Kernel Mode is the **privileged execution environment** where the OS's core components run.

**Key characteristics:**

- **Unrestricted access** to all hardware and memory
- Can execute any CPU instruction
- Can access any memory address
- A **bug here can crash the entire system**

**Who runs here:** OS kernel, device drivers, critical system services.

**Design principle:** Operating systems minimize the amount of code running at this level to reduce risk.

```
  KERNEL MODE
  ┌──────────────────────────────────────────────────┐
  │                 OS Kernel                        │
  │  ┌────────────┐  ┌───────────┐  ┌─────────────┐ │
  │  │  Scheduler │  │  Memory   │  │   Device    │ │
  │  │            │  │  Manager  │  │   Drivers   │ │
  │  └────────────┘  └───────────┘  └─────────────┘ │
  │         Full access to RAM, CPU, all hardware    │
  └──────────────────────────────────────────────────┘
```

---

## 4. Why Do We Need Two Modes?

### System Protection

Without restrictions, a buggy app could modify critical system files or corrupt kernel data structures, crashing everything. The two-mode separation creates a **safety barrier** isolating user programs from critical system components.

> If every program could directly control the hard disk, a buggy app might overwrite the OS itself — making your computer unbootable.

### Hardware Protection

If multiple programs simultaneously tried to use the printer or write to the same disk sector, chaos would ensue. The kernel **mediates all hardware access**, ensuring orderly operations. User programs must request hardware services through system calls.

### Memory Isolation

User Mode prevents one process from reading or modifying another process's memory.

| Without isolation                        | With isolation (two modes)                      |
| ---------------------------------------- | ----------------------------------------------- |
| Process A can read Process B's passwords | Each process has its own protected memory space |
| Buggy app corrupts other app's data      | Only the crashing app is affected               |
| Malware can steal data from any process  | Malware is confined to its own memory           |

---

## 5. How Mode Switching Works

Programs frequently switch between User Mode and Kernel Mode to request system services.

### User → Kernel Transition

When a user program needs a privileged operation (read a file, allocate memory), it makes a **system call**. This triggers an automatic mode switch.

```
  User Program                      Kernel
       │                               │
       │  1. Makes system call         │
       │──────────────────────────────►│
       │                               │  2. CPU switches to Kernel Mode
       │                               │  3. Kernel executes operation
       │  4. Returns result            │
       │◄──────────────────────────────│
       │                               │  5. CPU switches back to User Mode
       │  6. Program resumes           │
```

### Kernel → User Transition

After the kernel completes the operation, it returns control to the user program. The kernel verifies completion (or returns an error code) before switching back.

**This switching happens thousands of times per second but is imperceptible to users.**

---

## 6. Practical Example: Reading a File

When a text editor opens `document.txt`, here is exactly what happens:

```
// User program running in User Mode
file_handle = open_file("document.txt")  // System call triggered
// ── CPU switches to Kernel Mode ──

// Kernel Mode operations:
// 1. Verify file exists
// 2. Check user has permission to read it
// 3. Locate file on disk
// 4. Load data from disk into memory buffer
// 5. Return file handle to user program

// ── CPU switches back to User Mode ──
data = read(file_handle, 1024)           // Another system call
// ── CPU switches to Kernel Mode again ──

// Kernel Mode operations:
// 1. Read 1024 bytes from file
// 2. Copy data to user program's memory space
// 3. Return number of bytes read

// ── CPU switches back to User Mode ──
// User program now has the data and continues execution
```

**Each system call (`open_file`, `read`) causes a mode switch.** The user program cannot directly access the disk — it must ask the kernel.

This ensures:

- File permissions are checked
- Disk operations are synchronized
- File system stays consistent across multiple programs

---

## 7. Triggers for Mode Switching

Three events cause the CPU to switch between modes:

### System Calls

- Primary way user programs request kernel services
- Examples: file operations, process creation, memory allocation, network communication
- CPU automatically switches to Kernel Mode → kernel executes → switches back

### Hardware Interrupts

- Keyboards, mice, network cards generate interrupts when they need attention
- Forces the CPU into Kernel Mode regardless of what was running
- Kernel handles the interrupt, then returns to whatever was running before

### Exceptions and Errors

- Illegal operation (divide by zero, access invalid memory) → CPU generates exception → switches to Kernel Mode
- Kernel decides: terminate the program, send a signal, or attempt recovery
- Prevents errors from affecting other processes or the system

```
  Triggers for Mode Switch:

  System Call        Hardware Interrupt      Exception/Error
       │                    │                     │
       ▼                    ▼                     ▼
  ┌────────────────────────────────────────────────────┐
  │              CPU switches to KERNEL MODE           │
  │        Kernel handles the request/event            │
  │        CPU switches back to USER MODE              │
  └────────────────────────────────────────────────────┘
```

---

## 8. User Mode vs Kernel Mode Comparison

| Aspect           | User Mode                        | Kernel Mode                       |
| ---------------- | -------------------------------- | --------------------------------- |
| Privilege Level  | Restricted                       | Unrestricted                      |
| Memory Access    | Limited to own address space     | Can access all memory             |
| Hardware Access  | Not allowed directly             | Full access to all hardware       |
| CPU Instructions | Subset of instructions only      | All CPU instructions available    |
| Who Runs Here    | User applications, programs      | OS kernel, device drivers         |
| Crash Impact     | Only that program crashes        | Entire system may crash           |
| Entry Method     | Normal program execution         | System call, interrupt, exception |
| Performance      | Faster (no mode switch overhead) | Slower due to mode switching      |

---

## 9. Real-World Analogy

**Bank vault analogy:**

```
  BANK
  ┌───────────────────────────────────────────────────┐
  │  LOBBY (User Mode)                                │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
  │  │Customer A│  │Customer B│  │Customer C│        │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
  │       │  "I need    │              │              │
  │       │  my box"    │              │              │
  ├───────┼─────────────┼──────────────┼──────────────┤
  │  VAULT AREA (Kernel Mode)                         │
  │       ▼                                           │
  │  ┌──────────┐  Verifies ID → Opens vault →        │
  │  │ Employee │  Retrieves box → Brings to counter  │
  │  └──────────┘                                     │
  └───────────────────────────────────────────────────┘
```

- **Customers (User Mode):** Can use the lobby, fill out forms, make requests — cannot enter the vault
- **Employees (Kernel Mode):** Have keys to everything, perform privileged operations on behalf of customers
- **Controlled interaction:** Customers access their resources through supervised requests — never directly

---

## 10. Performance Considerations

### Mode Switching Overhead

Each switch between User Mode and Kernel Mode requires:

- Saving CPU registers
- Updating memory mappings
- Flushing certain caches (like the TLB)

A mode switch takes **hundreds to thousands of CPU cycles** compared to just a few for a regular function call.

### How to Minimize Overhead

| Technique                | How it helps                                                         |
| ------------------------ | -------------------------------------------------------------------- |
| Buffer reads/writes      | Read large chunks instead of one byte at a time — fewer system calls |
| Batch operations         | Combine multiple small operations into one system call               |
| Cache data in user space | Avoid repeatedly asking the kernel for the same info                 |

**Well-designed applications minimize system calls by caching data and batching operations.**

---

## 11. Common Misconceptions

### Misconception: Kernel Mode is Faster

Privilege level doesn't affect execution speed directly. What matters is the **overhead of switching between modes**. Code running entirely in User Mode (without switching) can be more efficient. Kernel Mode is necessary for privileged operations, not for performance.

### Misconception: User Programs Never Run in Kernel Mode

When a user program makes a system call, **the kernel runs code on behalf of that program** — but in Kernel Mode. The user program's own code never executes with kernel privileges. Only kernel code runs in Kernel Mode, even when servicing user requests.

---

## 11. Code Examples

> Working code that demonstrates user mode vs kernel mode switching in practice.

### C++ — Simple Version

Simulate a privilege flag, a user-mode app making a syscall, and the kernel handler taking over.

```cpp
#include <iostream>
#include <string>
using namespace std;

// Simulate the CPU privilege level
enum Mode { USER_MODE, KERNEL_MODE };

// Global simulated CPU mode flag
Mode currentMode = USER_MODE;

string modeName(Mode m) {
    return (m == USER_MODE) ? "USER_MODE" : "KERNEL_MODE";
}

// Simulate the kernel handling a syscall
// In a real CPU this code runs with ring-0 privileges
void kernelHandler(const string& syscallName) {
    currentMode = KERNEL_MODE;  // --- mode switch: USER -> KERNEL ---
    cout << "  [KERNEL] Mode = " << modeName(currentMode) << "\n";
    cout << "  [KERNEL] Handling syscall: '" << syscallName << "'\n";

    // Perform privileged work (read file, allocate memory, etc.)
    if (syscallName == "read")  cout << "  [KERNEL] Reading 512 bytes from disk...\n";
    if (syscallName == "write") cout << "  [KERNEL] Writing to stdout...\n";
    if (syscallName == "exit")  cout << "  [KERNEL] Freeing resources...\n";

    currentMode = USER_MODE;    // --- mode switch: KERNEL -> USER ---
    cout << "  [KERNEL] Done. Returning to user. Mode = "
         << modeName(currentMode) << "\n";
}

// Simulate a user-mode application making a system call
void userApp() {
    cout << "[USER] Running. Mode = " << modeName(currentMode) << "\n";
    cout << "[USER] Needs to read a file -> issuing syscall 'read'\n";

    // User-mode app CANNOT do the I/O itself.
    // It calls a library function (e.g. read()) which issues an INT 0x80 / SYSCALL.
    kernelHandler("read");    // mode switch happens here

    cout << "[USER] Back in user mode. Mode = " << modeName(currentMode) << "\n";
    cout << "[USER] Needs to print -> issuing syscall 'write'\n";
    kernelHandler("write");

    cout << "[USER] Finishing -> syscall 'exit'\n";
    kernelHandler("exit");
    cout << "[USER] Process exited.\n";
}

int main() {
    cout << "=== User Mode / Kernel Mode Demo ===\n\n";
    userApp();
    return 0;
}
// Compile: g++ -std=c++17 mode_switch.cpp -o mode_switch
```

### C++ — Medium / LeetCode Style

Syscall dispatcher with multiple syscall types, privilege validation, and mode-switch logging.

```cpp
#include <iostream>
#include <unordered_map>
#include <functional>
#include <string>
#include <stdexcept>
using namespace std;

// Privilege levels (simulates CPU rings)
enum Mode { USER = 0, KERNEL = 1 };

Mode cpuMode = USER;  // CPU starts in user mode

// Syscall table: maps syscall number -> (name, handler)
// In Linux: read=0, write=1, open=2, close=3, exit=60
struct Syscall {
    string name;
    function<void()> handler;
};

unordered_map<int, Syscall> syscallTable = {
    {0, {"read",  []{ cout << "    [kernel:read]  Reading from file descriptor...\n"; }}},
    {1, {"write", []{ cout << "    [kernel:write] Writing to stdout...\n"; }}},
    {2, {"open",  []{ cout << "    [kernel:open]  Opening file, returning fd...\n"; }}},
    {60,{"exit",  []{ cout << "    [kernel:exit]  Cleaning up process resources...\n"; }}},
};

// Called when user code executes SYSCALL / INT 0x80 instruction
// Simulates the entire mode switch + kernel handling + return
// Time: O(1) per syscall dispatch
void issueSyscall(int syscallNum) {
    if (cpuMode != USER)
        throw logic_error("Syscall issued from kernel mode — not expected");

    if (!syscallTable.count(syscallNum))
        throw invalid_argument("Unknown syscall: " + to_string(syscallNum));

    const auto& sc = syscallTable[syscallNum];

    cout << "  -> Syscall " << syscallNum << " ('" << sc.name << "') issued\n";

    // --- TRAP: USER -> KERNEL ---
    cpuMode = KERNEL;
    cout << "  -> Mode switch: USER -> KERNEL\n";

    sc.handler();  // privileged kernel work done here

    // --- IRET: KERNEL -> USER ---
    cpuMode = USER;
    cout << "  -> Mode switch: KERNEL -> USER (returned from syscall)\n";
}

int main() {
    cout << "=== Syscall Dispatcher Demo ===\n";
    cout << "CPU mode: USER\n\n";

    cout << "[user app] Opening a file:\n";
    issueSyscall(2);  // open

    cout << "\n[user app] Reading the file:\n";
    issueSyscall(0);  // read

    cout << "\n[user app] Printing result:\n";
    issueSyscall(1);  // write

    cout << "\n[user app] Exiting:\n";
    issueSyscall(60); // exit

    return 0;
}
// Compile: g++ -std=c++17 syscall_dispatcher.cpp -o syscall_dispatcher
```

### Python — Simple Version

Mode flag simulation — user code requests a syscall, kernel handler runs with elevated privileges.

```python
# User mode / kernel mode simulation

cpu_mode = "USER"   # global CPU mode flag


def kernel_handler(syscall_name: str) -> None:
    """Runs in KERNEL mode — user code cannot call privileged ops directly."""
    global cpu_mode

    # --- Mode switch: USER -> KERNEL ---
    cpu_mode = "KERNEL"
    print(f"  [KERNEL] Mode = {cpu_mode}")
    print(f"  [KERNEL] Handling: '{syscall_name}'")

    # Privileged work
    if syscall_name == "read":  print("  [KERNEL] Reading 512 bytes from disk...")
    if syscall_name == "write": print("  [KERNEL] Writing to stdout...")
    if syscall_name == "exit":  print("  [KERNEL] Freeing resources...")

    # --- Mode switch: KERNEL -> USER ---
    cpu_mode = "USER"
    print(f"  [KERNEL] Returning to user. Mode = {cpu_mode}")


def user_app() -> None:
    """Runs in USER mode — cannot touch hardware directly."""
    print(f"[USER] Running. Mode = {cpu_mode}")

    print("[USER] Reading a file -> syscall 'read'")
    kernel_handler("read")

    print(f"\n[USER] Back in user mode = {cpu_mode}")
    print("[USER] Writing output -> syscall 'write'")
    kernel_handler("write")

    print("\n[USER] Exiting -> syscall 'exit'")
    kernel_handler("exit")


user_app()
```

### Python — Medium Level

Decorator-based syscall table — any function marked `@kernel_only` runs in kernel mode automatically.

```python
import functools

# Global CPU privilege state
_cpu_mode = "USER"


def kernel_only(fn):
    """Decorator: wraps a function so it always runs in KERNEL mode.
    Triggers mode switch before, reverts after — just like a real syscall.
    """
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        global _cpu_mode
        if _cpu_mode != "USER":
            raise PermissionError("Nested kernel call not allowed")

        print(f"  [trap]  {fn.__name__}: USER -> KERNEL")
        _cpu_mode = "KERNEL"
        try:
            result = fn(*args, **kwargs)          # privileged work
        finally:
            _cpu_mode = "USER"                    # always restore
            print(f"  [iret]  {fn.__name__}: KERNEL -> USER")
        return result
    return wrapper


# --- Syscall implementations (run in kernel mode) ---

@kernel_only
def sys_read(fd: int, n_bytes: int) -> bytes:
    print(f"  [kernel:read]  fd={fd}, reading {n_bytes} bytes from disk")
    return b"hello world"  # simulated data


@kernel_only
def sys_write(fd: int, data: bytes) -> int:
    print(f"  [kernel:write] fd={fd}, writing {len(data)} bytes")
    return len(data)


@kernel_only
def sys_exit(code: int) -> None:
    print(f"  [kernel:exit]  exit code={code}, releasing resources")


# --- User application ---

def user_app() -> None:
    print(f"[user] Running. mode={_cpu_mode}\n")

    print("[user] Opening and reading file:")
    data = sys_read(3, 512)
    print(f"[user] Got {len(data)} bytes. mode={_cpu_mode}\n")

    print("[user] Writing to stdout:")
    sys_write(1, data)
    print(f"[user] Done. mode={_cpu_mode}\n")

    print("[user] Exiting:")
    sys_exit(0)


user_app()
```

---

## 12. Key Takeaways

- **Two privilege levels:** User Mode (restricted) and Kernel Mode (unrestricted) — enforced by the CPU at hardware level
- **User Mode** is where your apps live — safe, isolated, crash of one app doesn't affect others
- **Kernel Mode** is where the OS lives — full hardware access, a bug here crashes everything
- **Mode switching** happens via system calls, hardware interrupts, and exceptions
- **Each switch has overhead** — efficient programs minimize system calls through buffering and batching
- **Memory isolation** in User Mode is what prevents one app from stealing data from another
- Understanding these two modes is foundational for topics like system calls, process management, and OS security
