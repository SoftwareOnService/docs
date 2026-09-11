---
title: "Part 23 — Concurrency & Parallelism: IO Bound vs CPU Bound (Part 1)"
tags: [backend, video-notes, concurrency, parallelism]
course: "[[_00 - Backend from First Principles - Index]]"
source: "https://www.youtube.com/watch?v=bs9MEYRTA30"
video_id: "bs9MEYRTA30"
playlist_position: 23
duration_seconds: 5285
published: "2026-01-01"
status: completed
---

# Part 23 — Concurrency & Parallelism: IO Bound vs CPU Bound

> [!info] Video Reference
> **Title:** 22. Concurrency & Parallelism: IO Bound vs CPU Bound  
> **Channel:** Sriniously  
> **Playlist:** Backend from First Principles (Video 23 of 29)  
> **URL:** https://www.youtube.com/watch?v=bs9MEYRTA30  
> **Duration:** 1 hour 28 minutes 5 seconds  
> **Published:** 2026-01-01

> [!abstract] In This Chapter (Part 1)
> This first half of the video builds a rigorous mental model for concurrency and parallelism from first principles. We cover:
> - **Why concurrency matters**: The fundamental requirement that every backend must handle multiple requests simultaneously, and the catastrophic waste of a sequential single-threaded server.
> - **The cost of not using concurrency**: Quantitative analysis showing how a modern 3 GHz CPU wastes 300 million instructions per 100 ms of database wait time, and how typical backend workloads spend 70–95% of their time waiting on IO.
> - **IO-bound vs CPU-bound operations**: Clear definitions with concrete examples (database queries, network calls, file system access vs. JSON parsing, image processing, encryption). The critical insight that most backend applications are overwhelmingly IO-bound.
> - **Concurrency vs parallelism**: The crucial distinction—parallelism requires multiple CPU cores executing instructions *at the exact same moment*; concurrency is about *structuring* a program to start, pause, and resume multiple tasks so they *appear* to progress simultaneously, achievable even on a single core.
> - **Visualizing concurrent request handling**: A detailed timeline diagram showing how two requests interleave on a single CPU core—CPU bursts alternating with IO waits—demonstrating how concurrency eliminates idle cycles.
> - **Threading model deep dive**: OS threads as independent units of execution, each with its own stack (≈8 MB on Linux), instruction pointer, and register state. Thread creation via syscall, kernel-managed preemptive scheduling, time slices, and context switching.
> - **Thread overhead & context switching**: Memory overhead (stack per thread), creation overhead (syscall + kernel bookkeeping), and the dominant cost—context switching (1–10 μs per switch, exploding with thousands of threads). Why thread-per-request models struggle at high concurrency.
> - **Event loop model deep dive**: Single-threaded, callback/queue-based concurrency. Non-blocking IO via OS primitives (epoll on Linux, kqueue on macOS). The event loop iteration: check IO completions → run callbacks → repeat. Efficiency gains (no context switches, minimal memory) and the critical constraint: **never block the event loop**.

## [00:00] Introduction: Why Concurrency Matters

Every backend system you will ever build shares one fundamental requirement: **it must be able to handle multiple things at once**. In the context of an HTTP-based web server (which is what we mean by "backend application" here), if your server can process only one request at a time, then a thousand other users trying to send requests must either wait or receive a "server busy" error. This is untenable in production.

Understanding how servers—and the operating systems beneath them—achieve multitasking, and how different languages and runtimes (Node.js, Python, Rust, Go) expose this capability, shapes your mental model of request processing. This mental model is not about quick tips or copy-paste techniques; it is the foundation for debugging, structuring applications, and making architectural decisions.

Most developers learn `async`/`await`, threads, and concurrency primitives from language documentation without grasping what these keywords actually do mechanically. We know they work, but understanding the *mechanics* puts you a step ahead. This video builds that mental model from the ground up—from the hardware level up to your application code.

---

### The Naive Sequential Server: A Concrete Scenario

Imagine a single client (a browser) sending a request to your backend server. The request traverses your routing, service, handler, and repository layers. To process it, the server must query a database: send a query, **wait** for the database to execute and respond, then return the result.

On a local network (localhost), a simple query might take **1–2 ms**. In production with the database in a different availability zone, **20–30 ms**. Across regions, **90–100 ms**.

**The critical question:** While the server waits those 2, 30, or 100 ms, what is the CPU doing?

With a naive synchronous, single-request-at-a-time approach: **nothing**. The CPU sits completely idle, waiting for network packets from the database.

---

## [02:51] The Cost of Not Using Concurrency: Quantifying the Waste

A modern CPU executes roughly **3 billion instructions per second** (3 GHz ≈ 3 million instructions per millisecond).

During a **100 ms** cross-region database wait, the CPU *could have executed*:
```
3,000,000 instructions/ms × 100 ms = 300,000,000 instructions
```

But with sequential processing, it executes **zero**. That is the waste.

A typical mid-complexity API call involves **3–5 database queries** plus **1–2 external service calls** (email, Redis cache, etc.)—call it **5 network operations** total. If each averages **50 ms** of wait time, that's **250 ms** of pure IO wait per request. Meanwhile, actual CPU work (validation, JSON serialization, business logic) might only take **10 ms**.

**Result: The server's CPU and memory are idle 95% of the time.**

> **Key Insight:** In a typical backend application, **>70% of request lifecycle time is spent on IO** (database, external APIs, file system, logging, stdin/stdout). Concurrency exists to reclaim that 70%—to use the CPU and memory for *other* work while one request waits on the network.

---

## [05:55] IO-Bound vs CPU-Bound Operations

### Definitions

| Category | Definition | Examples |
|----------|------------|----------|
| **IO-Bound** | Operations that wait for external resources (network, disk, devices). The CPU is useless during the wait. | Database queries, HTTP API calls, file reads/writes, logging (stdout), cache (Redis) operations, stdin/stdout |
| **CPU-Bound** | Operations that perform actual computation on the CPU—crunching numbers, transforming data in memory. | JSON parsing/validation, image/video processing (matrix multiplication), encryption (JWT verification), compression, ML inference |

### How to Tell the Difference

- **IO-bound**: The operation involves a **syscall** that hands control to the kernel/device driver, then waits for an interrupt/signal when data arrives. The CPU is free to do other work.
- **CPU-bound**: The operation stays in **user space**, executing instructions continuously on the CPU core. No waiting on external resources.

### The Backend Reality

> **Most backend applications are overwhelmingly IO-bound.** Unless you are doing video encoding, heavy encryption, image processing, or ML inference, your bottleneck is IO, not CPU.

**Implication:**
- For **IO-bound** workloads → **Concurrency is mandatory**. Without it, you waste 95% of your hardware. The bottleneck is *always* IO, whether you have 1 core or 64.
- For **CPU-bound** workloads → **Parallelism is beneficial**. Multiple cores executing simultaneously finish heavy computation faster. Concurrency alone (time-slicing on one core) doesn't speed up CPU-bound work—it just interleaves it.

You need **both**: concurrency for IO-bound dominance, parallelism for the occasional CPU-heavy task.

---

## [09:44] Concurrency vs Parallelism Explained

These terms are often conflated. The distinction is precise and mechanical.

### Parallelism
> **Executing multiple instructions *at the exact same moment*.**

Requires **hardware-level support**: at least **two CPU cores**. One core can execute only one instruction at a time. To run two instructions simultaneously, you need two cores.

### Concurrency
> **Dealing with multiple things at once by structuring a program to start, pause, and resume tasks.**

Achievable on **a single CPU core**. The program is written (or the runtime structures it) so that Task A runs briefly, yields (e.g., waiting on IO), Task B runs, yields, Task A resumes, etc. At any *instant*, only one instruction executes—but *over time*, multiple tasks make progress. It **feels like** simultaneous execution.

### The "Doing" vs "Dealing" Distinction

- **Parallelism** = *doing* multiple things at once (simultaneous execution)
- **Concurrency** = *dealing with* multiple things at once (interleaved progress)

---

### Visualizing the Difference: Timeline Diagrams

#### Concurrency on a Single Core (Two Requests A & B)

```
Time (ms) →  0    5    10   15   20   25   30   35   40   45   50   55   60
             │    │    │    │    │    │    │    │    │    │    │    │    │
Request A:   █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
             CPU  │← 40 ms waiting for DB →│   CPU   │
Request B:   ░░░░░███████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
             wait │← 15 ms CPU →│          wait  │← CPU →│
```

**What this diagram shows:** Request A uses CPU for 5 ms (validation, routing), then issues a DB query and **waits 40 ms** (IO). The OS scheduler immediately gives the CPU to Request B, which uses 15 ms of CPU, then also waits on IO. At 40 ms, A's DB responds; after scheduling latency, A resumes CPU work. At 50 ms, B's DB responds; B resumes. **Zero CPU idle time** despite only one core. Both requests are "in progress" from the client's perspective, but at any micro-instant, only one runs.

#### Parallelism on Two Cores (Same Two Requests)

```
Time (ms) →  0    5    10   15   20   25   30   35   40   45   50
             │    │    │    │    │    │    │    │    │    │    │
Core 1:      █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
             Req A CPU    │← A waits →│                    A CPU
Core 2:      ░░░░░███████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                  Req B CPU    │← B waits →│           B CPU
```

**What this diagram shows:** Both requests get CPU *simultaneously* on different cores. No waiting for the other to yield. True simultaneous execution. This is parallelism.

---

## [15:16] Visualizing Concurrent Request Handling: The "Conveyor Belt" Mental Model

The timeline above is the **conveyor belt** model. Think of the CPU as a single worker at a station. Tasks (requests) arrive on a belt. The worker processes a task until it hits an "IO wait" station—then the task moves off the belt into a waiting area, and the worker immediately grabs the next task. When a waiting task's IO completes, it re-enters the belt.

**Key properties:**
- **Single worker (core)** → concurrency only
- **Multiple workers (cores)** → concurrency + parallelism
- **IO wait = task leaves the belt** → CPU free for others
- **Scheduler = traffic controller** deciding which task gets the worker next

This model explains why **thread-per-request** works for moderate loads (each thread = a task on the belt) but collapses under high concurrency (too many tasks, scheduler overhead dominates).

---

## [19:14] Threading Model Deep Dive

### What Is a Thread?

A **thread** is an **independent piece of execution** provided by the **operating system**. It is not a language construct—it is an OS primitive. When you create a thread, the OS allocates:

1. **Stack** (≈8 MB on Linux, mostly virtual memory until touched)
   - Tracks function call chain (call frames pushed/popped)
   - Holds local variables (e.g., `let a = 3` lives on the stack)
2. **Instruction Pointer (Program Counter)**
   - Points to the next instruction to execute
   - Saved/restored on context switch so execution can resume exactly where it left off
3. **Register State** (CPU registers)
4. **Thread Control Block (TCB)** / kernel bookkeeping structures
   - Thread ID, priority, state (running, runnable, blocked), scheduling info

### Thread Lifecycle & Scheduling

- **Creation**: User code → syscall (`clone()`/`pthread_create()`) → kernel allocates stack, TCB, adds to scheduler run queue.
- **States**: Running (on CPU), Runnable (ready, waiting for CPU), Blocked (waiting on IO/syscall).
- **Scheduler**: Kernel-controlled, **preemptive**. Each thread gets a **time slice** (typically a few ms). When the slice expires, the scheduler:
  1. Saves current thread's registers & instruction pointer
  2. Updates bookkeeping (state → runnable)
  3. Selects next thread (priority, fairness, etc.)
  4. Restores next thread's registers & instruction pointer
  5. Jumps to its instruction pointer

This is **preemptive scheduling**—threads are paused *whether they like it or not*.

### Blocking IO in the Threading Model

When a thread performs a blocking IO operation (e.g., `read()` on a socket, DB query):
1. Thread tells kernel: "I'm blocked on IO"
2. Kernel marks thread **Blocked**, removes from run queue
3. Scheduler picks another **Runnable** thread → runs it
4. When IO completes (interrupt from device/network card), kernel marks thread **Runnable**, re-adds to run queue
5. Scheduler eventually picks it → thread resumes after the syscall returns

**Critical property:** Threads **within the same process share memory** (heap, globals). Thread A allocates an object on the heap → Thread B can access it via pointer/address. No copying, no serialization—fast communication via **shared memory**.

> ⚠️ **Danger:** Shared memory + preemptive scheduling = **race conditions**. Two threads reading/writing the same variable without coordination corrupt data. We'll cover locks/mutexes/channels in Part 2.

### Threads Across Processes

- **Processes** are isolated: separate virtual address spaces. Thread from Process 1 *cannot* see Thread from Process 2's memory (security, stability).
- **Threads within a process** share the process's heap/global memory. This is why threads are "lightweight processes"—same address space, separate stacks.

### Parallelism with Threads

If you have **N CPU cores**, up to **N threads can run truly in parallel** (simultaneously). This is how threading achieves parallelism for CPU-bound work. But for IO-bound work, even on one core, many threads can be *concurrent* (interleaved via blocking/waking).

---

## [23:22] Thread Overhead & Context Switching: Why Threads Aren't Free

Three major costs make naive thread-per-request models fail at scale.

### 1. Memory Overhead (Stack)

Each thread → **~8 MB stack** (Linux default). Even with virtual memory (physical pages allocated on demand), a thread doing modest work still consumes **hundreds of KB to 1+ MB** of physical RAM.

| Threads | Approx. Physical Memory (at 1 MB/thread) |
|---------|------------------------------------------|
| 1,000   | ~1 GB                                    |
| 10,000  | ~10 GB                                   |
| 100,000 | ~100 GB (impossible on typical servers)  |

A traffic spike of 10,000 concurrent requests → 10,000 threads → **10+ GB just for stacks**, leaving nothing for application data, caches, OS, etc. **OOM crash.**

### 2. Creation Overhead (Syscall + Kernel Work)

Creating a thread = syscall → kernel:
- Allocates stack virtual memory region
- Initializes TCB, register save area
- Sets up signal masks, thread-local storage
- Inserts into scheduler data structures

**Cost: ~few μs to few ms per thread.** Not negligible when creating threads per request under load.

### 3. Context Switching Overhead (The Dominant Cost)

Every time the scheduler switches from Thread A → Thread B:

```
1. Save Thread A's CPU registers (16+ general purpose, SIMD, control regs)
2. Save instruction pointer, stack pointer, flags
3. Update Thread A's TCB (state = runnable, accounting)
4. Run scheduler algorithm (pick next thread)
5. Load Thread B's registers, instruction pointer, stack pointer
6. Update Thread B's TCB (state = running)
7. Flush/invalidate CPU pipeline, TLB entries (sometimes)
8. Resume execution at Thread B's instruction pointer
```

**Cost: 1–10 μs on modern hardware** (higher if cache/TLB misses—data in RAM not L1/L2).

With **100 threads on 4 cores**: scheduler juggles constantly → thousands of switches/second → **milliseconds of pure overhead per second**.

With **1,000+ threads**: context switch time **exceeds milliseconds**, becoming a major latency component. This is **unproductive work**—maintenance, not request processing.

> **Result:** Thread-per-request models hit a concurrency ceiling. The more threads, the more context switching, the *slower* the system gets for IO-bound work.

---

## [37:15] Event Loop Model Deep Dive

### Core Idea

Instead of many threads (each with stack, registers, kernel TCB), use **one thread** + **non-blocking IO** + **event notification** from the OS.

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT LOOP (single thread)               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Check IO   │ →  │ Run Ready   │ →  │   Loop      │     │
│  │  Completions│    │  Callbacks  │    │   Back      │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### How It Works

1. **Application code** initiates IO (DB query, HTTP call) via **non-blocking syscall** (e.g., `epoll_ctl` to register interest).
2. Instead of blocking, the code **registers a callback** (or uses `async`/`await` which desugars to a callback/state machine) and **yields control back to the event loop**.
3. The **event loop** calls the OS's **IO multiplexing primitive**:
   - **Linux**: `epoll` (edge/level-triggered)
   - **macOS/BSD**: `kqueue`
   - **Windows**: IOCP (I/O Completion Ports)
4. The OS kernel monitors **thousands of file descriptors/sockets** simultaneously. When data arrives on any, the kernel marks it "ready."
5. Event loop wakes, collects ready descriptors, **runs their callbacks** (which execute the "what to do when IO completes" logic).
6. Callbacks run **to completion** (no preemption mid-callback—cooperative scheduling).
7. Loop repeats.

### Why It's Efficient for IO-Bound Work

| Resource | Threading Model | Event Loop |
|----------|-----------------|------------|
| **Stacks** | 1 per thread (MB each) | 1 total (single thread) |
| **Context Switches** | Preemptive, frequent (μs each) | **None** (cooperative, only at `await`) |
| **Memory** | Grows linearly with concurrency | **Constant** (single stack + callback closures) |
| **Scheduler** | Kernel (complex, general-purpose) | **User-space** (simple, app-specific) |
| **Scalability** | ~10K threads practical limit | **100K–1M+ connections** (C10K problem solved) |

### The Critical Constraint: Never Block the Event Loop

Because there is **only one thread**, if any callback performs a **CPU-bound operation taking 100 ms** (e.g., heavy image processing, synchronous crypto, large JSON parse), **the entire event loop freezes**. No other request makes progress. All connections stall.

**Rule:** In event-loop architectures (Node.js, Python asyncio, Go's netpoller, Java virtual threads), **all IO must be non-blocking**, and **CPU-bound work must be offloaded** (worker threads, thread pool, separate process).

### Async/Await as Syntactic Sugar

`async`/`await`, promises, callbacks—these are **syntactic conveniences** over the event loop. They let you write sequential-looking code that the compiler/runtime transforms into a **state machine** yielding at each `await`.

```javascript
// What you write
async function handleRequest(req) {
  const user = await db.query('SELECT * FROM users WHERE id = ?', req.id);
  const posts = await fetch(`https://api.example.com/posts/${user.id}`);
  return { user, posts };
}

// What the runtime sees (simplified state machine)
function handleRequest(req) {
  return db.query(...).then(user => {
    return fetch(...).then(posts => ({ user, posts }));
  });
}
```

At each `await` (or `.then()`), the function **returns a promise**, **yields to the event loop**, and **registers the continuation** as a callback to run when the IO completes.

---

## [42:44] Threading vs Event Loop: Handling Requests in Practice

### Threading Model (e.g., Java servlet containers, Python sync, Ruby, PHP-FPM)

```
Request A arrives
    → Thread pool assigns Thread 1
    → Thread 1: parse JSON (CPU)
    → Thread 1: DB query (blocks) → kernel marks Thread 1 blocked
    → Scheduler runs Thread 2 (Request B)
    → Thread 2: parse JSON (CPU)
    → Thread 2: DB query (blocks)
    → ...when DB responds, kernel unblocks thread...
```

**Characteristics:**
- Natural sequential code style
- Blocking IO is fine (thread just sleeps)
- Shared memory = easy communication, but race conditions
- Scales to ~hundreds-low thousands concurrent requests

### Event Loop Model (e.g., Node.js, Python asyncio, Go netpoller)

```
Request A arrives
    → Event loop: parse JSON (CPU, fast)
    → Event loop: initiate DB query (non-blocking), register callback
    → Event loop: immediately free for Request B
    → Request B arrives → parse JSON → initiate DB query → register callback
    → ...event loop checks epoll/kqueue, runs ready callbacks...
```

**Characteristics:**
- Must use `async`/`await` or callbacks everywhere
- Single thread → no races on single-threaded data (but shared state across callbacks still possible)
- Massive concurrency (100K+) with minimal memory
- CPU-bound work blocks everything → must offload

### What Comes Next (Part 2)

The transcript continues into:
- **Async/await explained** (state machine transformation)
- **Virtual threads & Goroutines** (Go runtime, Java Project Loom)
- **How async/await works under the hood** (compiler-generated state machines)
- **Race conditions & shared state problems**
- **Solutions: Locks, mutexes, channels**
- **Summary & key takeaways**

---

