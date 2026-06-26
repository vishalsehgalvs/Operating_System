# Signal Handling in Operating Systems

> **Signals** are asynchronous software notifications sent to a process to tell it something happened — they carry no data (just a signal number), can arrive at any moment during execution, and the process can either run a custom handler, use the default action, or ignore most of them (except SIGKILL and SIGSTOP, which cannot be caught).

---

## Table of Contents

1. [What is a Signal?](#1-what-is-a-signal)
2. [Common Signal Types](#2-common-signal-types)
3. [How Signals Are Generated](#3-how-signals-are-generated)
4. [Signal Handling: Three Responses](#4-signal-handling-three-responses)
5. [Writing a Signal Handler](#5-writing-a-signal-handler)
6. [Signal Masking and Blocking](#6-signal-masking-and-blocking)
7. [Signals vs Message Passing](#7-signals-vs-message-passing)
8. [Best Practices and Pitfalls](#8-best-practices-and-pitfalls)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is a Signal?

A signal is an **asynchronous software interrupt** delivered to a process by the OS, by hardware, or by another process. It carries only one piece of information: **which signal number** it is. No data payload.

```
  Normal process execution:
  [instruction 1] → [instruction 2] → [instruction 3] → ...

  Signal arrives (e.g., user presses Ctrl+C):
  [instruction 1] → [instruction 2] ← signal interrupts here!
                                    → [run signal handler]
                                    → [return and resume instruction 3]

  The process doesn't need to "check for signals" — the OS injects them.
  This is what "asynchronous" means: can happen at any point.
```

**Analogy:** You're reading a book and your phone rings. You put your finger on the page (the CPU saves your state), pick up the phone (run signal handler), hang up, and go back to where you were. The call came in without warning — you didn't have to be checking your phone.

**Key characteristics:**

- Asynchronous (delivered at any time)
- Carry minimal info (just signal number)
- Lightweight (no data payload)
- Used for notifications and interrupts, NOT data transfer

---

## 2. Common Signal Types

| Signal        | Number | Description                                              | Default Action        |
| ------------- | ------ | -------------------------------------------------------- | --------------------- |
| **SIGINT**    | 2      | Keyboard interrupt (Ctrl+C)                              | Terminate process     |
| **SIGTERM**   | 15     | Polite termination request                               | Terminate process     |
| **SIGKILL**   | 9      | Force kill — **cannot be caught or ignored**             | Terminate immediately |
| **SIGSEGV**   | 11     | Segmentation fault (invalid memory access)               | Terminate + core dump |
| **SIGFPE**    | 8      | Floating point / arithmetic exception (division by zero) | Terminate + core dump |
| **SIGILL**    | 4      | Illegal instruction                                      | Terminate + core dump |
| **SIGPIPE**   | 13     | Write to pipe/socket with no reader                      | Terminate             |
| **SIGALRM**   | 14     | Timer alarm expired (from `alarm()`)                     | Terminate             |
| **SIGCHLD**   | 17     | Child process changed state (exit, stop)                 | Ignore                |
| **SIGSTOP**   | 19     | Pause process — **cannot be caught or ignored**          | Stop (pause)          |
| **SIGCONT**   | 18     | Resume a stopped process                                 | Continue              |
| **SIGTSTP**   | 20     | Keyboard stop (Ctrl+Z) — **can** be caught               | Stop                  |
| **SIGUSR1/2** | 10/12  | User-defined signals (app-specific use)                  | Terminate (default)   |

**Two uncatchable signals:**

```
  SIGKILL (9):  process is killed immediately — no cleanup possible
  SIGSTOP (19): process is paused immediately — cannot resist

  These cannot be caught, blocked, or ignored by any process.
  They are the OS's "override" mechanism.

  Use SIGTERM first → give process a chance to clean up
  Use SIGKILL only if process doesn't respond → no cleanup happens
```

---

## 3. How Signals Are Generated

### From Hardware

```
  Invalid memory access (null pointer dereference):
    CPU detects the violation → CPU raises exception → OS delivers SIGSEGV

  Division by zero:
    CPU detects divide-by-zero instruction → OS delivers SIGFPE

  Illegal CPU instruction:
    CPU can't execute the instruction → OS delivers SIGILL
```

### From Software / OS Kernel

```
  Process writes to a pipe whose read end is closed:
  → OS detects the broken write → delivers SIGPIPE

  Timer set with alarm(5) expires:
  → OS delivers SIGALRM to the process after 5 seconds

  Child process exits:
  → OS delivers SIGCHLD to the parent process
```

### From User / Other Processes

```bash
# Keyboard shortcuts
Ctrl+C   → sends SIGINT  to foreground process
Ctrl+Z   → sends SIGTSTP to foreground process (suspend)
Ctrl+\   → sends SIGQUIT to foreground process (quit + core dump)

# kill command (sends signal to process by PID)
kill 1234            # sends SIGTERM (default) to PID 1234
kill -9 1234         # sends SIGKILL to PID 1234
kill -SIGINT 1234    # sends SIGINT to PID 1234
kill -SIGUSR1 1234   # sends SIGUSR1 (custom signal) to PID 1234
```

---

## 4. Signal Handling: Three Responses

When a signal arrives, a process has three options:

```
  Option 1: DEFAULT ACTION (do nothing special)
  → OS carries out the built-in response (usually terminate)

  Option 2: IGNORE the signal
  → Process says "I don't care about this signal"
  → Signal is silently discarded
  → (Cannot ignore SIGKILL or SIGSTOP)

  Option 3: CUSTOM HANDLER (signal handler function)
  → Process registers its own function
  → When signal arrives, OS calls that function
  → After function returns, process resumes where it was
```

```c
#include <signal.h>

// Register ignore:
signal(SIGPIPE, SIG_IGN);   // ignore broken pipe (common in network servers)

// Register default:
signal(SIGINT, SIG_DFL);    // restore default for SIGINT

// Register custom handler:
void my_handler(int signum) {
    // runs when SIGINT arrives
}
signal(SIGINT, my_handler);
```

---

## 5. Writing a Signal Handler

### Example 1: Graceful Shutdown on Ctrl+C

```c
#include <signal.h>
#include <unistd.h>
#include <stdio.h>

// volatile: tells compiler "this CAN change outside normal program flow"
// (signal handler runs asynchronously, compiler must not cache it)
volatile int shutdown_requested = 0;

void handle_sigint(int signum) {
    // Signal handlers: keep them VERY short
    // Only set a flag; let main loop do the real work
    shutdown_requested = 1;
    write(STDOUT_FILENO, "\nShutdown requested...\n", 23); // write() is async-signal-safe
    // DO NOT use printf() here — it's NOT async-signal-safe!
}

int main() {
    signal(SIGINT, handle_sigint);  // install handler

    while (!shutdown_requested) {
        // do work
        printf("Working...\n");
        sleep(1);
    }

    // graceful cleanup
    printf("Cleaning up and exiting.\n");
    // close files, release locks, save state here
    return 0;
}
```

**Without this handler:** Ctrl+C kills the process immediately — no cleanup.  
**With this handler:** process gets a chance to save state and exit cleanly.

### Example 2: Timeout Using SIGALRM

```c
volatile int timeout_occurred = 0;

void handle_alarm(int signum) {
    timeout_occurred = 1;
}

int main() {
    signal(SIGALRM, handle_alarm);
    alarm(5);               // deliver SIGALRM in 5 seconds

    while (!timeout_occurred) {
        if (do_work_chunk()) {    // returns true when work is done
            alarm(0);            // cancel alarm — work finished in time
            printf("Done in time!\n");
            return 0;
        }
    }

    printf("Operation timed out after 5 seconds.\n");
    return 1;
}
```

### Example 3: SIGUSR1 for Dynamic Config Reload

```c
// Pattern used by web servers (nginx, Apache) and daemons:
// Instead of restarting, send SIGUSR1 to reload config without downtime

void handle_sigusr1(int signum) {
    reload_flag = 1;   // set flag; main loop will reload config safely
}

// Usage:
// kill -SIGUSR1 $(pidof nginx)   → nginx reloads config without stopping
```

---

## 6. Signal Masking and Blocking

Sometimes you need a critical section of code to run without being interrupted by a signal. **Blocking** holds pending signals in a queue until you unblock them.

```
  Signal mask = set of signals currently blocked for this process

  ┌─────────────────────────────────┐
  │ Signal arrives                  │
  │          ↓                      │
  │ Is this signal blocked?         │
  │    Yes → add to pending set     │  (hold, deliver later)
  │    No  → deliver now            │  (run handler or default)
  └─────────────────────────────────┘
```

```c
#include <signal.h>

sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGINT);   // add SIGINT to the mask

// Block SIGINT during critical section
sigprocmask(SIG_BLOCK, &mask, NULL);

// ── CRITICAL SECTION ──────────────────────────────
// Update data structure that must not be interrupted mid-way
// Any SIGINT that arrives here will be HELD in pending queue
// ──────────────────────────────────────────────────

// Unblock SIGINT — any pending SIGINT is delivered now
sigprocmask(SIG_UNBLOCK, &mask, NULL);
```

**Important:** Multiple instances of the same signal while blocked → only ONE instance is delivered when unblocked (standard signals do NOT queue). Real-time signals (`SIGRTMIN` through `SIGRTMAX`) DO queue.

---

## 7. Signals vs Message Passing

| Aspect            | Signals                                     | Message Passing (pipes, queues)         |
| ----------------- | ------------------------------------------- | --------------------------------------- |
| **Data transfer** | No — just notification (signal number only) | Yes — arbitrary data payload            |
| **Delivery**      | Asynchronous only                           | Sync or async                           |
| **Overhead**      | Very low (minimal kernel work)              | Moderate (copy data through kernel)     |
| **Reliability**   | Can be lost/coalesced (standard signals)    | Reliable and ordered                    |
| **Use case**      | "Something happened" notifications          | Actual data exchange, commands          |
| **API**           | `signal()`, `sigaction()`, `kill()`         | `read()`, `write()`, `send()`, `recv()` |
| **Complexity**    | Simple mechanism                            | More complex but richer                 |

```
  Use SIGNALS for:
  - "Your child process exited" (SIGCHLD)
  - "Please terminate gracefully" (SIGTERM)
  - "Timer expired" (SIGALRM)
  - "User pressed Ctrl+C" (SIGINT)
  - "Reload your config file" (SIGUSR1)

  Use MESSAGE PASSING for:
  - Sending actual data between processes
  - Request-response patterns
  - Streaming data
  - Cross-machine communication
```

---

## 8. Best Practices and Pitfalls

### Best Practices

| Practice                                | Reason                                                                    |
| --------------------------------------- | ------------------------------------------------------------------------- |
| Keep signal handlers short              | Handlers interrupt at unpredictable times; complex code = race conditions |
| Use `volatile` for shared flags         | Compiler won't optimize away changes made by async handler                |
| Only call async-signal-safe functions   | `write()`, `_exit()` are safe; `printf()`, `malloc()` are NOT             |
| Use `sigaction()` instead of `signal()` | `sigaction()` is more reliable and portable                               |
| Block signals during critical sections  | Protects data structure updates from being interrupted mid-way            |
| Handle SIGTERM for graceful shutdown    | Lets your process clean up before exiting                                 |
| Ignore SIGPIPE in network servers       | Prevents crash when client disconnects                                    |

### Common Pitfalls

**Pitfall 1: Calling unsafe functions in handler**

```c
// WRONG — printf uses internal buffers with locks
// If main is inside printf when signal arrives → deadlock!
void bad_handler(int sig) {
    printf("Got signal %d\n", sig);  // NOT async-signal-safe!
}

// CORRECT — write() is async-signal-safe
void good_handler(int sig) {
    write(STDOUT_FILENO, "Got signal\n", 11);  // OK
    shutdown_requested = 1;                    // OK (volatile flag)
}
```

**Pitfall 2: Missing volatile keyword**

```c
int flag = 0;         // WRONG — compiler may cache in register
volatile int flag = 0; // CORRECT — always read from memory

void handler(int sig) { flag = 1; }
while (!flag) { ... }  // Without volatile, loop may run forever
```

**Pitfall 3: SIGKILL misconception**

```bash
# People sometimes try:
kill -9 PID   # expecting graceful shutdown
# No cleanup possible! Always try SIGTERM first:
kill PID      # sends SIGTERM → process can clean up
sleep 5
kill -9 PID 2>/dev/null  # escalate to SIGKILL only if still running
```

---

## 8. Code Examples

> Working code that demonstrates signal handling concepts in practice.

### C++ — Simple Version
Simulate a per-process signal table: register custom handlers, ignore signals, deliver them — including uncatchable SIGKILL.

```cpp
#include <iostream>
#include <functional>
#include <unordered_map>
#include <set>
#include <string>

// ── Signal number constants (matching POSIX values) ────────────────────────────
constexpr int SIG_INT  =  2;   // SIGINT  — keyboard interrupt (Ctrl+C)
constexpr int SIG_TERM = 15;   // SIGTERM — polite termination request
constexpr int SIG_KILL =  9;   // SIGKILL — force kill (CANNOT be caught)
constexpr int SIG_USR1 = 10;   // SIGUSR1 — user-defined signal 1
constexpr int SIG_ALRM = 14;   // SIGALRM — timer alarm

const std::unordered_map<int, std::string> SIG_NAMES = {
    {2, "SIGINT"}, {9, "SIGKILL"}, {10, "SIGUSR1"}, {14, "SIGALRM"}, {15, "SIGTERM"}
};

// ── Per-process signal table ──────────────────────────────────────────────────
// In real OS: a table entry per signal stores disposition + handler pointer.
// Dispositions: DEFAULT (terminate), IGNORE, or CUSTOM (run handler function).

using Handler = std::function<void(int)>;

struct SignalEntry {
    enum class Disp { DEFAULT, IGNORE, CUSTOM } disp = Disp::DEFAULT;
    Handler fn;
};

class Process {
    std::string name;
    std::unordered_map<int, SignalEntry> table;
    const std::set<int> uncatchable = { SIG_KILL };  // OS overrides everything
    bool alive = true;

    void default_action(int sig) {
        std::cout << "  [OS default] " << SIG_NAMES.at(sig)
                  << " → process '" << name << "' terminated\n";
        alive = false;
    }

public:
    explicit Process(const std::string& n) : name(n) {}

    // register_handler() simulates sigaction(sig, &sa, NULL)
    void register_handler(int sig, Handler h) {
        if (uncatchable.count(sig)) {
            std::cout << "  [WARN] Cannot catch "
                      << SIG_NAMES.at(sig) << " — OS ignores this\n";
            return;
        }
        table[sig] = { SignalEntry::Disp::CUSTOM, h };
        std::cout << "  [" << name << "] Custom handler registered for "
                  << SIG_NAMES.at(sig) << "\n";
    }

    void ignore_signal(int sig) {
        if (uncatchable.count(sig)) { std::cout << "  [WARN] Cannot ignore SIGKILL\n"; return; }
        table[sig].disp = SignalEntry::Disp::IGNORE;
        std::cout << "  [" << name << "] Will ignore " << SIG_NAMES.at(sig) << "\n";
    }

    // deliver() simulates kill(pid, sig) from another process or the kernel
    void deliver(int sig) {
        auto sname = SIG_NAMES.count(sig) ? SIG_NAMES.at(sig) : std::to_string(sig);
        std::cout << "\n→ Delivering " << sname << " to '" << name << "'\n";

        if (uncatchable.count(sig)) {
            std::cout << "  [OS] " << sname << " cannot be blocked — killed immediately\n";
            alive = false;
            return;
        }

        auto it = table.find(sig);
        if (it == table.end() || it->second.disp == SignalEntry::Disp::DEFAULT) {
            default_action(sig);
        } else if (it->second.disp == SignalEntry::Disp::IGNORE) {
            std::cout << "  [" << name << "] " << sname << " ignored\n";
        } else {
            std::cout << "  [" << name << "] Running handler for " << sname << "\n";
            it->second.fn(sig);
        }
    }

    bool is_alive() const { return alive; }
};

int main() {
    Process proc("myapp");

    // Register handlers (like calling sigaction() at program startup)
    proc.register_handler(SIG_INT,  [](int) { std::cout << "  → SIGINT: saving state, exiting cleanly\n"; });
    proc.register_handler(SIG_TERM, [](int) { std::cout << "  → SIGTERM: releasing resources\n"; });
    proc.register_handler(SIG_USR1, [](int) { std::cout << "  → SIGUSR1: reloading config file\n"; });
    proc.ignore_signal(SIG_ALRM);   // process chose to ignore timer alarms

    std::cout << "\n=== Sending signals ===\n";
    proc.deliver(SIG_USR1);    // custom handler runs
    proc.deliver(SIG_ALRM);    // ignored
    proc.deliver(SIG_INT);     // custom handler runs

    std::cout << "\n=== Trying to catch SIGKILL ===\n";
    proc.register_handler(SIG_KILL, [](int) { std::cout << "  This should never run!\n"; });
    proc.deliver(SIG_KILL);    // OS kills immediately regardless

    std::cout << "\nProcess alive: " << (proc.is_alive() ? "YES" : "NO") << "\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style
Simulate signal masking: block signals during a critical section, queue them as pending, deliver when unblocked.

```cpp
#include <iostream>
#include <unordered_map>
#include <unordered_set>
#include <functional>
#include <string>

// ── Signal dispatcher with mask + pending set ─────────────────────────────────
// Models POSIX per-process signal state:
//   mask    — signals currently blocked  (sigprocmask)
//   pending — signals received while blocked (standard signals: max 1 per signum)
//   table   — registered handler functions

using Handler = std::function<void(int)>;
const std::unordered_map<int, std::string> NAMES =
    {{2,"SIGINT"},{10,"SIGUSR1"},{12,"SIGUSR2"},{14,"SIGALRM"},{15,"SIGTERM"}};

class SignalDispatcher {
    std::unordered_map<int, Handler> table;
    std::unordered_set<int> mask;       // blocked signals
    std::unordered_set<int> pending;    // pending: one slot per signal number

    void dispatch(int sig) {
        auto it = table.find(sig);
        if (it != table.end()) {
            std::cout << "    [handler] " << NAMES.at(sig) << " executing\n";
            it->second(sig);
        } else {
            std::cout << "    [default] " << NAMES.at(sig) << " — terminate\n";
        }
    }

public:
    void set_handler(int sig, Handler h) { table[sig] = h; }

    // sigprocmask(SIG_BLOCK, ...)
    void block(int sig) {
        mask.insert(sig);
        std::cout << "  [Mask] " << NAMES.at(sig) << " BLOCKED\n";
    }

    // sigprocmask(SIG_UNBLOCK, ...) — delivers pending signal if one is waiting
    void unblock(int sig) {
        mask.erase(sig);
        if (pending.count(sig)) {
            pending.erase(sig);
            std::cout << "  [Mask] " << NAMES.at(sig)
                      << " UNBLOCKED → delivering pending signal\n";
            dispatch(sig);
        } else {
            std::cout << "  [Mask] " << NAMES.at(sig) << " UNBLOCKED (no pending)\n";
        }
    }

    // kill(pid, sig) — deliver a signal to this process
    void send(int sig) {
        auto name = NAMES.count(sig) ? NAMES.at(sig) : std::to_string(sig);
        std::cout << "\n→ " << name << " arriving ";
        if (mask.count(sig)) {
            // Standard signals: only ONE pending instance kept (no queueing)
            if (pending.count(sig))
                std::cout << "(blocked — already pending, duplicate dropped)\n";
            else {
                pending.insert(sig);
                std::cout << "(blocked — queued as pending)\n";
            }
        } else {
            std::cout << "(not blocked — dispatching now)\n";
            dispatch(sig);
        }
    }

    // sigpending()
    void show_pending() const {
        std::cout << "  Pending: [";
        for (int s : pending) std::cout << NAMES.at(s) << " ";
        std::cout << "]\n";
    }
};

int main() {
    SignalDispatcher proc;
    proc.set_handler(2,  [](int){ std::cout << "      cleaning up temp files\n"; });
    proc.set_handler(10, [](int){ std::cout << "      reloading config\n"; });
    proc.set_handler(12, [](int){ std::cout << "      dumping stats to log\n"; });

    std::cout << "=== Phase 1: No mask — immediate dispatch ===\n";
    proc.send(10);   // SIGUSR1 runs handler immediately

    std::cout << "\n=== Phase 2: Block SIGINT + SIGUSR1 (critical section) ===\n";
    proc.block(2);
    proc.block(10);
    proc.send(2);    // blocked → queued
    proc.send(10);   // blocked → queued
    proc.send(10);   // duplicate — dropped (standard signals don't queue)
    proc.send(12);   // SIGUSR2 NOT blocked → runs immediately
    proc.show_pending();

    std::cout << "\n=== Phase 3: Unblock — pending signals delivered ===\n";
    proc.unblock(10);  // SIGUSR1 pending → dispatched now
    proc.unblock(2);   // SIGINT  pending → dispatched now

    std::cout << "\n=== Phase 4: All clear ===\n";
    proc.show_pending();

    return 0;
}
```

### Python — Simple Version
Simulate a signal table: define handlers for SIGINT, SIGTERM, SIGUSR1; show SIGKILL cannot be caught.

```python
from __future__ import annotations
from typing import Callable

# ── Signal number constants ────────────────────────────────────────────────────
SIGINT  =  2   # keyboard interrupt (Ctrl+C)
SIGTERM = 15   # polite shutdown request
SIGKILL =  9   # force kill — cannot be caught or ignored
SIGUSR1 = 10   # user-defined signal 1
SIGALRM = 14   # timer alarm

NAMES = {2: "SIGINT", 9: "SIGKILL", 10: "SIGUSR1", 14: "SIGALRM", 15: "SIGTERM"}
Handler = Callable[[int], None]

class Process:
    """Simulates a process's signal disposition table."""

    UNCATCHABLE = {SIGKILL}   # OS overrides — cannot be caught or ignored

    def __init__(self, name: str):
        self.name = name
        # table: {signum → ("default" | "ignore" | "custom", handler_fn | None)}
        self._table: dict[int, tuple[str, Handler | None]] = {}

    def signal(self, signum: int, handler: Handler | None) -> None:
        """Register a handler — like signal() or sigaction() in C."""
        sname = NAMES.get(signum, str(signum))
        if signum in self.UNCATCHABLE:
            print(f"  [WARN] Cannot register handler for {sname} — OS ignores this")
            return
        if handler is None:
            self._table[signum] = ("ignore", None)
            print(f"  [{self.name}] Will IGNORE {sname}")
        else:
            self._table[signum] = ("custom", handler)
            print(f"  [{self.name}] Custom handler registered for {sname}")

    def deliver(self, signum: int) -> None:
        """Deliver a signal — like kill(pid, sig) from another process."""
        sname = NAMES.get(signum, str(signum))
        print(f"\n→ Delivering {sname} to '{self.name}'")

        if signum in self.UNCATCHABLE:
            print(f"  [OS] {sname} cannot be blocked — process killed immediately")
            return

        entry = self._table.get(signum)
        if entry is None or entry[0] == "default":
            print(f"  [OS default] {sname} — process terminated")
        elif entry[0] == "ignore":
            print(f"  [{self.name}] {sname} ignored")
        else:
            print(f"  [{self.name}] Running handler for {sname}")
            entry[1](signum)

proc = Process("myapp")

# Register handlers (what a real program does at startup)
proc.signal(SIGINT,  lambda s: print("  → Ctrl+C caught: saving state and exiting cleanly"))
proc.signal(SIGTERM, lambda s: print("  → SIGTERM: releasing resources, graceful shutdown"))
proc.signal(SIGUSR1, lambda s: print("  → SIGUSR1: reloading configuration file"))
proc.signal(SIGALRM, None)   # ignore timer alarms

print("\n=== Sending signals ===")
proc.deliver(SIGUSR1)   # custom handler runs
proc.deliver(SIGALRM)   # ignored
proc.deliver(SIGTERM)   # custom handler runs

print("\n=== Trying to catch SIGKILL ===")
proc.signal(SIGKILL, lambda s: print("  This should NEVER run"))
proc.deliver(SIGKILL)   # OS overrides — kills immediately
```

### Python — Medium Level
Simulate a signal mask: block signals during a critical section, pending set, deliver on unmask.

```python
from __future__ import annotations
from typing import Callable

Handler = Callable[[int], None]
NAMES = {2: "SIGINT", 10: "SIGUSR1", 12: "SIGUSR2", 14: "SIGALRM", 15: "SIGTERM"}

class SignalDispatcher:
    """
    Models POSIX per-process signal state:
      _table   — registered handlers
      _mask    — currently blocked signal numbers  (sigprocmask)
      _pending — signals received while blocked    (max 1 per signum, no queuing)
    """

    def __init__(self):
        self._table:   dict[int, Handler] = {}
        self._mask:    set[int] = set()    # blocked
        self._pending: set[int] = set()    # waiting for delivery

    def set_handler(self, signum: int, handler: Handler) -> None:
        self._table[signum] = handler

    def block(self, signum: int) -> None:
        """sigprocmask(SIG_BLOCK, ...) — add signum to the mask."""
        self._mask.add(signum)
        print(f"  [Mask] {NAMES.get(signum, signum)} BLOCKED")

    def unblock(self, signum: int) -> None:
        """sigprocmask(SIG_UNBLOCK, ...) — delivers pending signal if any."""
        self._mask.discard(signum)
        sname = NAMES.get(signum, signum)
        if signum in self._pending:
            self._pending.discard(signum)
            print(f"  [Mask] {sname} UNBLOCKED → delivering pending signal")
            self._dispatch(signum)
        else:
            print(f"  [Mask] {sname} UNBLOCKED (no pending)")

    def send(self, signum: int) -> None:
        """Deliver a signal — like kill(pid, sig) from another process."""
        sname = NAMES.get(signum, signum)
        print(f"\n→ {sname} arriving ", end="")
        if signum in self._mask:
            if signum in self._pending:
                print("(blocked — already pending, duplicate dropped)")
            else:
                self._pending.add(signum)
                print("(blocked — queued as pending)")
        else:
            print("(not blocked — dispatching now)")
            self._dispatch(signum)

    def _dispatch(self, signum: int) -> None:
        handler = self._table.get(signum)
        if handler:
            handler(signum)
        else:
            print(f"  [default] {NAMES.get(signum, signum)} — terminate")

    def show_pending(self) -> None:
        names = [NAMES.get(s, str(s)) for s in self._pending]
        print(f"  Pending: {names if names else '(none)'}")

# ── Demo ───────────────────────────────────────────────────────────────────────
proc = SignalDispatcher()
proc.set_handler(2,  lambda s: print("    SIGINT:  cleaning up, exiting"))
proc.set_handler(10, lambda s: print("    SIGUSR1: reloading config"))
proc.set_handler(12, lambda s: print("    SIGUSR2: dumping stats"))

print("=== Phase 1: No mask — immediate dispatch ===")
proc.send(10)   # SIGUSR1 runs immediately

print("\n=== Phase 2: Block SIGINT + SIGUSR1 (critical section) ===")
proc.block(2)    # SIGINT
proc.block(10)   # SIGUSR1
proc.send(2)     # blocked → queued
proc.send(10)    # blocked → queued
proc.send(10)    # duplicate → dropped (standard signals don't queue)
proc.send(12)    # SIGUSR2 NOT blocked → runs immediately
proc.show_pending()

print("\n=== Phase 3: Unblock — pending signals delivered ===")
proc.unblock(10)  # SIGUSR1 pending → delivered now
proc.unblock(2)   # SIGINT  pending → delivered now

print("\n=== Phase 4: All clear ===")
proc.show_pending()
```

---

## 9. Key Takeaways

- **Signal** = asynchronous notification to a process — no data, just a signal number; arrives at any time
- **Three responses:** default action (usually terminate), ignore, or custom handler function
- **SIGKILL (9) and SIGSTOP (19)** cannot be caught, blocked, or ignored — the OS overrides everything
- **SIGTERM (15)** is the polite way to ask a process to exit; use SIGKILL only as a last resort
- **Signal handlers must be short and simple** — set a flag, let the main loop do real work
- **Only use async-signal-safe functions** inside handlers (`write()` yes, `printf()` / `malloc()` no)
- **`volatile`** is required for variables shared between a signal handler and the main program
- **Signal masking/blocking** protects critical sections from being interrupted mid-way
- **Pending signals** are held while blocked; only ONE instance of the same signal is kept (standard signals don't queue)
- **Signals vs message passing:** signals are for notifications ("something happened"), message passing is for actual data transfer
- **Common uses:** graceful shutdown (SIGTERM), timeout implementation (SIGALRM), child process tracking (SIGCHLD), config reload (SIGUSR1), broken pipe prevention (ignore SIGPIPE)
- **`sigaction()` is preferred** over `signal()` for writing reliable, portable signal-handling code
