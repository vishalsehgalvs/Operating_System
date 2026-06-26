# IPC: Message Passing vs Shared Memory

> **Inter-Process Communication (IPC)** lets separate processes exchange data — **message passing** uses the OS as a safe middleman (slower, easier to use, works across networks), while **shared memory** gives processes direct access to the same RAM (faster, but requires manual synchronization to avoid race conditions).

---

## Table of Contents

1. [What is IPC?](#1-what-is-ipc)
2. [Message Passing](#2-message-passing)
3. [Shared Memory](#3-shared-memory)
4. [Synchronization in Shared Memory](#4-synchronization-in-shared-memory)
5. [Comparison: Message Passing vs Shared Memory](#5-comparison-message-passing-vs-shared-memory)
6. [When to Use Which](#6-when-to-use-which)
7. [Hybrid Approaches](#7-hybrid-approaches)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What is IPC?

Every process in a modern OS runs in its own **protected memory space**. Process A cannot read or write Process B's memory directly — the OS enforces this isolation.

```
  Process A              Process B
  ┌─────────┐            ┌─────────┐
  │ RAM: A  │    ✗ X     │ RAM: B  │
  │ (private)│   direct  │ (private)│
  └─────────┘   access   └─────────┘

  Without IPC: processes are completely isolated islands.

  With IPC:
  Process A ──[IPC mechanism]──► Process B
             (controlled, safe channel)
```

**Why IPC is needed:**

- Browser (one process) needs to tell the printer spooler (another process) to print
- Database server and connection pool processes need to share cached data
- Multiple worker processes need to coordinate on a shared task queue
- Copy-paste between two applications (clipboard = shared data)

Two fundamental approaches: **Message Passing** and **Shared Memory**.

---

## 2. Message Passing

### How It Works

The OS acts as a **middleman**. Processes never touch each other's memory. All communication goes through the kernel.

```
  Process A wants to send "Hello" to Process B:

  Step 1: A calls send(process_B_id, "Hello")
  Step 2: OS copies "Hello" from A's memory → kernel buffer
  Step 3: B calls receive(process_A_id, buffer)
  Step 4: OS copies "Hello" from kernel buffer → B's memory

  Process A memory: ["Hello"]     Process B memory: [      ]
         copy ↓                                       ↑ copy
              [kernel buffer: "Hello"]────────────────

  Two copies happen. Both processes work with their OWN data.
```

**Analogy:** Sending an email. You write the message, send it to the mail server (kernel buffer), and the recipient checks their inbox (calls receive). The mail server handles delivery. You never handed your laptop to the recipient.

### Code Example (Conceptual)

```c
// Process A (Sender)
char message[] = "Hello from Process A";
send(process_B_id, message);       // kernel copies message out
printf("Message sent\n");
// Process A continues immediately — doesn't wait for B to read it (async)

// Process B (Receiver)
char buffer[100];
receive(process_A_id, buffer);     // kernel copies message in
printf("Received: %s\n", buffer);
// Output: Received: Hello from Process A
```

### Message Passing Modes

| Mode                            | Behavior                                           | Use case                                               |
| ------------------------------- | -------------------------------------------------- | ------------------------------------------------------ |
| **Synchronous (blocking)**      | Sender waits until receiver gets the message       | When you need confirmation the message was received    |
| **Asynchronous (non-blocking)** | Sender continues immediately; message queued       | High-throughput systems; fire-and-forget notifications |
| **Direct**                      | Name the specific recipient: `send(processB, msg)` | Point-to-point communication                           |
| **Indirect (mailbox/port)**     | Send to a named queue: `send(queue_name, msg)`     | Multiple senders/receivers; decoupled design           |

### Real-World Examples of Message Passing

| Example                        | How it uses message passing                        |
| ------------------------------ | -------------------------------------------------- |
| Web browser ↔ web server       | HTTP requests/responses over network sockets       |
| Email client ↔ SMTP server     | Email messages via SMTP protocol                   |
| Microservices                  | REST API calls, message queues (Kafka, RabbitMQ)   |
| Shell pipes (`ls \| grep txt`) | Pipe connects stdout of `ls` to stdin of `grep`    |
| Android Intents                | Apps communicate by sending Intent messages via OS |

---

## 3. Shared Memory

### How It Works

The OS creates a single physical memory region and **maps it into the address space of multiple processes**. Each process sees the same RAM — reads and writes happen at full memory speed with NO kernel involvement after setup.

```
  Setup phase (OS does this once):
  ┌───────────────────────────────────────┐
  │   Physical RAM: Shared Segment        │  ← one real block of RAM
  └───────────────────────────────────────┘
         ↑ mapped into         ↑ mapped into

  Process A's virtual memory:       Process B's virtual memory:
  ┌──────────────────────┐          ┌──────────────────────┐
  │  A's private memory  │          │  B's private memory  │
  │  [shared_segment ptr]│          │  [shared_segment ptr]│
  └──────────────────────┘          └──────────────────────┘
           ↓ both point to same physical block

  Runtime (NO kernel involvement):
  A writes: shared_data = 42;   → instantly visible to B!
  B reads:  int x = shared_data; → x is 42
```

**Analogy:** A physical whiteboard in a shared office. Anyone in the room can walk up and read or write. Much faster than emailing notes back and forth, but if two people write at the same time, you get a mess.

### Code Example (Conceptual)

```c
// Process A (Writer)
int* shared_data = attach_shared_memory("segment1");
*shared_data = 42;
printf("Process A wrote: %d\n", *shared_data);
// Output: Process A wrote: 42
detach_shared_memory(shared_data);

// Process B (Reader) — running at same time as A
int* shared_data = attach_shared_memory("segment1");
printf("Process B read: %d\n", *shared_data);
// Output: Process B read: 42
detach_shared_memory(shared_data);
```

**Linux API (POSIX shared memory):**

```c
// Create/open shared memory
int fd = shm_open("/my_segment", O_CREAT | O_RDWR, 0666);
ftruncate(fd, sizeof(int));
int* data = mmap(NULL, sizeof(int), PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

// Write to it (no system call needed for each access!)
*data = 42;

// Cleanup
munmap(data, sizeof(int));
shm_unlink("/my_segment");
```

---

## 4. Synchronization in Shared Memory

Shared memory is fast but **dangerous without synchronization**. Two processes writing at the same time = race condition = data corruption.

### The Race Condition Problem

```
  Shared counter starts at 5.
  Process A and Process B both want to increment it.

  Expected result: 7 (5 + 1 + 1)

  Without synchronization:
  Time: A reads 5
  Time: B reads 5  (before A writes back!)
  Time: A computes 5+1=6, writes 6
  Time: B computes 5+1=6, writes 6

  Result: 6  ← ONE INCREMENT WAS LOST!
```

### Solution: Use a Semaphore or Mutex

```c
// SAFE version with semaphore protection:

// Process A:
semaphore_wait(lock);       // acquire lock (blocks if B holds it)
int* counter = attach_shared_memory("counter");
(*counter)++;               // now only A is incrementing
semaphore_signal(lock);     // release lock

// Process B:
semaphore_wait(lock);       // waits if A holds it
(*counter)++;               // only B is incrementing now
semaphore_signal(lock);     // release lock

// Result: counter goes from 5 → 6 → 7 correctly
```

**Key rule:** Always protect shared memory access with a synchronization primitive (mutex, semaphore, spinlock). With message passing, the OS handles this automatically.

---

## 5. Comparison: Message Passing vs Shared Memory

| Aspect                        | Message Passing                        | Shared Memory                          |
| ----------------------------- | -------------------------------------- | -------------------------------------- |
| **Speed**                     | Slower (2+ data copies through kernel) | Faster (direct RAM access after setup) |
| **Synchronization**           | OS handles automatically               | Programmer must implement explicitly   |
| **Safety**                    | Safer (processes stay isolated)        | Risky (race conditions possible)       |
| **Ease of use**               | Easier to write correctly              | More complex; bugs are hard to find    |
| **Best data size**            | Small messages                         | Large data blocks                      |
| **Across network?**           | Yes (sockets, HTTP, MQ)                | No (same machine only)                 |
| **OS overhead per operation** | High (system call each time)           | Low (only during setup/teardown)       |
| **Debugging**                 | Easier (clear boundaries)              | Harder (memory bugs, heisenbugs)       |

### Performance Numbers (Approximate)

```
  Same machine, 1 MB data transfer:

  Message passing (pipe):        ~5,000 microseconds  (1 MB / 200 MB/s effective)
  Message passing (socket):      ~8,000 microseconds
  Shared memory:                 ~100 microseconds    (limited by cache/RAM bandwidth)

  → Shared memory ≈ 50–80× faster for large local transfers

  But for small messages (< 1 KB): difference is negligible.
  For cross-machine transfers: message passing is the ONLY option.
```

---

## 6. When to Use Which

### Use Message Passing When:

| Situation                                 | Why message passing fits                     |
| ----------------------------------------- | -------------------------------------------- |
| Processes on different machines           | Shared memory can't cross network boundaries |
| Small, infrequent messages                | Overhead is negligible; code is cleaner      |
| Safety matters more than speed            | OS prevents accidental memory corruption     |
| Team is less experienced with concurrency | Less chance of subtle race condition bugs    |
| Microservices architecture                | Services must be independently deployable    |
| Real-time event notification              | Async queues excel at event-driven patterns  |

### Use Shared Memory When:

| Situation                                  | Why shared memory fits                           |
| ------------------------------------------ | ------------------------------------------------ |
| Large data blocks on same machine          | Avoid copying megabytes through kernel           |
| High-frequency reads of same data          | Multiple consumers read same data with zero copy |
| Producer-consumer, high throughput         | One process writes, many read at memory speed    |
| Shared cache (e.g., database buffer pool)  | Multiple workers share same cached pages         |
| Scientific computing / parallel processing | Multiple threads/processes on same large array   |

---

## 7. Hybrid Approaches

Many real-world systems use BOTH. The pattern:

```
  Control channel:  Message Passing  (small, infrequent signals/commands)
  Data channel:     Shared Memory    (large, frequent data transfers)

  Example — Video processing pipeline:

  Encoder process                     Player process

  encodes frame → [shared memory]  ← reads frame
                ──[pipe: "frame_ready" signal]──►
                                   player knows when to read

  The DATA (large frame) = shared memory (fast, no copy)
  The SIGNAL ("frame is ready") = message passing (simple, safe)
```

**Other examples:**

- **Database:** Shared memory for buffer pool (data), message queue for query dispatch (control)
- **Browser:** Shared memory for rendered bitmaps, IPC messages for DOM events
- **MPI (parallel computing):** Shared memory within one node, message passing across nodes

---

## 8. Key Takeaways

- **IPC** is needed because processes run in isolated memory spaces — they cannot read each other's RAM without OS-mediated mechanisms
- **Message passing:** OS copies data through kernel buffer; safe, easy, works over network, but slower due to copying overhead
- **Shared memory:** OS maps same physical RAM into multiple process address spaces; fastest possible IPC, but requires explicit synchronization
- **Two copies happen** in message passing (sender→kernel, kernel→receiver); **zero copies** in shared memory after initial setup
- **Race conditions** are the core danger of shared memory — always protect with semaphores, mutexes, or other locks
- **Message passing = automatic synchronization** (OS manages it); **shared memory = manual synchronization** (your job as programmer)
- **Cross-network IPC:** message passing only (pipes, sockets, HTTP, message queues like Kafka/RabbitMQ)
- **High-throughput local IPC:** shared memory wins (50–100× faster than pipes for large data)
- **Best practice:** use message passing for control flow and small signals; use shared memory for bulk data transfer — combine both in the same system
- **Real-world tools:** Linux pipes, POSIX message queues, Unix sockets (message passing); POSIX `shm_open`/`mmap`, `shmget`/`shmat` (shared memory)
