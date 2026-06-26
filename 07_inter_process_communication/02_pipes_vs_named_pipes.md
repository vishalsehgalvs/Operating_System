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

## 7. Code Examples

> Working code that demonstrates pipes and named pipes in practice.

### C++ — Simple Version

Simulate an anonymous pipe: one thread writes into a FIFO buffer, another reads — includes EOF handling.

```cpp
#include <iostream>
#include <queue>
#include <string>
#include <mutex>
#include <condition_variable>
#include <thread>

// ── Anonymous Pipe simulation ─────────────────────────────────────────────────
// Real pipe: pipe() returns two FDs — write_fd and read_fd.
// Unidirectional FIFO. When all write ends close, reader gets EOF.
// Simulation: a shared queue protected by mutex + condvar (identical semantics).

class AnonymousPipe {
    std::queue<std::string> buffer;
    std::mutex mtx;
    std::condition_variable cv;
    bool write_end_closed = false;   // simulates: all write FDs closed → EOF

public:
    // write() — puts data into the pipe (like write(write_fd, ...))
    void write(const std::string& data) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            buffer.push(data);
        }
        cv.notify_one();
    }

    // close_write() — closing write end signals EOF to reader
    void close_write() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            write_end_closed = true;
        }
        cv.notify_all();
    }

    // read() — blocks until data arrives; returns "" on EOF
    std::string read() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [&] { return !buffer.empty() || write_end_closed; });
        if (!buffer.empty()) {
            std::string data = buffer.front();
            buffer.pop();
            return data;
        }
        return "";   // empty string = EOF
    }
};

AnonymousPipe pipe_channel;

// Parent "process" — holds the write end of the pipe
void parent_process() {
    std::string lines[] = { "Hello from parent", "Line 2", "Line 3", "Last line" };
    for (const auto& line : lines) {
        pipe_channel.write(line);
        std::cout << "[Parent → Pipe] wrote: \"" << line << "\"\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    pipe_channel.close_write();    // close write end → child sees EOF
    std::cout << "[Parent] Closed write end\n";
}

// Child "process" — holds the read end of the pipe
void child_process() {
    std::cout << "[Child] Waiting to read from pipe...\n";
    while (true) {
        std::string data = pipe_channel.read();
        if (data.empty()) {
            std::cout << "[Child] Got EOF — write end closed\n";
            break;
        }
        std::cout << "[Child ← Pipe] read: \"" << data << "\"\n";
    }
}

int main() {
    std::cout << "=== Anonymous Pipe Simulation ===\n";
    std::cout << "Data: Parent ──[pipe]──► Child  (unidirectional FIFO)\n\n";

    std::thread writer(parent_process);
    std::thread reader(child_process);
    writer.join();
    reader.join();

    return 0;
}
```

### C++ — Medium / LeetCode Style

Simulate a named pipe with a registry: two unrelated writers connect by name (no fork required), one reader drains until EOF.

```cpp
#include <iostream>
#include <queue>
#include <string>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <map>

// ── Named Pipe: identified by a filesystem path, not by inherited FD ──────────
// Any process can open it by name — no parent-child relationship needed.
// Created with mkfifo(); persists on disk until unlink() removes it.

class NamedPipe {
    std::string path;
    std::queue<std::string> buffer;
    std::mutex mtx;
    std::condition_variable cv;
    int active_writers = 0;

public:
    explicit NamedPipe(const std::string& p) : path(p) {}

    void open_write() {
        std::lock_guard<std::mutex> lock(mtx);
        active_writers++;
    }

    void close_write() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            active_writers--;
        }
        cv.notify_all();   // may unblock reader waiting for EOF
    }

    void write(const std::string& data) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            buffer.push(data);
        }
        cv.notify_one();
    }

    // Returns false when all writers closed and buffer is drained (EOF)
    bool read(std::string& out) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [&] { return !buffer.empty() || active_writers == 0; });
        if (!buffer.empty()) { out = buffer.front(); buffer.pop(); return true; }
        return false;   // EOF
    }

    const std::string& name() const { return path; }
};

// ── Global registry — simulates the filesystem (mkfifo creates the entry) ─────
std::map<std::string, NamedPipe*> pipe_registry;
std::mutex registry_mtx;

NamedPipe* mkfifo(const std::string& path) {
    std::lock_guard<std::mutex> lock(registry_mtx);
    if (!pipe_registry.count(path))
        pipe_registry[path] = new NamedPipe(path);
    return pipe_registry[path];
}

NamedPipe* open_pipe(const std::string& path) {
    std::lock_guard<std::mutex> lock(registry_mtx);
    return pipe_registry.at(path);   // throws if not found (like ENOENT)
}

// ── Two UNRELATED writers: they find the pipe by name, not via fork() ─────────
void writer_A() {
    NamedPipe* p = open_pipe("/tmp/data_pipe");
    p->open_write();
    for (int i = 1; i <= 3; i++) {
        std::string msg = "WriterA_item" + std::to_string(i);
        p->write(msg);
        std::cout << "[Writer A] wrote: " << msg << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(120));
    }
    p->close_write();
    std::cout << "[Writer A] closed\n";
}

void writer_B() {
    NamedPipe* p = open_pipe("/tmp/data_pipe");
    p->open_write();
    std::this_thread::sleep_for(std::chrono::milliseconds(60));   // stagger
    for (int i = 1; i <= 3; i++) {
        std::string msg = "WriterB_item" + std::to_string(i);
        p->write(msg);
        std::cout << "[Writer B] wrote: " << msg << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(150));
    }
    p->close_write();
    std::cout << "[Writer B] closed\n";
}

void reader_process() {
    NamedPipe* p = open_pipe("/tmp/data_pipe");
    std::cout << "[Reader] Opened '" << p->name() << "' — waiting...\n";
    std::string line;
    while (p->read(line))
        std::cout << "[Reader] received: " << line << "\n";
    std::cout << "[Reader] EOF — all writers closed\n";
}

int main() {
    std::cout << "=== Named Pipe Simulation ===\n";
    std::cout << "Two UNRELATED writers + one reader — connected only by pipe name\n\n";

    mkfifo("/tmp/data_pipe");   // create the named pipe (like mkfifo in shell)

    std::thread r(reader_process);
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    std::thread wa(writer_A);
    std::thread wb(writer_B);

    wa.join(); wb.join(); r.join();

    std::cout << "\nKey: Writers A and B have no parent-child relationship.\n";
    std::cout << "They connected to the same pipe purely by its filesystem name.\n";

    return 0;
}
```

### Python — Simple Version

Simulate an anonymous pipe: producer writes, consumer reads with FIFO order and EOF signalling.

```python
import threading
import queue
import time

# ── Anonymous Pipe simulation ─────────────────────────────────────────────────
# Real POSIX pipe: pipe() returns (read_fd, write_fd) — unidirectional FIFO.
# Closing all write ends → reader gets EOF after draining remaining data.
# Python's queue.Queue behaves identically: FIFO, blocking reads, sentinel EOF.

class AnonymousPipe:
    def __init__(self):
        self._buf    = queue.Queue()        # FIFO buffer (kernel pipe buffer)
        self._closed = False

    def write(self, data: str):
        """Write end: put data into pipe (like write(write_fd, data))."""
        self._buf.put(data)
        print(f"[Write End] wrote: '{data}'")

    def close(self):
        """Close write end: send EOF sentinel so reader unblocks."""
        self._closed = True
        self._buf.put(None)               # None = EOF sentinel

    def read(self) -> str | None:
        """Read end: blocks until data available. Returns None on EOF."""
        return self._buf.get()            # None = EOF

pipe = AnonymousPipe()

def parent_writes():
    """Parent process — holds write end."""
    for line in ["Hello from parent", "Line 2", "Final line"]:
        pipe.write(line)
        time.sleep(0.1)
    pipe.close()
    print("[Parent] Closed write end")

def child_reads():
    """Child process — holds read end."""
    print("[Child] Waiting to read from pipe...")
    while True:
        data = pipe.read()
        if data is None:
            print("[Child] Got EOF — write end closed")
            break
        print(f"[Child] read: '{data}'")

print("=== Anonymous Pipe Simulation ===")
print("Data: Parent ──[pipe]──► Child  (unidirectional FIFO)\n")
t1 = threading.Thread(target=parent_writes)
t2 = threading.Thread(target=child_reads)
t1.start(); t2.start()
t1.join();  t2.join()
```

### Python — Medium Level

Simulate a named pipe with a registry: multiple unrelated processes connect by path, reader gets EOF when all writers close.

```python
import threading
import queue
import time

# ── Named pipe registry — simulates filesystem entries created by mkfifo() ────
_registry: dict[str, "NamedPipe"] = {}
_registry_lock = threading.Lock()

class NamedPipe:
    """Named pipe: any code opens it by path — no fork/parent relationship."""

    def __init__(self, path: str):
        self.path    = path
        self._buf    = queue.Queue()
        self._writers = 0
        self._lock   = threading.Lock()

    def open_write(self):
        with self._lock:
            self._writers += 1

    def close_write(self):
        with self._lock:
            self._writers -= 1
            if self._writers == 0:
                self._buf.put(None)       # EOF sentinel for reader

    def write(self, data: str):
        self._buf.put(data)

    def read(self) -> str | None:
        """Blocks until data available. Returns None on EOF."""
        return self._buf.get()

def mkfifo(path: str) -> NamedPipe:
    """Create named pipe (like mkfifo /tmp/path in shell)."""
    with _registry_lock:
        if path not in _registry:
            _registry[path] = NamedPipe(path)
        return _registry[path]

def open_fifo(path: str) -> NamedPipe:
    """Open existing named pipe by path — any process can call this."""
    with _registry_lock:
        if path not in _registry:
            raise FileNotFoundError(f"Named pipe '{path}' not found")
        return _registry[path]

# ── Demo: two completely unrelated "processes" write, one reads ───────────────
PIPE = "/tmp/log_pipe"

def daemon_logger():
    """Unrelated daemon — no fork(), just opens the pipe by name."""
    p = open_fifo(PIPE)
    p.open_write()
    for i in range(1, 4):
        msg = f"[Logger] event_{i}"
        p.write(msg)
        print(f"[Logger daemon] wrote: {msg}")
        time.sleep(0.15)
    p.close_write()
    print("[Logger daemon] closed")

def app_process():
    """Unrelated app — also opens the same pipe by name independently."""
    p = open_fifo(PIPE)
    p.open_write()
    time.sleep(0.08)                      # stagger so messages interleave
    for i in range(1, 4):
        msg = f"[App] request_{i}"
        p.write(msg)
        print(f"[App process]    wrote: {msg}")
        time.sleep(0.2)
    p.close_write()
    print("[App process] closed")

def log_monitor():
    """Monitor — reads everything until both writers have closed."""
    p = open_fifo(PIPE)
    print(f"[Monitor] Opened '{PIPE}' — reading...\n")
    while True:
        data = p.read()
        if data is None:
            print("[Monitor] EOF — all writers closed")
            break
        print(f"[Monitor] received: {data}")

mkfifo(PIPE)   # create the named pipe first (like: mkfifo /tmp/log_pipe)

print("=== Named Pipe Simulation ===")
print("Logger + App (UNRELATED) both write; Monitor reads until EOF\n")

monitor = threading.Thread(target=log_monitor)
logger  = threading.Thread(target=daemon_logger)
app     = threading.Thread(target=app_process)

monitor.start()
time.sleep(0.02)
logger.start(); app.start()
logger.join(); app.join(); monitor.join()

print("\nKey: Logger and App are unrelated — connected only by pipe path.")
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
