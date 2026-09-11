# Go Complete Notes 🚀

A single, unified, deeply detailed resource for learning Go from **zero** to
**production backend developer** — plus the **go-zero** framework and hands-on
**projects**.

This resource is the result of blending two existing note sets into ONE
coherent reference:

1. **Go-Language-Notes** — rich study notes with theory, Mermaid diagrams,
   Modern Practices & Common Mistakes sections.
2. **Go-Mastery-Course** — a 49-article course taking you to production-grade
   backends (gRPC, Gin, PostgreSQL, auth, production engineering).

Everything is merged here so you only need **this** folder. The two originals
are left untouched.

> 💡 **Pro tip:** Read the Parts in order for a guided path, or jump to any topic using the Table of Contents below.

## How to use this resource

Read the Parts in order for a guided path, or jump to any topic. Every file
follows a consistent style:

- Theory with **Mermaid diagrams** (render on GitHub/GitLab/VS Code with the
  Markdown Preview Mermaid extension)
- Code you can copy into `.go` files
- **Modern Practices** — current, idiomatic Go
- **Common Mistakes** — concrete errors to avoid
- Quick reference tables and exercises

> 🧠 **Think of it as:** one self-contained study guide — theory, diagrams, and idiomatic code side by side.

## Table of Contents

| Part | Folder | Covers |
|------|--------|--------|
| 1 | `01-go-fundamentals` | Getting started, toolchain, modules, variables, types, strings, control flow, functions |
| 2 | `02-collections` | Arrays, slices, maps, strings deep, data structures |
| 3 | `03-types-pointers-functions` | Structs, methods, pointers, interfaces, generics, error handling |
| 4 | `04-concurrency` | Goroutines, channels, sync, concurrency patterns |
| 5 | `05-standard-library` | io/os, strings/strconv, time/json, context/slog, packages, project structure |
| 6 | `06-testing` | Unit tests, benchmarks, fuzzing, TDD, backend/HTTP testing |
| 7 | `07-http-backend` | net/http, middleware, graceful shutdown, REST APIs |
| 8 | `08-grpc-protobuf` | gRPC & Protocol Buffers end-to-end |
| 9 | `09-gin-framework` | The Gin web framework |
| 10 | `10-databases` | SQL, PostgreSQL, database/sql |
| 11 | `11-go-zero` | The go-zero web & microservices framework |
| 12 | `12-architecture-auth-production` | App architecture, auth/security, production backend |
| 13 | `13-projects` | Hands-on projects of increasing difficulty |

## Learning path

```
Go basics
  -> fundamentals
    -> collections & types
      -> concurrency
        -> standard library
          -> testing
            -> HTTP with net/http
              -> gRPC / Protocol Buffers
                -> Gin framework
                  -> PostgreSQL
                    -> go-zero framework
                      -> architecture, auth & production
                        -> projects
```

## Prerequisites

- [Install Go](https://go.dev/dl/) (Go 1.22+ recommended for the newest
  features — per-iteration loop vars, method-based routing)
- A text editor or IDE (VS Code + Go extension / gopls, or GoLand)
- Basic programming experience in any language

> ⚠️ **Watch out:** Mermaid diagrams render on GitHub/GitLab or in VS Code with the Markdown Preview Mermaid extension — they won't render in a plain text editor.

> Throughout the notes, toolchain commands assume Go 1.18+ (generics) and
> often Go 1.21+ (`slices`/`maps`, `log/slog`, `errors.Join`) and Go 1.22+
> (loop variables, method-based routing, `r.PathValue`).
