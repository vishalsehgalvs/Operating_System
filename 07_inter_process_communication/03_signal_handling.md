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
