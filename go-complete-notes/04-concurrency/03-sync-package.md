# 03 - The sync Package

## Introduction

Go's concurrency model is built on goroutines and channels. For most coordination tasks, channels are the right tool. But some problems are better solved with shared memory and explicit locks, and the `sync` package provides the primitives for that.

> **General rule**: use channels when goroutines need to *communicate*; use `sync` primitives when goroutines need to *coordinate access to shared state*.

---

## The Problem: Data Races

When multiple goroutines read and write the same variable without synchronization, the behavior is **undefined** (a data race). Results are unpredictable and bugs are hard to reproduce.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var counter int
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++   // RACE!
        }()
    }
    wg.Wait()
    fmt.Println(counter)   // unpredictable, often != 1000
}
```

Run with `-race` to detect:
```sh
go run -race main.go
# WARNING: DATA RACE
```

### The Memory Model: Why Synchronization Is Needed

Two goroutines updating the same variable concurrently produce a **data race** (race condition): the result depends on the interleaving of their reads/writes and is **undefined**. Synchronization primitives establish **happens-before** relationships that make a write in one goroutine *visible* to a read in another.

- A `Mutex.Unlock` **happens-before** the matching `Lock`.
- A `WaitGroup.Done` **happens-before** a `Wait` that returns after it.
- `atomic` operations establish ordering for the specific location.
- A goroutine's `go` statement **happens-before** the goroutine body.

Without these, the compiler and CPU are free to reorder operations, so a goroutine may observe stale or partial values.

> 🔑 **Key idea:** Synchronization primitives don't just serialize code — they establish happens-before relationships that make one goroutine's writes visible to another.

---

## sync.Mutex (Mutual Exclusion Lock)

`sync.Mutex` ensures only one goroutine can execute a **critical section** at a time.

### Methods

| Method | Purpose |
|--------|---------|
| `Lock()` | Acquire lock (blocks if already held) |
| `Unlock()` | Release lock |
| `TryLock()` | Non-blocking attempt (Go 1.18+); returns bool |

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var mu sync.Mutex
    var counter int
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            counter++       // now safe
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Println(counter)   // 1000 (correct)
}
```

### Best Practices

```go
// Always defer Unlock immediately after Lock
mu.Lock()
defer mu.Unlock()
// ... critical section
```

**Keep the critical section as short as possible.** Do I/O and slow work *after* unlocking:

```go
// Bad - holding lock too long
mu.Lock()
data := readFromDisk() // slow!
counter += data
mu.Unlock()

// Good - minimize critical section
data := readFromDisk()
mu.Lock()
counter += data
mu.Unlock()
```

### Locking a Whole Struct

```go
type SafeCounter struct {
    mu      sync.Mutex
    counter int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.counter++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.counter
}
```

> Always pass mutex-protected structs by pointer, never by value. `go vet` catches copying a mutex with the `-copylocks` analyzer.

### Common Mutex Pitfalls

- **Must not copy a Mutex** after first use. The zero value is a usable unlocked mutex.
- Don't forget to `Unlock` — otherwise **deadlock**. Use `defer`.
- Don't call a function that also locks while already holding (self-deadlock unless it's a `RWMutex` or careful).
- **Mutex is not reentrant** — locking the same mutex twice in one goroutine hangs forever.

> ⚠️ **Gotcha:** Mutexes are not reentrant — locking the same mutex twice in one goroutine deadlocks silently instead of erroring.

---

## sync.RWMutex (Read-Write Mutex)

`sync.RWMutex` allows **multiple readers** or **one writer** at a time. Writers get exclusive access; readers share access (as long as no writer is active).

### Methods

| Method | Purpose |
|--------|---------|
| `Lock()` / `Unlock()` | Writer lock (exclusive) |
| `RLock()` / `RUnlock()` | Reader lock (shared) |
| `TryLock()`, `TryRLock()` | Non-blocking variants |

```go
type SafeCache struct {
    mu    sync.RWMutex
    store map[string]string
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()          // shared read lock
    defer c.mu.RUnlock()
    v, ok := c.store[key]
    return v, ok
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()           // exclusive write lock
    defer c.mu.Unlock()
    c.store[key] = value
}
```

### RWMutex vs Mutex: When to Use Which

| Use RWMutex when | Use Mutex when |
|-------------------|----------------|
| Reads vastly outnumber writes | Reads and writes are roughly balanced |
| Configuration caches, lookup tables | Critical section is tiny |
| Benchmark confirms it | Simpler code matters more |

> `RWMutex` has higher overhead per operation. A plain `Mutex` can be faster when operations are balanced. **Always benchmark.**

### Mixing Lock Types

```go
// BAD — will deadlock
mu.RLock()
mu.Unlock() // wrong! must call RUnlock()

// GOOD
mu.RLock()
defer mu.RUnlock()
```

---

## sync.Once

`sync.Once` ensures a function runs **exactly once**, even across multiple goroutines. Great for lazy initialization.

```go
var (
    once   sync.Once
    config *Config
)

func getConfig() *Config {
    once.Do(func() {
        config = loadConfig()   // runs only once
    })
    return config
}
```

### Key Properties

- `Do` blocks until the function completes. Other goroutines calling `Do` will block.
- If the function panics, subsequent calls to `Do` remain blocked — the `Once` is effectively stuck after a panic.
- `sync.Once` does not return a value. Store the result in a package-level variable.

> 💡 **Pro tip:** For lazy initialization that returns a value, prefer `sync.OnceValue` (Go 1.21+) — it wraps `Once.Do` and gives you the result directly.

### Singleton Pattern

```go
var (
    once     sync.Once
    instance *MyService
)

func GetInstance() *MyService {
    once.Do(func() {
        instance = &MyService{}
    })
    return instance
}
```

> This replaces double-checked locking, which is racy in Go without `sync.Once`.

---

## sync.OnceValue / OnceFunc / OnceValues (Go 1.21+)

Helper wrappers for memoization with cleaner syntax:

```go
var now = sync.OnceValue(func() time.Time {
    return time.Now()   // computed once, cached forever
})

fmt.Println(now()) // same time everywhere
```

```go
var dbConn = sync.OnceValues(func() (*sql.DB, error) {
    return sql.Open("postgres", dsn)
})

conn, err := dbConn()
```

---

## sync.WaitGroup (Deeper Patterns)

### Bounded Concurrency

```go
func processFiles(filenames []string) error {
    var wg sync.WaitGroup
    sem := make(chan struct{}, 10) // limit to 10 concurrent goroutines

    var mu sync.Mutex
    var firstErr error

    for _, name := range filenames {
        wg.Add(1)
        sem <- struct{}{} // acquire
        go func(n string) {
            defer wg.Done()
            defer func() { <-sem }() // release

            if err := processFile(n); err != nil {
                mu.Lock()
                if firstErr == nil {
                    firstErr = err
                }
                mu.Unlock()
            }
        }(name)
    }

    wg.Wait()
    return firstErr
}
```

### Pipeline Stage with WaitGroup

```go
func pipeline(input <-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    for i := 0; i < 4; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range input {
                out <- v * v
            }
        }()
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

---

## sync.Map

`sync.Map` is a **concurrency-safe map** optimized for two patterns: (1) keys written once, read many times, and (2) disjoint key sets across goroutines.

```go
var m sync.Map

// Store
m.Store("key1", "value1")

// Load
v, ok := m.Load("key1")

// Delete
m.Delete("key1")

// LoadOrStore — get or set atomically
actual, loaded := m.LoadOrStore("key2", "default")

// Range (like iterating)
m.Range(func(k, v interface{}) bool {
    fmt.Println(k, v)
    return true   // continue
})
```

> In Go 1.24+, `Iter` method also exists. Methods take `any` (interface{}), so you must type-assert keys/values.

```go
if v, ok := m.Load("key1"); ok {
    s := v.(string)   // type assertion
    fmt.Println(s)
}
```

### sync.Map vs Mutex+map

| Aspect | sync.Map | Mutex + map |
|--------|----------|-------------|
| Read-heavy, write-rare | Faster | Slower |
| Write-heavy | Slower | Faster |
| Type safety | `interface{}` | Typed |
| Simplicity | Less familiar | More familiar |

For most cases, a `sync.Mutex`-guarded regular map is simpler and often faster:

```go
type TypedMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (t *TypedMap) Get(key string) (int, bool) {
    t.mu.RLock()
    defer t.mu.RUnlock()
    v, ok := t.m[key]
    return v, ok
}

func (t *TypedMap) Set(key string, value int) {
    t.mu.Lock()
    defer t.mu.Unlock()
    t.m[key] = value
}
```

---

## sync.Pool

`sync.Pool` is a cache of temporary objects to reduce GC pressure. Great for reusing expensive-to-allocate objects (buffers, encoders, large structs).

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func formatMessage(msg string) string {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer func() {
        buf.Reset()
        pool.Put(buf)
    }()

    buf.WriteString("Message: ")
    buf.WriteString(msg)
    return buf.String()
}
```

### Caveats

- **Not a cache.** Objects may be removed at any time, especially during GC.
- Do not store objects with external resources without cleanup.
- The `New` function must return a non-nil value.
- **Always reset objects** before returning them to the pool.

```go
// BAD
buf := pool.Get().(*bytes.Buffer)
buf.WriteString("data")
pool.Put(buf) // buffer still contains old data

// GOOD
buf := pool.Get().(*bytes.Buffer)
buf.Reset()
defer func() {
    buf.Reset()
    pool.Put(buf)
}()
buf.WriteString("data")
```

---

## sync/atomic — Atomic Operations

The `sync/atomic` package provides lock-free atomic operations using hardware-level instructions. Faster than mutexes for single-variable operations.

### Core Operations

```go
import "sync/atomic"

var counter int64

// atomic increment
atomic.AddInt64(&counter, 1)

// atomic read / write
v := atomic.LoadInt64(&counter)
atomic.StoreInt64(&counter, 100)

// compare-and-swap (CAS) — fundamental lock-free primitive
swapped := atomic.CompareAndSwapInt64(&counter, 100, 200)

// swap
old := atomic.SwapInt64(&counter, newValue)
```

> 🧠 **Think of it as:** Atomics are like a tiny hardware-level lock on a single word — faster than a mutex, but only work when you're protecting one independent value.

### Full Type Table

| Function | Purpose |
|----------|---------|
| `atomic.AddInt32/AddInt64/AddFloat64` | Atomically add |
| `atomic.LoadInt32/Int64/Uint32/Uint64`, `LoadPointer` | Atomically read |
| `atomic.StoreInt32/Int64/...` | Atomically write |
| `atomic.CompareAndSwapInt32/Int64/...` | CAS |
| `atomic.SwapInt32/Int64/...` | Swap |
| `atomic.Value` | Holds any value atomically (Load/Store) |

### Atomic Counter Without a Mutex

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

var counter int64

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1)
        }()
    }
    wg.Wait()
    fmt.Println(counter)  // 1000 (always correct, no mutex)
}
```

### CompareAndSwap Pattern

CAS is the building block for lock-free algorithms:

```go
for {
    old := atomic.LoadInt64(&counter)
    new := old + 1
    if atomic.CompareAndSwapInt64(&counter, old, new) {
        break
    }
}
```

Use `AddInt64` for simple increments; reserve CAS for complex conditional updates.

### atomic.Value (For Arbitrary Types)

Stores any value atomically. Useful for configuration reloads:

```go
var config atomic.Value

config.Store(someConfig)

// Load with type assertion
cfg := config.Load().(Config)
```

> Note: `sync/atomic` types avoid the alignment issues when handling 64-bit values on 32-bit platforms. Use `atomic.LoadInt64` instead of plain reads for `int64` on 32-bit.

---

## When to Use What: Decision Guide

### Atomic vs Mutex

| Use atomic when | Use mutex when |
|-----------------|----------------|
| Single variable operations (counter, flag) | Multiple related variables updated together |
| No compound operations needed | Critical section has multiple statements |
| Performance is critical and profiling confirms it | Readability matters more |
| Implementing lock-free data structures | Building normal application logic |

```go
// Atomic - simple flag
var ready int32
atomic.StoreInt32(&ready, 1)

// Mutex - multiple variables updated together
var mu sync.Mutex
var total int64
var count int
mu.Lock()
total += amount
count++
mu.Unlock()
```

### Complete Primitive Selection Guide

| Need | Tool |
|------|------|
| Coordinate N goroutines finishing | `sync.WaitGroup` |
| Exactly-once lazy init | `sync.Once` / `sync.OnceValue` |
| Simple counter increment | `atomic.AddInt64` |
| Protect a struct's invariant | `sync.Mutex` (or `RWMutex` if read-heavy) |
| Read-heavy shared data | `sync.RWMutex` or `atomic.Value` |
| General shared state writes | `sync.Mutex` |
| Concurrency-safe key/value store | `sync.Mutex` + map (or `sync.Map` for specific cases) |
| Reusable object cache | `sync.Pool` |
| Pass data between goroutines | Channels (see [02-channels.md](02-channels.md)) |
| Cancel a group of goroutines | `context.Context` (see [04-concurrency-patterns.md](04-concurrency-patterns.md)) |
| Collect errors from goroutines | `errgroup` (see [04-concurrency-patterns.md](04-concurrency-patterns.md)) |

---

## Practical Example: Thread-Safe Cache with Expiration

```go
type Cache struct {
    mu      sync.RWMutex
    entries map[string]entry
    ttl     time.Duration
}

type entry struct {
    value     interface{}
    expiresAt time.Time
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    e, ok := c.entries[key]
    c.mu.RUnlock()
    if !ok || time.Now().After(e.expiresAt) {
        return nil, false
    }
    return e.value, true
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    c.entries[key] = entry{value: value, expiresAt: time.Now().Add(c.ttl)}
    c.mu.Unlock()
}

func (c *Cache) cleanup() {
    ticker := time.NewTicker(c.ttl / 2)
    defer ticker.Stop()
    for range ticker.C {
        c.mu.Lock()
        now := time.Now()
        for k, e := range c.entries {
            if now.After(e.expiresAt) {
                delete(c.entries, k)
            }
        }
        c.mu.Unlock()
    }
}
```

---

## Modern Practices

- Use **`sync.Mutex`** judiciously — prefer it over busy-waiting or sleep-based synchronization.
- Use **`atomic` operations** (`sync/atomic`) for simple counters and flags — they're lock-free and faster than mutexes for single-value updates.
- Use **`sync.OnceValue`** (Go 1.21+) for lazy singleton initialization with cleaner syntax than manual `Once.Do`.
- Use **`sync.Pool`** to reduce GC pressure for expensive-to-allocate objects (buffers, encoders, large structs).
- Keep **critical sections as short as possible** — do slow work (I/O, allocations) outside the lock.
- Prefer **channels for signaling** (done, cancel, start) and mutexes for protecting data.
- Use **`sync.RWMutex`** only when reads vastly outnumber writes; otherwise plain `Mutex` has less overhead.
- Return a **copy of data** read under a lock if callers keep it beyond the lock scope.
- Use `go vet` to catch copied locks.

---

## Common Mistakes

- **Copying a mutex after first use** — `go vet` catches this; always pass mutex-protected structs by pointer, never by value.
- **Not unlocking on early return** — use `defer mu.Unlock()` immediately after `Lock()` to ensure the lock is released on all paths.
- **Deadlock from nested locks** — locking the same `Mutex` twice in one goroutine (non-reentrant) hangs forever; use `RWMutex` or restructure.
- **Using `RWMutex` when writes dominate** — the reader lock overhead makes `RWMutex` slower than plain `Mutex` if most operations are writes.
- **Mutex in a value receiver** — methods with value receivers copy the struct (including the mutex), breaking synchronization.
- **Non-atomic reads of `int64` on 32-bit platforms** — use `atomic.LoadInt64` instead of plain reads to avoid torn reads.
- **Mixing RLock/RUnlock with Lock/Unlock** — `mu.RLock()` must pair with `mu.RUnlock()`, not `mu.Unlock()`.
- **Returning pooled objects without resetting** — reuse is broken if previous state leaks.
- **Double-checked locking without `sync.Once`** — the first check is racy; use `Once.Do` instead.
- **`WaitGroup.Add` race** — call `Add` in the tracker goroutine *before* starting the workers, never inside them.

---

## Key Takeaways

1. Data races cause unpredictable behavior — always synchronize shared access.
2. `sync.Mutex`: exclusive access; use `defer Unlock()`. Keep critical sections short.
3. `sync.RWMutex`: many readers / one writer; use when reads vastly dominate.
4. `sync.Once` / `sync.OnceValue`: run exactly once (lazy init, singleton).
5. `sync.WaitGroup`: wait for goroutines; always pass by pointer, `Add` before launching.
6. `sync.Map`: concurrency-safe map for specific patterns (write-once-read-many).
7. `sync.Pool`: reusable object cache to reduce GC pressure; always reset before returning.
8. `atomic`: lock-free fast operations on simple types; use for counters/flags.
9. Run `go test -race` to catch races automatically.
10. Never copy a `sync.Mutex` or `sync.WaitGroup` after first use.

## Next

Continue to [04-concurrency-patterns.md](04-concurrency-patterns.md) — worker pools, pipelines, cancellation with context, and more.
