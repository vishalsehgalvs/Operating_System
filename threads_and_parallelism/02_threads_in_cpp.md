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
10. [Key Takeaways](#10-key-takeaways)

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

## 10. Key Takeaways

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
