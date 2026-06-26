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
