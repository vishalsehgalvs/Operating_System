# Pipes vs Named Pipes in OS

> **Pipes** are one-directional, in-memory channels for data between processes — an **ordinary pipe** works only between parent and child processes (anonymous, temporary), while a **named pipe (FIFO)** has a file system entry and can connect any two processes regardless of relationship.

---

## Table of Contents

1. [What is a Pipe?](#1-what-is-a-pipe)
2. [Ordinary Pipes (Unnamed)](#2-ordinary-pipes-unnamed)
3. [Named Pipes (FIFOs)](#3-named-pipes-fifos)
4. [How Pipes Work Internally](#4-how-pipes-work-internally)
5. [Ordinary vs Named Pipes Comparison](#5-ordinary-vs-named-pipes-comparison)
6. [Advantages and Limitations](#6-advantages-and-limitations)
7. [Real-World Usage](#7-real-world-usage)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What is a Pipe?

A pipe is a **communication channel** in an OS that lets data flow from one process to another through a shared buffer in memory — no disk I/O, no network. Data moves in **one direction only** (unidirectional), in FIFO order (first in, first out).

```
  Process A (Writer)                    Process B (Reader)
  ┌──────────┐                          ┌──────────┐
  │          │──write──►[PIPE BUFFER]──►│          │
  │          │          (in memory)     │          │
  └──────────┘                          └──────────┘

  One direction only. If both need to talk both ways → need TWO pipes.
```

**Analogy:** A one-way tube connecting two rooms. Person A stuffs notes in one end; Person B picks them up on the other end. Notes come out in the order they went in. Nobody can reverse-stuff notes back in.

---

## 2. Ordinary Pipes (Unnamed)

### Key Properties

- **Anonymous** — no name, no file system entry
- **Temporary** — destroyed when both process ends close
- **Requires parent-child relationship** — both processes must share a common ancestor (typically created via `fork()`)
- **Unidirectional** — one write end, one read end

### How They're Created (C, Linux/Unix)

```c
int fd[2];  // fd[0] = read end, fd[1] = write end

if (pipe(fd) == -1) {
    perror("pipe failed");
    return -1;
}

int pid = fork();  // create child process

if (pid == 0) {
    // CHILD PROCESS — reads from pipe
    close(fd[1]);           // close write end (child only reads)
    char buffer[100];
    read(fd[0], buffer, sizeof(buffer));
    printf("Child received: %s\n", buffer);
    close(fd[0]);

} else {
    // PARENT PROCESS — writes to pipe
    close(fd[0]);           // close read end (parent only writes)
    char msg[] = "Hello from parent!";
    write(fd[1], msg, sizeof(msg));
    close(fd[1]);
}
```

### Lifecycle

```
  1. Parent calls pipe(fd) → OS creates pipe buffer, gives two file descriptors
  2. Parent calls fork() → child inherits both file descriptors
  3. Parent closes fd[0], child closes fd[1] (each keeps only what it needs)
  4. Communication flows: parent writes fd[1], child reads fd[0]
  5. Both close their ends → OS destroys pipe buffer → automatic cleanup
```

### Shell Example (Ordinary Pipe in Action)

```bash
ls -l | grep ".txt"
```

What the shell does internally:

```
  1. shell calls pipe(fd)
  2. shell forks child1 → runs 'ls -l', stdout connected to fd[1]
  3. shell forks child2 → runs 'grep ".txt"', stdin connected to fd[0]
  4. Output of ls flows through pipe to input of grep
  5. When ls exits, pipe signals EOF to grep → grep exits
```

---

## 3. Named Pipes (FIFOs)

### Key Properties

- **Has a name** in the filesystem (appears as a special file type)
- **Persists** beyond the lifetime of the processes that created it
- **No relationship required** — any two processes with permission can use it
- **Unidirectional** (but two named pipes can create bidirectional channel)

### How They're Created

```bash
# Command line
mkfifo /tmp/myfifo
ls -la /tmp/myfifo
# Output: prw-r--r-- 1 user group 0 Jan 1 12:00 /tmp/myfifo
#         ^─ 'p' means pipe (FIFO special file)

# Clean up
rm /tmp/myfifo
```

```c
// C code
#include <sys/stat.h>

// Create named pipe (permissions: 0666)
int result = mkfifo("/tmp/myfifo", 0666);

// Writer process (can be ANY process)
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "Hello via named pipe!", 21);
close(fd);

// Reader process (can be ANY other process)
int fd = open("/tmp/myfifo", O_RDONLY);  // blocks until writer opens write end
char buffer[100];
read(fd, buffer, sizeof(buffer));
printf("Received: %s\n", buffer);
close(fd);
```

### Named Pipe Use Case: Two Unrelated Programs

```
  Process: log_generator (running independently, writing log lines)
  Process: log_monitor (separate program, reading and alerting on errors)

  Without named pipe:
  → log_generator has no way to talk to log_monitor (no shared parent)
  → Would need full sockets or files

  With named pipe:
  /tmp/logs_pipe  ← created once

  log_generator: open("/tmp/logs_pipe", O_WRONLY) → writes log lines
  log_monitor:   open("/tmp/logs_pipe", O_RDONLY) → reads and monitors

  No parent-child relationship needed!
```

### Blocking Behavior of Named Pipes

```
  open(named_pipe, O_RDONLY):
  → BLOCKS until some other process opens the same pipe for writing

  open(named_pipe, O_WRONLY):
  → BLOCKS until some other process opens the same pipe for reading

  This ensures both sides are connected before any data flows.
  (Use O_NONBLOCK flag to avoid blocking if needed)
```

---

## 4. How Pipes Work Internally

```
  Kernel-managed circular buffer (typically 64 KB on Linux):

  ┌────────────────────────────────────────┐
  │  [d][a][t][a][  ][  ][  ][  ][  ][  ] │
  │   ↑ read ptr                           │
  │            ↑ write ptr                 │
  └────────────────────────────────────────┘

  Writer: writes at write_ptr, advances write_ptr
  Reader: reads at read_ptr, advances read_ptr

  When buffer is FULL:
  → Writer blocks (waits) until reader consumes some data

  When buffer is EMPTY:
  → Reader blocks (waits) until writer writes more data

  When ALL write ends are closed:
  → Reader gets EOF (read() returns 0)

  When ALL read ends are closed:
  → Writer gets SIGPIPE signal (broken pipe)
```

This **automatic blocking and synchronization** is one of the main advantages of pipes — no explicit locks needed.

---

## 5. Ordinary vs Named Pipes Comparison

| Property                 | Ordinary Pipe                            | Named Pipe (FIFO)                             |
| ------------------------ | ---------------------------------------- | --------------------------------------------- |
| **Name**                 | No (anonymous)                           | Yes (filesystem path)                         |
| **Persistence**          | Temporary (destroyed when processes end) | Persists until explicitly deleted             |
| **Process relationship** | Must share ancestor (parent-child)       | Any two processes                             |
| **Creation**             | `pipe()` system call                     | `mkfifo()` system call or command             |
| **Filesystem entry**     | No                                       | Yes (appears as `p` type file)                |
| **Direction**            | Unidirectional                           | Unidirectional                                |
| **Use case**             | Shell pipelines, parent-child IPC        | Unrelated process IPC, log pipelines, daemons |
| **Overhead**             | Very low                                 | Low                                           |
| **Network?**             | No                                       | No (local machine only)                       |

---

## 6. Advantages and Limitations

### Advantages

| Advantage                     | Why it matters                                            |
| ----------------------------- | --------------------------------------------------------- | ------------------------------------ |
| **Simple API**                | read/write work just like regular files                   |
| **Automatic synchronization** | Blocking read/write prevents manual lock management       |
| **Fast**                      | In-memory buffer, no disk I/O                             |
| **Built-in flow control**     | Full buffer → writer blocks; empty buffer → reader blocks |
| **Shell integration**         | `                                                         | ` operator in every Unix/Linux shell |

### Limitations

| Limitation                           | Explanation                                                        |
| ------------------------------------ | ------------------------------------------------------------------ |
| **Unidirectional**                   | One direction only; need two pipes for bidirectional               |
| **Ordinary pipe: parent-child only** | Cannot connect two unrelated arbitrary processes                   |
| **No random access**                 | Sequential only; read data is consumed and gone                    |
| **Local only**                       | Pipes are within one machine; can't cross network boundaries       |
| **No persistence** (ordinary)        | Data is gone when both ends close                                  |
| **Fixed buffer size**                | Typically 64 KB; large data may stall writer until reader consumes |

---

## 7. Real-World Usage

### Shell Pipelines (Ordinary Pipes)

```bash
# Count files ending in .py in current dir recursively
find . -name "*.py" | wc -l

# Show top 5 largest files
du -sh * | sort -rh | head -5

# Filter nginx access log for 404s and count
grep " 404 " /var/log/nginx/access.log | wc -l

# Each | creates one ordinary pipe between adjacent commands
```

### Named Pipe Example: Real-time log streaming

```bash
# Terminal 1: Create named pipe and continuously write
mkfifo /tmp/applog
./my_application > /tmp/applog   # app writes log lines to pipe

# Terminal 2: Read and filter in real-time (separate process, no parent-child!)
grep "ERROR" /tmp/applog | tee error_log.txt
```

### Named Pipes in Programs (Python example)

```python
# Writer (producer.py) — any process
import os

pipe_path = "/tmp/data_pipe"
if not os.path.exists(pipe_path):
    os.mkfifo(pipe_path)

with open(pipe_path, 'w') as pipe:
    for i in range(10):
        pipe.write(f"item_{i}\n")
        pipe.flush()

# Reader (consumer.py) — completely separate process
with open("/tmp/data_pipe", 'r') as pipe:
    for line in pipe:
        print(f"Processing: {line.strip()}")
```

---

## 8. Key Takeaways

- **Pipe** = unidirectional, in-memory FIFO channel between two processes — data flows one way only
- **Ordinary pipe (unnamed):** anonymous, temporary, requires parent-child relationship (created via `pipe()` + `fork()`), used heavily in shell `|` operator
- **Named pipe (FIFO):** has a filesystem path, persists until deleted, any two processes with permission can use it (created via `mkfifo()`)
- **Both are unidirectional** — for bidirectional communication, create two pipes in opposite directions
- **Automatic synchronization:** reading from empty pipe → blocks; writing to full pipe → blocks; no explicit locks needed
- **Writer exits** (all write ends closed) → reader gets EOF; **reader exits** (all read ends closed) → writer gets SIGPIPE
- **Pipes cannot cross network boundaries** — for inter-machine IPC, use sockets or message queues
- **Shell `|` operator** creates ordinary pipes; `ls | grep` is two processes connected by a pipe the shell creates
- **When to use ordinary pipes:** parent-child communication, shell scripting, simple data pipelines
- **When to use named pipes:** two unrelated daemons/services communicating, log monitoring tools, producer-consumer between separate programs
