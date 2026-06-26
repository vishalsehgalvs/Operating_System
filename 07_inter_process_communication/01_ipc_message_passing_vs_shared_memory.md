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

## 7. Code Examples

> Working code that demonstrates IPC message passing vs shared memory in practice.

### C++ — Simple Version

Simulate both IPC mechanisms with threads: message passing via a queue buffer, shared memory via a shared array.

```cpp
#include <iostream>
#include <queue>
#include <mutex>
#include <thread>
#include <string>
#include <chrono>

// ── Message Passing: data flows through a QUEUE buffer (like kernel copies) ───
std::queue<std::string> message_box;   // simulates kernel-managed message buffer
std::mutex mp_lock;

void mp_producer() {
    for (int i = 1; i <= 5; i++) {
        std::string msg = "Message_" + std::to_string(i);
        {
            std::lock_guard<std::mutex> g(mp_lock);
            message_box.push(msg);          // data is COPIED into the buffer
        }
        std::cout << "[MP Producer] sent: " << msg << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

void mp_consumer() {
    int received = 0;
    while (received < 5) {
        std::lock_guard<std::mutex> g(mp_lock);
        if (!message_box.empty()) {
            std::string msg = message_box.front();
            message_box.pop();              // data is COPIED out of the buffer
            std::cout << "[MP Consumer] received: " << msg << "\n";
            received++;
        }
    }
}

// ── Shared Memory: both sides access the SAME array directly (no copy) ────────
int  shared_array[5];          // the shared region — both threads access this
bool slot_ready[5] = {};       // flags: producer sets true when slot is filled
std::mutex sm_lock;

void sm_writer() {
    for (int i = 0; i < 5; i++) {
        {
            std::lock_guard<std::mutex> g(sm_lock);
            shared_array[i] = i * 10;      // DIRECT WRITE into shared region
            slot_ready[i]   = true;
        }
        std::cout << "[SM Writer] wrote " << i * 10 << " at slot " << i << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(80));
    }
}

void sm_reader() {
    for (int i = 0; i < 5; i++) {
        while (true) {
            std::lock_guard<std::mutex> g(sm_lock);
            if (slot_ready[i]) {
                std::cout << "[SM Reader] read " << shared_array[i]
                          << " from slot " << i << "\n";  // DIRECT READ — no copy
                break;
            }
        }
    }
}

int main() {
    std::cout << "=== IPC: Message Passing (data copied through buffer) ===\n";
    std::thread t1(mp_producer), t2(mp_consumer);
    t1.join(); t2.join();

    std::cout << "\n=== IPC: Shared Memory (direct access, no copy) ===\n";
    std::thread t3(sm_writer), t4(sm_reader);
    t3.join(); t4.join();

    return 0;
}
```

### C++ — Medium / LeetCode Style

Benchmark throughput of message passing vs shared memory transferring N items — measure the copy overhead.

```cpp
#include <iostream>
#include <queue>
#include <vector>
#include <mutex>
#include <thread>
#include <chrono>
#include <atomic>

constexpr int N = 100'000;   // number of items to transfer in each benchmark

// ── Message Passing: mailbox with a kernel-like queue buffer ─────────────────
struct Mailbox {
    std::queue<int> buf;
    std::mutex mtx;
};

void mp_sender(Mailbox& box) {
    for (int i = 0; i < N; i++) {
        std::lock_guard<std::mutex> g(box.mtx);
        box.buf.push(i);                // COPY 1: sender → buffer
    }
}

long long mp_receiver(Mailbox& box) {
    long long sum = 0;
    int count = 0;
    while (count < N) {
        std::lock_guard<std::mutex> g(box.mtx);
        if (!box.buf.empty()) {
            sum += box.buf.front();     // COPY 2: buffer → receiver
            box.buf.pop();
            count++;
        }
    }
    return sum;
}

// ── Shared Memory: lock-free ring — both sides access the same vector ─────────
struct SharedRegion {
    std::vector<int> data = std::vector<int>(N);
    std::atomic<int> produced{0};       // how many slots the writer has filled
};

void sm_writer(SharedRegion& r) {
    for (int i = 0; i < N; i++) {
        r.data[i] = i;                  // DIRECT WRITE — no copy at all
        r.produced.fetch_add(1, std::memory_order_release);
    }
}

long long sm_reader(SharedRegion& r) {
    long long sum = 0;
    for (int i = 0; i < N; i++) {
        while (r.produced.load(std::memory_order_acquire) <= i) {}  // spin-wait
        sum += r.data[i];               // DIRECT READ — no copy
    }
    return sum;
}

int main() {
    // Benchmark message passing
    Mailbox box;
    auto mp_start = std::chrono::high_resolution_clock::now();
    std::thread s(mp_sender, std::ref(box));
    long long mp_sum = mp_receiver(box);
    s.join();
    auto mp_ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::high_resolution_clock::now() - mp_start).count();

    // Benchmark shared memory
    SharedRegion region;
    auto sm_start = std::chrono::high_resolution_clock::now();
    std::thread w(sm_writer, std::ref(region));
    long long sm_sum = sm_reader(region);
    w.join();
    auto sm_ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::high_resolution_clock::now() - sm_start).count();

    std::cout << "Items:           " << N << "\n";
    std::cout << "Message Passing: sum=" << mp_sum << " | time=" << mp_ms << "ms\n";
    std::cout << "Shared Memory  : sum=" << sm_sum << " | time=" << sm_ms << "ms\n";
    std::cout << "Sums match: " << (mp_sum == sm_sum ? "YES" : "NO") << "\n";
    if (sm_ms > 0)
        std::cout << "Shared memory ~" << mp_ms / sm_ms << "x faster\n";

    return 0;
}
```

### Python — Simple Version

Demonstrate both IPC mechanisms side by side: queue for message passing, shared list for shared memory.

```python
import threading
import queue
import time

# ── Message Passing: producer and consumer share a QUEUE BUFFER ───────────────
# Two copies: sender→buffer, buffer→receiver (like kernel copies in real IPC)

message_box = queue.Queue()   # simulates kernel-managed message buffer

def mp_producer():
    for i in range(1, 6):
        msg = f"Message_{i}"
        message_box.put(msg)              # COPY: data goes into the buffer
        print(f"[MP Producer] sent: {msg}")
        time.sleep(0.1)

def mp_consumer():
    for _ in range(5):
        msg = message_box.get()           # COPY: data comes out of the buffer
        print(f"[MP Consumer] received: {msg}")

# ── Shared Memory: producer and consumer access the SAME LIST directly ─────────
# No copy — but requires explicit synchronization (mutex) to avoid races

shared_list = [None] * 5
slot_ready  = [False] * 5
sm_lock     = threading.Lock()

def sm_writer():
    for i in range(5):
        with sm_lock:
            shared_list[i] = i * 10       # DIRECT WRITE — no copy
            slot_ready[i]  = True
        print(f"[SM Writer] wrote {i * 10} at slot {i}")
        time.sleep(0.08)

def sm_reader():
    for i in range(5):
        while True:
            with sm_lock:
                if slot_ready[i]:
                    print(f"[SM Reader] read {shared_list[i]} from slot {i}")
                    break                 # DIRECT READ — no copy

print("=== Message Passing (data copied through queue buffer) ===")
t1 = threading.Thread(target=mp_producer)
t2 = threading.Thread(target=mp_consumer)
t1.start(); t2.start(); t1.join(); t2.join()

print("\n=== Shared Memory (direct access, no copy) ===")
t3 = threading.Thread(target=sm_writer)
t4 = threading.Thread(target=sm_reader)
t3.start(); t4.start(); t3.join(); t4.join()
```

### Python — Medium Level

Compare throughput of both mechanisms with timing — shows why shared memory wins for bulk data.

```python
import threading
import queue
import time

N = 50_000   # items to transfer in each benchmark

# ── Message Passing: mailbox with buffered queue ──────────────────────────────
def run_message_passing():
    mailbox = queue.Queue()
    results = {}

    def sender():
        for i in range(N):
            mailbox.put(i)                # COPY: item goes into queue buffer

    def receiver():
        total = 0
        for _ in range(N):
            total += mailbox.get()        # COPY: item comes out of buffer
        results["sum"] = total

    t1, t2 = threading.Thread(target=sender), threading.Thread(target=receiver)
    start = time.perf_counter()
    t1.start(); t2.start(); t1.join(); t2.join()
    return results["sum"], time.perf_counter() - start

# ── Shared Memory: both threads access the same list directly ─────────────────
def run_shared_memory():
    shared   = [0] * N         # both threads access the SAME list — no copy
    produced = [0]             # tracks how many slots writer has filled
    lock     = threading.Lock()
    results  = {}

    def writer():
        for i in range(N):
            with lock:
                shared[i]   = i       # DIRECT WRITE — no copy
                produced[0] = i + 1   # signal that slot i is ready

    def reader():
        total = 0
        for i in range(N):
            while True:               # spin-wait until writer fills slot i
                with lock:
                    if produced[0] > i:
                        total += shared[i]   # DIRECT READ — no copy
                        break
        results["sum"] = total

    t1, t2 = threading.Thread(target=writer), threading.Thread(target=reader)
    start = time.perf_counter()
    t1.start(); t2.start(); t1.join(); t2.join()
    return results["sum"], time.perf_counter() - start

mp_sum, mp_time = run_message_passing()
sm_sum, sm_time = run_shared_memory()

print(f"Items:           {N:,}")
print(f"Message Passing: sum={mp_sum:,} | time={mp_time:.3f}s")
print(f"Shared Memory  : sum={sm_sum:,} | time={sm_time:.3f}s")
print(f"Sums match: {mp_sum == sm_sum}")
print(f"Faster by:  {mp_time / sm_time:.1f}x  (shared memory wins for bulk data)")
```

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
