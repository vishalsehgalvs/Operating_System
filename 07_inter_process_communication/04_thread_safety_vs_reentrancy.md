# Thread Safety vs Reentrancy

> **Thread safety** means a function works correctly when multiple threads call it at the same time — typically achieved with locks protecting shared data. **Reentrancy** is a stronger guarantee: a function can be safely interrupted mid-execution and called again (even on the same thread) because it uses ONLY local/parameter data and no shared state at all.

---

## Table of Contents

1. [What is Thread Safety?](#1-what-is-thread-safety)
2. [What is Reentrancy?](#2-what-is-reentrancy)
3. [Thread Safety vs Reentrancy: The Key Difference](#3-thread-safety-vs-reentrancy-the-key-difference)
4. [Common Thread Safety Issues](#4-common-thread-safety-issues)
5. [Writing Reentrant Functions](#5-writing-reentrant-functions)
6. [Practical Scenarios](#6-practical-scenarios)
7. [Best Practices](#7-best-practices)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What is Thread Safety?

A function (or data structure) is **thread-safe** if multiple threads can call it simultaneously and it always produces correct results — no data corruption, no lost updates, no race conditions.

**Analogy:** A shared bathroom with a lock. Multiple people can use the bathroom, but the lock guarantees only one uses it at a time. Everything stays clean and organized because access is controlled.

### Thread-Unsafe Counter (Race Condition)

```c
int counter = 0;  // shared global

void increment_counter() {
    int temp = counter;   // Step 1: read
    temp = temp + 1;      // Step 2: add
    counter = temp;       // Step 3: write back
}
```

**What goes wrong with two threads:**

```
  Thread A reads counter = 0
  Thread B reads counter = 0    ← B reads BEFORE A writes back!
  Thread A computes 0+1 = 1, writes counter = 1
  Thread B computes 0+1 = 1, writes counter = 1

  Expected result: 2
  Actual result:   1  ← ONE INCREMENT LOST

  This is a race condition: outcome depends on timing of thread scheduling.
```

### Thread-Safe Counter (Fixed with Mutex)

```c
int counter = 0;
mutex counter_lock;

void increment_counter() {
    lock(counter_lock);      // Block other threads here
    counter = counter + 1;   // Only one thread here at a time
    unlock(counter_lock);    // Let next thread proceed
}
```

```
  Thread A acquires lock → increments counter to 1 → releases lock
  Thread B was BLOCKED → now acquires lock → increments to 2 → releases

  Result: 2 ✓ correct regardless of timing
```

---

## 2. What is Reentrancy?

A function is **reentrant** if it can be interrupted mid-execution and called again — either:

1. From a signal handler that interrupts the current call
2. Recursively (from itself)
3. From another thread

…and all invocations work correctly without interfering with each other.

**The rules for a reentrant function:**

1. Uses ONLY local variables and parameters (no globals, no static variables)
2. Does not modify non-constant global data
3. Does not call non-reentrant functions
4. Does not use shared resources without restoring them

**Analogy:** You're reading a book and someone interrupts you to ask about another page in the same book. You put your finger on your current page (save state on the stack), look up the other page, answer the question, and return to your bookmark. The book can handle being "read at two places simultaneously" because each reader has their own bookmark (stack frame).

### Non-Reentrant Function (Static Buffer)

```c
// NOT REENTRANT — static buffer is shared across ALL calls
char* get_error_message(int error_code) {
    static char error_buffer[100];  // ← ONE buffer for ALL calls!

    if (error_code == 1)
        strcpy(error_buffer, "File not found");
    else if (error_code == 2)
        strcpy(error_buffer, "Access denied");

    return error_buffer;  // returns pointer to static buffer
}

// What goes wrong:
// Main program: ptr = get_error_message(1);  // buffer = "File not found"
// Signal arrives! Handler calls: ptr2 = get_error_message(2);  // buffer = "Access denied"
// Back to main: printf("%s", ptr);  // prints "Access denied" — WRONG!
// The signal handler clobbered the static buffer!
```

### Reentrant Version (Caller Provides Buffer)

```c
// REENTRANT — each caller provides its OWN buffer
void get_error_message(int error_code, char* buffer, int buffer_size) {
    if (error_code == 1)
        strncpy(buffer, "File not found", buffer_size);
    else if (error_code == 2)
        strncpy(buffer, "Access denied", buffer_size);

    buffer[buffer_size - 1] = '\0';
    // Returns nothing — each caller owns their own copy
}

// Now:
char buf1[100]; get_error_message(1, buf1, 100);  // "File not found"
// Signal! Handler:
char buf2[100]; get_error_message(2, buf2, 100);  // "Access denied"
// Return to main: buf1 is still "File not found" — CORRECT!
```

---

## 3. Thread Safety vs Reentrancy: The Key Difference

| Aspect                       | Thread-Safe                                   | Reentrant                       |
| ---------------------------- | --------------------------------------------- | ------------------------------- |
| **Can use shared data?**     | Yes (with locks)                              | No — only local/parameter data  |
| **Uses synchronization?**    | Yes (mutex, semaphore)                        | No locks needed                 |
| **Performance**              | Has locking overhead                          | Faster (no locks)               |
| **Safe in signal handlers?** | Maybe not (lock might deadlock)               | Yes (no locks, no shared state) |
| **Safe for recursion?**      | Maybe not (re-acquiring same lock = deadlock) | Yes                             |
| **Stronger property?**       | Weaker                                        | Stronger                        |

**The key insight:**

```
  Reentrant ⊂ Thread-Safe

  Reentrant → Thread-Safe  (reentrant functions are automatically thread-safe)
  Thread-Safe ↛ Reentrant  (thread-safe functions are NOT necessarily reentrant)

  Example: Function using mutex to protect static data
  → Thread-safe: multiple threads safely share the data
  → NOT reentrant: if interrupted and re-entered, the mutex deadlocks!
```

**Concrete example — thread-safe but NOT reentrant:**

```c
static int cached_result = -1;
mutex cache_lock;

int compute_with_cache(int input) {
    lock(cache_lock);

    if (cached_result == -1) {
        cached_result = expensive_compute(input);
    }
    int result = cached_result;

    unlock(cache_lock);
    return result;
}

// Thread-safe: multiple threads get correct result (mutex protects)
// NOT reentrant: if a signal fires WHILE holding cache_lock,
//               and signal handler calls compute_with_cache() again,
//               it tries to lock cache_lock — DEADLOCK!
```

---

## 4. Common Thread Safety Issues

### Race Condition (Shared Write)

```c
int balance = 1000;  // shared

void withdraw(int amount) {
    if (balance >= amount) {           // Thread A checks: 1000 >= 600 ✓
        // ← Thread B also checks: 1000 >= 600 ✓ (before A writes)
        balance = balance - amount;    // Thread A: balance = 400
                                       // Thread B: balance = 400 (should be -200!)
        printf("Withdrawn %d\n", amount);
    }
}
// Both threads withdraw 600, but balance should be 400 → 400 → crash/overdraft
```

**Fix:** Wrap the entire check-and-update in a mutex lock.

### Data Corruption (Partial Update)

```c
struct user_data {
    char name[50];
    int age;
    char email[100];
};

struct user_data shared_user;

void update_user(char* name, int age, char* email) {
    strcpy(shared_user.name, name);  // Thread A is HERE
    // ← Thread B reads user — gets new name but OLD age and OLD email!
    shared_user.age = age;
    strcpy(shared_user.email, email);
}
```

**Fix:** Protect the entire struct update with a single lock.

### Deadlock from Inconsistent Lock Order

```c
mutex lock_a, lock_b;

void thread_1() {
    lock(lock_a);         // holds A, wants B
    lock(lock_b);
    // ...
    unlock(lock_b);
    unlock(lock_a);
}

void thread_2() {
    lock(lock_b);         // holds B, wants A
    lock(lock_a);         // ← DEADLOCK: B waits for A, A waits for B
    // ...
}
```

**Fix:** Always acquire locks in a CONSISTENT global order (e.g., always lock A before B).

---

## 5. Writing Reentrant Functions

### Rule 1: No Global or Static Variables for Mutable State

```c
// BAD — static result shared across calls
int factorial_bad(int n) {
    static int result = 1;   // mutable static = NOT reentrant
    if (n <= 1) return result;
    result = n * factorial_bad(n - 1);
    return result;
}

// GOOD — only parameters and local stack variables
int factorial_good(int n) {
    if (n <= 1) return 1;
    return n * factorial_good(n - 1);  // stack frame per call = safe
}
```

### Rule 2: Use Caller-Provided Buffers Instead of Internal Static Ones

```c
// BAD — internal static buffer (common in old C library functions)
char* format_time_bad(int hours, int minutes) {
    static char buffer[20];           // one buffer, overwritten on each call
    sprintf(buffer, "%02d:%02d", hours, minutes);
    return buffer;
}

// GOOD — caller provides their own storage
void format_time_good(int hours, int minutes, char* buffer, int size) {
    snprintf(buffer, size, "%02d:%02d", hours, minutes);
}
```

### Rule 3: Use Reentrant Standard Library Versions

| Non-reentrant (avoid) | Reentrant alternative |
| --------------------- | --------------------- |
| `strtok()`            | `strtok_r()`          |
| `rand()`              | `rand_r()`            |
| `strerror()`          | `strerror_r()`        |
| `gmtime()`            | `gmtime_r()`          |
| `localtime()`         | `localtime_r()`       |

Pattern: Many POSIX functions have an `_r` suffix version that takes a caller-provided state pointer instead of using internal static storage.

---

## 6. Practical Scenarios

### Signal Handlers Must Use Reentrant Functions

```c
// UNSAFE — printf and malloc are NOT reentrant/async-signal-safe
void bad_handler(int sig) {
    printf("Signal %d received\n", sig);  // may deadlock if interrupted mid-printf
    char* p = malloc(100);                // may corrupt heap
}

// SAFE — only async-signal-safe functions + volatile flag
volatile sig_atomic_t signal_count = 0;

void good_handler(int sig) {
    signal_count++;                                   // sig_atomic_t is safe
    write(STDOUT_FILENO, "Signal!\n", 8);             // write() IS safe
    // Nothing else — main loop checks signal_count and acts
}
```

**Rule:** Signal handlers MUST be reentrant (or at minimum only call async-signal-safe functions). Thread-safety with locks is NOT sufficient here.

### Multi-Threaded Web Server

```c
// Shared response cache (thread-safe with mutex)
struct cache {
    char data[1000][100];
    int count;
};

struct cache global_cache;
mutex cache_lock;

void add_to_cache(char* item) {
    lock(cache_lock);
    if (global_cache.count < 1000) {
        strcpy(global_cache.data[global_cache.count], item);
        global_cache.count++;
    }
    unlock(cache_lock);
}
// Thread-safe: multiple request threads safely add to cache
// NOT reentrant: uses shared state (but that's acceptable here — no signal interrupts)
```

### Recursive Parsing (Needs Reentrancy)

```c
// Parsing nested expressions like: 3 + (4 * (2 + 1))
// parse_expr() calls itself recursively — MUST be reentrant

int parse_expr(const char** pos) {
    int left = parse_term(pos);  // all state on stack — safe to recurse
    while (**pos == '+' || **pos == '-') {
        char op = *(*pos)++;
        int right = parse_term(pos);
        left = (op == '+') ? left + right : left - right;
    }
    return left;
}
// No static/global state → safe for recursion → reentrant
```

---

## 7. Best Practices

| Practice                                      | Why                                                                                                 |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Minimize shared state**                     | Less sharing → fewer synchronization problems                                                       |
| **Use thread-local storage**                  | Each thread gets its own copy — no sharing needed                                                   |
| **Keep locks fine-grained**                   | Smaller lock scope → less contention, better performance                                            |
| **Consistent lock ordering**                  | Prevents deadlocks                                                                                  |
| **Use atomic operations for simple counters** | `atomic_int` in C11/C++11; no lock overhead                                                         |
| **Prefer immutable data**                     | Read-only data is inherently thread-safe and reentrant                                              |
| **Document thread-safety level**              | State clearly whether function is reentrant, thread-safe, or neither                                |
| **Test with multiple threads**                | Race conditions often don't appear in single-threaded tests; use tools like TSan (Thread Sanitizer) |

---

## 7. Code Examples

> Working code that demonstrates thread safety and reentrancy in practice.

### C++ — Simple Version
Non-thread-safe function uses a shared global buffer; thread-safe version uses local variables — run both with multiple threads and observe the difference.

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <vector>
#include <string>
#include <cstdio>
#include <chrono>

// ── NON-THREAD-SAFE: uses a GLOBAL static buffer ─────────────────────────────
// All threads share the same buffer → one thread overwrites while another reads
char global_buf[64];   // shared by ALL threads — danger zone!

std::string format_unsafe(int value) {
    snprintf(global_buf, sizeof(global_buf), "value=%d", value);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));  // simulate work
    return std::string(global_buf);   // another thread may have clobbered this!
}

// ── THREAD-SAFE: uses LOCAL variables only ────────────────────────────────────
// Each thread call gets its own stack frame → no sharing → no race condition

std::string format_safe(int value) {
    char local_buf[64];   // lives on THIS thread's stack — not shared
    snprintf(local_buf, sizeof(local_buf), "value=%d", value);
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    return std::string(local_buf);   // always correct — it's ours alone
}

std::mutex print_lock;   // only for clean console output

void run_unsafe(int id) {
    std::string result = format_unsafe(id);
    std::lock_guard<std::mutex> g(print_lock);
    std::string expected = "value=" + std::to_string(id);
    std::cout << "  Thread " << id << " unsafe: " << result
              << (result != expected ? "  ← CORRUPTED!" : "  ✓") << "\n";
}

void run_safe(int id) {
    std::string result = format_safe(id);
    std::lock_guard<std::mutex> g(print_lock);
    std::cout << "  Thread " << id << " safe:   " << result << "  ✓\n";
}

int main() {
    constexpr int N = 6;

    std::cout << "=== Non-thread-safe (global buffer) ===\n";
    std::cout << "Expected: each thread sees its own value. Actual:\n";
    std::vector<std::thread> unsafe_threads;
    for (int i = 1; i <= N; i++) unsafe_threads.emplace_back(run_unsafe, i);
    for (auto& t : unsafe_threads) t.join();

    std::cout << "\n=== Thread-safe (local buffer) ===\n";
    std::cout << "Each thread uses its own stack — always correct:\n";
    std::vector<std::thread> safe_threads;
    for (int i = 1; i <= N; i++) safe_threads.emplace_back(run_safe, i);
    for (auto& t : safe_threads) t.join();

    std::cout << "\nConclusion: same logic, different storage → completely different behavior.\n";
    return 0;
}
```

### C++ — Medium / LeetCode Style
Thread-safe singleton, thread-safe stack, and reentrant vs non-reentrant tokenizer with nested-call demonstration.

```cpp
#include <iostream>
#include <mutex>
#include <stack>
#include <thread>
#include <vector>
#include <string>

// ── 1. Thread-safe Singleton (Meyer's Singleton — C++11 guaranteed) ───────────
class Config {
    Config() { std::cout << "  [Config] instance created exactly once\n"; }
public:
    static Config& instance() {
        static Config inst;   // C++11: initialized exactly once even with races
        return inst;
    }
    std::string get(const std::string& key) const { return "val_" + key; }
    Config(const Config&) = delete;
    Config& operator=(const Config&) = delete;
};

void use_config(int tid) {
    Config& cfg = Config::instance();   // all threads get the SAME object
    std::cout << "  Thread " << tid << ": " << cfg.get("timeout") << "\n";
}

// ── 2. Thread-safe Stack ──────────────────────────────────────────────────────
template<typename T>
class ThreadSafeStack {
    std::stack<T> data;
    mutable std::mutex mtx;
public:
    void push(T val) {
        std::lock_guard<std::mutex> lock(mtx);
        data.push(std::move(val));
    }
    bool pop(T& out) {
        std::lock_guard<std::mutex> lock(mtx);
        if (data.empty()) return false;
        out = std::move(data.top());
        data.pop();
        return true;
    }
    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx);
        return data.empty();
    }
};

// ── 3a. NON-REENTRANT tokenizer (like C's strtok) ─────────────────────────────
// Uses a static variable → a nested call OVERWRITES the saved state

char* strtok_nonreentrant(char* str, char delim) {
    static char* saved = nullptr;   // STATIC: shared by all callers — danger!
    if (str) saved = str;
    if (!saved || *saved == '\0') return nullptr;
    char* start = saved;
    while (*saved && *saved != delim) saved++;
    if (*saved) { *saved = '\0'; saved++; } else { saved = nullptr; }
    return start;
}

// ── 3b. REENTRANT tokenizer (like POSIX strtok_r) ─────────────────────────────
// Caller provides saveptr → each call site has its OWN state → safe to nest

char* strtok_reentrant(char* str, char delim, char** saveptr) {
    if (str) *saveptr = str;        // caller-owned state — no hidden globals
    if (!*saveptr || **saveptr == '\0') return nullptr;
    char* start = *saveptr;
    while (**saveptr && **saveptr != delim) (*saveptr)++;
    if (**saveptr) { **saveptr = '\0'; (*saveptr)++; } else { *saveptr = nullptr; }
    return start;
}

int main() {
    // 1. Singleton
    std::cout << "=== 1. Thread-safe Singleton ===\n";
    std::vector<std::thread> cfg_threads;
    for (int i = 1; i <= 4; i++) cfg_threads.emplace_back(use_config, i);
    for (auto& t : cfg_threads) t.join();
    std::cout << "  (Created only once no matter how many threads raced)\n";

    // 2. Thread-safe stack
    std::cout << "\n=== 2. Thread-safe Stack ===\n";
    ThreadSafeStack<int> ts;
    auto pusher = [&](int base) { for (int i = base; i < base + 3; i++) ts.push(i); };
    std::thread p1(pusher, 10), p2(pusher, 20);
    p1.join(); p2.join();
    int val;
    std::cout << "  Popped:";
    while (ts.pop(val)) std::cout << " " << val;
    std::cout << "\n";

    // 3. Tokenizer reentrancy
    std::cout << "\n=== 3a. Non-reentrant tokenizer (nested call corrupts outer) ===\n";
    char s1[] = "a,b,c";
    char* tok = strtok_nonreentrant(s1, ',');
    while (tok) {
        std::cout << "  outer: " << tok << "\n";
        char tmp[] = "x,y";
        strtok_nonreentrant(tmp, ',');  // nested call OVERWRITES static saveptr
        tok = strtok_nonreentrant(nullptr, ',');  // outer iteration now broken
    }

    std::cout << "\n=== 3b. Reentrant tokenizer (nested call is safe) ===\n";
    char s2[] = "a,b,c";
    char *outer_save = nullptr, *inner_save = nullptr;
    tok = strtok_reentrant(s2, ',', &outer_save);
    while (tok) {
        std::cout << "  outer: " << tok << "\n";
        char tmp[] = "x,y";
        char* inner = strtok_reentrant(tmp, ',', &inner_save);
        while (inner) inner = strtok_reentrant(nullptr, ',', &inner_save);
        tok = strtok_reentrant(nullptr, ',', &outer_save);  // outer continues correctly
    }

    return 0;
}
```

### Python — Simple Version
Non-thread-safe function uses a module-level global; thread-safe version uses only local variables — corruption is visible with enough threads.

```python
import threading
import time

# ── NON-THREAD-SAFE: function uses a MODULE-LEVEL (shared) variable ───────────
# All threads read/modify the same global → race conditions guaranteed

_global_result = ""   # shared by ALL threads — danger!

def format_unsafe(value: int) -> str:
    global _global_result
    _global_result = f"value={value}"   # WRITE to shared global
    time.sleep(0.001)                   # another thread can overwrite here
    return _global_result               # may see another thread's value!

# ── THREAD-SAFE: function uses ONLY local variables ───────────────────────────
# Each call gets its own stack frame — nothing is shared between threads

def format_safe(value: int) -> str:
    local_result = f"value={value}"     # lives only in this call's scope
    time.sleep(0.001)
    return local_result                 # always correct — nothing else touches it

print_lock = threading.Lock()

def run_unsafe(thread_id: int, results: list):
    result   = format_unsafe(thread_id)
    expected = f"value={thread_id}"
    with print_lock:
        status = "✓" if result == expected else f"← CORRUPTED! (got '{result}')"
        print(f"  Thread {thread_id} unsafe: {result}  {status}")
        results.append(result == expected)

def run_safe(thread_id: int):
    result = format_safe(thread_id)
    with print_lock:
        print(f"  Thread {thread_id} safe:   {result}  ✓")

print("=== Non-thread-safe (global variable) ===")
print("Expected: each thread sees its own value. Actual:")
results = []
unsafe = [threading.Thread(target=run_unsafe, args=(i, results)) for i in range(1, 7)]
for t in unsafe: t.start()
for t in unsafe: t.join()
print(f"  Corruptions: {results.count(False)}/6\n")

print("=== Thread-safe (local variables only) ===")
safe = [threading.Thread(target=run_safe, args=(i,)) for i in range(1, 7)]
for t in safe: t.start()
for t in safe: t.join()

print("\nConclusion: global state → races; local state → thread-safe.")
```

### Python — Medium Level
Thread-safe stack with a mutex, reentrant vs non-reentrant tokenizer with nested-call demonstration, and a reentrant recursive expression evaluator.

```python
import threading
from typing import TypeVar, Generic

T = TypeVar("T")

# ── 1. Thread-safe Stack ──────────────────────────────────────────────────────
class ThreadSafeStack(Generic[T]):
    """Stack protected by a mutex — safe to push/pop from multiple threads."""

    def __init__(self):
        self._data: list[T] = []
        self._lock = threading.Lock()

    def push(self, item: T) -> None:
        with self._lock:        # acquire lock before touching shared data
            self._data.append(item)

    def pop(self) -> T | None:
        with self._lock:
            return self._data.pop() if self._data else None

    def size(self) -> int:
        with self._lock:
            return len(self._data)

# ── 2a. NON-REENTRANT tokenizer: uses module-level shared state ───────────────
_saved_str = ""
_saved_pos = 0

def strtok_nr(s: str | None, sep: str) -> str | None:
    """Like C's strtok — not reentrant because state is hidden in globals."""
    global _saved_str, _saved_pos
    if s is not None:
        _saved_str = s    # nested call will OVERWRITE this!
        _saved_pos = 0
    while _saved_pos < len(_saved_str) and _saved_str[_saved_pos] == sep:
        _saved_pos += 1
    if _saved_pos >= len(_saved_str):
        return None
    start = _saved_pos
    while _saved_pos < len(_saved_str) and _saved_str[_saved_pos] != sep:
        _saved_pos += 1
    return _saved_str[start:_saved_pos]

# ── 2b. REENTRANT tokenizer: all state provided by caller ────────────────────
def strtok_r(s: str | None, sep: str, state: list) -> str | None:
    """Reentrant: caller owns the state list [string, pos] — no hidden globals."""
    if s is not None:
        state[0], state[1] = s, 0    # caller stores their own state
    src, pos = state[0], state[1]
    while pos < len(src) and src[pos] == sep:
        pos += 1
    if pos >= len(src):
        return None
    start = pos
    while pos < len(src) and src[pos] != sep:
        pos += 1
    state[1] = pos
    return src[start:pos]

# ── 3. Reentrant recursive expression evaluator ───────────────────────────────
def eval_expr(expr: str) -> int:
    """
    Reentrant: uses ONLY local variables — safe for recursion and concurrent calls.
    Handles simple expressions like "3+5-2", "10+3-1+2".
    """
    tokens: list[int | str] = []    # LOCAL — not shared with any other call
    i = 0
    while i < len(expr):
        if expr[i].isdigit():
            j = i
            while j < len(expr) and expr[j].isdigit():
                j += 1
            tokens.append(int(expr[i:j]))
            i = j
        else:
            tokens.append(expr[i])
            i += 1
    result = tokens[0]              # LOCAL result — each call has its own
    idx = 1
    while idx < len(tokens):
        op  = tokens[idx]
        num = tokens[idx + 1]
        result = result + num if op == '+' else result - num
        idx += 2
    return result

# ── Demo ───────────────────────────────────────────────────────────────────────
print("=== 1. Thread-safe Stack ===")
stack: ThreadSafeStack[int] = ThreadSafeStack()
def pusher(base: int):
    for i in range(base, base + 3):
        stack.push(i)
threads = [threading.Thread(target=pusher, args=(b,)) for b in [10, 20, 30]]
for t in threads: t.start()
for t in threads: t.join()
print(f"  Stack size after 9 pushes from 3 threads: {stack.size()} (expected 9)")

print("\n=== 2a. Non-reentrant tokenizer ===")
tok = strtok_nr("a,b,c", ",")
while tok:
    print(f"  outer: {tok}")
    strtok_nr("x,y,z", ",")    # nested call OVERWRITES global state
    tok = strtok_nr(None, ",") # outer iteration is now hijacked
print("  (outer was corrupted by nested call)")

print("\n=== 2b. Reentrant tokenizer ===")
outer_state, inner_state = [None, 0], [None, 0]
tok = strtok_r("a,b,c", ",", outer_state)
while tok:
    print(f"  outer: {tok}")
    inner = strtok_r("x,y,z", ",", inner_state)
    while inner:
        inner = strtok_r(None, ",", inner_state)   # uses its OWN state
    tok = strtok_r(None, ",", outer_state)         # outer unaffected
print("  (outer continued correctly — nested call used its own state)")

print("\n=== 3. Reentrant expression evaluator ===")
for e in ["3+5", "10+3-1", "100-50+25-10"]:
    print(f"  eval('{e}') = {eval_expr(e)}")
print("  (all local variables → safe for recursion and concurrent calls)")
```

---

## 8. Key Takeaways

- **Thread safety:** function works correctly when called by multiple threads simultaneously — typically achieved with mutexes protecting shared data
- **Race condition:** two threads read/modify shared data without synchronization → outcome depends on scheduling order → data corruption
- **Reentrancy:** function can be safely interrupted and re-entered before its previous call completes — achieved by using ONLY local/parameter data, no shared state
- **Reentrant → Thread-safe** (always); **Thread-safe does NOT imply reentrant** (can still deadlock in recursive/signal contexts)
- **Key rule for reentrant functions:** no global/static mutable variables, no non-reentrant function calls, caller provides all storage
- **Signal handlers must be reentrant** — they interrupt at random points; calling non-reentrant functions (like `printf`, `malloc`) from a handler can deadlock or corrupt data
- **Common non-reentrant functions:** `strtok`, `rand`, `strerror`, `gmtime` — all have `_r` reentrant versions
- **Deadlocks in thread-safe code:** occur when locks are acquired in different orders across threads — always acquire locks in a consistent global order
- **`volatile sig_atomic_t`** is the correct type for flags shared between main code and signal handlers
- **When designing:**
  - If only concurrent thread access → thread safety (mutex) is sufficient
  - If signal handlers or recursion → need full reentrancy (no shared state at all)
