# Part 11 — go-zero Framework

The **go-zero** framework (by [Zeromicro](https://github.com/zeromicro/go-zero))
is a cloud-native Go web & microservices framework. It follows a philosophy of
**"prefer tools over conventions and documents"**: instead of hand-writing
routers, handlers, DTO structs, service stubs and client SDKs, you describe
your API in a tiny DSL (`.api`/`.proto`) and let **goctl** generate
production-ready code for HTTP APIs, gRPC (zRPC) services, DB model layers,
Dockerfiles, Kubernetes manifests and multi-language clients.

This part assumes you have already completed the earlier parts of
*Go Complete Notes* — especially **Part 7 (HTTP backend)** and
**Part 8 (gRPC/Protobuf)** — because go-zero builds HTTP and gRPC on top of
`net/http` and `google.golang.org/grpc`. If those concepts are unfamiliar,
review them first.

## Table of contents of this part

| File | Covers |
|------|--------|
| `01-introduction.md` | What go-zero is, philosophy, architecture, install |
| `02-api-dsl-and-goctl.md` | The `.api` DSL, goctl code generation, project layout |
| `03-http-api-services.md` | Building HTTP API services, routes, handlers, logic, config |
| `04-rpc-zrpc-services.md` | zRPC + Protocol Buffers, service discovery, API calling RPC |
| `05-advanced-patterns.md` | Middleware, interceptors, httpx helpers, validation, cache, errors |
| `06-production-patterns.md` | Metrics, tracing, Docker/K8s, deployment, modern practices & mistakes |
| `07-go-zero-vs-gin.md` | go-zero vs Gin: when to choose which |

> 🔑 **Remember:** each file follows the same arc — describe, generate, implement logic, then production concerns. Files 01–04 are the core loop; 05–06 harden it.

## Quick start (60 seconds)

```bash
# 1. Install goctl (go-zero's code-generation CLI)
go install github.com/zeromicro/go-zero/tools/goctl@latest

# 2. Scaffold a Hello-World HTTP API
goctl api new greet
cd greet
go mod tidy

# 3. Run it
go run greet.go

# 4. Call it
curl http://localhost:8888/from/you
# {"message":"Hello you"}
```

> 🧠 **Think of it as:** the "60 seconds" works because goctl writes the ~95% boilerplate — entrypoint, routes, handlers, types — and leaves you with just the 5% business logic.

That single `goctl api new greet` command produces a complete, production-ready
project layout:

```
greet/
├── etc/
│   └── greet-api.yaml      # configuration (port, etc.)
├── internal/
│   ├── config/             # config struct (mirrors the yaml)
│   ├── handler/            # HTTP handlers (auto-registered)
│   ├── logic/              # business logic ← you implement this
│   ├── middleware/         # custom middleware hooks
│   ├── svc/                # service context (shared deps: db, redis)
│   └── types/              # request/response types (from the DSL)
├── greet.go                # entrypoint
└── greet.api               # DSL source of truth
```

> **Rule to remember**: edit only the files in `internal/logic/` (and
> `internal/svc`, `internal/config`, `internal/middleware`). Everything else is
> regenerated from the `.api` file whenever you run `goctl api go -api greet.api -dir .`.
> goctl never overwrites your `logic/` files.

> 💡 **Pro tip:** keep one `etc/*.yaml` per environment (dev/prod) and pass it at startup with `-f` — ports, log level, and endpoints live in the yaml, not in code.

## Prerequisites

- Go **1.21+** (go-zero's minimum)
- `goctl` (the CLI) — install as above
- `protoc` + `protoc-gen-go` + `protoc-gen-go-grpc` — **only** for zRPC/RPC parts

## Recommended learning path

```
01-introduction → 02-api-dsl-and-goctl → 03-http-api-services
   → 04-rpc-zrpc-services → 05-advanced-patterns
   → 06-production-patterns → 07-go-zero-vs-gin
```

Then apply everything in the capstone project:
`../13-projects/06-go-zero-microservice.md`.

---

Each file in this part includes **Mermaid diagrams**, a **Modern Practices**
section, a **Common Mistakes** section, **Key Takeaways**, and exercises, in the
same style as the rest of *Go Complete Notes*.
