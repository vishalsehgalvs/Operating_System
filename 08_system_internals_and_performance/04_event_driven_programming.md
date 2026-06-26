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
