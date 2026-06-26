# Threads in Python — threading, multiprocessing, and concurrent.futures

> Python offers three layers of concurrency: `threading` for I/O-bound concurrent tasks (limited by the GIL for CPU work), `multiprocessing` for true CPU-bound parallelism (each process gets its own Python interpreter), and `concurrent.futures` for a unified high-level interface over both.

---

## Table of Contents

1. [threading Module — Basics](#1-threading-module--basics)
2. [Race Condition in Python](#2-race-condition-in-python)
3. [threading.Lock](#3-threadinglock)
4. [threading.Semaphore](#4-threadingsemaphore)
5. [threading.Event](#5-threadingevent)
6. [threading.Condition — Producer-Consumer](#6-threadingcondition--producer-consumer)
7. [The GIL Problem for CPU-Bound Tasks](#7-the-gil-problem-for-cpu-bound-tasks)
8. [multiprocessing Module](#8-multiprocessing-module)
9. [concurrent.futures — Unified Interface](#9-concurrentfutures--unified-interface)
10. [asyncio — Single-Threaded Concurrency](#10-asyncio--single-threaded-concurrency)
11. [Code Examples](#11-code-examples)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. threading Module — Basics

```python
import threading
import time

def greet(name, delay):
    time.sleep(delay)
    print(f"Hello from {name}!")

# Create threads
t1 = threading.Thread(target=greet, args=("Thread-1", 1))
t2 = threading.Thread(target=greet, args=("Thread-2", 2))

# Start both threads
t1.start()
t2.start()

# Main thread waits for both to finish
t1.join()
t2.join()

print("All threads done!")
```

**Output** (both run concurrently — Thread-1 finishes first):

```
Hello from Thread-1!   ← after 1 second
Hello from Thread-2!   ← after 2 seconds
All threads done!
```

**Using a class (OOP approach):**

```python
class WorkerThread(threading.Thread):
    def __init__(self, task_id):
        super().__init__()
        self.task_id = task_id

    def run(self):                          # Override run()
        print(f"Task {self.task_id} running on {self.name}")

workers = [WorkerThread(i) for i in range(5)]
for w in workers: w.start()
for w in workers: w.join()
```

**Thread lifecycle:**

```
  t = threading.Thread(target=func)    ← NEW (not started)
  t.start()                            ← RUNNABLE
  # ... func is running on a new OS thread ...  ← RUNNING
  t.join()                             ← main waits; thread terminates when func returns
```

---

## 2. Race Condition in Python

```python
import threading

counter = 0   # Shared variable

def increment():
    global counter
    for _ in range(100_000):
        counter += 1   # NOT atomic in Python!
                       # This is: LOAD counter → ADD 1 → STORE counter
                       # A context switch can happen between any of these

threads = [threading.Thread(target=increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Expected: 500000, Got: {counter}")
# Got: ~320000 — many increments lost!
```

**Why `counter += 1` is not atomic:**

```
  counter += 1   compiles to:
    LOAD_FAST  counter       ← loads value into temp
    LOAD_CONST 1
    BINARY_ADD               ← computes new value
    STORE_FAST counter       ← writes back

  Thread switch can happen between LOAD and STORE!
```

---

## 3. threading.Lock

```python
import threading

counter = 0
lock = threading.Lock()   # Binary mutex

def increment():
    global counter
    for _ in range(100_000):
        with lock:            # Acquire lock (blocks if held by another thread)
            counter += 1     # Critical section — only one thread here at a time
                             # Lock released automatically when 'with' block exits

threads = [threading.Thread(target=increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Counter: {counter}")   # Always 500000
```

**`with lock:` vs manual lock/unlock:**

```python
# Manual (error-prone — unlock might not happen on exception)
lock.acquire()
try:
    counter += 1
finally:
    lock.release()   # Must be in finally block

# With context manager (recommended — auto-releases even on exception)
with lock:
    counter += 1
```

**threading.RLock — reentrant lock:**

```python
rlock = threading.RLock()

def nested_function():
    with rlock:               # Can acquire same lock multiple times
        with rlock:           # Won't deadlock (unlike regular Lock)
            do_something()
    # Lock released when outermost 'with' exits
```

Use `RLock` when a function that holds a lock might call another function that also needs the same lock.

---

## 4. threading.Semaphore

Controls how many threads can access a resource simultaneously:

```python
import threading
import time
import random

# Only 3 threads can use the "database connection" at once
db_semaphore = threading.Semaphore(3)

def query_database(thread_id):
    print(f"Thread {thread_id}: waiting for connection...")
    with db_semaphore:    # Blocks if 3 threads already inside
        print(f"Thread {thread_id}: got connection, querying...")
        time.sleep(random.uniform(0.5, 1.5))  # Simulate query time
        print(f"Thread {thread_id}: done, releasing connection")

threads = [threading.Thread(target=query_database, args=(i,)) for i in range(8)]
for t in threads: t.start()
for t in threads: t.join()
```

**What happens:**

```
  Semaphore = 3

  Thread 0 enters → semaphore = 2
  Thread 1 enters → semaphore = 1
  Thread 2 enters → semaphore = 0
  Thread 3 waits  ← BLOCKED (semaphore = 0)
  Thread 4 waits  ← BLOCKED
  ...

  Thread 0 finishes → semaphore = 1 → Thread 3 unblocked
  Thread 1 finishes → semaphore = 1 → Thread 4 unblocked
  ...
```

---

## 5. threading.Event

A simple flag that threads can wait on — one thread signals an event, others wake up:

```python
import threading
import time

event = threading.Event()   # Initially not set (False)

def worker(name):
    print(f"{name}: waiting for start signal...")
    event.wait()             # Blocks until event.set() is called
    print(f"{name}: got signal, starting work!")

def starter():
    time.sleep(2)
    print("Starter: sending start signal!")
    event.set()              # Wakes all waiting threads simultaneously

threads = [threading.Thread(target=worker, args=(f"Worker-{i}",)) for i in range(4)]
threads.append(threading.Thread(target=starter))

for t in threads: t.start()
for t in threads: t.join()
```

**Output:**

```
Worker-0: waiting for start signal...
Worker-1: waiting for start signal...
Worker-2: waiting for start signal...
Worker-3: waiting for start signal...
Starter: sending start signal!
Worker-0: got signal, starting work!   ← all 4 wake up at once
Worker-1: got signal, starting work!
Worker-2: got signal, starting work!
Worker-3: got signal, starting work!
```

**Event methods:**
| Method | Effect |
|-------------------|-----------------------------------------------|
| `event.set()` | Set flag to True; wake all waiting threads |
| `event.clear()` | Reset flag to False |
| `event.wait()` | Block until flag is True |
| `event.wait(timeout)` | Block up to N seconds, then continue |
| `event.is_set()` | Check current flag value (non-blocking) |

---

## 6. threading.Condition — Producer-Consumer

```python
import threading
import time
from collections import deque

buffer = deque()
MAX_SIZE = 5
condition = threading.Condition()

def producer():
    for i in range(10):
        with condition:
            while len(buffer) >= MAX_SIZE:
                condition.wait()         # Release lock + sleep until notified
            buffer.append(i)
            print(f"Produced: {i}  | buffer size: {len(buffer)}")
            condition.notify_all()       # Wake up consumers

def consumer():
    consumed = 0
    while consumed < 10:
        with condition:
            while len(buffer) == 0:
                condition.wait()         # Release lock + sleep until notified
            item = buffer.popleft()
            consumed += 1
            print(f"  Consumed: {item} | buffer size: {len(buffer)}")
            condition.notify_all()       # Wake up producers

prod_thread = threading.Thread(target=producer)
cons_thread = threading.Thread(target=consumer)
prod_thread.start()
cons_thread.start()
prod_thread.join()
cons_thread.join()
```

**How `condition.wait()` works:**

```
  Inside 'with condition:' (lock is held)

  condition.wait():
    1. Release the lock    ← other threads can now acquire it
    2. Sleep this thread
    3. On notify_all():
       a. Re-acquire the lock
       b. Return from wait()
    4. Re-check the while condition (spurious wakeups possible!)
```

---

## 7. The GIL Problem for CPU-Bound Tasks

```python
import threading
import time

def cpu_work(n):
    """Heavy CPU computation — counts down from n"""
    total = 0
    for i in range(n):
        total += i * i
    return total

N = 50_000_000

# Single-threaded
start = time.time()
cpu_work(N)
print(f"Single-threaded: {time.time() - start:.2f}s")

# Multi-threaded (2 threads, each doing N/2)
start = time.time()
t1 = threading.Thread(target=cpu_work, args=(N // 2,))
t2 = threading.Thread(target=cpu_work, args=(N // 2,))
t1.start(); t2.start()
t1.join();  t2.join()
print(f"Multi-threaded:  {time.time() - start:.2f}s")
# Multi-threaded is NOT faster — GIL prevents true parallel Python bytecode
# It may even be SLOWER due to GIL contention overhead!
```

**GIL visualization:**

```
  IDEAL (no GIL):
  Core 0:  [T1 Python code ──────────────────────────────────────►]
  Core 1:  [T2 Python code ──────────────────────────────────────►]
  Total time: N/2 × work per item

  REALITY with GIL:
  Core 0:  [T1 runs][wait][T1 runs][wait][T1 runs]
  Core 1:  [wait][T2 runs][wait][T2 runs][wait]
  Total time: ≈ N × work per item (same as single-threaded, plus overhead!)
```

**Solution: use `multiprocessing` for CPU work** (see section 8).

---

## 8. multiprocessing Module

Each process has its own Python interpreter and GIL — true parallelism for CPU-bound tasks:

```python
import multiprocessing
import time

def cpu_work(n):
    total = 0
    for i in range(n):
        total += i * i
    return total

N = 50_000_000

if __name__ == "__main__":   # Required on Windows (avoids recursive spawning)

    # Single process
    start = time.time()
    cpu_work(N)
    print(f"Single process: {time.time() - start:.2f}s")

    # Multiple processes
    start = time.time()
    p1 = multiprocessing.Process(target=cpu_work, args=(N // 2,))
    p2 = multiprocessing.Process(target=cpu_work, args=(N // 2,))
    p1.start(); p2.start()
    p1.join();  p2.join()
    print(f"Multi-process:  {time.time() - start:.2f}s")
    # On a 2+ core machine: roughly 2x faster!
```

**Sharing data between processes:**

```python
from multiprocessing import Process, Value, Array, Queue

# Shared integer (uses shared memory)
counter = Value('i', 0)   # 'i' = C int

def increment(shared_val):
    for _ in range(100_000):
        with shared_val.get_lock():   # Must lock! Race conditions still exist
            shared_val.value += 1

if __name__ == "__main__":
    processes = [Process(target=increment, args=(counter,)) for _ in range(4)]
    for p in processes: p.start()
    for p in processes: p.join()
    print(f"Counter: {counter.value}")   # 400000

# Queue — safe inter-process communication
q = Queue()

def producer(queue):
    for i in range(5):
        queue.put(i)         # Thread/process-safe put
        print(f"Put: {i}")

def consumer(queue):
    for _ in range(5):
        item = queue.get()   # Blocks until item available
        print(f"Got: {item}")
```

**Process Pool — parallel map:**

```python
from multiprocessing import Pool

def square(x):
    return x * x

if __name__ == "__main__":
    with Pool(processes=4) as pool:
        # Distributes work across 4 worker processes
        results = pool.map(square, range(20))
        print(results)
        # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81, 100, ...]

        # Non-blocking version
        async_results = pool.map_async(square, range(20))
        # ... do other work ...
        print(async_results.get())
```

---

## 9. concurrent.futures — Unified Interface

`concurrent.futures` provides a consistent API for both threads and processes:

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time

def fetch_data(url_id):
    """Simulates an I/O-bound task (e.g., HTTP request)"""
    time.sleep(0.5)   # Simulate network wait
    return f"Data from url-{url_id}"

def heavy_compute(n):
    """CPU-bound task"""
    return sum(i * i for i in range(n))

# ── I/O-bound: use ThreadPoolExecutor ──────────────────────────────
with ThreadPoolExecutor(max_workers=5) as executor:
    # Submit tasks
    futures = [executor.submit(fetch_data, i) for i in range(10)]

    # Collect results as they complete
    for future in futures:
        print(future.result())   # Blocks until this specific task is done

# ── CPU-bound: use ProcessPoolExecutor ─────────────────────────────
if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:
        inputs = [1_000_000, 2_000_000, 3_000_000, 4_000_000]

        # map() keeps order, blocks until all done
        results = list(executor.map(heavy_compute, inputs))
        print(results)
```

**as_completed — process results in order of completion (not submission):**

```python
from concurrent.futures import as_completed

with ThreadPoolExecutor(max_workers=5) as executor:
    future_to_id = {executor.submit(fetch_data, i): i for i in range(10)}

    for future in as_completed(future_to_id):
        task_id = future_to_id[future]
        result = future.result()
        print(f"Task {task_id} finished: {result}")
```

**Comparison:**

| Feature       | `ThreadPoolExecutor`    | `ProcessPoolExecutor`      |
| ------------- | ----------------------- | -------------------------- |
| Best for      | I/O-bound tasks         | CPU-bound tasks            |
| Shared memory | Yes (beware race conds) | No (separate memory)       |
| Startup cost  | Low                     | High                       |
| GIL effect    | Limited parallelism     | No GIL — true parallel     |
| Communication | Direct (shared vars)    | Via serialization (pickle) |

---

## 10. asyncio — Single-Threaded Concurrency

`asyncio` handles many I/O tasks with a **single thread** using an event loop — no race conditions, no locks needed for most cases:

```python
import asyncio
import aiohttp   # pip install aiohttp (async HTTP client)

async def fetch_url(session, url_id):
    # 'await' suspends this coroutine and lets others run
    await asyncio.sleep(0.5)   # Simulates async I/O wait
    return f"Data from url-{url_id}"

async def main():
    async with aiohttp.ClientSession() as session:
        # Run 10 fetches concurrently in a single thread
        tasks = [fetch_url(session, i) for i in range(10)]
        results = await asyncio.gather(*tasks)   # Run all concurrently
        for r in results:
            print(r)

asyncio.run(main())
```

**asyncio vs threading:**

```
  threading:        One OS thread per concurrent task
                    [T1][T2][T3][T4][T5]  ← 5 threads, OS switches between them

  asyncio:          One OS thread, cooperative multitasking
                    [──event loop──────────────────────────────]
                     T1 runs until await → T2 runs until await → T3...
                     No OS context switches — much lower overhead for I/O
```

**When to use what for I/O tasks:**

| Scale of I/O tasks | Recommendation    |
| ------------------ | ----------------- |
| < 100 tasks        | `threading`       |
| 100–10,000 tasks   | `asyncio`         |
| 10,000+ tasks      | `asyncio`         |
| CPU-bound          | `multiprocessing` |

---

## 11. Code Examples

> Working code that demonstrates Python threading, multiprocessing, and concurrent.futures in practice.

### C++ — Simple Version
Create N `std::thread` objects (the C++ equivalent of `threading.Thread`) — no GIL means CPU-bound work truly scales across cores.

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>

// C++ equivalent of Python's threading.Thread — no GIL limitation for CPU work
void worker(int id, int work_units) {
    long long total = 0;
    for (int i = 0; i < work_units; i++) total += i;   // CPU-bound: parallel in C++
    std::cout << "Thread " << id << " done (sum=" << total << ")\n";
}

int main() {
    const int N = 4, WORK = 5'000'000;

    auto start = std::chrono::steady_clock::now();

    std::vector<std::thread> threads;
    for (int i = 0; i < N; i++)
        threads.emplace_back(worker, i, WORK);   // All start immediately

    for (auto& t : threads) t.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    std::cout << N << " CPU-bound threads done in " << ms << "ms\n";
    std::cout << "(In Python, threading for CPU work would NOT be faster due to the GIL)\n";
    return 0;
}
// Compile: g++ -std=c++17 -pthread -O2 cpp_threads.cpp -o cpp_threads
```

### C++ — Medium / LeetCode Style
`std::async` + `std::future` for task-based parallelism — the C++ equivalent of `concurrent.futures.ProcessPoolExecutor`.

```cpp
#include <iostream>
#include <future>
#include <vector>
#include <chrono>
#include <numeric>

long long partial_sum(long long start, long long end) {
    long long s = 0;
    for (long long i = start; i < end; i++) s += i * i;
    return s;
}

int main() {
    const long long N = 50'000'000LL;
    const int TASKS = 4;
    long long chunk = N / TASKS;

    auto t0 = std::chrono::steady_clock::now();

    // Launch TASKS futures in parallel (like ProcessPoolExecutor.map in Python)
    std::vector<std::future<long long>> futures;
    for (int i = 0; i < TASKS; i++) {
        long long from = i * chunk;
        long long to   = (i + 1 == TASKS) ? N : from + chunk;
        futures.push_back(std::async(std::launch::async, partial_sum, from, to));
    }

    // Collect results — blocks until each future is ready
    long long total = 0;
    for (auto& f : futures) total += f.get();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - t0).count();

    std::cout << "Sum of squares 0.." << N << " = " << total << "\n";
    std::cout << "Parallel time: " << ms << "ms  (expected ~" << TASKS << "x faster)\n";
    return 0;
}
// Compile: g++ -std=c++17 -pthread -O2 async_futures.cpp -o async_futures
```

### Python — Simple Version
Create N threads with `threading.Thread`; prove GIL impact — I/O-bound tasks run concurrently (fast), CPU-bound tasks do not speed up.

```python
import threading
import time

def io_task(tid):
    """I/O-bound: sleep releases the GIL so other threads can run."""
    time.sleep(0.5)
    print(f"  Thread {tid}: I/O done")

def cpu_task(n):
    """CPU-bound: GIL held — only one thread executes Python bytecode at a time."""
    return sum(i * i for i in range(n))

N, WORK = 4, 3_000_000

# ── I/O-bound: threading works great ──────────────────────────────────
print("=== I/O-Bound: threading.Thread ===")
start = time.time()
threads = [threading.Thread(target=io_task, args=(i,)) for i in range(N)]
for t in threads: t.start()
for t in threads: t.join()
print(f"  {N} threads × 0.5s → {time.time()-start:.2f}s  (expect ~0.5s)\n")

# ── CPU-bound: threading does NOT help ───────────────────────────────
print("=== CPU-Bound: Sequential vs Threads ===")
start = time.time()
[cpu_task(WORK) for _ in range(N)]    # Sequential baseline
seq = time.time() - start
print(f"  Sequential:  {seq:.2f}s")

start = time.time()
threads = [threading.Thread(target=cpu_task, args=(WORK,)) for _ in range(N)]
for t in threads: t.start()
for t in threads: t.join()
print(f"  {N} threads:  {time.time()-start:.2f}s  (GIL → similar or slower than sequential!)")
print("  → Use multiprocessing.Pool for CPU-bound parallelism")
```

### Python — Medium Level
`concurrent.futures`: `ThreadPoolExecutor` for I/O-bound, `ProcessPoolExecutor` for CPU-bound, plus `asyncio` for single-threaded massive concurrency — all compared.

```python
import time
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_work(n):
    """CPU-bound: GIL blocks multi-thread execution."""
    return sum(i * i for i in range(n))

def io_work(delay):
    """I/O-bound: GIL released during sleep."""
    import time; time.sleep(delay); return delay

async def async_io(delay):
    """asyncio coroutine: yields on await, letting event loop run others."""
    await asyncio.sleep(delay)
    return delay

TASKS, N = 4, 2_000_000

def bench(label, fn):
    t = time.time(); fn()
    print(f"  {label:<55} {time.time()-t:.2f}s")

if __name__ == "__main__":
    work, delays = [N] * TASKS, [0.5] * TASKS

    print("CPU-BOUND: Which tool wins?")
    print("-" * 65)
    bench("Sequential (baseline):",
          lambda: [cpu_work(n) for n in work])
    with ThreadPoolExecutor(TASKS) as ex:
        bench("ThreadPoolExecutor  (GIL blocked — no speedup):",
              lambda: list(ex.map(cpu_work, work)))
    with ProcessPoolExecutor(TASKS) as ex:
        bench("ProcessPoolExecutor (no GIL  — true parallel):",
              lambda: list(ex.map(cpu_work, work)))

    print("\nI/O-BOUND: Which tool wins?")
    print("-" * 65)
    bench("Sequential (baseline):",
          lambda: [io_work(d) for d in delays])
    with ThreadPoolExecutor(TASKS) as ex:
        bench("ThreadPoolExecutor  (concurrent — great speedup):",
              lambda: list(ex.map(io_work, delays)))
    bench("asyncio gather         (single-thread, cooperative):",
          lambda: asyncio.run(asyncio.gather(*[async_io(d) for d in delays])))

    print("\nDecision guide:")
    print("  CPU-bound              → ProcessPoolExecutor  (bypass GIL with processes)")
    print("  I/O-bound (< 100 tasks) → ThreadPoolExecutor")
    print("  I/O-bound (1000+ tasks) → asyncio  (less overhead per task)")
```

---

## 12. Key Takeaways

- `threading.Thread(target=func, args=(...))` + `.start()` + `.join()` — basic thread creation
- Use `with lock:` (context manager) — never manually call `acquire()`/`release()` without try/finally
- `threading.Lock` = binary mutex; `threading.RLock` = reentrant (same thread can acquire multiple times)
- `threading.Semaphore(n)` = allow up to N threads simultaneously
- `threading.Event` = simple flag; `event.set()` wakes all `event.wait()` callers at once
- `threading.Condition` = advanced wait/notify for producer-consumer patterns
- **GIL** = only one thread executes Python bytecode at a time → `threading` is NOT useful for CPU-bound tasks
- Use `multiprocessing` for CPU-bound parallelism — each process gets its own GIL and interpreter
- `multiprocessing.Pool.map()` = easy parallel map across processes
- `concurrent.futures.ThreadPoolExecutor` = high-level threads; `ProcessPoolExecutor` = high-level processes
- `asyncio` = single-threaded cooperative concurrency — best for massive I/O (10,000+ connections)
- Always put `if __name__ == "__main__":` guard when using `multiprocessing` on Windows
