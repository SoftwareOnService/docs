# Middleware

Middleware is the connective tissue of a Go HTTP application. It sits between
the raw HTTP request and your final handler, giving you a place to handle
cross-cutting concerns — logging, authentication, error recovery — without
duplicating logic across every handler.

---

## Table of Contents

1. [What Is Middleware?](#1-what-is-middleware)
2. [The Middleware Pattern](#2-the-middleware-pattern)
3. [Logging Middleware](#3-logging-middleware)
4. [CORS Middleware](#4-cors-middleware)
5. [Recovery Middleware](#5-recovery-middleware)
6. [Authentication Middleware](#6-authentication-middleware)
7. [Rate Limiting Middleware](#7-rate-limiting-middleware)
8. [Request ID Middleware](#8-request-id-middleware)
9. [Content-Type Middleware](#9-content-type-middleware)
10. [Middleware Chains](#10-middleware-chains)
11. [Practical Example: Complete Middleware Stack](#11-practical-example-complete-middleware-stack)
12. [Testing Middleware with httptest](#12-testing-middleware-with-httptest)

---

## 1. What Is Middleware?

A middleware is a function that wraps an `http.Handler` (or another middleware)
to add behavior before or after the request reaches the final handler.

```
Request → Middleware 1 → Middleware 2 → ... → Final Handler → Response
```

Each middleware does one thing: it receives a handler, performs some logic, then
calls the next handler in the chain. This design keeps responsibilities
isolated and composable.

```mermaid
graph LR
    A[Client Request] --> B[Recovery]
    B --> C[Request ID]
    C --> D[Logging]
    D --> E[CORS]
    E --> F[Auth]
    F --> G[Handler]
    G --> H[Client Response]
```

> 🔑 **Key idea:** Middleware is onion layers around your handler. Request travels
> inward through each wrapper; the response travels outward through the same
> layers in reverse — before/after `next.ServeHTTP`, that's where your logic runs.

---

## 2. The Middleware Pattern

The standard middleware signature in Go wraps `http.Handler`:

```go
func someMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // code before the handler
        next.ServeHTTP(w, r)
        // code after the handler
    })
}
```

This is the canonical pattern. The middleware receives the next handler and
returns a new handler that adds behavior around it. Every middleware in the
standard library and most third-party packages follow this shape.

> 🧠 **Memory aid:** "A middleware is a `func(http.Handler) http.Handler`" — it
> takes a handler, returns a handler. Once that signature is muscle memory, every
> middleware you read or write follows the same skeleton.

---

## 3. Logging Middleware

Logging middleware records details about each request: method, path, status
code, and how long the handler took to execute.

```go
func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        wrapped := &statusRecorder{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)

        log.Printf(
            "%s %s %d %s",
            r.Method,
            r.URL.Path,
            wrapped.statusCode,
            time.Since(start),
        )
    })
}

type statusRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (rec *statusRecorder) WriteHeader(code int) {
    rec.statusCode = code
    rec.ResponseWriter.WriteHeader(code)
}
```

The `statusRecorder` wraps `http.ResponseWriter` to intercept `WriteHeader`
calls so the middleware can log the actual status code the handler returned. If
the handler never explicitly sets a status code, it defaults to 200.

> 💡 **Pro tip:** Embedding `http.ResponseWriter` and overriding `WriteHeader` is
> the standard trick for intercepting status codes. Don't forget to also override
> `Write` if you want to capture body bytes — and to forward everything else to
> the embedded writer.

---

## 4. CORS Middleware

Cross-Origin Resource Sharing (CORS) headers tell browsers whether a
cross-origin request is allowed. A CORS middleware sets the appropriate headers
on every response.

```go
func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

The `OPTIONS` preflight check is a browser mechanism. The server must respond
to it with the correct headers and a 204 status before the browser proceeds
with the actual request. This middleware handles that automatically.

In production, replace `*` with specific allowed origins and restrict headers
to what your API actually needs.

> ⚠️ **Watch out:** `Access-Control-Allow-Origin: *` is a debugging convenience,
> not a production policy. Pair it with restrictions on `Allow-Credentials` —
> `*` plus credentials is silently rejected by browsers, and the audit trail gets
> messy.

---

## 5. Recovery Middleware

A panic in a handler goroutine will crash the entire server if not recovered.
Recovery middleware catches panics so the server keeps running and returns a
500 error to the client.

```go
func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic recovered: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()

        next.ServeHTTP(w, r)
    })
}
```

Always log the panic value and the stack trace. A silent recovery makes
debugging in production nearly impossible. Place this middleware early in the
chain so it wraps everything else.

> ⚠️ **Gotcha:** Only recover what you must — a silent `recover()` without logging
> the stack turns a crash into a mysterious 500 with no trace. `debug.Stack()` is
> your only breadcrumb; always log it.

---

## 6. Authentication Middleware

Authentication middleware checks for valid credentials before allowing a
request to reach protected handlers. A common pattern is checking an API key
or bearer token.

```go
func Auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "missing authorization header", http.StatusUnauthorized)
            return
        }

        if token != "Bearer my-secret-token" {
            http.Error(w, "invalid token", http.StatusForbidden)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

In a real application, the token check would involve JWT parsing, database
lookups, or calls to an auth service. The middleware pattern stays the same.

You can also extract user information from the token and attach it to the
request context so downstream handlers can access it:

```go
func AuthWithUser(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        user, err := validateToken(token)
        if err != nil {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }

        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

> ⚠️ **Watch out:** String keys in `context.WithValue` collide across packages.
> Define an unexported key type (`type ctxKey string`) and a typed accessor —
> otherwise two libraries using `"user"` silently stomp each other.

---

## 7. Rate Limiting Middleware

Rate limiting protects your server from abuse by capping the number of requests
a client can make within a time window. A simple in-memory implementation uses
a map of client IP to request counts.

```go
func RateLimit(maxRequests int, window time.Duration) func(http.Handler) http.Handler {
    type client struct {
        count    int
        lastSeen time.Time
    }

    var (
        mu      sync.Mutex
        clients = make(map[string]*client)
    )

    // goroutine to periodically clean up old entries
    go func() {
        for {
            time.Sleep(window)
            mu.Lock()
            for ip, c := range clients {
                if time.Since(c.lastSeen) > window {
                    delete(clients, ip)
                }
            }
            mu.Unlock()
        }
    }()

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := r.RemoteAddr

            mu.Lock()
            c, exists := clients[ip]
            if !exists {
                clients[ip] = &client{count: 1, lastSeen: time.Now()}
                mu.Unlock()
                next.ServeHTTP(w, r)
                return
            }

            if time.Since(c.lastSeen) > window {
                c.count = 1
                c.lastSeen = time.Now()
                mu.Unlock()
                next.ServeHTTP(w, r)
                return
            }

            if c.count >= maxRequests {
                mu.Unlock()
                http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
                return
            }

            c.count++
            c.lastSeen = time.Now()
            mu.Unlock()

            next.ServeHTTP(w, r)
        })
    }
}
```

Note the function signature: this is a middleware factory. It returns a
middleware function and closes over its own state. This lets you configure
different limits for different routes.

For production use, consider a token bucket algorithm (available in
`golang.org/x/time/rate`) or a distributed rate limiter backed by Redis.

> 💡 **Note:** The factory signature `func(...) func(http.Handler) http.Handler`
> closes over its own state — that's how each route gets its own limit. It also
> means shared mutable maps need a mutex; the counter map here is lock-guarded
> for a reason.

---

## 8. Request ID Middleware

A unique request ID makes it possible to trace a single request across log
lines, especially in systems with multiple services or goroutines.

```go
func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = generateID()
        }

        w.Header().Set("X-Request-ID", id)
        ctx := context.WithValue(r.Context(), "requestID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func generateID() string {
    b := make([]byte, 16)
    _, _ = rand.Read(b)
    return fmt.Sprintf("%x", b)
}
```

If the client already sends a request ID, honor it. Otherwise generate one.
The middleware stores it in the response header so clients can use it for
debugging, and in the request context so other middleware and handlers can
access it.

> 💡 **Pro tip:** Correlate IDs with your logs as early as possible — `RequestID`
> should sit near the top of the chain so every downstream log line carries the
> same trace ID. Honor inbound IDs for multi-service traces.

---

## 9. Content-Type Middleware

Content-Type middleware enforces that incoming requests have the correct
`Content-Type` header. This is useful for APIs that only accept JSON.

```go
func RequireJSON(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == http.MethodPost || r.Method == http.MethodPut || r.Method == http.MethodPatch {
            ct := r.Header.Get("Content-Type")
            if ct != "application/json" {
                http.Error(w, "Content-Type must be application/json", http.StatusUnsupportedMediaType)
                return
            }
        }

        next.ServeHTTP(w, r)
    })
}
```

This middleware only checks methods that typically carry a request body. GET
requests do not need a Content-Type header.

---

## 10. Middleware Chains

The real power of middleware is composition. You combine multiple middleware
into a chain by nesting them. The request passes through each layer in order.

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/users", handleUsers)
    mux.HandleFunc("/api/orders", handleOrders)

    // Build the chain: Request → Recovery → Logging → CORS → Auth → Handler
    var handler http.Handler = mux
    handler = Auth(handler)
    handler = CORS(handler)
    handler = Logging(handler)
    handler = Recovery(handler)

    log.Println("server listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", handler))
}
```

The order matters. `Recovery` is outermost so it catches panics from any
middleware below it. `Logging` wraps `Auth` so it can measure the total time
including authentication checks. `CORS` is inside `Logging` so preflight
OPTIONS requests are also logged.

> 🔑 **Remember:** Order is semantic — Recovery outermost, logging broadly, auth as
> a gate near the handler. Inverting them (e.g., Recovery inside Logging) silently
> changes what gets caught and what gets timed.

### Chain Helper Function

Another approach is a helper function that takes a handler and a list of
middleware:

```go
func Chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}
```

Usage:

```go
handler := Chain(
    mux,
    Recovery,
    Logging,
    CORS,
    Auth,
)
```

This reads left-to-right: Recovery is applied last (outermost), Auth is
applied first (innermost). The `Chain` function eliminates the error-prone
manual nesting.

> 🧠 **Think of it as:** `Chain` reads like a coat check: list the outermost layer
> first, and the helper walks the slice backwards so the first item ends up
> wrapping everything else.

### Middleware Wrapping Order

```mermaid
graph TB
    subgraph "Outermost"
        R[Recovery]
    end
    subgraph ""
        L[Logging]
    end
    subgraph ""
        C[CORS]
    end
    subgraph "Innermost"
        A[Auth]
    end
    subgraph "Handler Layer"
        M[ServeMux + Handlers]
    end
    R --> L --> C --> A --> M

    style R fill:#f66,color:#fff
    style L fill:#fa0,color:#000
    style C fill:#4a4,color:#fff
    style A fill:#48f,color:#fff
    style M fill:#888,color:#fff
```

---

## 11. Practical Example: Complete Middleware Stack

Here is a full working example that combines all the middleware discussed in
this article into a single application.

```go
package main

import (
    "context"
    "crypto/rand"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"
)

// --- Middleware ---

func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic recovered: %v", err)
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            b := make([]byte, 16)
            _, _ = rand.Read(b)
            id = fmt.Sprintf("%x", b)
        }
        w.Header().Set("X-Request-ID", id)
        ctx := context.WithValue(r.Context(), "requestID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &statusRecorder{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)
        log.Printf("[%s] %s %d %s", r.Header.Get("X-Request-ID"), r.URL.Path, wrapped.statusCode, time.Since(start))
    })
}

func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        next.ServeHTTP(w, r)
    })
}

func RequireJSON(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == http.MethodPost || r.Method == http.MethodPut {
            if ct := r.Header.Get("Content-Type"); ct != "application/json" {
                http.Error(w, "Content-Type must be application/json", http.StatusUnsupportedMediaType)
                return
            }
        }
        next.ServeHTTP(w, r)
    })
}

func Auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token != "Bearer secret" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

type statusRecorder struct {
    http.ResponseWriter
    statusCode int
}

func (rec *statusRecorder) WriteHeader(code int) {
    rec.statusCode = code
    rec.ResponseWriter.WriteHeader(code)
}

// --- Chain helper ---

func Chain(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

// --- Handlers ---

func publicHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "this endpoint requires no auth")
}

func protectedHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "welcome, authenticated user")
}

func main() {
    publicMux := http.NewServeMux()
    publicMux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ok")
    })
    publicMux.HandleFunc("/public", publicHandler)

    protectedMux := http.NewServeMux()
    protectedMux.HandleFunc("/api/data", protectedHandler)

    mux := http.NewServeMux()
    mux.Handle("/health", publicMux)
    mux.Handle("/public", publicMux)
    mux.Handle("/api/", Auth(protectedMux))

    handler := Chain(
        mux,
        Recovery,
        RequestID,
        Logging,
        CORS,
        RequireJSON,
    )

    log.Println("server listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", handler))
}
```

This example shows how different endpoints can have different middleware
requirements. The `/health` and `/public` routes skip `Auth`. The `/api/` routes
pass through authentication. All routes get recovery, logging, CORS, and
request ID middleware.

---

## 12. Testing Middleware with httptest

Middleware wraps a handler. Test it by wrapping a known handler and checking
behavior.

```go
func TestRequireAuth(t *testing.T) {
    var called bool
    wrapped := requireAuth(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        called = true
        w.WriteHeader(http.StatusOK)
    }))

    // Without token → 401, handler not called
    req := httptest.NewRequest(http.MethodGet, "/", nil)
    rec := httptest.NewRecorder()
    wrapped.ServeHTTP(rec, req)
    if rec.Code != http.StatusUnauthorized {
        t.Fatalf("want 401, got %d", rec.Code)
    }
    if called {
        t.Fatal("handler should not run without auth")
    }

    // With token → 200, handler called
    called = false
    req2 := httptest.NewRequest(http.MethodGet, "/", nil)
    req2.Header.Set("Authorization", "secret")
    rec2 := httptest.NewRecorder()
    wrapped.ServeHTTP(rec2, req2)
    if rec2.Code != http.StatusOK {
        t.Fatalf("want 200, got %d", rec2.Code)
    }
    if !called {
        t.Fatal("handler should have run")
    }
}
```

### Testing CORS with httptest

```go
func TestCORS(t *testing.T) {
    inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ok")
    })
    wrapped := CORS(inner)

    // Normal request → CORS headers set
    req := httptest.NewRequest(http.MethodGet, "/data", nil)
    rec := httptest.NewRecorder()
    wrapped.ServeHTTP(rec, req)

    if rec.Header().Get("Access-Control-Allow-Origin") != "*" {
        t.Fatal("missing Access-Control-Allow-Origin header")
    }

    // OPTIONS preflight → 204 with CORS headers
    req2 := httptest.NewRequest(http.MethodOptions, "/data", nil)
    rec2 := httptest.NewRecorder()
    wrapped.ServeHTTP(rec2, req2)

    if rec2.Code != http.StatusNoContent {
        t.Fatalf("want 204, got %d", rec2.Code)
    }
}
```

> 🔑 **Key idea:** To test middleware, wrap a spy handler (a `HandlerFunc` that
> flips a flag) and assert on both the recorded response AND whether the inner
> handler ran — that's how you prove gating like Auth actually short-circuits.

---

## Modern Practices

- **Place Recovery outermost** so it catches panics from all middleware and
  handlers below it in the chain.
- **Use middleware factories** (functions that return middleware) when you need
  configurable behavior — `RateLimit(100, time.Minute)` is clearer than
  hardcoding limits.
- **Pass user data via `context.WithValue`** in auth middleware, not via global
  variables. Use typed context keys to avoid collisions.
- **Use `httptest.NewRecorder`** to test middleware in isolation. Wrap a stub
  handler and assert on the recorded response.
- **Consider `golang.org/x/time/rate`** for production rate limiting — it
  implements a token bucket algorithm and is more efficient than simple
  counter-based approaches.
- **Log structured data** (request ID, method, path, status, duration) instead
  of free-form text. Use `log/slog` (Go 1.21+) for leveled, structured
  logging.
- **Apply middleware selectively** — use sub-muxes or route-specific wrapping
  so that public endpoints skip auth and internal endpoints skip CORS.

---

## Common Mistakes

- **Forgetting to call `next.ServeHTTP`.** If your middleware never calls the
  next handler, the request dies in the middleware and the client receives no
  response (or an empty one). Always call `next.ServeHTTP` unless you are
  intentionally short-circuiting the request.
- **Ordering middleware incorrectly.** Recovery must be outermost to catch
  panics from other middleware. If you put Recovery after Logging, a panic in
  Logging will not be caught. Think carefully about the order of your chain.
- **Writing headers after calling `next.ServeHTTP`.** Once you call
  `next.ServeHTTP`, the handler may have already written the response headers.
  Any `w.Header().Set()` calls after that point will have no effect. Set
  response headers before calling the next handler, or do it after the handler
  returns but before writing any new headers.
- **Using `sync.Mutex` incorrectly in concurrent middleware.** Rate limiters and
  caches accessed by multiple goroutines need proper synchronization. A plain map
  without a mutex will cause a data race. Use `sync.Mutex` or `sync.Map`.
- **Not setting `Content-Type` on error responses.** When a middleware returns
  early with `http.Error`, the default Content-Type is `text/plain`. If your API
  returns JSON, consider writing a JSON error body and setting the header
  explicitly.
- **Ignoring the request context.** Middleware that blocks for a long time without
  checking `r.Context().Done()` will not respect client disconnects or server
  shutdown signals. Use `select` with the context when doing work that could be
  slow.

---

## Exercises

1. **Conditional logging.** Modify the logging middleware to skip logging for
   requests to `/health`. The middleware should check the request path before
   calling the next handler.

2. **Panic recovery with JSON responses.** Write a recovery middleware that
   returns errors as JSON (`{"error": "Internal Server Error"}`) instead of
   plain text. Set the `Content-Type` header to `application/json`.

3. **JWT authentication middleware.** Replace the simple bearer token check
   with a middleware that parses and validates a JWT. Extract the user ID from
   the token claims and store it in the request context using
   `context.WithValue`.

4. **Middleware unit tests.** Write tests for the CORS middleware using
   `httptest.NewRecorder`. Verify that `Access-Control-Allow-Origin` is set on
   normal requests and that OPTIONS requests return a 204 with the correct
   headers.

5. **Sliding window rate limiter.** Replace the fixed-window rate limiter
   with a sliding window implementation. Use a sorted list of timestamps per
   client instead of a simple counter.

---

## Key Takeaways

1. Middleware wraps `http.Handler` — it receives a handler, returns a new handler
   with added behavior before/after calling the next handler.
2. **Logging** needs a `statusRecorder` to capture the status code. **Recovery**
   must be outermost. **CORS** handles OPTIONS preflight. **Auth** checks tokens
   and attaches user data to context.
3. **Middleware order matters** — Recovery outermost, logging wrapping auth,
   CORS inside logging. Use the `Chain` helper for clean composition.
4. **Middleware factories** (functions returning middleware) enable configurable
   behavior — `RateLimit(100, time.Minute)` closes over its own state.
5. **Test middleware** by wrapping a stub handler with `httptest.NewRecorder` and
   asserting on status codes and headers.

---

## Next

Continue to [04-graceful-shutdown-rest-api.md](04-graceful-shutdown-rest-api.md)
for graceful shutdown, signal handling, and a complete production-style REST
API.
