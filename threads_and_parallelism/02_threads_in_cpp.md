# Threads in C++ — std::thread and Synchronization

> C++ gives you direct control over threads via `std::thread` (C++11+) with no GIL — every thread you create can run truly in parallel on multiple cores, but you're responsible for all synchronization yourself.

---

## Table of Contents

1. [Creating and Joining Threads](#1-creating-and-joining-threads)
2. [Passing Arguments to Threads](#2-passing-arguments-to-threads)
3. [Race Condition Example](#3-race-condition-example)
4. [Fixing it with std::mutex](#4-fixing-it-with-stdmutex)
5. [std::lock_guard and std::unique_lock](#5-stdlock_guard-and-stdunique_lock)
6. [std::condition_variable](#6-stdcondition_variable)
7. [std::atomic](#7-stdatomic)
8. [std::async and std::future](#8-stdasync-and-stdfuture)
9. [Thread Pool Pattern](#9-thread-pool-pattern)
10. [Code Examples](#10-code-examples)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. Creating and Joining Threads

```cpp
#include <iostream>
#include <thread>

void greet(int id) {
    std::cout << "Hello from thread " << id << "\n";
}

int main() {
    std::thread t1(greet, 1);   // Create thread, run greet(1)
    std::thread t2(greet, 2);   // Create thread, run greet(2)

    t1.join();   // Main thread waits for t1 to finish
    t2.join();   // Main thread waits for t2 to finish

    std::cout << "Both threads done\n";
    return 0;
}
```

**Thread lifecycle with join/detach:**

```
  main()
    │
    ├── t1 = std::thread(greet, 1) ─────────────────────────────┐
    │                                                    thread t1 runs
    ├── t2 = std::thread(greet, 2) ───────────────────┐         │
    │                                          thread t2 runs    │
    ├── t1.join() ─────────────────────────────────────┼─────────┘ wait
    ├── t2.join() ─────────────────────────────────────┘ wait
    │
    └── "Both threads done"

  join()  = main waits for the thread to finish (safe)
  detach()= main doesn't wait; thread runs independently (fire and forget)
```

> **Important:** Always `join()` or `detach()` a thread before its `std::thread` object is destroyed, or the program will `std::terminate()`.

---

## 2. Passing Arguments to Threads

```cpp
#include <iostream>
#include <thread>
#include <string>

// Pass by value
void print_message(std::string msg, int repeat) {
    for (int i = 0; i < repeat; i++) {
        std::cout << msg << "\n";
    }
}

// Pass by reference (must use std::ref)
void double_value(int& x) {
    x *= 2;
}

int main() {
    // Pass by value — copies are made
    std::thread t1(print_message, "Hello!", 3);

    // Pass by reference — wraps in std::ref to avoid copy
    int num = 5;
    std::thread t2(double_value, std::ref(num));

    t1.join();
    t2.join();

    std::cout << "num after doubling: " << num << "\n";  // 10
    return 0;
}
```

**Using lambdas (most common modern approach):**

```cpp
int main() {
    int result = 0;

    std::thread t([&result]() {
        result = 42;          // Capture by reference
    });

    t.join();
    std::cout << result << "\n";  // 42
}
```

---

## 3. Race Condition Example

```cpp
#include <iostream>
#include <thread>

int counter = 0;   // Shared variable

void increment() {
    for (int i = 0; i < 100000; i++) {
        counter++;   // NOT atomic! This is: read → add 1 → write
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Counter: " << counter << "\n";
    // Expected: 200000
    // Actual:   ~120000–180000 (random each run!)
}
```

**Why it fails:**

```
  counter = 50000

  Thread 1:  reads counter → gets 50000
                                           Thread 2:  reads counter → gets 50000
  Thread 1:  calculates 50001
  Thread 1:  writes 50001
                                           Thread 2:  calculates 50001
                                           Thread 2:  writes 50001  ← overwrites T1's work!

  One increment was lost. Multiplied by thousands of iterations → large error.
```

---

## 4. Fixing it with std::mutex

```cpp
#include <iostream>
#include <thread>
#include <mutex>

int counter = 0;
std::mutex mtx;    // The lock — only one thread holds it at a time

void increment() {
    for (int i = 0; i < 100000; i++) {
        mtx.lock();      // Acquire lock — other threads BLOCK here
        counter++;       // Critical section — safe now
        mtx.unlock();    // Release lock — next waiting thread wakes up
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Counter: " << counter << "\n";  // Always 200000
}
```

**Mutex flow:**

```
  Thread 1                     Mutex                Thread 2
  ────────                     ─────                ────────
  mtx.lock()  ──────────────►  LOCKED (by T1)
  counter++                    (T2 waits here)  ◄── mtx.lock()
  mtx.unlock() ─────────────►  UNLOCKED
                               LOCKED (by T2)  ◄─── T2 wakes up
                               counter++
                               UNLOCKED
```

---

## 5. std::lock_guard and std::unique_lock

Calling `lock()` and `unlock()` manually is dangerous — if an exception is thrown between them, the mutex is never released (deadlock). Use RAII wrappers instead:

### std::lock_guard — simple, auto-releases on scope exit

```cpp
void increment() {
    for (int i = 0; i < 100000; i++) {
        std::lock_guard<std::mutex> guard(mtx);  // Locks here
        counter++;
        // guard destructor automatically calls mtx.unlock() when scope ends
        // Even if an exception is thrown!
    }
}
```

### std::unique_lock — more flexible (can unlock early, use with condition_variable)

```cpp
void increment() {
    for (int i = 0; i < 100000; i++) {
        std::unique_lock<std::mutex> lock(mtx);   // Locks here
        counter++;
        lock.unlock();    // Can manually unlock early
        // ... do other work without holding the lock ...
        lock.lock();      // Can re-lock
    }  // Auto-unlocks here if still locked
}
```

| Feature                  | `lock_guard` | `unique_lock` |
| ------------------------ | :----------: | :-----------: |
| Auto unlock on destroy   |      ✅      |      ✅       |
| Manual early unlock      |      ❌      |      ✅       |
| Works with condition_var |      ❌      |      ✅       |
| Overhead                 |   Minimal    | Slightly more |

---

## 6. std::condition_variable

Used when one thread needs to **wait for a condition** that another thread will signal.
Classic use case: **producer-consumer**.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> buffer;
const int MAX_SIZE = 5;

void producer() {
    for (int i = 1; i <= 10; i++) {
        std::unique_lock<std::mutex> lock(mtx);

        // Wait while buffer is full
        cv.wait(lock, [] { return buffer.size() < MAX_SIZE; });

        buffer.push(i);
        std::cout << "Produced: " << i << "\n";

        cv.notify_all();  // Wake up any waiting consumer
    }
}

void consumer() {
    for (int i = 0; i < 10; i++) {
        std::unique_lock<std::mutex> lock(mtx);

        // Wait while buffer is empty
        cv.wait(lock, [] { return !buffer.empty(); });

        int item = buffer.front();
        buffer.pop();
        std::cout << "Consumed: " << item << "\n";

        cv.notify_all();  // Wake up any waiting producer
    }
}

int main() {
    std::thread prod(producer);
    std::thread cons(consumer);
    prod.join();
    cons.join();
}
```

**How cv.wait() works:**

```
  cv.wait(lock, predicate):
    1. Check predicate — if TRUE, continue (no wait)
    2. If FALSE:
       a. Atomically release the lock
       b. Put thread to sleep
       c. When notified: re-acquire the lock
       d. Check predicate again (loop to handle spurious wakeups)
```

---

## 7. std::atomic

For simple operations on a single variable, `std::atomic` is **faster than a mutex** — no lock overhead, uses hardware atomic instructions.

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<int> counter(0);   // Thread-safe integer

void increment() {
    for (int i = 0; i < 100000; i++) {
        counter++;   // This IS atomic — no mutex needed
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();

    std::cout << "Counter: " << counter << "\n";  // Always 200000
}
```

**When to use atomic vs mutex:**

| Use Case                               | Use                                      |
| -------------------------------------- | ---------------------------------------- |
| Increment/decrement a single counter   | `std::atomic`                            |
| Read/write a single flag               | `std::atomic<bool>`                      |
| Protect a block of code (multiple ops) | `std::mutex`                             |
| Wait for a condition                   | `std::condition_variable` + `std::mutex` |

```
  std::atomic<int> x = 5;

  x++         ← one atomic instruction (test_and_set / fetch_add)
  x += 3      ← atomic
  int val = x ← atomic load

  x = x * 2  ← NOT atomic! Read + compute + write = 3 ops
              ← Need mutex for compound operations like this
```

---

## 8. std::async and std::future

`std::async` launches a task and gives you a `std::future` to retrieve the result later. Simpler than raw threads for task-based parallelism.

```cpp
#include <iostream>
#include <future>
#include <numeric>
#include <vector>

long long sum_range(int start, int end) {
    long long total = 0;
    for (int i = start; i <= end; i++) total += i;
    return total;
}

int main() {
    // Launch two tasks in parallel
    auto f1 = std::async(std::launch::async, sum_range, 1,    500000);
    auto f2 = std::async(std::launch::async, sum_range, 500001, 1000000);

    // Main thread can do other work here...

    long long result = f1.get() + f2.get();  // Block until both finish
    std::cout << "Sum 1 to 1000000: " << result << "\n";  // 500000500000
}
```

**Flow:**

```
  main thread
      │
      ├── async(sum_range, 1, 500000)     → spawns thread, returns future f1
      │                                      [background thread: summing 1..500000]
      ├── async(sum_range, 500001, 1000000)→ spawns thread, returns future f2
      │                                      [background thread: summing 500001..1000000]
      │
      ├── f1.get() ─────────── blocks until first half done ──────────► 125000250000
      ├── f2.get() ─────────── blocks until second half done ─────────► 375000250000
      │
      └── total = 500000500000
```

---

## 9. Thread Pool Pattern

Creating and destroying threads is expensive. A thread pool reuses a fixed set of worker threads for many tasks.

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <queue>
#include <functional>
#include <mutex>
#include <condition_variable>

class ThreadPool {
public:
    ThreadPool(int num_threads) : stop(false) {
        for (int i = 0; i < num_threads; i++) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex);
                        // Wait until a task arrives or pool is stopped
                        condition.wait(lock, [this] {
                            return stop || !tasks.empty();
                        });
                        if (stop && tasks.empty()) return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();   // Execute the task
                }
            });
        }
    }

    void enqueue(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            tasks.push(task);
        }
        condition.notify_one();   // Wake one sleeping worker
    }

    ~ThreadPool() {
        { std::lock_guard<std::mutex> lock(queue_mutex); stop = true; }
        condition.notify_all();
        for (auto& t : workers) t.join();
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};

int main() {
    ThreadPool pool(4);   // 4 worker threads

    for (int i = 0; i < 10; i++) {
        pool.enqueue([i] {
            std::cout << "Task " << i << " on thread "
                      << std::this_thread::get_id() << "\n";
        });
    }

    // Pool destructor waits for all tasks to finish
}
```

**Architecture:**

```
  ┌─────────────────────────────────────────────────────┐
  │                     THREAD POOL                     │
  │                                                     │
  │  Task Queue:  [Task5][Task4][Task3][Task2][Task1]   │
  │                  │     │     │     │     │          │
  │  Worker 0: ──────┘     │     │     │     │          │
  │  Worker 1: ────────────┘     │     │     │          │
  │  Worker 2: ──────────────────┘     │     │          │
  │  Worker 3: ────────────────────────┘     │          │
  │                                    (waiting)        │
  └─────────────────────────────────────────────────────┘

  New tasks go to queue → idle workers pick them up
```

---

## 10. Code Examples

> Working code that demonstrates C++ thread creation, lifecycle, and a thread pool pattern in practice.

### C++ — Simple Version
Create 5 threads, each prints its ID, join all — demonstrates the full thread lifecycle: create → run → join → terminate.

```cpp
#include <iostream>
#include <thread>
#include <vector>

// This function runs on each worker thread
void thread_job(int thread_id) {
    std::cout << "Thread " << thread_id << " is running\n";
    // When this function returns, the thread terminates automatically
}

int main() {
    const int N = 5;
    std::vector<std::thread> threads;

    // LIFECYCLE STEP 1: Create threads — each starts running immediately
    std::cout << "Creating " << N << " threads...\n";
    for (int i = 0; i < N; i++) {
        threads.emplace_back(thread_job, i);   // thread_job(i) on a new OS thread
    }

    // LIFECYCLE STEP 2: Join — main thread blocks until every thread finishes
    std::cout << "Waiting for threads...\n";
    for (auto& t : threads) {
        t.join();   // Must join (or detach) before the std::thread object is destroyed
    }

    // LIFECYCLE STEP 3: All threads terminated — safe to access shared state
    std::cout << "All " << N << " threads done!\n";
    return 0;
}
// Compile: g++ -std=c++17 -pthread thread_lifecycle.cpp -o thread_lifecycle
// Note: output lines appear in non-deterministic order (OS schedules threads freely)
```

### C++ — Medium / LeetCode Style
Thread pool with 4 workers: `std::queue` holds tasks, `std::mutex` protects the queue, `std::condition_variable` wakes sleeping workers — the classic interview implementation.

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <queue>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <chrono>

class ThreadPool {
public:
    explicit ThreadPool(int n) : running_(true) {
        for (int i = 0; i < n; i++)
            workers_.emplace_back([this]{ worker_loop(); });
    }

    // Submit a task — thread-safe, can be called from any thread
    void submit(std::function<void()> task) {
        { std::lock_guard<std::mutex> lk(mu_); queue_.push(std::move(task)); }
        cv_.notify_one();   // Wake one sleeping worker
    }

    ~ThreadPool() {
        { std::lock_guard<std::mutex> lk(mu_); running_ = false; }
        cv_.notify_all();                     // Wake all workers so they can exit
        for (auto& t : workers_) t.join();    // Wait for all to finish
    }

private:
    void worker_loop() {
        while (true) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lk(mu_);
                cv_.wait(lk, [this]{ return !queue_.empty() || !running_; });
                if (!running_ && queue_.empty()) return;
                task = std::move(queue_.front());
                queue_.pop();
            }
            task();   // Execute the task OUTSIDE the lock
        }
    }

    std::vector<std::thread>          workers_;
    std::queue<std::function<void()>> queue_;
    std::mutex                         mu_;
    std::condition_variable            cv_;
    bool                               running_;
};

int main() {
    ThreadPool pool(4);           // 4 worker threads
    std::atomic<int> done{0};

    // Submit 10 tasks — pool distributes them across 4 workers
    for (int i = 0; i < 10; i++) {
        pool.submit([i, &done]{
            std::cout << "Task " << i << " on thread "
                      << std::this_thread::get_id() << "\n";
            ++done;
        });
    }

    std::this_thread::sleep_for(std::chrono::milliseconds(200));
    std::cout << done << "/10 tasks completed\n";
    return 0;   // Pool destructor joins all workers when it goes out of scope
}
// Compile: g++ -std=c++17 -pthread thread_pool.cpp -o thread_pool
```

### Python — Simple Version
Create N threads with `threading.Thread`; prove GIL impact — I/O-bound tasks run concurrently (fast), CPU-bound tasks do not speed up with more threads.

```python
import threading
import time

def io_task(thread_id):
    """I/O-bound: sleep releases the GIL so other threads run Python code."""
    time.sleep(0.5)
    print(f"Thread {thread_id}: I/O done")

def cpu_task(n):
    """CPU-bound: GIL is held — only one thread executes Python bytecode at a time."""
    return sum(i * i for i in range(n))

N, WORK = 5, 3_000_000

# ── I/O-bound: threading works great ──────────────────────────────────
print("=== I/O-Bound (threading works well) ===")
start = time.time()
threads = [threading.Thread(target=io_task, args=(i,)) for i in range(N)]
for t in threads: t.start()
for t in threads: t.join()
print(f"  {N} threads × 0.5s → {time.time()-start:.2f}s  (expected ~0.5s)\n")

# ── CPU-bound: GIL prevents true parallelism ──────────────────────────
print("=== CPU-Bound (GIL prevents speedup) ===")
start = time.time()
cpu_task(WORK * N)                    # Sequential baseline
seq = time.time() - start
print(f"  Sequential:   {seq:.2f}s")

start = time.time()
threads = [threading.Thread(target=cpu_task, args=(WORK,)) for _ in range(N)]
for t in threads: t.start()
for t in threads: t.join()
print(f"  {N} threads:   {time.time()-start:.2f}s  (GIL → similar or slower than sequential!)")
print("  → Use multiprocessing.Pool for CPU-bound parallelism")
```

### Python — Medium Level
`ThreadPoolExecutor` for I/O-bound, `ProcessPoolExecutor` for CPU-bound, and `asyncio` for massive single-threaded concurrency — see which model wins for each task type.

```python
import time
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_work(n):
    return sum(i * i for i in range(n))

def io_work(delay):
    import time; time.sleep(delay); return delay

async def async_io(delay):
    await asyncio.sleep(delay)   # Suspend coroutine, event loop runs others
    return delay

TASKS, N = 4, 2_000_000

def bench(label, fn):
    t = time.time(); fn()
    print(f"  {label:<52} {time.time()-t:.2f}s")

if __name__ == "__main__":
    work, delays = [N] * TASKS, [0.5] * TASKS

    print("CPU-BOUND")
    bench("Sequential:", lambda: [cpu_work(n) for n in work])
    with ThreadPoolExecutor(TASKS) as ex:
        bench("ThreadPoolExecutor (GIL blocked):", lambda: list(ex.map(cpu_work, work)))
    with ProcessPoolExecutor(TASKS) as ex:
        bench("ProcessPoolExecutor (no GIL):",     lambda: list(ex.map(cpu_work, work)))

    print("\nI/O-BOUND")
    bench("Sequential:", lambda: [io_work(d) for d in delays])
    with ThreadPoolExecutor(TASKS) as ex:
        bench("ThreadPoolExecutor:",               lambda: list(ex.map(io_work, delays)))
    bench("asyncio (single thread, cooperative):",
          lambda: asyncio.run(asyncio.gather(*[async_io(d) for d in delays])))

    print("\nWinner: CPU → ProcessPoolExecutor | I/O → asyncio or ThreadPoolExecutor")
```

---

## 11. Key Takeaways

- `std::thread t(func, args...)` — create a thread; always call `t.join()` or `t.detach()`
- `std::mutex` + `lock()`/`unlock()` — basic mutual exclusion; use RAII wrappers to avoid forgetting unlock
- `std::lock_guard` — simplest RAII lock; auto-releases when scope ends
- `std::unique_lock` — flexible RAII lock; supports early unlock and use with condition variables
- `std::condition_variable` + `cv.wait(lock, pred)` — sleep until a condition is true; `notify_one()` / `notify_all()` to wake waiters
- `std::atomic<T>` — lock-free thread-safe operations on single variables; faster than mutex for counters and flags
- `std::async` + `std::future` — task-based parallelism; get results with `future.get()`
- **Thread pool** — reuse N worker threads for many tasks; avoids thread creation/destruction overhead
- C++ has **no GIL** — all threads can truly run in parallel on multiple cores
- **Build flag:** compile with `g++ -std=c++17 -pthread your_file.cpp` to enable threading support
