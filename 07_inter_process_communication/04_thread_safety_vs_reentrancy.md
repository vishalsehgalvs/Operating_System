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
