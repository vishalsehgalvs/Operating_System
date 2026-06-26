# System Calls: Types, Uses, and Examples

> A **system call** is a controlled request from a user program to the OS kernel for a privileged operation — the program can't access hardware directly (user mode restriction), so it asks the kernel to do it safely; the CPU switches to kernel mode, performs the operation, and returns the result.

---

## Table of Contents

1. [What is a System Call?](#1-what-is-a-system-call)
2. [How System Calls Work: Step by Step](#2-how-system-calls-work-step-by-step)
3. [Category 1: Process Control](#3-category-1-process-control)
4. [Category 2: File Management](#4-category-2-file-management)
5. [Category 3: Device Management](#5-category-3-device-management)
6. [Category 4: Information Maintenance](#6-category-4-information-maintenance)
7. [Category 5: Communication](#7-category-5-communication)
8. [System Call Interface and Libraries](#8-system-call-interface-and-libraries)
9. [Error Handling](#9-error-handling)
10. [Performance Considerations](#10-performance-considerations)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What is a System Call?

A **system call** (syscall) is the only legitimate way a user program can ask the OS kernel to perform a privileged operation.

```
  User Mode (restricted):          Kernel Mode (full hardware access):
  ┌────────────────────────┐       ┌────────────────────────────────────┐
  │ Your application       │       │ OS Kernel                          │
  │  - Read a file         │──────►│  - Actually reads disk hardware    │
  │  - Open a socket       │ syscall│  - Talks to network card           │
  │  - Create a process    │       │  - Manages process table           │
  │  - Get current time    │◄──────│  - Returns result                  │
  └────────────────────────┘       └────────────────────────────────────┘
            ↑ Cannot touch hardware directly
```

**Why system calls exist — three reasons:**

| Reason          | Explanation                                                                                          |
| --------------- | ---------------------------------------------------------------------------------------------------- |
| **Security**    | Without syscalls, any program could read another process's memory, access any file, crash the system |
| **Stability**   | Kernel validates all parameters before acting — prevents buggy apps from corrupting the system       |
| **Abstraction** | Programs don't need to know hardware details; same `read()` call works on any disk brand/type        |

**Analogy:** A hotel room service menu. You can't walk into the kitchen and cook your own meal. You call room service (system call), tell them what you want, they prepare it (kernel does the work), and deliver it (return result). The kitchen enforces rules about what's available and checks your room number.

---

## 2. How System Calls Work: Step by Step

```
  User program calls read() to read a file:

  1. Program calls read() library wrapper function
  2. Library puts syscall number (e.g., 0 on x86-64 Linux) and args in registers
  3. Library executes SYSCALL instruction (software interrupt / trap)

  CPU MODE SWITCH: User Mode → Kernel Mode

  4. CPU jumps to kernel's syscall handler via interrupt vector table
  5. Kernel saves process state (registers, stack pointer)
  6. Kernel reads syscall number → dispatches to correct handler (sys_read)
  7. Kernel validates parameters (valid fd? valid buffer? sufficient permissions?)
  8. Kernel performs actual disk read (talks to device driver, DMA, etc.)
  9. Kernel copies data into user-space buffer
  10. Kernel restores process state, places return value in register

  CPU MODE SWITCH: Kernel Mode → User Mode

  11. Program receives return value (number of bytes read, or -1 on error)
  12. Program continues executing
```

**Time cost:** A system call takes roughly 1–10 microseconds due to the mode switch overhead — much more than a regular function call (~nanoseconds), but negligible compared to actual I/O work (milliseconds).

---

## 3. Category 1: Process Control

These syscalls create, manage, and terminate processes.

| System Call            | Purpose               | Description                                          |
| ---------------------- | --------------------- | ---------------------------------------------------- |
| `fork()`               | Create new process    | Creates an exact copy (child) of the calling process |
| `exec()` / `execve()`  | Replace process image | Loads and runs a new program in current process      |
| `exit()` / `_exit()`   | Terminate process     | Process signals completion and frees resources       |
| `wait()` / `waitpid()` | Wait for child        | Parent pauses until child process finishes           |
| `getpid()`             | Get own PID           | Returns the process's own process ID                 |
| `getppid()`            | Get parent PID        | Returns parent process's ID                          |
| `kill()`               | Send signal           | Sends a signal to a process (not just SIGKILL)       |

**Classic fork-exec-wait pattern (how the shell runs commands):**

```c
// Shell runs: ls -l

pid_t pid = fork();   // create a copy of the shell

if (pid == 0) {
    // CHILD process — becomes the new program
    execv("/bin/ls", args);   // replaces child's code with 'ls'
    // If exec succeeds, the line below never runs
    exit(1);  // exec failed
} else {
    // PARENT process — the shell waits
    wait(NULL);   // shell blocks until ls finishes
    // Shell shows next prompt after wait() returns
}
```

```
  shell process ──fork()──► [child]──exec("ls")──► ls process
                 ──wait()──►                              │
                                                          │ exits
                 ◄──────────────────────────────────────┘
  shell continues (shows next prompt)
```

---

## 4. Category 2: File Management

These syscalls handle creating, opening, reading, writing, and deleting files.

| System Call           | Purpose                  | Description                                                |
| --------------------- | ------------------------ | ---------------------------------------------------------- |
| `open()`              | Open a file              | Returns a file descriptor (fd) for subsequent operations   |
| `read()`              | Read from file/fd        | Reads bytes into a buffer                                  |
| `write()`             | Write to file/fd         | Writes bytes from a buffer                                 |
| `close()`             | Close file descriptor    | Releases the kernel resource                               |
| `unlink()`            | Delete a file            | Removes the directory entry; file deleted when no fds open |
| `stat()`              | Get file metadata        | Returns size, permissions, timestamps, type                |
| `lseek()`             | Move read/write position | Random access within a file                                |
| `mkdir()` / `rmdir()` | Create/delete directory  | Manages directory entries                                  |

**File descriptor system:**

```
  open() returns a small integer (the file descriptor):

  Process's file descriptor table:
  fd 0 → stdin  (keyboard)
  fd 1 → stdout (terminal)
  fd 2 → stderr (terminal)
  fd 3 → (your newly opened file)   ← open() returns 3
  fd 4 → ...

  All subsequent read(3, ...) and write(3, ...) calls use fd=3
  close(3) removes this entry
```

**Reading a config file:**

```c
// Step 1: Open
int fd = open("/etc/config.txt", O_RDONLY);
if (fd < 0) {
    // error — file doesn't exist or no permission
    return -1;
}

// Step 2: Read
char buffer[1024];
ssize_t bytes_read = read(fd, buffer, sizeof(buffer));

// Step 3: Process
process_config(buffer, bytes_read);

// Step 4: Close (ALWAYS close — otherwise fd leak)
close(fd);
```

---

## 5. Category 3: Device Management

In Unix/Linux, devices are files (e.g., `/dev/sda`, `/dev/null`, `/dev/tty`). Most device I/O uses the same `read()`/`write()` syscalls as files, plus `ioctl()` for device-specific operations.

| System Call | Purpose                 | Example                                     |
| ----------- | ----------------------- | ------------------------------------------- |
| `read()`    | Read from device        | Get keyboard input, read from serial port   |
| `write()`   | Write to device         | Send data to printer, write to display      |
| `ioctl()`   | Device-specific control | Set baud rate, configure terminal, eject CD |
| `mmap()`    | Map device memory       | Map framebuffer directly for GPU access     |

**ioctl example (configure terminal):**

```c
#include <termios.h>

struct termios settings;
ioctl(STDIN_FILENO, TCGETS, &settings);  // get current terminal settings
settings.c_lflag &= ~ECHO;              // turn off echo (for password input)
ioctl(STDIN_FILENO, TCSETS, &settings); // apply new settings
```

**"Everything is a file" (Unix philosophy):**

```
  /dev/sda       → hard disk     (read/write blocks)
  /dev/tty       → terminal      (read/write characters)
  /dev/null      → null device   (discards all writes, reads return EOF)
  /dev/random    → random source (reads return random bytes)
  /dev/printer0  → printer       (write sends to printer)

  Same open/read/write/close syscalls work for ALL of them!
```

---

## 6. Category 4: Information Maintenance

These syscalls retrieve or set system and process metadata.

| System Call              | Purpose                      | Example                      |
| ------------------------ | ---------------------------- | ---------------------------- |
| `getpid()`               | Current process ID           | Logging, IPC addressing      |
| `getppid()`              | Parent process ID            | Daemon parent-child tracking |
| `getuid()` / `geteuid()` | User ID (real / effective)   | Permission checking          |
| `time()`                 | Current Unix timestamp       | Event timestamping           |
| `gettimeofday()`         | High-precision time          | Performance measurement      |
| `uname()`                | System info (kernel, arch)   | Detecting platform           |
| `sysinfo()`              | System stats (RAM, CPU load) | Resource monitoring          |

**Logging with system info:**

```c
#include <unistd.h>
#include <time.h>

pid_t pid   = getpid();
uid_t uid   = getuid();
time_t now  = time(NULL);

char log_msg[200];
snprintf(log_msg, sizeof(log_msg),
         "[PID %d] [UID %d] Action at %ld\n", pid, uid, (long)now);

write(log_fd, log_msg, strlen(log_msg));
```

---

## 7. Category 5: Communication

These syscalls let processes exchange data with each other.

| System Call                           | Mechanism      | Purpose                                       |
| ------------------------------------- | -------------- | --------------------------------------------- |
| `pipe()`                              | Pipe           | Create anonymous pipe (parent-child IPC)      |
| `mkfifo()`                            | Named pipe     | Create persistent named pipe                  |
| `socket()` / `connect()` / `accept()` | Socket         | Network or local IPC                          |
| `send()` / `recv()`                   | Socket data    | Read/write socket data                        |
| `shmget()` / `shmat()`                | Shared memory  | Map same physical RAM into multiple processes |
| `msgget()` / `msgsnd()` / `msgrcv()`  | Message queue  | POSIX message queue IPC                       |
| `mmap()`                              | Memory mapping | Share file-backed memory between processes    |

**Summary — how different IPC mechanisms use syscalls:**

```
  Pipe (anonymous):
  pipe(fd) → fork() → write(fd[1]) / read(fd[0])

  Named pipe:
  mkfifo("/tmp/pipe", 0666) → open("/tmp/pipe") → read/write → close

  Network socket:
  socket() → connect() → send()/recv() → close()

  Shared memory:
  shmget() → shmat() → [direct memory access, no syscall per operation!] → shmdt() → shmctl(IPC_RMID)

  POSIX shared memory:
  shm_open() → ftruncate() → mmap() → [direct access] → munmap() → shm_unlink()
```

---

## 8. System Call Interface and Libraries

Most programmers never call syscalls directly. They call **library wrapper functions** that handle the low-level details.

```
  Application programmer writes:
    printf("Hello!\n");         ← standard library function

  Library (stdio) does:
    - formats the string
    - adds to output buffer
    - when buffer flushes, calls: write(STDOUT_FILENO, "Hello!\n", 8)

  Kernel receives:
    syscall(SYS_write, 1, "Hello!\n", 8)
```

**Three-layer stack:**

```
  ┌──────────────────────────────────────┐
  │  Application: printf("Hello\n")      │  Level 3: What you write
  ├──────────────────────────────────────┤
  │  C Library (glibc): fprintf/write    │  Level 2: Buffering, formatting
  ├──────────────────────────────────────┤
  │  System call: write(1, "Hello", 6)   │  Level 1: Kernel interface
  ├──────────────────────────────────────┤
  │  Kernel: sys_write → device driver   │  Level 0: Hardware operations
  └──────────────────────────────────────┘
```

---

## 9. Error Handling

All system calls return a special value on failure (usually `-1` or `NULL`), and set the global `errno` variable to indicate the specific error.

```c
#include <errno.h>
#include <string.h>

int fd = open("data.txt", O_RDONLY);

if (fd < 0) {
    // Check errno for specific error
    switch (errno) {
        case ENOENT:  // Error NO ENTry
            fprintf(stderr, "Error: File not found\n");
            break;
        case EACCES:  // Error ACCESs
            fprintf(stderr, "Error: Permission denied\n");
            break;
        default:
            fprintf(stderr, "Error: %s\n", strerror(errno));
    }
    exit(1);
}

// Success path
read(fd, buffer, size);
```

**Common errno values:**

| errno       | Meaning                     |
| ----------- | --------------------------- |
| `ENOENT`    | File or directory not found |
| `EACCES`    | Permission denied           |
| `EEXIST`    | File already exists         |
| `EINVAL`    | Invalid argument            |
| `ENOMEM`    | Out of memory               |
| `EBADF`     | Bad file descriptor         |
| `EPIPE`     | Broken pipe (reader closed) |
| `ETIMEDOUT` | Operation timed out         |

---

## 10. Performance Considerations

**System calls are slow compared to regular function calls:**

```
  Regular function call:   ~1 nanosecond     (just a CALL + RET instruction)
  System call:             ~1,000 nanoseconds (mode switch, kernel execution, return)

  System call ≈ 1,000× more expensive than a function call
  BUT: actual I/O operations (disk, network) ≈ millions× slower than syscall overhead

  So syscall overhead matters only in tight loops with many small I/O operations.
```

**Batching to reduce syscalls:**

```c
// BAD: 1000 system calls
for (int i = 0; i < 1000; i++) {
    write(fd, &data[i], 1);   // one syscall per byte → 1000 mode switches
}

// GOOD: 1 system call
write(fd, data, 1000);         // one mode switch, writes all 1000 bytes

// EVEN BETTER: Let stdio buffer for you
for (int i = 0; i < 1000; i++) {
    fputc(data[i], file);      // buffered in user space; write() called infrequently
}
fflush(file);                  // forces buffered data to kernel
```

---

## 10. Code Examples

> Working code that demonstrates system calls in practice.

### C++ — Simple Version

Simulate the user → kernel → user mode switch for `sys_open`, `sys_read`, `sys_write`, and `sys_fork`.

```cpp
// Simulate system call interface: user process calls syscall → CPU traps to kernel → kernel runs handler → returns to user.
// No actual OS calls — everything is simulated with plain functions and print statements.

#include <iostream>
#include <string>
#include <unordered_map>
#include <functional>

// ── Kernel-mode handler functions ─────────────────────────────────────────────
// Each function below represents what the OS kernel does in kernel mode.

void kernel_sys_write(const std::string& msg) {
    // Kernel validates the buffer address and writes to the output device
    std::cout << "    [KERNEL] sys_write: writing to stdout → \"" << msg << "\"\n";
}

void kernel_sys_read(const std::string& source) {
    // Kernel reads from device/file into a user-space buffer
    std::cout << "    [KERNEL] sys_read: reading from \"" << source << "\" → 42 bytes copied to user buffer\n";
}

void kernel_sys_open(const std::string& filename) {
    // Kernel checks permissions, creates a file descriptor entry, returns fd number
    std::cout << "    [KERNEL] sys_open: opening \"" << filename << "\" → fd = 3\n";
}

void kernel_sys_fork() {
    // Kernel duplicates the process: copies PCB, page tables, file descriptors
    std::cout << "    [KERNEL] sys_fork: duplicating process → child PID = 1234\n";
}

// ── Syscall trap dispatcher ───────────────────────────────────────────────────
// In real hardware, INT 0x80 or SYSCALL instruction triggers this sequence:
//   1. CPU saves user-mode state (PC, registers, flags) on kernel stack
//   2. CPU loads kernel stack pointer and switches to ring 0 (kernel mode)
//   3. Kernel reads syscall number from register (eax / rax)
//   4. Kernel dispatches to the correct handler
//   5. IRET instruction restores user state and switches back to ring 3

void syscall_trap(const std::string& name, const std::string& arg = "") {
    std::cout << "\n[USER]   calling " << name << "(" << arg << ")\n";
    std::cout << "[CPU]    SYSCALL instruction → saving registers, mode switch: USER → KERNEL (ring 3 → ring 0)\n";

    // Dispatch to correct kernel handler
    if      (name == "sys_write") kernel_sys_write(arg);
    else if (name == "sys_read")  kernel_sys_read(arg);
    else if (name == "sys_open")  kernel_sys_open(arg);
    else if (name == "sys_fork")  kernel_sys_fork();
    else    std::cout << "    [KERNEL] unknown syscall: " << name << " → ENOSYS\n";

    std::cout << "[CPU]    IRET instruction → restoring registers, mode switch: KERNEL → USER (ring 0 → ring 3)\n";
}

int main() {
    std::cout << "=== System Call Simulation ===\n";

    // A user program makes several system calls — each causes a mode switch
    syscall_trap("sys_open",  "data.txt");     // open a file → get fd
    syscall_trap("sys_read",  "data.txt");     // read its contents
    syscall_trap("sys_write", "Hello, OS!");   // write output to stdout
    syscall_trap("sys_fork");                  // create a child process

    std::cout << "\n[USER]   program continues in user mode — all syscalls complete\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style

Syscall table dispatcher using function pointers (mirrors Linux's `sys_call_table`), plus `libc` wrapper functions.

```cpp
// Syscall table: maps syscall number → kernel handler (exactly like Linux's sys_call_table[]).
// Shows how libc wrappers (open/read/write) translate to raw syscall numbers.
// C++17, uses std::function, lambdas, unordered_map.

#include <iostream>
#include <functional>
#include <unordered_map>
#include <vector>
#include <string>

// ── Syscall argument / result types ──────────────────────────────────────────
struct SyscallArgs   { std::vector<std::string> v; };
struct SyscallResult { int retval; std::string msg; };

// ── Kernel handler function type ──────────────────────────────────────────────
using KernelHandler = std::function<SyscallResult(const SyscallArgs&)>;

// ── Syscall table ─────────────────────────────────────────────────────────────
// Linux x86-64 numbers: sys_read=0, sys_write=1, sys_open=2, sys_fork=57, sys_exit=60
std::unordered_map<int, std::pair<std::string, KernelHandler>> syscall_table;

void build_syscall_table() {
    syscall_table[0] = {"sys_read", [](const SyscallArgs& a) -> SyscallResult {
        std::cout << "    [kernel] sys_read: fd=" << a.v[0] << " count=" << a.v[1] << "\n";
        return {42, "42 bytes read"};
    }};
    syscall_table[1] = {"sys_write", [](const SyscallArgs& a) -> SyscallResult {
        std::cout << "    [kernel] sys_write: fd=" << a.v[0] << " buf=\"" << a.v[1] << "\"\n";
        return {(int)a.v[1].size(), "bytes written"};
    }};
    syscall_table[2] = {"sys_open", [](const SyscallArgs& a) -> SyscallResult {
        std::cout << "    [kernel] sys_open: path=\"" << a.v[0] << "\" → fd=5\n";
        return {5, "file descriptor"};
    }};
    syscall_table[3] = {"sys_close", [](const SyscallArgs& a) -> SyscallResult {
        std::cout << "    [kernel] sys_close: fd=" << a.v[0] << "\n";
        return {0, "closed"};
    }};
    syscall_table[57] = {"sys_fork", [](const SyscallArgs&) -> SyscallResult {
        std::cout << "    [kernel] sys_fork → child PID=7890\n";
        return {7890, "child PID"};
    }};
    syscall_table[60] = {"sys_exit", [](const SyscallArgs& a) -> SyscallResult {
        std::cout << "    [kernel] sys_exit: code=" << a.v[0] << "\n";
        return {0, "exited"};
    }};
}

// ── Raw syscall dispatcher (simulates SYSCALL instruction) ────────────────────
SyscallResult raw_syscall(int num, SyscallArgs args = {}) {
    std::cout << "\n  [trap]  SYSCALL #" << num;
    if (!syscall_table.count(num)) {
        std::cout << " → ENOSYS (no such syscall)\n";
        return {-1, "ENOSYS"};
    }
    auto& [name, handler] = syscall_table[num];
    std::cout << " (" << name << ")\n";
    SyscallResult r = handler(args);
    std::cout << "  [trap]  return retval=" << r.retval << " (" << r.msg << ")\n";
    return r;
}

// ── libc wrapper functions — what programmers actually call ───────────────────
// These hide the raw syscall numbers and provide a friendly C API.
int libc_open(const std::string& path)            { return raw_syscall(2, {{path, "O_RDONLY"}}).retval; }
int libc_read(int fd, int count)                   { return raw_syscall(0, {{std::to_string(fd), std::to_string(count)}}).retval; }
int libc_write(int fd, const std::string& buf)     { return raw_syscall(1, {{std::to_string(fd), buf}}).retval; }
int libc_close(int fd)                             { return raw_syscall(3, {{std::to_string(fd)}}).retval; }
int libc_fork()                                    { return raw_syscall(57).retval; }

int main() {
    build_syscall_table();
    std::cout << "=== Syscall Table Dispatcher ===\n";
    std::cout << "  (programmer uses libc wrappers; libc issues raw SYSCALL instructions)\n";

    // Programmer uses libc wrappers (exactly like real code)
    int fd    = libc_open("readme.txt");
    int bytes = libc_read(fd, 128);
    libc_write(1, "Hello from user space!");
    libc_close(fd);
    int child = libc_fork();

    // Direct raw syscall (like using syscall(2, ...) in Linux)
    std::cout << "\n  --- Direct raw syscall ---";
    raw_syscall(60, {{"0"}});   // sys_exit(0)

    // Unknown syscall
    std::cout << "\n  --- Unknown syscall ---";
    raw_syscall(999);

    return 0;
}
```

### Python — Simple Version

Simulate the mode switch and kernel dispatch for `sys_open`, `sys_read`, `sys_write`, and `sys_fork` with clear print output at each step.

```python
# Simulates user-space → kernel-space system call interface.
# Each syscall() call prints: user call → mode switch → kernel handler → mode switch back → return value.
# Self-contained: no imports needed beyond builtins.

# ── Kernel-mode handler functions ──────────────────────────────────────────────
# These represent what the OS kernel does in privileged (kernel) mode.

def kernel_sys_write(message):
    """Kernel validates buffer pointer and writes to output device."""
    print(f"    [KERNEL] sys_write: writing \"{message}\" to stdout")
    return len(message)   # bytes written

def kernel_sys_read(fd, count):
    """Kernel reads 'count' bytes from fd into user buffer, returns byte count."""
    fake_data = "Hello, file!"[:count]
    print(f"    [KERNEL] sys_read: fd={fd} count={count} → {len(fake_data)} bytes copied")
    return len(fake_data)

def kernel_sys_open(path, mode):
    """Kernel checks permissions, allocates a file descriptor, returns fd."""
    fd = 5  # pretend we assigned fd=5
    print(f"    [KERNEL] sys_open: \"{path}\" mode={mode} → fd={fd}")
    return fd

def kernel_sys_fork():
    """Kernel duplicates the calling process (PCB, page tables, open files)."""
    child_pid = 4321
    print(f"    [KERNEL] sys_fork: child created → child PID={child_pid}")
    return child_pid   # parent receives child PID; child would receive 0


# ── Syscall dispatcher ────────────────────────────────────────────────────────
# Simulates INT 0x80 or SYSCALL instruction triggering a trap to kernel mode.

def syscall(name, *args):
    """Simulate the full mode-switch cycle for one system call."""
    arg_str = ", ".join(str(a) for a in args)
    print(f"\n[USER]   {name}({arg_str})")
    print( "  [CPU]  SYSCALL instruction → mode switch: USER → KERNEL")

    dispatch = {
        "sys_write": kernel_sys_write,
        "sys_read":  kernel_sys_read,
        "sys_open":  kernel_sys_open,
        "sys_fork":  kernel_sys_fork,
    }

    if name in dispatch:
        result = dispatch[name](*args)
    else:
        print(f"    [KERNEL] syscall '{name}' not found → errno = ENOSYS")
        result = -1

    print( "  [CPU]  IRET → mode switch: KERNEL → USER")
    print(f"[USER]   return value = {result}")
    return result


# ── User program ───────────────────────────────────────────────────────────────
if __name__ == "__main__":
    print("=== System Call Simulation ===")

    fd    = syscall("sys_open",  "data.txt", "r")    # open file
    n     = syscall("sys_read",  fd, 5)              # read 5 bytes
    w     = syscall("sys_write", "Hello, OS!")       # write to stdout
    child = syscall("sys_fork")                      # create child process

    syscall("sys_unknown")   # demonstrate unknown syscall error

    print("\n[USER] program complete — returned from all syscalls")
```

### Python — Medium Level

Syscall table with a `@register` decorator, typed results, and thin `libc` wrapper functions — mirrors the real Linux kernel design.

```python
# Syscall table dispatcher: maps syscall numbers to kernel handlers via decorator.
# libc wrapper functions hide the raw numbers and provide programmer-friendly API.
# Mirrors Linux's sys_call_table[] and glibc wrapper pattern.

from dataclasses import dataclass, field
from typing import Callable, Any

# ── Result type ───────────────────────────────────────────────────────────────
@dataclass
class SyscallResult:
    retval: int          # ≥0 = success, -1 = error
    errno:  str = ""     # error name (e.g. "ENOENT") if retval == -1
    data:   Any = None   # optional payload (e.g., bytes read)

# ── Syscall table ─────────────────────────────────────────────────────────────
# Linux x86-64 numbers: read=0, write=1, open=2, close=3, fork=57, exit=60
_syscall_table: dict[int, tuple[str, Callable]] = {}

def register(num: int, name: str):
    """Decorator: registers a function as the kernel handler for syscall #num."""
    def decorator(fn: Callable) -> Callable:
        _syscall_table[num] = (name, fn)
        return fn
    return decorator

# ── Kernel handlers ───────────────────────────────────────────────────────────
@register(0, "sys_read")
def kernel_read(fd: int, count: int) -> SyscallResult:
    print(f"    [kernel] sys_read: fd={fd} count={count}")
    return SyscallResult(retval=count, data="x" * count)  # fake data

@register(1, "sys_write")
def kernel_write(fd: int, buf: str) -> SyscallResult:
    print(f"    [kernel] sys_write: fd={fd} buf=\"{buf}\"")
    return SyscallResult(retval=len(buf))

@register(2, "sys_open")
def kernel_open(path: str, flags: int = 0) -> SyscallResult:
    fd = abs(hash(path)) % 90 + 3   # deterministic fake fd
    print(f"    [kernel] sys_open: \"{path}\" flags={flags} → fd={fd}")
    return SyscallResult(retval=fd)

@register(3, "sys_close")
def kernel_close(fd: int) -> SyscallResult:
    print(f"    [kernel] sys_close: fd={fd}")
    return SyscallResult(retval=0)

@register(57, "sys_fork")
def kernel_fork() -> SyscallResult:
    print("    [kernel] sys_fork: duplicating process → child PID=8888")
    return SyscallResult(retval=8888)

@register(60, "sys_exit")
def kernel_exit(code: int = 0) -> SyscallResult:
    print(f"    [kernel] sys_exit: code={code}")
    return SyscallResult(retval=0)

# ── Raw syscall dispatcher ────────────────────────────────────────────────────
def raw_syscall(num: int, *args) -> SyscallResult:
    """Simulate SYSCALL instruction: trap to kernel, dispatch, return."""
    if num not in _syscall_table:
        print(f"  [trap]  SYSCALL #{num} → ENOSYS")
        return SyscallResult(-1, errno="ENOSYS")
    name, handler = _syscall_table[num]
    print(f"\n  [trap]  SYSCALL #{num} ({name}) args={args}")
    result = handler(*args)
    print(f"  [trap]  retval={result.retval}" + (f" errno={result.errno}" if result.errno else ""))
    return result

# ── libc wrappers ─────────────────────────────────────────────────────────────
# What programmers call — libc translates to raw_syscall() internally.
def open(path: str, flags: int = 0) -> int:  return raw_syscall(2, path, flags).retval
def read(fd: int, count: int)        -> str:  return raw_syscall(0, fd, count).data or ""
def write(fd: int, buf: str)         -> int:  return raw_syscall(1, fd, buf).retval
def close(fd: int)                   -> int:  return raw_syscall(3, fd).retval
def fork()                           -> int:  return raw_syscall(57).retval
def exit_process(code: int = 0):             raw_syscall(60, code)

# ── User program ───────────────────────────────────────────────────────────────
if __name__ == "__main__":
    print("=== Syscall Table Dispatcher ===")
    print("  programmer calls libc wrappers → libc issues raw SYSCALL → kernel handler runs\n")

    fd   = open("readme.txt")
    data = read(fd, 12)
    print(f"  [user]  read data: \"{data}\"")
    write(1, "Hello from user space!")
    close(fd)
    child = fork()
    print(f"  [user]  child PID = {child}")
    exit_process(0)

    print("\n  --- unknown syscall ---")
    raw_syscall(999)
```

---

## 11. Key Takeaways

- **System call** = controlled request from user mode to kernel mode to perform a privileged operation
- **Why they exist:** security (restrict hardware access), stability (validate requests), abstraction (hide hardware details)
- **Mode switch:** every syscall triggers user mode → kernel mode → user mode; costs ~1,000 ns per call
- **5 categories:**
  1. **Process control:** `fork`, `exec`, `exit`, `wait`, `kill`
  2. **File management:** `open`, `read`, `write`, `close`, `unlink`, `stat`
  3. **Device management:** `read`, `write`, `ioctl` (devices are files in Unix)
  4. **Information maintenance:** `getpid`, `time`, `getuid`, `uname`
  5. **Communication:** `pipe`, `socket`, `shmget`, `msgget`
- **Library vs syscall:** `printf()` is a library function that eventually calls the `write()` syscall — programmers usually work with library wrappers, not raw syscalls
- **Error handling:** check return value for -1 or NULL; read `errno` for specific error code; always handle errors
- **Performance:** minimize syscall frequency with buffering; batch small writes into one large write; let stdio manage buffering automatically
- **Everything is a file (Unix):** devices, pipes, sockets all use the same `read`/`write`/`close` interface — unified and elegant
- **fork-exec-wait** is the fundamental pattern for process creation in Unix: fork makes a copy, exec replaces it with new program, wait waits for completion
