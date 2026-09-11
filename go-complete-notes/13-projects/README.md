# Part 13: Projects

Hands-on projects that tie together everything from Parts 01–12. Each project
builds on the skills of the previous ones. Work through them in order for a
guided progression, or jump to whichever matches your interest.

---

## Project Index

| #  | Project                          | Difficulty | Lines of Code | Description                                                        |
|----|----------------------------------|------------|---------------|--------------------------------------------------------------------|
| 01 | [CLI Todo App](01-cli-todo.md)   | Beginner   | ~200          | Stdlib-only command-line todo list with JSON persistence           |
| 02 | [HTTP URL Shortener](02-http-url-shortener.md) | Intermediate | ~350 | URL shortener with net/http, middleware, graceful shutdown         |
| 03 | [Concurrent Web Crawler](03-concurrency-web-crawler.md) | Intermediate | ~400 | Worker-pool crawler with channels, deduplication, rate limiting    |
| 04 | [REST API Todo + PostgreSQL](04-rest-api-todo-postgres.md) | Intermediate-Advanced | ~500 | Full CRUD REST API with database/sql, JWT auth, migrations         |
| 05 | [gRPC Greeter Service](05-grpc-greeter-service.md) | Intermediate | ~450 | Protobuf definitions, server, client, streaming, interceptors      |
| 06 | [go-zero Microservice](06-go-zero-microservice.md) | Intermediate | ~350 | CRUD API with go-zero framework: API definition, codegen, config   |
| 07 | [Capstone: Notes App](07-fullstack-backend-notes-app.md) | Advanced | ~800 | End-to-end REST backend: auth, PostgreSQL, Redis, Docker, metrics  |

> 💡 **Tip:** Work through the projects in order — each one builds on
> patterns from the last, and Project 07 assumes you can do everything
> before it.

---

## Difficulty Levels

- **Beginner** — only stdlib basics (Part 01–03). Great warm-up.
- **Intermediate** — requires concurrency, HTTP, or gRPC (Parts 04–08).
- **Intermediate-Advanced** — combines multiple parts (database, auth, patterns).
- **Advanced** — full production-style system (Parts 07–12 all exercised).

---

## Prerequisites Map

| Project | Parts Used                                        |
|---------|---------------------------------------------------|
| 01      | 01 (Go Fundamentals), 02 (Collections), 03 (Types/Pointers/Functions) |
| 02      | 07 (HTTP Backend — net/http, middleware, testing)  |
| 03      | 04 (Concurrency), 05 (Standard Library — net/http) |
| 04      | 05 (Std Lib), 06 (Testing), 07 (HTTP), 09 (Gin), 10 (Databases), 12 (Auth) |
| 05      | 08 (gRPC/Protobuf)                                |
| 06      | 11 (go-zero)                                      |
| 07      | 04, 05, 06, 07, 08, 09, 10, 11, 12               |

> 🔑 **Checkpoint:** If a project's prerequisites list looks unfamiliar,
> skim the referenced part first instead of pushing through — the projects
> assume those concepts, not re-teach them.

---

## Suggested Order

```
Project 01  (stdlib basics — build confidence)
    ↓
Project 02  (HTTP server — first real networked app)
    ↓
Project 03  (concurrency — master goroutines + channels)
    ↓
Project 04  (database + auth — production API)
    ↓
Project 05  (gRPC — binary protocol, streaming)
    ↓
Project 06  (go-zero — codegen, microservice patterns)
    ↓
Project 07  (capstone — everything together)
```

---

## Tips

1. **Type every code block yourself.** Don't copy-paste. Muscle memory matters.
2. **Run the tests.** Every project includes `go test ./...` commands.
3. **Break things on purpose.** Delete an import, rename a function, corrupt
   the JSON file — see what happens and learn from the errors.
4. **Run `go vet ./...` and `staticcheck ./...`** after each project.
5. **Commit each project separately** with a meaningful message.

> ⚠️ **Watch out:** Don't skip the `go test ./...` step — a project that
> "looks right" but fails its tests is unfinished. Break things on purpose
> so you recognize the errors later.

---

## Next

Start with [01-cli-todo.md](01-cli-todo.md) — a beginner CLI todo app using
only the Go standard library.
