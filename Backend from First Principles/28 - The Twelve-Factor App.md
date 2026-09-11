---
title: "The Twelve-Factor App"
tags: [backend, video-notes, twelve-factor, architecture]
course: "[[_00 - Backend from First Principles - Index]]"
source: "https://www.youtube.com/watch?v=2_fcv3-XfN4"
video_id: "2_fcv3-XfN4"
playlist_position: 28
duration_seconds: 5598
published: "2026-09-02"
status: completed
---

# Part 28 — The Twelve-Factor App

> [!info] Video Reference
> **Title:** 27. The Twelve-Factor App  
> **Channel:** Sriniously  
> **Playlist:** Backend from First Principles (Video 28 of 29)  
> **URL:** https://www.youtube.com/watch?v=2_fcv3-XfN4  
> **Duration:** 1 hour 33 minutes 18 seconds (5598 seconds)  
> **Published:** 2026-09-02

> [!abstract] In This Chapter
> This chapter provides an exhaustive walkthrough of the **Twelve-Factor App** methodology — a contract between your application and the platform that runs it. Originally authored by Heroku co-founder Adam Wiggins in 2011, the twelve factors codify patterns observed from running hundreds of thousands of apps on one of the first PaaS platforms. The video covers:
> 
> - **Historical context:** Heroku 2011, "software erosion," snowflake servers, and the pre-cloud deployment nightmare
> - **The 2024 open-sourcing** and community revision (GitHub `twelve-factor/twelve-factor`, `next` branch)
> - **All twelve factors** with original 2011 text, modern reinterpretation, anti-patterns, and where each has aged well or poorly:
>   1. **Codebase** — One codebase tracked in revision control, many deploys
>   2. **Dependencies** — Explicitly declare and isolate dependencies
>   3. **Config** — Store config in the environment (the open-source litmus test, secrets vs. config)
>   4. **Backing Services** — Treat backing services as attached resources (resource handles)
>   5. **Build, Release, Run** — Strictly separate build, release, and run stages (immutable releases, instant rollbacks)
>   6. **Processes** — Execute the app as one or more stateless processes (share-nothing, sticky sessions, WebSocket exception)
>   7. **Port Binding** — Export services via port binding (self-contained HTTP server, serverless exception)
>   8. **Concurrency** — Scale out via the process model (process types, process formation, horizontal scaling)
>   9. **Disposability** — Maximize robustness with fast startup and graceful shutdown (SIGTERM, crash-only design, re-entrancy)
>   10. **Dev/Prod Parity** — Keep development, staging, and production as similar as possible (time/person/tools gaps, managed services trade-off)
>   11. **Logs** — Treat logs as event streams (stdout, unbuffered, platform captures/routing, observability evolution)
>   12. **Admin Processes** — Run admin/management tasks as one-off processes (identical environment, database migrations, Procfile demo)
> - **Live demo** of a real `Procfile` formation: boot, one-off job, graceful shutdown
> - **Key Takeaways**, **Related Notes**, and **Source Fidelity** note

## 1. Origin Story: Heroku, 2011, and Software Erosion [05:07]

### The Pre-Cloud Deployment Nightmare
Before platforms like Heroku, deploying an application meant **renting a physical server** (or worse, running one in your office). AWS existed but wasn't yet standard practice. The deployment process was entirely manual:

1. **Provision hardware** — rent a server or use an on-premise machine
2. **Install OS by hand** — Ubuntu or another Linux distribution
3. **Install language runtime** — Java, Go, Python, etc., manually
4. **Install database and supporting libraries** — PostgreSQL, Redis, ImageMagick, etc.
5. **Copy application code** — often via FTP (File Transfer Protocol)
6. **Manually edit config files** on the server via SSH
7. **Run the application** — hope it works

Every server built this way was a **snowflake server** — unique, hand-built, and **impossible to reproduce**. Even with documented runbooks, slight variations in manual steps led to drift. If the machine died, you started from scratch. Onboarding a new developer took days, and their local environment never matched production — hence the infamous phrase: **"It works on my machine."**

### Software Erosion
The Twelve-Factor document names this phenomenon **software erosion**: an application eventually stops working even though no one changed the code. Causes include:
- OS patches breaking compatibility
- Shared libraries updated by another app on the same server
- The sole engineer who understood the infrastructure leaving without documentation
- Manual changes that were never tracked

The twelve factors are **defenses against erosion** — a contract ensuring any machine, platform, or teammate can run, scale, and update your app without surprises.

### The Heroku Perspective
Heroku (founded ~2008) was a **Platform as a Service (PaaS)**: you give them code; they handle building, environment prep, config, database connections, networking, and making it internet-reachable. By 2011, they'd seen hundreds of thousands of apps deploy, scale, break, and corrupt across languages and frameworks. This visibility enabled Adam Wiggins (co-founder) to distill patterns into the twelve factors, published at **12factor.net**.

### The 2024 Open-Sourcing and Community Revision [12:05]
For 13 years the document was frozen (last updated 2017, minimally). Meanwhile, the ecosystem transformed: containers, Kubernetes, serverless, managed cloud everywhere. In **November 2024**, Heroku (now owned by Salesforce) open-sourced the document:
- Moved to a **public repo** under neutral organization: `github.com/twelve-factor/twelve-factor`
- Wrote a **governance document**
- Invited maintainers from AWS, Google Cloud, and the community
- Revision happens in public on the **`next` branch**

The rewrite **keeps all 12 names and one-liners** but expands each factor into a numbered list of principles, **separating timeless principles from 2011-era specifics** and adding modern principles applicable to today's applications. As of the video recording, the live site still shows the classic 2011 text; the rewrite is a working draft.

> [!note] Video Structure
> For each factor, the video covers: (1) original 2011 text, (2) the problem it solved, (3) where we've seen it in this playlist, (4) what the community rewrite changes, (5) what aged well vs. what aged poorly.

## 2. Factor I — Codebase: One Codebase, Many Deploys [14:23]

### Original 2011 Definition
> **"One codebase tracked in revision control, many deploys."**

Two parts here:

#### Part 1: One Codebase Tracked in Revision Control
In 2011, this was not obvious. Many production servers were just a folder on someone's computer or a ZIP file named `final_v2_really_final.zip`. Making this a rule forced teams to treat version control (Git, then Subversion/Mercurial) as a standard, not an afterthought.

#### Part 2: Many Deploys
A **deploy** = one running instance of your application. Production is a deploy; staging is a deploy; a developer's laptop is a deploy; a PR preview environment is a deploy. The key insight: **the same codebase serves all deploys**. What differs between deploys is **never the code** — it's **config** (Factor III).

**Example:**
- Repo at commit 100
- Production running commit 97 (deployed Friday)
- Staging running commit 99 (testing tomorrow's release)
- Laptop running commit 100 + 2 unpushed commits
- On deploy day: staging → 100, then production → 100

**Takeaway:** If you cannot describe your environment as "the same codebase at commit X," the pattern is broken.

### Boundaries: What This Factor Forbids
1. **Multiple codebases for one app** → That's a **distributed system**, not a single app. Each component must follow the twelve factors independently.
2. **Shared code via copy-paste** → Shared code must be refactored into a **library** and consumed via a **dependency manager** (Factor II).

### Modern Reinterpretation (2024 Rewrite) & Monorepos
The rewrite clarifies: a **codebase** is "a repository or any set of repositories which shares a root commit." The litmus test: **one-to-one mapping between codebase and app**.

**Monorepos are NOT an anti-pattern.** If a monorepo builds five different apps, it's equivalent to five separate codebases that happen to share a repo — each has its own twelve factors. The anti-pattern is **the same app existing in two places** (copy-paste sharing), not multiple apps in one repo.

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Git as universal VCS | ✅ Aged perfectly |
| Many deploys from one codebase | ✅ Core to CI/CD, preview environments |
| Monorepo confusion | ⚠️ Clarified by rewrite: not forbidden |

## 3. Factor II — Dependencies: Explicitly Declare and Isolate [20:58]

### Original 2011 Definition
> **"Explicitly declare and isolate dependencies."**

Two critical words: **declare** and **isolate**.

#### Declaration: The Manifest File
Somewhere in your repo, a file must **completely and accurately list every library** your app needs to run. We know these as **manifest files**:
- Node.js: `package.json`
- Go: `go.mod`
- Python: `requirements.txt` / `pyproject.toml`
- Ruby: `Gemfile`
- Java: `pom.xml` / `build.gradle`
- Rust: `Cargo.toml`

**Problem solved:** Without a manifest, a new teammate (or new laptop) must dig through every `import` statement, guess versions, resolve conflicts — a 2–3 day setup ordeal. With a manifest, **one command** (`npm install`, `go mod download`, `pip install -r requirements.txt`) pulls exact versions and gets the app running.

#### Isolation: No Leakage from the System
Declaration alone fails if your app uses **whatever is installed on the host system**.

**Classic Python example:** Your `requirements.txt` declares `requests==2.x`, but the system Python has `requests==1.x` installed by another tool. Python's import resolution finds the system copy first — your app runs the wrong version.

**Classic system-tool example (ImageMagick):** Your app shells out to `convert` (ImageMagick v6 on Ubuntu) but your Mac has v7 via Homebrew where the binary is `magick`. Same code, different behavior — major version breaking changes.

**Isolation mechanisms in 2011:**
- Ruby: `bundle exec`
- Python: `virtualenv`
- **Vendoring**: Bundle system-level dependencies (like ImageMagick) into your app

**Rule:** Declaration without isolation = broken reproducibility. Isolation without declaration = weeks reproducing versions. **Both are mandatory.**

### Modern Reinterpretation: Containers Changed Everything [27:34]
**Containers (Docker) practically solved this factor.** A Docker image freezes not just your libraries, but:
- Operating system
- System tools
- Language runtime
- Every transitive dependency (libraries of libraries)

`package-lock.json` / `go.sum` pin exact versions of **the entire dependency tree**. A Docker image is the **peak of declaration and isolation** — the highest fidelity achievable.

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Manifest files as standard practice | ✅ Universal |
| Lock files for transitive pinning | ✅ Standard now |
| Language-level isolation (venv, bundle exec) | ⚠️ Superseded by containers for production; still useful locally |
| Vendoring system tools | ⚠️ Containers handle this better |

## 4. Factor III — Config: Store Config in the Environment [28:10]

### What Counts as Config?
**Config = everything likely to differ between deploys.** Examples:
- Database connection URL (local: `localhost:5432` in Docker; prod: Amazon RDS VPC endpoint)
- Email provider credentials (dev: sandbox API; prod: real SendGrid/Resend)
- Object storage (local: MinIO container; prod: AWS S3)
- Redis (local: container; prod: managed Redis)
- Any third-party API keys, timeouts, feature flags, hostnames, ports

**Code stays the same; config changes per deploy.**

### The Open-Source Litmus Test
The document gives a brilliant practical test:

> **"Can you open-source your production codebase right now without compromising a single credential?"**

Not *will* you — *can* you? If **no**, then config (secrets) lives in your code where it shouldn't.

### Why Environment Variables?
The factor prescribes **environment variables** for config, with three justifications:

1. **Easy to change between deploys** without touching files
2. **Cannot be accidentally committed** to the repo (unlike `config/database.yml` or `.properties` files — the #1 cause of credential leaks)
3. **Language- and framework-agnostic** — works identically in Go, Node.js, Rust, Python, Bash, Java, etc.

**2011 context:** Every framework had its own config format (Java `.properties`, Rails `config/database.yml`, etc.), and **all of them got committed** — leaking credentials daily.

### Granularity: No Named Environment Groups
The 2011 text **warns against grouping config into named environments** (e.g., `config/environments/production.rb`, `staging.rb`, `development.rb`).

**Why?** It doesn't scale. Need a new deploy that's "like staging but with a different DB"? You'd have to create `staging_pr_42.rb`. The factor demands **granular, independent environment variables** — compose a deploy by **setting values**, not by **choosing a mode**.

The modern rewrite reinforces: **env vars are granular controls; never group them as environments.**

### The Controversy: Secrets in Environment Variables? [34:56]
Environment variables are great for **config** (DB host, timeouts, feature flags). But **secrets** (DB passwords, API keys, private keys)?

**Diogo Mônica (Docker security lead, 2017): "Don't use env vars for secrets."** Reasons:
- **Child processes inherit the entire environment** — shell commands leak secrets
- **Crash reporters / debug pages** often dump the full environment
- **Linux `/proc/<pid>/environ`** is readable by anyone with process access
- Various tools log environment variables

**Correct pattern for secrets:** A **separate secret manager** (HashiCorp Vault, AWS Parameter Store, AWS Secrets Manager, Doppler, 1Password CLI). App reads secrets at **boot time**, keeps them **in memory only** (not in OS environment), rotates periodically, audits access.

**Where we stand:**
- **Config (non-secret) → Environment variables** ✅ Default, correct
- **Secrets → Secret manager** ✅ Mandatory as you scale
- **Small team / solo / platform-injected secrets** → Platform secret injection (e.g., Render/Heroku/Fly "secrets" feature) is **better than committing to repo**, but not the final destination

> [!note] ⇢ *inferred* The video implies the platform's secret injection (e.g., `fly secrets set`) likely writes to the container's environment at runtime — acceptable for small scale but not a substitute for a proper secret manager at scale.

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Config separated from code | ✅ Non-negotiable, timeless |
| Env vars as config mechanism | ✅ Universal standard |
| No named environment groups | ✅ Scales to preview deploys, PR environments |
| Env vars for **secrets** | ❌ **Aged poorly** — superseded by secret managers |
| Secret manager pattern | ✅ Modern best practice |

## 5. Factor IV — Backing Services: Treat as Attached Resources [38:06]

### Definition
> **"A backing service is anything your application consumes over the network for doing any kind of operation."**

Examples from a typical stack:
- **Datastores:** PostgreSQL, MySQL, Redis, MongoDB, Cassandra
- **Message queues:** RabbitMQ, Kafka, SQS, NATS
- **Email:** SMTP, SendGrid, Resend, Postmark
- **Object storage:** S3, GCS, MinIO, Cloudflare R2
- **Third-party APIs:** Stripe, Twilio, GitHub, Auth0

**If your app talks to it over the network via a protocol, it's a backing service.**

### The Rule: No Distinction Between Local and Managed
Your codebase **must not make any distinction** between:
- A PostgreSQL you run in a local Docker container
- Amazon RDS PostgreSQL
- Neon / PlanetScale / Supabase managed PostgreSQL

**All are "attached resources" reachable via a resource handle.**

### Resource Handle = URL + Credentials
A **resource handle** is the complete connection string. For PostgreSQL:
```
postgres://user:password@host:5432/database
```
Five parts:
1. **Scheme** — `postgres` (implies wire protocol)
2. **Credentials** — `user:password`
3. **Host** — hostname/IP
4. **Port** — `5432` (default)
5. **Database name** — specific DB within the cluster

This **one string** contains everything the app needs to attach to the resource. Per Factor III, it lives in **config (environment variables)**, not code.

### The Litmus Test
> **"You should be able to switch your local Postgres URL to any managed service (Neon, PlanetScale, Supabase, RDS) without changing a single line of code. Only the resource handle in config changes."**

If true, your app adheres to this factor.

### Historical Context: A 2011 Prediction [41:42]
In 2011, this was considered **eccentric**. The standard was: **app and database on the same physical server**. Heroku saying "the database is someone else's computer; your app just talks via URL" was radical. 

**Today:** Running app + DB on the same instance is an **anti-pattern** for scaling. Managed services (RDS, Neon, Upstash, Resend, S3) are the norm. This factor **aged remarkably well — it predicted the future.**

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Backing services as attached resources | ✅ Universal pattern |
| Resource handle (connection string) | ✅ Standard |
| Swapping local ↔ managed without code changes | ✅ Enabled by ORMs, drivers, Factor III |
| Prediction of managed service dominance | ✅ Prescient |

## 6. Factor V — Build, Release, Run: Strictly Separate Stages [43:06]

### The Three Stages

```mermaid
flowchart LR
    subgraph Pipeline["CI/CD Pipeline"]
        direction TB
        Build["Build Stage\n• Git pull at commit\n• Fetch dependencies\n• Compile / bundle\n→ Produces: Build Artifact"]
        Release["Release Stage\n• Build Artifact +\n  Deploy Config\n→ Produces: Release (immutable)"]
        Run["Run Stage\n• Release →\n  Start Process"]
    end
    Build --> Release --> Run
```

**What this diagram shows:** The three strictly separated stages of deploying a twelve-factor app. **Build** transforms code + dependencies into an immutable artifact. **Release** combines that artifact with deploy-specific config to create a uniquely identified, immutable release. **Run** starts a process from that release. No stage bleeds into another.

---

#### Stage 1: Build
- Input: **Code at a specific commit** (via git pull)
- Actions: Fetch dependencies (Factor II), compile (if compiled language), bundle assets
- Output: **Build artifact** (called a "build" in the document)
  - Compiled language (Go): single binary
  - Interpreted (Node.js): app + `node_modules`
  - Container world: **Docker image**
- **Key property:** A build contains **only code + dependencies**. **No config.**

#### Stage 2: Release
- Input: **Build artifact** + **Deploy-specific config** (Factor III)
- Action: Combine them
- Output: **Release** — an exact build + exact config, **ready to run**
  - Build @ commit 97 + Production config = **Release v100 (Production)**
  - Same build + Staging config = **Release v101 (Staging)**
- **Key properties:**
  - **Immutable** — once created, never modified
  - **Append-only ledger** — each release gets a unique ID (timestamp, incrementing number)
  - **Rollback = instant** — just re-run a previous release; the build hasn't changed

#### Stage 3: Run
- Input: **Release**
- Action: Launch process(es) from that release
- Runtime concerns: Process management, scaling, restart on crash — **platform's job** (Factor VIII, IX)

---

### Why This Separation? Two Critical Benefits

#### Benefit 1: Immutable Builds → Instant, Zero-Risk Rollbacks
**Scenario:**
- 2:00 PM — Build commit 97 + config → **Release 100** (production)
- 4:00 PM — DB password rotated → config changes → **Release 101** (same build, new config)
- 6:00 PM — New code deployed → **Release 102** (new build, same config)
- 6:15 PM — 405 errors spike, users complain in Slack

**Without strict separation:** Debug in production while users suffer.  
**With strict separation:** **Re-run Release 101** (takes seconds — build is immutable, meaning Release 101 is *exactly* what ran at 4 PM). Production stable. Debug Release 102 at leisure.

> The document emphasizes: **releases are an append-only ledger**. Never mutate a release. Every release has an ID, timestamp, or incrementing number.

#### Benefit 2: No Code Editing on Production Servers
In the snowflake era, engineers **SSH'd into prod, hot-patched files, reloaded the server**. This caused **drift** between the repo and the running environment — a primary cause of software erosion.

**Strict build/release/run separation makes this impossible:** There is **no path from the running machine back into the build**. Code only enters at the Build stage.

---

### Modern Reinterpretation: Immutability Made Explicit
The rewrite adds one principle: **Immutability is mandatory**. It was implied in 2011; now it's a first-class requirement. Every release = unique, immutable snapshot.

**Modern implementations:** Heroku Releases, Docker image tags (`myapp:v1.2.3`), Kubernetes rollouts (`kubectl rollout undo`), Render/Railway/Fly deployments — all embody this pattern.

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Three-stage separation | ✅ Foundation of all modern CI/CD |
| Immutable builds/releases | ✅ Enables GitOps, instant rollback |
| Append-only release ledger | ✅ Audit trail, reproducibility |
| No prod hot-patching | ✅ Eliminated entire class of drift bugs |

## 7. Factor VI — Processes: Execute as Stateless, Share-Nothing Processes [50:16]

### Original Definition
> **"Execute the app as one or more stateless processes."**

**Stateless** = processes **do not remember anything between requests**. The document says: **"Share nothing between these processes"** — no shared memory, no shared disk. Anything that must persist goes to a **stateful backing service** (Factor IV).

**Core expectation:** Any request at any moment can be served by any process. Any process can disappear at any moment (destroyed, crashed, scaled down).

---

### What Processes *Can* Hold (Exceptions)
The document permits **memory and local disk as a cache for a single transaction**:
- Downloading a file → processing → writing result to DB → temp file deleted
- In-memory cache for the duration of one request

**Why allowed?** After the transaction, **nothing else expects that data to exist**. It's in a temp directory that disappears anyway.

**Crosses the line:** If a **future request or process depends on that file/memory** — that's state in the wrong place.

---

### The Classic Anti-Pattern: Sticky Sessions [52:25]

```mermaid
flowchart TD
    User["User logs in"]
    LB["Load Balancer\n(Sticky Session)"]
    Inst1["Instance 1\nSession in memory"]
    Inst2["Instance 2\nNo session"]
    Inst3["Instance 3\nNo session"]
    
    User --> LB
    LB -->|Routes to| Inst1
    LB -.->|Next request<br/>forced to| Inst1
    LB -.->|If Inst1 dies| User2["User instantly logged out"]
```

**Sticky session** = Load balancer remembers which instance a user hit first and **forces all subsequent requests to that same instance** — to preserve in-process session state.

**Why the document calls it a violation:**
1. **Defeats scaling purpose** — if Instance 1 dies, all its users instantly lose state (logged out, cart lost, etc.)
2. **New instances get zero existing users** — they only get *new* users, wasting capacity
3. **Scaling up doesn't help current load** — existing users stuck on overloaded instance

**Fix:** Session data = **state** → belongs in a **backing service** (Redis, PostgreSQL, database). Every instance reads/writes the same session store. Any instance can serve any request.

---

### The WebSocket Exception [55:24]
WebSockets **hold state in process memory** (the open socket). This **breaks statelessness** by definition.

**Video's take:** This factor is a **default you apply everywhere**, but you **may break it when the feature itself IS state** (real-time, WebSockets). The workaround (covered in the real-time video): **Pub/Sub** — after each event, publish to a message bus; other instances subscribe and propagate. The socket-holding instance remains special, but state is shared.

> [!note] ⇢ *inferred* The video suggests WebSockets are the primary legitimate exception to Factor VI. Other long-lived connections (SSE, gRPC streaming) likely fall in the same category.

---

### Thumb Rule
> **"If you cannot kill any process at any moment without a user losing data, you have state where it doesn't belong."**

---

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Stateless processes / share-nothing | ✅ Foundation of horizontal scaling, Kubernetes, serverless |
| Sticky sessions = anti-pattern | ✅ Universally rejected; Redis-backed sessions standard |
| Temp disk/memory per-transaction | ✅ Still valid (e.g., `/tmp` in containers) |
| WebSocket exception | ⚠️ Acknowledged exception; modern solutions (Socket.io + Redis adapter, Pusher, Ably) mitigate |

## 8. Factor VII — Port Binding: Export Services via Port Binding [56:39]

### Historical Context: The "Guest" Pattern [57:50]
In 2011, many apps **didn't run themselves**:
- **PHP** → Apache runs; PHP is a module *inside* Apache
- **Java** → Tomcat runs; `.war` file dropped into Tomcat's folder
- **Python** → mod_wsgi / uWSGI behind Nginx/Apache

The **web server (Apache, Tomcat, Nginx) was the program**; your app was a **guest** inside it. The server's version, config, and quirks became part of your runtime environment — **back to snowflake servers / software erosion**.

### The Rule: Self-Contained + Port Binding
> **"Your app should be completely self-contained: bring its own HTTP server as a library (via Factor II's manifest) and export itself as a service by binding to a port and listening for connections."**

**Examples:**
- Go: `http.ListenAndServe(":8080", handler)` — standard library HTTP server
- Node.js: `app.listen(4000)` — Express/Fastify binds port
- Python: `uvicorn.run(app, port=8000)` — ASGI server in-process
- Rust: `axum::Server::bind(&addr).serve(app).await`
- Java (modern): Spring Boot embedded Tomcat/Jetty (`java -jar app.jar`)

**Dev:** Access via `localhost:8080`  
**Prod:** Reverse proxy (Nginx, Traefik, Cloudflare, ALB) forwards public hostname → your process's port

---

### Powerful Consequence: Any App Can Be a Backing Service
Since **every service binds a port**, any app can become a backing service for another:
- Your database binds a port (5432)
- Your cache binds a port (6379)
- Your email service binds a port
- **Your microservice binds a port** → another service calls it via `http://service-name:port`

This is **exactly how modern service meshes and microservices work** — everything speaks HTTP/gRPC over ports.

---

### What Aged Poorly: Serverless [1:01:09]
**AWS Lambda / serverless functions do NOT bind a port.** The platform **invokes an exported function** (`handler(event, context)`). No long-running process, no port.

**2011 prediction:** Port-bound, 24/7 servers would persist. **Reality:** Serverless is a major paradigm.

**Modern rewrite adjustment:** Keeps "port binding" as headline but broadens to: **"App should be self-contained and export its service through a declared interface"** — covering both:
- Long-running port-bound server
- Serverless exported function invoked by platform

---

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Self-contained HTTP server in-process | ✅ Standard in Go, Node, Rust, Spring Boot, .NET Core |
| Port binding as service export | ✅ Universal for containers, VMs, PaaS |
| Reverse proxy in front | ✅ Standard (TLS termination, routing) |
| Serverless exception | ❌ **Aged poorly** — major paradigm not anticipated |
| Rewrite's "declared interface" broadening | ✅ Correctly captures both models |

## 9. Factor VIII — Concurrency: Scale Out via the Process Model [1:02:09]

### 2011 Context: Every Runtime Had Its Own Scaling Story
- **Java:** One massive JVM process, huge heap, scales via **threads** (multiplexing inside one process)
- **PHP:** Apache spins **child processes on demand** per request
- **Node.js:** Single-threaded **event loop**
- **Go:** Goroutines (green threads) within one process

**Problem:** No universal scaling primitive. Operations teams had to learn each runtime's knobs.

### The Rule: Process = First-Class Citizen
> **"Keep the process as a first-class citizen. Design services to scale as separate processes from scratch."**

Inspired by **Unix/Linux daemon model**: different work types → different process types.

#### Process Types (Examples)
| Process Type | Work | Example |
|--------------|------|---------|
| **web** | HTTP requests | API server, webhook handler |
| **worker** | Heavy background jobs | Video encoding, image resize, PDF generation |
| **scheduler** | Time-based tasks | Daily email at midnight, cleanup cron |
| **clock** | Periodic triggers | Every 5 min: check queue health |

**Each type scales independently.**

---

### Why Independent Scaling Matters: The Food Delivery Example [1:04:50]

```mermaid
flowchart TD
    subgraph Monolith["Monolithic Process (Web + Worker together)"]
        MWeb["Web Process\n500 MB"]
        MWorker["Worker Process\n500 MB"]
    end
    
    subgraph Split["Separate Process Types"]
        SWeb["Web Process\n500 MB × 10 = 5 GB"]
        SWorker["Worker Process\n500 MB × 2 = 1 GB"]
    end
    
    Monolith -->|Lunch rush: scale to 10| MWeb
    Monolith -.->|Worker ALSO scaled to 10| MWorker
    Split -->|Lunch rush: web needs 10| SWeb
    Split -.->|Worker stays at 2| SWorker
```

**Scenario:** Each process = 500 MB RAM. Food delivery app at **lunchtime**:
- **Web traffic spikes** → need 10 web processes (5 GB)
- **Background jobs low** → only need 2 workers (1 GB)

**If combined in one process:** Scaling web to 10 **forces workers to 10** = 5 GB wasted RAM (10 GB total vs. 6 GB needed).  
**If separate:** Scale web to 10, keep workers at 2 = **6 GB total**.  
**At night:** Reverse — web at 5, workers at 20 for batch jobs.

---

### Process Formation
> **"The process of setting the type and count of processes (e.g., `web=3, worker=2`) is called the process formation."**

**Still used today:**
- **Procfile** (Heroku origin): `web: ./server`, `worker: ./worker`, `clock: ./scheduler`
- **Kubernetes:** `replicas: 3` in Deployment spec = process formation in YAML
- **Docker Compose:** `deploy: replicas: 3`
- **Render/Fly/Railway:** UI or config for process types + counts

---

### Threads/Goroutines/Event Loops: Use Freely, But Scale at Process Level
The factor **does not forbid** intra-process concurrency:
- JVM threads ✅
- Go goroutines ✅
- Node event loop ✅

**But:** The **unit of horizontal scaling = process**, not thread/goroutine. Why? **One machine has vertical limits** (RAM, CPU). Share-nothing processes can scale **across multiple machines** without code changes.

---

### 12-Factor App Must Not Daemonize / Manage Its Own Lifecycle
- **No `daemon()` calls**, no PID file writing
- **Process management = Platform's job** (systemd, Kubernetes, Docker, Overmind, Supervisord)
- App runs in **foreground**, does its work, exits when told

---

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Process as scaling primitive | ✅ Universal (K8s pods, containers, serverless invocations) |
| Process types (web/worker/clock) | ✅ Standard architecture |
| Process formation (Procfile → K8s replicas) | ✅ Direct lineage |
| No self-daemonization | ✅ Platforms manage lifecycle |
| Threads/goroutines for intra-process concurrency | ✅ Explicitly allowed |
| Vertical scaling limits argument | ✅ Still true; horizontal = processes across machines |

## 10. Factor IX — Disposability: Maximize Robustness with Fast Startup & Graceful Shutdown [1:09:59]

### Why Disposability? The Platform Kills Processes Constantly [1:10:39]
Modern orchestrators (Kubernetes, ECS, Fly, Render) **start/stop/move processes constantly**:
- Deploy 5×/week × 10 instances = 50 stop/starts/week **just from deployments**
- Autoscaling adds/removes instances
- Node upgrades, spot instance reclamation, OOM kills
- **Process death is routine, not an incident**

If startup is slow or shutdown is ungraceful, **every deploy/scale event becomes painful**.

---

### Three Pillars of Disposability

#### 1. Fast Startup (Boot in Seconds)
- Target: **< 10 seconds**, ideally **2–3 seconds**
- **Why?** Rolling deploy of 10 instances × 60 sec boot = **10 minutes** of deploy time. Autoscaler reacts 1 minute late to traffic spike.
- **If 2 seconds:** Negligible overhead.

#### 2. Graceful Shutdown (SIGTERM Handling)
When orchestrator wants to stop a process → sends **SIGTERM** (polite signal, ~30 sec grace period). Process must:

**Web Process:**
1. **Stop accepting new requests** (close listener / remove from load balancer)
2. **Finish in-flight requests**
3. **Exit cleanly**

**Worker Process:**
1. **Stop pulling new jobs** from queue
2. **Finish current job** OR **return it to queue** (re-entrancy required — see below)
3. **Exit cleanly**

> **Re-entrancy** = Job can be safely processed multiple times. Achieved by:
> - Wrapping in a **transaction** (all-or-nothing)
> - Making operations **idempotent** (re-running produces same result; detect-and-skip duplicates)

#### 3. Robustness Against Sudden Death (Crash-Only Design)
SIGTERM is the **polite case**. **Hardware failure, OOM kill, power loss = no signal, instant death.**

**Design for:** Any process can be **killed mid-instruction at any time**.
- **Crash-only design:** If you survive crashes, you don't need a separate graceful path — the crash path *is* the shutdown path.
- Use **transactions**, **idempotency**, **write-ahead logs** so partial work is either completed or cleanly rolled back on restart.

---

### Graceful Shutdown Lifecycle (Recap from Video 19)

```mermaid
sequenceDiagram
    participant Orchestrator
    participant Process
    participant LoadBalancer
    participant Queue
    
    Orchestrator->>Process: SIGTERM
    Process->>LoadBalancer: Deregister / stop accepting
    Process->>Process: Finish in-flight requests
    alt Worker Process
        Process->>Queue: Return current job (re-entrant)
    end
    Process->>Orchestrator: Exit(0)
    Note over Orchestrator: If timeout (30s): SIGKILL
```

**What this diagram shows:** The graceful shutdown sequence. On SIGTERM, the process immediately stops accepting new work, finishes existing work (returning queue jobs if worker), then exits. The orchestrator escalates to SIGKILL only if the grace period expires.

---

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Fast startup as requirement | ✅ Critical for K8s, serverless cold starts |
| SIGTERM handling | ✅ Universal standard |
| In-flight request draining | ✅ Standard (Go `http.Server.Shutdown`, Node `server.close`) |
| Re-entrancy / idempotency for workers | ✅ Essential for exactly-once / at-least-once semantics |
| Crash-only design | ✅ Gaining traction (systems like FoundationDB, TigerBeetle) |
| 30-second grace period | ⚠️ Varies by platform (K8s `terminationGracePeriodSeconds` default 30s) |

## 11. Factor X — Dev/Prod Parity: Keep Development, Staging, and Production as Similar as Possible [1:16:48]

### The Three Gaps (Original 2011 Text)

| Gap | Traditional App | 12-Factor App |
|-----|-----------------|---------------|
| **Time** | Deploy every week/two weeks | Deploy every few hours |
| **Person** | Devs write code; Ops deploys; different person sees errors | **Same people** write, deploy, and debug (you deploy, you read logs) |
| **Tools** | Dev: Mac/Windows, native DB; Prod: Linux, external managed services | **Same backing services, same stack** everywhere |

**Parity = sameness.** Shrinking these gaps makes continuous deployment smooth.

---

### The Tools Gap: The Hardest to Close [1:19:18]
**Time & Person gaps** largely closed by modern CI/CD and DevOps culture.  
**Tools gap** persists — especially **backing services**.

#### The SQLite vs. PostgreSQL Trap
- **Dev:** SQLite (lightweight, file-based, zero-setup via ORMs like Prisma, Drizzle, GORM, SQLAlchemy)
- **Prod:** PostgreSQL

**ORMs abstract the DB** — but **behavioral differences leak through silently**:

| Difference | SQLite | PostgreSQL |
|------------|--------|------------|
| `LIKE` operator | Case-insensitive (ASCII) | **Case-sensitive** |
| `ILIKE` | Not standard | Case-insensitive variant |
| `LIMIT ... OFFSET` | Supported | Supported |
| `RETURNING` clause | ❌ | ✅ |
| JSONB operations | Limited | Full support |
| Concurrency/locking | File-level | Row-level, MVCC |

**Result:** Search feature works locally (case-insensitive `LIKE`), **breaks silently in prod** (case-sensitive). **No error, no warning — just wrong behavior.**

#### The Document's Advice
> **"Backing services (DB, cache, queue) should be the SAME in local, staging, and production."**

**2026 reality:** `docker run postgres` is **one command**. No excuse for SQLite in dev.

---

### The Managed Service Complication [1:22:08]
**Production uses managed services** (AWS DynamoDB, Cloudflare D1, Upstash Redis, Resend) — **no local equivalent exists**.

- **LocalStack** emulates AWS locally — but **not the real implementation** (proprietary), only documented behaviors
- **DynamoDB local** — same issue

**No straightforward answer.** Trade-off:
- **Managed service benefits** (scaling, cost, ops burden) vs. **Parity cost** (behavioral drift)

**Nuanced takeaway:** If managed service benefits are **large**, accept the parity gap. Mitigate with:
- Integration tests against real managed service in CI
- Staging environment using real managed service
- Feature flags for risky migrations

---

### Modern Reinterpretation
The rewrite acknowledges: **Perfect parity is impossible with proprietary managed services.** The principle becomes: **"Minimize unnecessary differences; make remaining differences explicit and tested."**

---

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Time gap (deploy frequency) | ✅ Closed by CI/CD, trunk-based development |
| Person gap (DevOps / you build it you run it) | ✅ Cultural standard |
| Tools gap — containerizable services (Postgres, Redis, Kafka) | ✅ **Docker closed this** — run same image everywhere |
| Tools gap — proprietary managed services | ❌ **Widened** — DynamoDB, Spanner, Cloudflare D1, etc. have no local twin |
| Rewrite's pragmatic stance | ✅ Acknowledges reality; focuses on testable differences |

## 12. Factor XI — Logs: Treat Logs as Event Streams [1:24:04]

### The Mental Shift: Logs ≠ Files
Historically, **logs = files** (`/var/log/app.log`, `log/production.log`). The factor says:

> **"The file was never really the log. The file is just one output format. The log itself is a stream of events, ordered by time, flowing continuously as long as the application runs — no beginning, no end."**

---

### The Prescription: Write to Stdout, Unbuffered
**Your app's only job:** Write every log event to **standard output (stdout)**, **unbuffered** (no batching, no delay).

**Platform's job:** Capture that stream and decide:
- **Dev (terminal):** Show live in console
- **Container (Docker/K8s):** Capture, tag with container ID/pod name, forward to collector
- **Prod (orchestrator/cloud):** Merge all instance streams → single time-ordered stream → send to **log drain** (Loki, Datadog, Elasticsearch, CloudWatch, Splunk)

```mermaid
flowchart LR
    App["App\nwrites to stdout"] -->|Dev| Term["Terminal\nLive view"]
    App -->|Container| Runtime["Container Runtime\nTag with pod/container ID"]
    App -->|Prod| Orchestrator["Orchestrator\n(K8s, ECS, Fly, Render)"]
    Runtime --> Collector["Log Collector\n(Fluent Bit, Vector, Promtail)"]
    Orchestrator --> Collector
    Collector --> Backend["Log Backend\nLoki, Datadog,\nElastic, CloudWatch"]
    Backend --> UI["Query / Alert / Dashboard"]
```

**What this diagram shows:** The log flow in a twelve-factor app. The application writes unstructured or structured events to stdout. The execution environment (terminal, container runtime, orchestrator) captures the stream, enriches it with metadata (pod name, container ID, timestamp), and routes it to a centralized log backend. The application code never knows or cares about files, rotation, or destinations.

---

### Why This Division? Two Reasons

1. **Deploy without code changes** — Logging destination is **config**, not code. Switch from terminal → Loki → Datadog by changing platform config, not app code.
2. **Streams compose; files don't** — A stream can be:
   - Teed to terminal **and** file **and** network simultaneously
   - Merged across 100 instances into one time-ordered stream
   - Filtered, sampled, transformed in-flight
   - Files require `tail -f`, `logrotate`, manual correlation across instances

---

### Modern Evolution: Observability (Logs + Metrics + Traces) [1:29:05]
**2011:** Only logs existed as a practiced discipline.  
**Today:** **Observability** = **Logs + Metrics + Traces** (OpenTelemetry, Prometheus, Jaeger, Tempo).

**The factor isn't limiting you** — treat metrics/traces as **additional event streams** following the same principle: **app emits; platform routes.**

---

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| Stdout as log destination | ✅ Universal (Docker, K8s, all PaaS) |
| Unbuffered, streaming | ✅ Standard (libraries default to line-buffered on TTY, unbuffered when piped) |
| Platform owns routing/storage | ✅ Sidecar agents (Fluent Bit, Vector), DaemonSets, managed logging |
| Merged time-ordered stream across instances | ✅ Solves "which instance?" debugging |
| Structured logging (JSON) | ✅ Implicitly encouraged; not explicit in 2011 but standard now |
| Metrics & traces as sibling streams | ⚠️ Not in 2011 text; rewrite should include |

## 13. Factor XII — Admin Processes: Run Admin/Management Tasks as One-Off Processes [1:29:28]

### Definition
> **"A one-off process = a task you run once, next to your long-running app processes, then it exits."**

**Examples:**
- Database migrations (`./migrate up`)
- One-time data fix script (`./fix-user-emails`)
- REPL / console (`rails console`, `php artisan tinker`, `go run main.go console`)
- Ad-hoc query / report generation

---

### The Rule: Identical Environment
A one-off admin process **must run in an environment identical to the regular app**:
- **Same codebase** (same commit)
- **Same config** (same env vars)
- **Same dependency isolation** (same manifest/lock file)
- **Same backing service handles** (same DB URL, Redis URL)

**Why so strict?** The old way: **SSH into prod server → run raw SQL / interactive shell → no review, no record, no reproducibility.** This caused **drift** between what's in code and what's in production.

---

### Canonical Example: Database Migrations
Migrations **ship with the code**, run as a **one-off process before the app starts** (in the release phase or init container), then exit. This is the **gold-standard implementation** of Factor XII.

---

### Live Demo: Real Procfile Formation [1:31:48]
The video demonstrates a real `tasker` app (GitHub: `sriniously/tasker`) running on a Linux machine via SSH, driven by a **Procfile** (Heroku's invention for declaring process types):

```procfile
# Procfile
web: ./server
worker: ./worker
clock: ./scheduler
migrate: ./migrate up
console: ./console
```

**Process manager:** `overmind` (reads Procfile, starts formation)

**Boot sequence shown:**
1. Schema check (migrations)
2. Background job server starts
3. Web server binds to port
4. All logs → stdout

**One-off admin process:** `overmind run migrate` — runs from **same code, same config, same env**, exits cleanly.

**Graceful shutdown (Ctrl+C):**
- Connection pool closes
- Job server waits for workers to finish jobs
- Clean exit reported

This **is Factors VI, VII, IX, XII in action**.

---

### What Aged Well / What Aged Poorly
| Aspect | Verdict |
|--------|---------|
| One-off processes in identical env | ✅ Standard (K8s Jobs, `kubectl exec`, `fly ssh console`, `heroku run`) |
| Migrations as one-off | ✅ Universal pattern |
| Procfile as process formation declaration | ✅ Direct ancestor of K8s Deployment + CronJob + Job specs |
| No SSH raw commands in prod | ✅ Eliminated via GitOps, CI/CD, controlled access |
| Interactive REPL as one-off | ✅ Supported by all platforms |

## Key Takeaways

1. **The Twelve-Factor App is a contract** between your application and the platform — not a list of "good software principles." It defines the **interface** (codebase, deps, config, backing services, build/release/run, processes, port, concurrency, disposability, parity, logs, admin).

2. **Software erosion is real.** Snowflake servers, manual config, hot-patched prod, and undocumented infrastructure cause apps to rot without code changes. The twelve factors are **defenses against erosion**.

3. **Config ≠ Secrets.** Environment variables are the right mechanism for **config** (DB host, timeouts, feature flags). **Secrets** (passwords, API keys, private keys) belong in a **secret manager** (Vault, AWS Secrets Manager, Doppler) — read at boot, kept in memory.

4. **Build → Release → Run separation enables instant, safe rollbacks.** Immutable builds + append-only release ledger = re-running a previous release takes seconds, not minutes. No path from running machine back to build = no prod hot-patching.

5. **Stateless, share-nothing processes are the unit of horizontal scale.** Threads/goroutines/event loops are for intra-process concurrency; **processes** scale across machines. Sticky sessions defeat this; session state belongs in Redis/DB.

6. **Port binding made every service a potential backing service.** Self-contained HTTP servers (Go stdlib, Node Express, Spring Boot, Axum) enabled the microservice/service-mesh world. **Serverless is the exception** — the rewrite broadens to "declared interface."

7. **Process formation (web=3, worker=2) is the scaling control plane.** Procfile → Kubernetes replicas → Render/Fly UI. Platform manages lifecycle; app runs in foreground.

8. **Disposability = fast boot + graceful SIGTERM + crash-only robustness.** Process death is routine (deploys, autoscaling, spot reclamation). Slow boot = slow deploys + late autoscale. Ungraceful shutdown = dropped requests. Sudden death = design for it (transactions, idempotency).

9. **Dev/prod parity closed time & person gaps; tools gap remains for proprietary managed services.** Run Postgres/Redis/Kafka locally via Docker (one command). Accept parity gaps for DynamoDB/Spanner/D1 only when managed-service ROI is high — mitigate with CI integration tests.

10. **Logs are streams, not files.** App writes to stdout; platform routes. This composes: terminal → container → centralized backend (Loki, Datadog) without code changes. Modern observability adds metrics + traces as sibling streams.

11. **Admin tasks = one-off processes in identical environment.** Migrations, scripts, REPLs run via `kubectl exec`, `fly ssh console`, `heroku run` — same code, config, deps as the app. No SSH raw SQL in prod.

12. **The 2024 community revision (github.com/twelve-factor/twelve-factor, `next` branch)** separates timeless principles from 2011-era specifics, adds serverless, secret managers, observability, and makes immutability explicit. The contract evolves; the interface endures.

---

## Related Notes

- **Course MOC:** [[_00 - Backend from First Principles - Index]]
- **Previous:** [[27 - Testing for Backend Engineers]]
- **Next:** [[29 - OpenAPI - The Universal Contract Between Clients and Servers]]
- **External Resources:**
  - [The Twelve-Factor App (classic site)](https://12factor.net)
  - [Open-source repo & revision](https://github.com/twelve-factor/twelve-factor)
  - [Open-sourcing announcement (Nov 2024)](https://12factor.net/blog/open-source-announcement)
  - [Vish Abrams at KubeCon NA 2024](https://www.youtube.com/watch?v=_V_s4VeJvjU)
  - [Why you shouldn't use ENV variables for secret data (Diogo Mônica, 2017)](https://blog.diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/)
  - [Tasker demo app](https://github.com/sriniously/tasker)

---

> [!note] Source Fidelity
> This chapter was produced by reading the **entire timestamped transcript** (3,978 cue lines, ~93 minutes) of the video *"27. The Twelve-Factor App"* (video ID: `2_fcv3-XfN4`, playlist position 28 of 29 in "Backend from First Principles" by Sriniously, published 2026-09-02). Every factor, example, historical anecdote, anti-pattern, modern reinterpretation, and live-demo detail from the transcript has been captured. Mermaid diagrams were added for the Build/Release/Run pipeline, sticky-session anti-pattern, graceful shutdown sequence, and log flow — each followed by a plain-language explanation. Ambiguous or inferred points are marked with `⇢ *inferred*`. The YAML frontmatter metadata matches `/tmp/opencode/bfp_meta/28_summary.txt`.

