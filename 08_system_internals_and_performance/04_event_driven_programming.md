# Event-Driven Programming in Operating Systems

> **Event-driven programming** is a paradigm where execution flow is controlled by events (key presses, timer ticks, network packets, disk completions) rather than a fixed sequence — the OS waits idle until an event arrives, dispatches it to the right handler, then waits again — this is how modern OSes stay responsive and efficient without burning CPU cycles.

---

## Table of Contents

1. [What is Event-Driven Programming?](#1-what-is-event-driven-programming)
2. [Key Components](#2-key-components)
3. [The Event Loop](#3-the-event-loop)
4. [Event Handlers and Callbacks](#4-event-handlers-and-callbacks)
5. [Event Queues and Priorities](#5-event-queues-and-priorities)
6. [Real-World Examples in OS](#6-real-world-examples-in-os)
7. [Event-Driven vs Polling](#7-event-driven-vs-polling)
8. [Advantages and Challenges](#8-advantages-and-challenges)
9. [Practical Design Considerations](#9-practical-design-considerations)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What is Event-Driven Programming?

**Event-driven programming** is a paradigm where the program's flow is determined by **events** — things that happen — rather than a pre-planned sequential execution.

```
  Traditional sequential program:
  main() {
    do_step_1();
    do_step_2();
    do_step_3();
    ...
  }

  Event-driven program:
  main() {
    register_handler(KEYBOARD_EVENT, on_key);
    register_handler(MOUSE_EVENT,    on_mouse);
    register_handler(TIMER_EVENT,    on_timer);
    register_handler(NETWORK_EVENT,  on_packet);

    event_loop();  // wait forever, dispatch events as they arrive
  }
  // Execution order is NOT predetermined — it depends on what events arrive
```

**Analogy:** A restaurant kitchen. Chefs don't cook random dishes non-stop. They wait for orders (events) to arrive, prepare the specific dish for each order (run handler), and go back to waiting. The kitchen responds to actual demand — not guessing what might be needed.

**Why OS needs event-driven design:**

- CPU runs at billions of operations/second
- User presses a key maybe 10 times/second
- Disk I/O completes a few times per second
- → If OS constantly checks for these, 99.99% of CPU time is wasted
- → Event-driven: CPU sleeps until interrupt/event arrives → handle it → sleep again

---

## 2. Key Components

| Component         | Role                                             | OS Example                                                |
| ----------------- | ------------------------------------------------ | --------------------------------------------------------- |
| **Event Source**  | Generates events                                 | Keyboard, timer chip, NIC, disk controller                |
| **Event**         | An occurrence that needs attention               | Key pressed, timer expired, packet arrived                |
| **Event Queue**   | Stores pending events until processed            | Kernel's interrupt pending queue, X11 event queue         |
| **Event Loop**    | Central dispatcher — reads queue, calls handlers | OS main scheduler loop, `select()`/`epoll()` in Linux     |
| **Event Handler** | Function that responds to a specific event       | ISR (Interrupt Service Routine), signal handler, callback |

```
  Flow:

  Event Source ──generates──► Event ──placed in──► Event Queue
                                                          ↓
                                                    Event Loop
                                                      reads it
                                                          ↓
                                                  Dispatch to Handler
                                                          ↓
                                                  Handler runs
                                                          ↓
                                             Loop waits for next event
```

---

## 3. The Event Loop

The **event loop** is the core of an event-driven OS subsystem. It continuously:

1. Waits for an event
2. Identifies the event type
3. Dispatches to the appropriate handler
4. Repeats

```c
// Simplified conceptual event loop (not actual kernel code)
while (system_is_running) {
    // Wait for an event (CPU can sleep in low-power state here)
    event = wait_for_event();    // blocks until interrupt/event

    if (event != NULL) {
        switch (get_event_type(event)) {
            case KEYBOARD_EVENT: handle_keyboard(event); break;
            case MOUSE_EVENT:    handle_mouse(event);    break;
            case TIMER_EVENT:    handle_timer(event);    break;
            case DISK_EVENT:     handle_disk_io(event);  break;
            case NETWORK_EVENT:  handle_network(event);  break;
            default:             handle_unknown(event);
        }
    }
}
```

**Key insight:** `wait_for_event()` **blocks** until something happens. The CPU is NOT spinning in a busy loop — it can enter a low-power C-state (sleep mode) until the next interrupt fires.

```
  CPU during idle event loop:
  [C0: active] → [C1: halt] → [C3: deeper sleep] → [interrupt arrives!]
  → [C0: wake and run handler] → [C3: back to sleep]

  This is why laptop battery lasts much longer when idle vs actively typing.
```

---

## 4. Event Handlers and Callbacks

**Event handlers** (callbacks) are functions registered to run when a specific event occurs. The event loop calls them — they're "called back" by the system.

```c
// Define event handlers
void keyboard_handler(Event* e) {
    char key = e->data.key_code;
    // Forward keypress to the currently active application window
    send_to_active_window(key);
}

void mouse_handler(Event* e) {
    int x = e->data.mouse_x;
    int y = e->data.mouse_y;
    update_cursor_position(x, y);     // move cursor on screen
    check_click_target(x, y);         // check if click hits a button/window
}

// Register handlers
register_event_handler(KEYBOARD_EVENT, keyboard_handler);
register_event_handler(MOUSE_EVENT,    mouse_handler);
```

### Synchronous vs Asynchronous Handlers

| Mode             | Behavior                                                                            | Use case                                     |
| ---------------- | ----------------------------------------------------------------------------------- | -------------------------------------------- |
| **Synchronous**  | Handler completes fully before event loop moves to next event                       | Simple, fast operations                      |
| **Asynchronous** | Handler starts an operation and returns immediately; completion fires another event | Slow operations (file save, network request) |

**Example — asynchronous file save:**

```
  Click "Save" button
  ↓
  handle_save_click() runs:
    1. Initiates async disk write (returns immediately!)
    2. Shows "Saving..." status bar
    3. Returns to event loop
  ↓
  Event loop continues handling other events (mouse moves, keystrokes)
  ↓
  Disk write completes → fires DISK_WRITE_COMPLETE event
  ↓
  handle_save_complete() runs:
    Updates status bar to "Saved ✓"

  User can keep working while the file saves in the background!
```

---

## 5. Event Queues and Priorities

Multiple events can arrive before being processed. The **event queue** buffers them. Priority determines which events get handled first.

| Queue Strategy        | Order                             | Best For                                |
| --------------------- | --------------------------------- | --------------------------------------- |
| **FIFO**              | First in, first out               | Events of equal importance              |
| **Priority Queue**    | Higher priority first             | Mix of critical and non-critical events |
| **Circular Queue**    | Ring buffer, fixed size           | Memory-constrained systems              |
| **Multi-level Queue** | Separate queue per priority level | Complex systems with many event types   |

**OS event priorities (approximate):**

```
  Highest priority: Hardware failures (NMI — power failure, ECC error)
                 ↓  Timer interrupt (preemptive scheduling)
                 ↓  Keyboard/mouse interrupts (interactive responsiveness)
                 ↓  Disk I/O completion (data ready for process)
                 ↓  Network packet arrival
                 ↓  Software timers
  Lowest:           Screen redraw, background tasks
```

**Event storm:** If events arrive faster than handlers can process them, the queue fills up. OS responses: drop new events, drop oldest events, or throttle event sources. Must be carefully designed to avoid cascading failures.

---

## 6. Real-World Examples in OS

### GUI Button Click (Two-Stage Async Pattern)

```c
// Stage 1: User clicks Save button
void handle_button_click(Event* e) {
    if (e->data.button->id == SAVE_BUTTON) {
        char* content = get_document_content();
        // Initiate async disk write
        queue_disk_write(filename, content, on_save_complete);
        update_status_bar("Saving...");
        // Return immediately — don't block the event loop!
    }
}

// Stage 2: Disk write finished (fires a new event)
void on_save_complete(Event* e) {
    if (e->data.success)
        update_status_bar("Saved ✓");
    else
        show_error_dialog("Save failed!");
}
```

### Network Server (Event-Driven I/O with epoll)

```c
// Linux epoll: event-driven way to handle many connections
int epfd = epoll_create1(0);

// Add server socket to epoll
struct epoll_event ev = { .events = EPOLLIN, .data.fd = server_fd };
epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

while (running) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1);  // blocks until event

    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == server_fd) {
            // New connection
            int client_fd = accept(server_fd, NULL, NULL);
            struct epoll_event cev = { .events = EPOLLIN, .data.fd = client_fd };
            epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &cev);
        } else {
            // Data arrived on existing connection
            handle_client_data(events[i].data.fd);
        }
    }
}
// One thread handles THOUSANDS of connections with zero busy-waiting
```

### Timer Events

```c
// Periodic system update using timer events
Timer* display_timer = create_timer(100);  // 100 ms interval
register_timer_handler(display_timer, periodic_update);

void periodic_update(Event* e) {
    update_clock_display();      // refresh HH:MM:SS on taskbar
    update_battery_indicator();  // refresh battery % icon
    check_scheduled_tasks();     // run any due cron jobs
    // Handler runs 10 times/second driven purely by events — no spinning loop
}
```

---

## 7. Event-Driven vs Polling

| Aspect               | Event-Driven                  | Polling                          |
| -------------------- | ----------------------------- | -------------------------------- |
| **CPU usage**        | Low — CPU idle between events | High — CPU constantly checking   |
| **Response time**    | Immediate when event arrives  | Delayed by polling interval      |
| **Power efficiency** | Excellent — CPU can sleep     | Poor — CPU always awake          |
| **Implementation**   | Complex (ISRs, event queues)  | Simple (loop + check)            |
| **Scalability**      | High — handles many sources   | Poor — each source needs polling |

```
  Polling (bad for slow devices):

  while (true) {
      if (keyboard_has_key()) handle_key();   // check 3 billion times/second
      if (disk_done()) handle_disk();         // disk completes once per 10 ms
  }
  // 99.99% of CPU time wasted checking nothing-happened

  Event-driven (efficient):

  CPU: zzz (sleeping)
       ← keyboard interrupt fires! →
  CPU: wakes up → reads key → back to sleep
       ← disk interrupt fires! →
  CPU: wakes up → processes data → back to sleep
  // CPU only works when there's actual work to do
```

**Hybrid approaches:** Some high-performance network drivers use interrupts for normal traffic but switch to polling temporarily when packet bursts arrive (reduces interrupt overhead). Linux's NAPI (New API) driver model does exactly this.

---

## 8. Advantages and Challenges

### Advantages

| Advantage               | Explanation                                                      |
| ----------------------- | ---------------------------------------------------------------- |
| **CPU efficiency**      | CPU sleeps when idle; only active during real work               |
| **Power savings**       | Critical for mobile devices and data centers                     |
| **Responsiveness**      | User input handled immediately when it arrives                   |
| **Natural concurrency** | Multiple event sources handled without complex thread management |
| **Scalability**         | One thread can manage thousands of event sources (e.g., `epoll`) |

### Challenges

| Challenge                                 | Explanation                                                           |
| ----------------------------------------- | --------------------------------------------------------------------- |
| **Callback complexity ("callback hell")** | Many nested callbacks → hard-to-follow code flow                      |
| **Non-linear debugging**                  | Execution order depends on events, not code order — harder to trace   |
| **Error propagation**                     | Errors in async handlers must be carefully propagated                 |
| **Event storms**                          | Burst of events can overwhelm queues; need rate limiting/dropping     |
| **Handler must be fast**                  | Slow handlers block other events; must delegate to background threads |

---

## 9. Practical Design Considerations

### Keep Handlers Fast

```
  Event handlers run with interrupts disabled (hardware ISRs) or block the loop.

  Rule: Handler should do minimal work (set a flag, add to queue) and return.
  Delegate heavy work to worker threads or background processes.

  WRONG: event handler that reads an entire large file
  RIGHT: event handler that queues a read request → worker thread reads the file
          → completion event fires → handler processes the result
```

### Event Ordering

```
  Keyboard events must preserve typing order: 'H','e','l','l','o'
  → Use FIFO queue for same-source events

  Mouse events and timer events may be processed in any order
  → No ordering needed between different sources
  → OS can process unrelated events in parallel (worker threads)
```

### Preventing Event Queue Overflow

```
  Solutions when events arrive faster than handlers process:

  1. Drop new events (tail-drop): easy but loses recent data
  2. Drop old events (head-drop): keeps newest; useful for video frames
  3. Rate-limit source: throttle the event source (e.g., mouse debounce)
  4. Back-pressure: block source until queue drains (socket flow control)
```

---

## 9. Code Examples

> Working code that demonstrates event-driven programming in practice.

### C++ — Simple Version
Implement a simple event loop — event queue, four event types (TIMER, IO_READY, USER_INPUT, QUIT), register one handler per type, dispatch events FIFO.

```cpp
// Simple event loop: register handlers for event types, post events to a FIFO queue,
// run the loop — it dequeues events and dispatches to the registered handler.
// Models how a GUI framework (Qt, GTK) or game loop works internally.

#include <iostream>
#include <functional>
#include <unordered_map>
#include <queue>
#include <string>
#include <vector>

// ── Event types ───────────────────────────────────────────────────────────────
enum class EventType {
    TIMER,        // periodic tick — used for animations, timeouts
    IO_READY,     // a file descriptor / socket has data ready to read
    USER_INPUT,   // keyboard or mouse event
    QUIT,         // application should exit
};

std::string to_str(EventType t) {
    switch (t) {
        case EventType::TIMER:      return "TIMER";
        case EventType::IO_READY:   return "IO_READY";
        case EventType::USER_INPUT: return "USER_INPUT";
        case EventType::QUIT:       return "QUIT";
    }
    return "?";
}

// ── Event ─────────────────────────────────────────────────────────────────────
struct Event {
    EventType   type;
    std::string data;   // payload: key pressed, file name, message, etc.
};

// ── Event handler type ────────────────────────────────────────────────────────
using Handler = std::function<void(const Event&)>;

// ── Event Loop ────────────────────────────────────────────────────────────────
class EventLoop {
    std::queue<Event>                    queue;      // FIFO event buffer
    std::unordered_map<int, Handler>     handlers;   // EventType (as int) → callback
    bool                                 running = false;

public:
    // Register a callback for an event type (only one handler per type here)
    void on(EventType type, Handler h) {
        handlers[static_cast<int>(type)] = h;
        std::cout << "  Registered handler for " << to_str(type) << "\n";
    }

    // Post an event to the queue — non-blocking, just enqueues it
    void post(EventType type, const std::string& data = "") {
        queue.push({type, data});
    }

    // Run the loop until QUIT is received or queue is empty
    void run() {
        running = true;
        std::cout << "\n--- Event loop started ---\n";

        while (running && !queue.empty()) {
            Event ev = queue.front();    // take the next event (FIFO order)
            queue.pop();

            std::cout << "\n[loop] event: " << to_str(ev.type);
            if (!ev.data.empty()) std::cout << " (\"" << ev.data << "\")";
            std::cout << "\n";

            auto it = handlers.find(static_cast<int>(ev.type));
            if (it != handlers.end()) {
                it->second(ev);          // call registered handler
            } else {
                std::cout << "  [loop] no handler — event dropped\n";
            }

            if (ev.type == EventType::QUIT) running = false;
        }

        std::cout << "\n--- Event loop stopped ---\n";
    }
};

int main() {
    std::cout << "=== Simple Event Loop ===\n\n";
    EventLoop loop;

    // Register event handlers (callbacks registered before the loop starts)
    loop.on(EventType::TIMER, [](const Event&) {
        std::cout << "  [handler] TIMER: update animation frame, check scheduled tasks\n";
    });
    loop.on(EventType::IO_READY, [](const Event& e) {
        std::cout << "  [handler] IO_READY: reading data from \"" << e.data << "\"\n";
    });
    loop.on(EventType::USER_INPUT, [](const Event& e) {
        std::cout << "  [handler] USER_INPUT: key pressed = '" << e.data << "'\n";
    });
    loop.on(EventType::QUIT, [](const Event&) {
        std::cout << "  [handler] QUIT: saving state, releasing resources\n";
    });

    // Post events (simulating what OS / devices / user would generate)
    loop.post(EventType::TIMER);
    loop.post(EventType::USER_INPUT, "H");
    loop.post(EventType::USER_INPUT, "i");
    loop.post(EventType::IO_READY,   "socket:8080");
    loop.post(EventType::TIMER);
    loop.post(EventType::IO_READY,   "file:data.txt");
    loop.post(EventType::QUIT);

    loop.run();
    return 0;
}
```

### C++ — Medium / LeetCode Style
Non-blocking I/O event loop — single thread monitors multiple "sockets" using select()-style readiness; timers fire at scheduled ticks; shows how Node.js and nginx handle many connections without blocking.

```cpp
// Single-threaded non-blocking I/O event loop.
// One thread monitors many file descriptors (fds) for readiness — like epoll/select().
// No thread blocks waiting for one fd; instead, the loop checks all fds each tick.
// This is the core pattern behind Node.js, nginx, Redis, and most async servers.

#include <iostream>
#include <functional>
#include <unordered_map>
#include <queue>
#include <vector>
#include <string>
#include <memory>

// ── Simulated file descriptor (socket / file) ─────────────────────────────────
struct FD {
    int                  id;
    std::string          label;            // human-readable name
    std::queue<std::string> buffer;        // data waiting to be read
    bool                 closed = false;
};

// ── Timer (like setTimeout in JavaScript) ─────────────────────────────────────
struct Timer {
    int  fire_at_tick;
    int  id;
    std::function<void()> callback;
    bool operator>(const Timer& o) const { return fire_at_tick > o.fire_at_tick; }
};

// ── Non-blocking event loop ────────────────────────────────────────────────────
class EventLoop {
    std::unordered_map<int, FD>                  fds;
    std::unordered_map<int, std::function<void(int, const std::string&)>> read_cbs;
    std::unordered_map<int, std::function<void(int)>>                     close_cbs;
    std::priority_queue<Timer, std::vector<Timer>, std::greater<Timer>>   timers;

    int next_fd    = 3;   // 0=stdin, 1=stdout, 2=stderr
    int next_timer = 0;
    int tick       = 0;

public:
    int register_fd(const std::string& label) {
        int id = next_fd++;
        fds[id] = {id, label};
        std::cout << "  EventLoop: watching fd=" << id << " (" << label << ")\n";
        return id;
    }

    void on_readable(int fd, std::function<void(int, const std::string&)> cb) {
        read_cbs[fd] = cb;
    }
    void on_close(int fd, std::function<void(int)> cb) {
        close_cbs[fd] = cb;
    }

    // Inject data into a fd (simulates network packet / file data arriving)
    void push_data(int fd, const std::string& data) {
        if (fds.count(fd)) fds[fd].buffer.push(data);
    }
    void close_fd(int fd) {
        if (fds.count(fd)) fds[fd].closed = true;
    }

    // Schedule a callback after delay_ticks ticks (like setTimeout)
    void set_timeout(int delay_ticks, std::function<void()> cb) {
        timers.push({tick + delay_ticks, next_timer++, cb});
    }

    // The event loop — one iteration per "tick" (like a select()/epoll() cycle)
    void run(int max_ticks = 8) {
        std::cout << "\n--- Non-blocking I/O event loop started ---\n";

        for (; tick < max_ticks; ++tick) {
            std::cout << "\n[tick=" << tick << "] epoll_wait() — polling readiness\n";
            bool idle = true;

            // 1. Fire due timers
            while (!timers.empty() && timers.top().fire_at_tick <= tick) {
                Timer t = timers.top(); timers.pop();
                std::cout << "  [timer] timer#" << t.id << " fired\n";
                t.callback();
                idle = false;
            }

            // 2. Check which fds are readable (like epoll returning ready fds)
            for (auto& [id, fd] : fds) {
                if (!fd.buffer.empty() && read_cbs.count(id)) {
                    std::string data = fd.buffer.front(); fd.buffer.pop();
                    std::cout << "  [epoll] fd=" << id << " (" << fd.label << ") READABLE\n";
                    read_cbs[id](id, data);
                    idle = false;
                }
                if (fd.closed && close_cbs.count(id)) {
                    std::cout << "  [epoll] fd=" << id << " (" << fd.label << ") CLOSED\n";
                    close_cbs[id](id);
                    close_cbs.erase(id);
                    idle = false;
                }
            }

            if (idle) std::cout << "  [loop]  nothing ready — thread sleeps (epoll blocks)\n";
        }

        std::cout << "\n--- Event loop ended ---\n";
    }
};

int main() {
    std::cout << "=== Non-Blocking I/O Event Loop ===\n\n";
    EventLoop loop;

    // Register multiple client connections (like a web server accepting connections)
    int c1 = loop.register_fd("client-1");
    int c2 = loop.register_fd("client-2");
    int c3 = loop.register_fd("client-3");

    // Register handlers — called only when a fd is readable (no blocking wait!)
    loop.on_readable(c1, [](int, const std::string& d){ std::cout << "    [handler] c1: " << d << " → send 200 OK\n"; });
    loop.on_readable(c2, [](int, const std::string& d){ std::cout << "    [handler] c2: " << d << " → validate credentials\n"; });
    loop.on_readable(c3, [](int, const std::string& d){ std::cout << "    [handler] c3: " << d << " → query database\n"; });
    loop.on_close(c2,    [](int fd){ std::cout << "    [handler] c2 disconnected (fd=" << fd << ")\n"; });

    // Schedule data to arrive asynchronously at different ticks
    loop.set_timeout(1, [&]{ loop.push_data(c1, "GET /index.html HTTP/1.1"); });
    loop.set_timeout(2, [&]{ loop.push_data(c2, "POST /login {user:'alice'}"); });
    loop.set_timeout(3, [&]{ loop.push_data(c3, "GET /api/data?limit=10"); });
    loop.set_timeout(4, [&]{ loop.push_data(c1, "GET /style.css"); });
    loop.set_timeout(5, [&]{ loop.close_fd(c2); });

    loop.run(7);
    return 0;
}
```

### Python — Simple Version
Implement a simple event loop with four event types, FIFO dispatch, and registered handler callbacks — shows the three core components: event source, event queue, event loop.

```python
# Simple event loop: register handlers for event types, post events to a FIFO queue,
# run the loop — it dispatches each event to its registered handler.
# Models how GUIs, game engines, and embedded systems process events.

from collections import deque
from enum import Enum, auto

# ── Event types ───────────────────────────────────────────────────────────────
class EventType(Enum):
    TIMER      = auto()   # periodic tick
    IO_READY   = auto()   # file/socket has data ready
    USER_INPUT = auto()   # keyboard or mouse input
    QUIT       = auto()   # application should exit

# ── Event ─────────────────────────────────────────────────────────────────────
class Event:
    def __init__(self, etype: EventType, data: str = ""):
        self.type = etype
        self.data = data

# ── Event Loop ────────────────────────────────────────────────────────────────
class EventLoop:
    def __init__(self):
        # FIFO queue — events processed in arrival order (same source preserves order)
        self.queue:    deque[Event]              = deque()
        # Handler map — one callback registered per event type
        self.handlers: dict[EventType, callable] = {}

    def on(self, etype: EventType, handler: callable):
        """Register a callback that runs when etype occurs."""
        self.handlers[etype] = handler
        print(f"  Registered handler for {etype.name}")

    def post(self, etype: EventType, data: str = ""):
        """Add an event to the queue — non-blocking, returns immediately."""
        self.queue.append(Event(etype, data))

    def run(self):
        """
        Main event loop:
        - Block until an event is ready (here: just process queue)
        - Dispatch to handler
        - Repeat until QUIT
        In a real OS, this blocks in epoll_wait() / WaitForSingleObject() to save CPU.
        """
        print("\n--- Event loop started ---")
        while self.queue:
            ev = self.queue.popleft()   # FIFO

            data_str = f' ("{ev.data}")' if ev.data else ""
            print(f"\n[loop] event: {ev.type.name}{data_str}")

            handler = self.handlers.get(ev.type)
            if handler:
                handler(ev)
            else:
                print("  [loop] no handler registered — event ignored")

            if ev.type == EventType.QUIT:
                print("  [loop] QUIT received — stopping")
                break

        print("\n--- Event loop stopped ---")

if __name__ == "__main__":
    print("=== Simple Event Loop ===\n")
    loop = EventLoop()

    # Register handlers (callbacks — called when each event type fires)
    loop.on(EventType.TIMER,      lambda e: print("  [handler] TIMER: animate frame, check timeouts"))
    loop.on(EventType.IO_READY,   lambda e: print(f"  [handler] IO_READY: reading from \"{e.data}\""))
    loop.on(EventType.USER_INPUT, lambda e: print(f"  [handler] USER_INPUT: key='{e.data}'"))
    loop.on(EventType.QUIT,       lambda e: print("  [handler] QUIT: saving state, cleanup"))

    # Post events (as if the OS/devices/user generated them)
    loop.post(EventType.TIMER)
    loop.post(EventType.USER_INPUT, "H")
    loop.post(EventType.USER_INPUT, "i")
    loop.post(EventType.IO_READY,   "socket:8080")
    loop.post(EventType.TIMER)
    loop.post(EventType.IO_READY,   "file:data.txt")
    loop.post(EventType.QUIT)

    loop.run()
```

### Python — Medium Level
Non-blocking I/O event loop — single thread monitors multiple file descriptors for readiness using a timer-driven select()-style model; shows how Node.js-style async works.

```python
# Single-threaded non-blocking I/O event loop — simulates epoll()/select().
# One thread monitors many "sockets" simultaneously without blocking on any of them.
# Data arrives asynchronously via scheduled timers; handlers run only when fd is readable.
# This is the foundation of Node.js, nginx, Redis, asyncio (Python), and libuv.

import heapq
from collections import deque
from dataclasses import dataclass, field
from typing import Callable

# ── Simulated file descriptor ─────────────────────────────────────────────────
@dataclass
class FD:
    fd_num:  int
    label:   str
    buffer:  deque = field(default_factory=deque)   # queued incoming data
    closed:  bool  = False

# ── Timer (like setTimeout / setInterval) ─────────────────────────────────────
@dataclass
class Timer:
    fire_at:  int
    timer_id: int
    callback: Callable = field(compare=False, repr=False)
    def __lt__(self, o): return (self.fire_at, self.timer_id) < (o.fire_at, o.timer_id)

# ── Non-blocking event loop ────────────────────────────────────────────────────
class EventLoop:
    def __init__(self):
        self._fds:       dict[int, FD]       = {}
        self._read_cbs:  dict[int, Callable] = {}
        self._close_cbs: dict[int, Callable] = {}
        self._timers:    list[Timer]         = []   # min-heap by fire_at
        self._next_fd    = 3
        self._next_timer = 0
        self.tick        = 0

    def register_fd(self, label: str) -> int:
        fd = self._next_fd; self._next_fd += 1
        self._fds[fd] = FD(fd, label)
        print(f"  EventLoop: watching fd={fd} ({label})")
        return fd

    def on_readable(self, fd: int, cb: Callable): self._read_cbs[fd]  = cb
    def on_close(self,    fd: int, cb: Callable): self._close_cbs[fd] = cb

    def push_data(self, fd: int, data: str):
        """Inject data into an fd (simulates OS kernel writing received data)."""
        if fd in self._fds: self._fds[fd].buffer.append(data)

    def close_fd(self, fd: int):
        if fd in self._fds: self._fds[fd].closed = True

    def set_timeout(self, delay_ticks: int, cb: Callable):
        """Schedule a callback to fire after delay_ticks ticks (like setTimeout)."""
        heapq.heappush(self._timers, Timer(self.tick + delay_ticks, self._next_timer, cb))
        self._next_timer += 1

    def run(self, max_ticks: int = 8):
        """
        Main event loop body — one iteration per tick:
        1. Fire any timers due at this tick
        2. Check each fd for readiness (like epoll_wait() returning ready events)
        3. If nothing is ready → sleep until next timer or I/O event
        """
        print("\n--- Non-blocking I/O event loop ---")
        while self.tick < max_ticks:
            print(f"\n[tick={self.tick}] epoll_wait() — polling readiness")
            idle = True

            # Step 1: fire due timers
            while self._timers and self._timers[0].fire_at <= self.tick:
                t = heapq.heappop(self._timers)
                print(f"  [timer]  timer#{t.timer_id} fired")
                t.callback()
                idle = False

            # Step 2: check fd readiness (like epoll returning EPOLLIN events)
            for fd_num, fd in list(self._fds.items()):
                if fd.buffer and fd_num in self._read_cbs:
                    data = fd.buffer.popleft()
                    print(f"  [epoll]  fd={fd_num} ({fd.label}) READABLE")
                    self._read_cbs[fd_num](fd_num, data)
                    idle = False
                if fd.closed and fd_num in self._close_cbs:
                    print(f"  [epoll]  fd={fd_num} ({fd.label}) CLOSED")
                    self._close_cbs.pop(fd_num)(fd_num)
                    idle = False

            if idle:
                print("  [loop]   nothing ready — thread sleeps (epoll blocks, CPU idle)")
            self.tick += 1
        print("\n--- Event loop ended ---")

if __name__ == "__main__":
    print("=== Non-Blocking I/O Event Loop ===\n")
    loop = EventLoop()

    # Simulate a web server: one thread, many client connections
    c1 = loop.register_fd("client-1 (GET /index)")
    c2 = loop.register_fd("client-2 (POST /login)")
    c3 = loop.register_fd("client-3 (GET /api)")

    loop.on_readable(c1, lambda fd, d: print(f"    [handler] c1: {d} → send 200 OK"))
    loop.on_readable(c2, lambda fd, d: print(f"    [handler] c2: {d} → validate credentials"))
    loop.on_readable(c3, lambda fd, d: print(f"    [handler] c3: {d} → query DB"))
    loop.on_close(c2,    lambda fd:    print(f"    [handler] c2 disconnected (fd={fd})"))

    # Data arrives asynchronously — scheduled by timers (like OS waking loop on I/O)
    loop.set_timeout(1, lambda: loop.push_data(c1, "GET /index.html HTTP/1.1"))
    loop.set_timeout(2, lambda: loop.push_data(c2, "POST /login {user:'alice'}"))
    loop.set_timeout(3, lambda: loop.push_data(c3, "GET /api/data?limit=10"))
    loop.set_timeout(4, lambda: loop.push_data(c1, "GET /style.css"))
    loop.set_timeout(5, lambda: loop.close_fd(c2))

    loop.run(max_ticks=7)
```

---

## 10. Key Takeaways

- **Event-driven programming** = execution flow determined by events, not a preset sequence; OS registers handlers for event types and dispatches them as events arrive
- **Three core components:** event source (generates events), event queue (buffers pending events), event loop (reads queue, dispatches to handlers)
- **Event loop blocks** on `wait_for_event()` — CPU can enter low-power sleep state between events → huge power savings
- **Callbacks/handlers** are registered functions called when a specific event occurs; should be short and fast
- **Async pattern:** handler starts an operation and returns immediately; operation fires a completion event when done → UI stays responsive
- **Priority queues** ensure urgent events (hardware failures, timer ticks) are processed before lower-priority ones (screen redraws)
- **Event-driven vs polling:** polling wastes CPU; event-driven sleeps until needed — orders of magnitude more efficient
- **ISRs are the kernel's event handlers:** hardware interrupt → OS ISR runs → places event in higher-level queue → OS event loop dispatches to process/driver
- **`epoll`/`kqueue`/IOCP:** OS-level APIs that allow one thread to efficiently wait on thousands of file descriptors/sockets using event-driven I/O
- **Challenges:** callback complexity, non-linear debugging, event storms; solutions include async/await patterns, worker thread delegation, and queue limits
- **Event vs interrupt:** interrupt is low-level (hardware IRQ, CPU signal); event is higher-level abstraction; interrupts often generate events, but not all events come from hardware interrupts
