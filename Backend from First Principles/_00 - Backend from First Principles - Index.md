---
title: Backend from First Principles — Index
tags:
  - moc
  - backend
  - course
status: completed
---

# Backend from First Principles — Index

> [!info] A complete, chapter-by-chapter study guide built from the YouTube playlist **"Backend from first principles"** by **Sriniously**.
> Each chapter is derived from the full video transcript (timestamp-by-timestamp), so no point is skipped. Read them in order — each chapter builds on the previous ones.

- **Playlist:** https://youtube.com/playlist?list=PLui3EUkuMTPgZcV0QhQrOcwMPcBCcd_Q1
- **Channel:** [Sriniously](https://www.youtube.com/@sriniously)

> [!tip] How to use this vault
> - Start with **Part 01** below and follow the numbered order.
> - Every chapter links forward/backward to its neighbours, so you can navigate organically.
> - Mermaid diagrams are rendered natively in Obsidian; each diagram is followed by a written explanation.
> - Timestamps `[mm:ss]` inside chapters point to moments in the original video.

---

## Part 1 — Foundations

| # | Chapter | What it covers |
|---|---------|----------------|
| 01 | [[01 - Roadmap for Backend from First Principles]] | Full roadmap of every backend topic covered in the series. |
| 02 | [[02 - Walk the Path of a True Backend Engineer]] | Vision and end goal of the series. |
| 03 | [[03 - What is a Backend, How Do They Work and Why Do We Need Them]] | What a backend is, how it works, why frontends can't do it alone. |
| 04 | [[04 - Benefits of Learning Backend Engineering from First Principles]] | Why first-principles learning is the fool-proof strategy. |

## Part 2 — The Request Lifecycle

| # | Chapter | What it covers |
|---|---------|----------------|
| 05 | [[05 - Understanding HTTP for Backend Engineers]] | HTTP from the ground up: messages, headers, methods, status codes, caching, negotiation, compression, TLS. |
| 06 | [[06 - What is Routing in Backend]] | How requests find their way to the right handler. |
| 07 | [[07 - Serialization and Deserialization for Backend Engineers]] | What serialization is, formats, use cases, gotchas. |
| 08 | [[08 - Authentication and Authorization for Backend Engineers]] | Full identity story: authn vs authz, sessions, tokens, JWTs, password storage. |
| 09 | [[09 - Validations and Transformations for Backend Engineers]] | Client/server validation, normalization, sanitization. |
| 10 | [[10 - Controllers, Services, Repositories, Middlewares and Request Context]] | The whole in-server request lifecycle, layer by layer. |
| 11 | [[11 - Complete REST API Design]] | REST end-to-end: resources, methods, versioning, design trade-offs. |

## Part 3 — Data & the Business Layer

| # | Chapter | What it covers |
|---|---------|----------------|
| 12 | [[12 - Mastering Databases with Postgres]] | Persistence, schemas, SQL, indexing, transactions — with Postgres. |
| 13 | [[13 - Caching, The Secret Behind It All]] | Why caching, cache types, strategies, Redis internals. |
| 14 | [[14 - Task Queues and Background Jobs]] | Why async work, queue components, design parameters. |
| 15 | [[15 - Full Text Search Using Elasticsearch]] | Why search engines, how inverted indexes work. |

## Part 4 — Production Readiness

| # | Chapter | What it covers |
|---|---------|----------------|
| 16 | [[16 - Error Handling and Building Fault Tolerant Systems]] | Error types, propagating errors, fault tolerance. |
| 17 | [[17 - Production-Grade Configuration Management]] | Config sources, hierarchy, secrets handling. |
| 18 | [[18 - Logging, Monitoring and Observability]] | Logs, metrics, traces, structured logging. |
| 19 | [[19 - Graceful Shutdown]] | Draining requests, closing connections/DBs cleanly. |
| 20 | [[20 - Backend Security - Everything You Need to Know]] | Attack surface, auth security, injection, rate limiting, CORS, OWASP. |

## Part 5 — Scale, Speed & Real-Time

| #   | Chapter                                                        | What it covers                                                     |
| --- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| 21  | [[21 - Backend Scaling and Performance Engineering (Part 1)]]  | Performance mindset, profiling, bottlenecks.                       |
| 22  | [[22 - Backend Scaling and Performance Engineering (Part 2)]]  | Hard scaling topics: load balancing, DB scaling, caching at scale. |
| 23  | [[23 - Object Storage - Everything You Need to Know (Part 1)]] | Why object storage, how it works, cloud providers.                 |
| 24  | [[24 - Object Storage - Everything You Need to Know (Part 2)]] | Object storage internals, security, cost optimisation.             |
| 25  | [[25 - Real-Time Backends]]                                    | WebSockets, SSE, pub/sub, real-time architecture.                  |

## Part 6 — Testing, Contracts & Delivery

| # | Chapter | What it covers |
|---|---------|----------------|
| 26 | [[26 - Testing for Backend Engineers]] | Unit/integration/E2E, mocks, TDD, coverage. |
| 27 | [[27 - The Twelve-Factor App]] | The 12 principles for modern SaaS applications. |
| 28 | [[28 - OpenAPI - The Universal Contract Between Clients and Servers]] | OpenAPI spec, tooling, contract-first development. |
| 29 | [[29 - Webhooks - How the Server Calls You]] | Webhooks, retries, security, provider design. |

---

## Also in this folder (quick-reference draft notes)

These are earlier quick notes kept for reference:
- [[Understanding of backend systems]]
- [[HTTP]]
- [[Routing]]
- [[CORS]]
- [[Preflight request]]
- [[3 way handshake]]
- [[_Roadmap]]

---

> [!info] Series style guide
> - Chapters mirror the video order 1:1 (video-by-video).
> - Mermaid diagrams are always accompanied by an explanation.
> - Inferences made where auto-captions were unclear are marked with ⇢ *inferred*.