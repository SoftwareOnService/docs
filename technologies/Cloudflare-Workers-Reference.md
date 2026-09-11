---
title: Cloudflare Workers — Comprehensive Reference
tags:
  - cloudflare
  - workers
  - edge-computing
  - serverless
  - cdn
  - reference
created: 2026-09-11
status: draft
---

# Cloudflare Workers — Comprehensive Reference

> [!abstract] Purpose
> Complete reference for Cloudflare Workers, Workers KV, Durable Objects, D1, Queues, R2, and related edge services. Covers architecture, APIs, patterns, limits, and integration with our EduPlatform stack.

---

## 1. Core Concepts

### 1.1 What are Workers?

- **Isolates**, not containers — V8 isolates (same as Chrome/Node) but **no Node.js runtime**
- Startup: **~0-5ms** cold start (vs 100ms+ for containers)
- Execution: **CPU limit** (10ms free, 50ms Bundled, 30s Unbound)
- Memory: **128 MB** limit
- Runs on **275+ locations** globally

### 1.2 Execution Models

| Model | CPU Time | Use Case | Pricing |
|-------|----------|----------|---------|
| **Free** | 10 ms | Simple redirects, headers, auth checks | Free tier: 100k req/day |
| **Bundled** | 50 ms | Most apps, API gateways, SSR | $5/mo + $0.50/million |
| **Unbound** | 30 s | Heavy compute, video processing, ML | $0.18/million CPU-hours |

### 1.3 Worker Types

```
┌─────────────────────────────────────────────────────────┐
│  Service Workers (fetch event)                         │
│  ├── fetch(request, env, ctx) → Response               │
│  └── Handles HTTP requests at edge                     │
├─────────────────────────────────────────────────────────┤
│  Scheduled Workers (cron triggers)                     │
│  ├── scheduled(controller, env, ctx)                   │
│  └── Cron: "0 * * * *" (minute granularity)           │
├─────────────────────────────────────────────────────────┤
│  Queue Consumers                                       │
│  ├── async queue(batch, env, ctx)                      │
│  └── Process Cloudflare Queues messages                │
├─────────────────────────────────────────────────────────┤
│  Durable Objects (stateful)                            │
│  ├── class MyDO { constructor(state, env) }           │
│  └── WebSocket, coordination, per-entity state         │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Project Structure (Wrangler)

```
my-worker/
├── wrangler.toml          # Configuration (required)
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts           # Entry point (fetch handler)
│   ├── routes/            # Route handlers
│   ├── middleware/        # Auth, logging, rate-limit
│   ├── services/          # Business logic
│   ├── bindings/          # Type-safe env bindings
│   └── utils/
├── test/
│   └── *.test.ts
├── migrations/            # D1 SQL migrations
└── wrangler.d.ts          # Auto-generated types
```

### 2.1 wrangler.toml — Complete Example

```toml
name = "eduplatform-api"
main = "src/index.ts"
compatibility_date = "2026-09-01"
compatibility_flags = ["nodejs_compat", "durable_object_migrations"]

# ── Build ──────────────────────────────────────────────
[build]
command = "npm run build"
watch_dir = "src"

# ── Environment ────────────────────────────────────────
[vars]
ENVIRONMENT = "production"
API_VERSION = "v1"

# ── Bindings ───────────────────────────────────────────
[[kv_namespaces]]
binding = "SESSIONS"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
preview_id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy"

[[d1_databases]]
binding = "DB"
database_name = "eduplatform-db"
database_id = "zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz"
migrations_dir = "./migrations"

[[r2_buckets]]
binding = "ASSETS"
bucket_name = "eduplatform-assets"
preview_bucket_name = "eduplatform-assets-dev"

[[queues]]
binding = "EMAIL_QUEUE"
queue_name = "email-notifications"
max_batch_size = 100
max_batch_timeout = 30

[[durable_objects.bindings]]
name = "RATE_LIMITER"
class_name = "RateLimiter"

# ── Triggers ───────────────────────────────────────────
[triggers]
crons = ["0 * * * *"]  # hourly

# ── Observability ──────────────────────────────────────
[observability]
enabled = true
head_sampling_rate = 0.1
tail_sampling_rate = 1.0

# ── Limits ─────────────────────────────────────────────
[limits]
cpu_ms = 50  # bundled
```

---

## 3. TypeScript Bindings (wrangler.d.ts)

Generated automatically — **do not edit manually**. Run `wrangler types` after config changes.

```typescript
// wrangler.d.ts (generated)
interface Env {
  // KV Namespaces
  SESSIONS: KVNamespace;
  CACHE: KVNamespace;
  ENTITLEMENTS: KVNamespace;

  // D1 Database
  DB: D1Database;

  // R2 Buckets
  ASSETS: R2Bucket;
  UPLOADS: R2Bucket;

  // Queues
  EMAIL_QUEUE: Queue<EmailPayload>;
  WEBHOOK_QUEUE: Queue<WebhookPayload>;

  // Durable Objects
  RATE_LIMITER: DurableObjectNamespace<RateLimiter>;
  TIMETABLE_SOLVER: DurableObjectNamespace<TimetableSolver>;

  // Service Bindings (Worker-to-Worker)
  AUTH_SERVICE: Fetcher;
  PAYMENT_SERVICE: Fetcher;

  // Secrets (via `wrangler secret put`)
  JWT_SECRET: string;
  ZITADEL_CLIENT_SECRET: string;
  STRIPE_WEBHOOK_SECRET: string;

  // Vars
  ENVIRONMENT: string;
  API_VERSION: string;
}

// Queue payload types
interface EmailPayload {
  to: string;
  template: string;
  data: Record<string, unknown>;
  tenant_id: string;
}

interface WebhookPayload {
  url: string;
  payload: unknown;
  headers: Record<string, string>;
  retries: number;
}
```

---

## 4. Core APIs

### 4.1 Fetch Handler (Service Worker)

```typescript
// src/index.ts
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);
    
    // CORS preflight
    if (request.method === "OPTIONS") {
      return handleCORS(request);
    }

    // Route matching
    const route = matchRoute(url.pathname);
    if (!route) {
      return new Response("Not Found", { status: 404 });
    }

    // Auth middleware
    const authResult = await authenticate(request, env);
    if (!authResult.ok) {
      return authResult.response;
    }

    // Rate limiting (Durable Object)
    const rateLimit = await checkRateLimit(request, env, authResult.userId);
    if (!rateLimit.allowed) {
      return new Response("Too Many Requests", { 
        status: 429,
        headers: { "Retry-After": rateLimit.retryAfter.toString() }
      });
    }

    // Execute handler
    try {
      const response = await route.handler(request, env, ctx, authResult);
      return addSecurityHeaders(response);
    } catch (err) {
      return handleError(err, env);
    }
  },

  // Scheduled handler
  async scheduled(controller: ScheduledController, env: Env, ctx: ExecutionContext): Promise<void> {
    if (controller.cron === "0 * * * *") {
      await runHourlyMaintenance(env, ctx);
    }
  },

  // Queue consumer
  async queue(batch: MessageBatch<EmailPayload>, env: Env, ctx: ExecutionContext): Promise<void> {
    await processEmailBatch(batch, env, ctx);
  }
} satisfies ExportedHandler<Env>;
```

### 4.2 Routing Patterns

```typescript
// src/routes/router.ts
type Handler = (req: Request, env: Env, ctx: ExecutionContext, auth: AuthContext) => Promise<Response>;

interface Route {
  pattern: RegExp;
  handler: Handler;
  methods: string[];
}

const routes: Route[] = [
  { pattern: /^\/api\/v1\/students$/, methods: ["GET", "POST"], handler: studentsHandler },
  { pattern: /^\/api\/v1\/students\/([^/]+)$/, methods: ["GET", "PATCH", "DELETE"], handler: studentHandler },
  { pattern: /^\/api\/v1\/invoices$/, methods: ["GET", "POST"], handler: invoicesHandler },
  { pattern: /^\/api\/v1\/webhooks\/stripe$/, methods: ["POST"], handler: stripeWebhookHandler },
];

export function matchRoute(pathname: string): Route | null {
  for (const route of routes) {
    if (route.pattern.test(pathname)) return route;
  }
  return null;
}
```

### 4.3 Middleware

```typescript
// src/middleware/auth.ts
export interface AuthContext {
  userId: string;
  tenantId: string;
  roles: string[];
  sessionId: string;
}

export async function authenticate(request: Request, env: Env): Promise<AuthResult> {
  const cookie = request.headers.get("Cookie");
  const sessionId = parseCookie(cookie, "__Host-sid");
  
  if (!sessionId) {
    return { ok: false, response: unauthorized() };
  }

  // Check KV session cache
  const session = await env.SESSIONS.get(`session:${sessionId}`, "json") as SessionData | null;
  if (!session || session.expiresAt < Date.now()) {
    return { ok: false, response: unauthorized() };
  }

  // Verify tenant from subdomain
  const tenantId = extractTenantFromHost(request.headers.get("Host"));
  if (session.tenantId !== tenantId) {
    return { ok: false, response: forbidden() };
  }

  return { ok: true, userId: session.userId, tenantId: session.tenantId, roles: session.roles, sessionId };
}
```

---

## 5. Storage APIs

### 5.1 Workers KV — Key-Value Store

```typescript
// Sessions, caching, rate-limit counters
const SESSION_TTL = 30 * 60; // 30 minutes

export async function createSession(env: Env, data: SessionData): Promise<string> {
  const sessionId = crypto.randomUUID();
  const key = `session:${sessionId}`;
  await env.SESSIONS.put(key, JSON.stringify(data), { 
    expirationTtl: SESSION_TTL,
    metadata: { userId: data.userId }
  });
  return sessionId;
}

export async function getSession(env: Env, sessionId: string): Promise<SessionData | null> {
  return env.SESSIONS.get(`session:${sessionId}`, "json");
}

export async function deleteSession(env: Env, sessionId: string): Promise<void> {
  await env.SESSIONS.delete(`session:${sessionId}`);
}

// Cache-aside pattern
export async function getCached<T>(env: Env, key: string, fetcher: () => Promise<T>, ttl = 300): Promise<T> {
  const cached = await env.CACHE.get(key, "json") as T | null;
  if (cached) return cached;
  
  const fresh = await fetcher();
  await env.CACHE.put(key, JSON.stringify(fresh), { expirationTtl: ttl });
  return fresh;
}

// Atomic counters for rate limiting
export async function incrementCounter(env: Env, key: string, windowSec: number): Promise<number> {
  const current = await env.CACHE.get(key);
  const count = (current ? parseInt(current) : 0) + 1;
  await env.CACHE.put(key, count.toString(), { expirationTtl: windowSec });
  return count;
}
```

### 5.2 D1 — SQLite at Edge

```typescript
// src/services/database.ts
export async function query<T>(env: Env, sql: string, params: unknown[] = []): Promise<T[]> {
  const { results } = await env.DB.prepare(sql).bind(...params).all();
  return results as T[];
}

export async function exec(env: Env, sql: string, params: unknown[] = []): Promise<D1Result> {
  return env.DB.prepare(sql).bind(...params).run();
}

export async function transaction<T>(env: Env, fn: (tx: D1Database) => Promise<T>): Promise<T> {
  // D1 doesn't have explicit transactions yet — use batch
  // Workaround: single batch with multiple statements
  const statements = []; // collected by fn
  return env.DB.batch(statements).then(() => fn(env.DB));
}

// Prepared statements (recommended)
const INSERT_STUDENT = `
  INSERT INTO students (id, admission_no, first_name, last_name, dob, status, tenant_id)
  VALUES (?, ?, ?, ?, ?, ?, ?)
`;

export async function createStudent(env: Env, student: StudentInput): Promise<string> {
  const id = crypto.randomUUID();
  await env.DB.prepare(INSERT_STUDENT)
    .bind(id, student.admissionNo, student.firstName, student.lastName, student.dob, "active", student.tenantId)
    .run();
  return id;
}
```

### 5.3 R2 — Object Storage

```typescript
// src/services/storage.ts
export async function uploadFile(env: Env, key: string, data: ReadableStream | ArrayBuffer, options: R2PutOptions = {}): Promise<R2Object> {
  return env.ASSETS.put(key, data, {
    httpMetadata: { contentType: options.contentType },
    customMetadata: options.metadata,
  });
}

export async function getFile(env: Env, key: string): Promise<R2Object | null> {
  return env.ASSETS.get(key);
}

export async function deleteFile(env: Env, key: string): Promise<void> {
  await env.ASSETS.delete(key);
}

export async function presignUpload(env: Env, key: string, expiresIn = 3600): Promise<string> {
  // Requires Workers Paid plan + R2 presigned URLs
  const url = new URL(`https://${env.ASSETS.bucketName}.r2.cloudflarestorage.com/${key}`);
  // Use AWS4 signing or R2 presign API
  return url.toString();
}

// Multipart for large files
export async function createMultipartUpload(env: Env, key: string): Promise<R2MultipartUpload> {
  return env.ASSETS.createMultipartUpload(key);
}

export async function uploadPart(env: Env, upload: R2MultipartUpload, partNumber: number, data: ArrayBuffer): Promise<R2UploadedPart> {
  return upload.uploadPart(partNumber, data);
}

export async function completeMultipartUpload(env: Env, upload: R2MultipartUpload): Promise<R2Object> {
  return upload.complete();
}
```

### 5.4 Queues — Message Queue

```typescript
// Producer
export async function enqueueEmail(env: Env, payload: EmailPayload): Promise<void> {
  await env.EMAIL_QUEUE.send(payload);
}

// Batch producer
export async function enqueueEmails(env: Env, payloads: EmailPayload[]): Promise<void> {
  await env.EMAIL_QUEUE.sendBatch(payloads);
}

// Consumer (in queue handler)
async function processEmailBatch(batch: MessageBatch<EmailPayload>, env: Env, ctx: ExecutionContext): Promise<void> {
  for (const message of batch.messages) {
    try {
      await sendEmail(message.body, env);
      message.ack();
    } catch (err) {
      message.retry({ delaySeconds: 60 }); // exponential backoff
    }
  }
}

// Dead letter handling
async function handleDeadLetter(batch: MessageBatch<EmailPayload>, env: Env): Promise<void> {
  for (const message of batch.messages) {
    await logDeadLetter(message, env);
  }
}
```

---

## 6. Durable Objects — Stateful Workers

### 6.1 Rate Limiter DO

```typescript
// src/durable-objects/rate-limiter.ts
export class RateLimiter {
  constructor(private state: DurableObjectState, private env: Env) {}

  async checkLimit(key: string, limit: number, windowMs: number): Promise<{ allowed: boolean; retryAfter: number }> {
    const now = Date.now();
    const windowStart = now - windowMs;

    // Get stored requests
    let requests: number[] = (await this.state.storage.get<number[]>(key)) || [];
    
    // Filter to current window
    requests = requests.filter(ts => ts > windowStart);

    if (requests.length >= limit) {
      const oldest = requests[0];
      return { allowed: false, retryAfter: Math.ceil((oldest + windowMs - now) / 1000) };
    }

    // Add current request
    requests.push(now);
    await this.state.storage.put(key, requests);

    return { allowed: true, retryAfter: 0 };
  }

  async reset(key: string): Promise<void> {
    await this.state.storage.delete(key);
  }
}

// Usage in fetch handler
const limiterId = env.RATE_LIMITER.idFromName(`ratelimit:${tenantId}:${userId}`);
const limiter = env.RATE_LIMITER.get(limiterId);
const result = await limiter.checkLimit("api", 100, 60_000); // 100 req/min
```

### 6.2 Timetable Solver DO (Long-Running)

```typescript
// src/durable-objects/timetable-solver.ts
export class TimetableSolver {
  private jobs: Map<string, SolverJob> = new Map();

  constructor(private state: DurableObjectState, private env: Env) {
    // Load persisted jobs
    const stored = await this.state.storage.get<Map<string, SolverJob>>("jobs");
    if (stored) this.jobs = stored;
  }

  async startGeneration(input: GenerationInput): Promise<string> {
    const jobId = crypto.randomUUID();
    const job: SolverJob = {
      id: jobId,
      status: "pending",
      input,
      progress: 0,
      createdAt: Date.now(),
    };
    this.jobs.set(jobId, job);
    await this.persist();

    // Start async work (non-blocking)
    this.runSolver(jobId, input);
    return jobId;
  }

  async getProgress(jobId: string): Promise<SolverJob | null> {
    return this.jobs.get(jobId) || null;
  }

  private async runSolver(jobId: string, input: GenerationInput): Promise<void> {
    const job = this.jobs.get(jobId)!;
    job.status = "running";
    await this.persist();

    try {
      // Call OR-Tools via WASM or external service
      const result = await this.callSolver(input);
      
      job.status = "completed";
      job.result = result;
      job.progress = 100;
    } catch (err) {
      job.status = "failed";
      job.error = err.message;
    }
    await this.persist();
  }

  private async persist(): Promise<void> {
    await this.state.storage.put("jobs", this.jobs);
  }
}
```

### 6.3 DO Migration (wrangler.toml)

```toml
[[durable_objects.bindings]]
name = "RATE_LIMITER"
class_name = "RateLimiter"

[[durable_objects.bindings]]
name = "TIMETABLE_SOLVER"
class_name = "TimetableSolver"

# Run: wrangler deploy --dry-run  (shows migration plan)
# Run: wrangler deploy             (applies migrations)
```

---

## 7. Integration Patterns

### 7.1 Worker → Go Backend (gRPC/REST)

```typescript
// src/services/backend.ts
export async function callGoService<T>(
  env: Env, 
  service: "core" | "student" | "fees" | "attendance",
  path: string,
  options: RequestInit = {}
): Promise<T> {
  const baseUrl = `https://${service}.${env.BACKEND_DOMAIN}`;
  const url = `${baseUrl}${path}`;
  
  // Get service-to-service token
  const token = await getServiceToken(env, service);
  
  const response = await fetch(url, {
    ...options,
    headers: {
      "Authorization": `Bearer ${token}`,
      "X-Tenant-ID": env.TENANT_ID, // from request context
      "Content-Type": "application/json",
      ...options.headers,
    },
    cf: {
      // Edge-specific options
      cacheTtl: 0,
      cacheEverything: false,
    }
  });

  if (!response.ok) {
    throw new BackendError(response.status, await response.text());
  }
  return response.json();
}

// Service token cache (KV)
async function getServiceToken(env: Env, service: string): Promise<string> {
  const cacheKey = `svc_token:${service}`;
  let token = await env.CACHE.get(cacheKey);
  
  if (!token) {
    token = await mintServiceToken(env, service);
    await env.CACHE.put(cacheKey, token, { expirationTtl: 300 }); // 5 min
  }
  return token;
}
```

### 7.2 Zitadel OIDC at Edge

```typescript
// src/middleware/zitadel.ts
const ZITADEL_ISSUER = "https://zitadel.example.com";

export async function validateZitadelToken(token: string, env: Env): Promise<TokenClaims | null> {
  // Fetch JWKS (cached in KV)
  const jwks = await getJWKS(env);
  
  // Verify JWT (use jose library)
  const { payload } = await jose.jwtVerify(token, jwks, {
    issuer: ZITADEL_ISSUER,
    audience: env.ZITADEL_CLIENT_ID,
  });
  
  return payload as TokenClaims;
}

async function getJWKS(env: Env): Promise<jose.JWKS> {
  const cached = await env.CACHE.get("jwks:zitadel", "json") as jose.JWKS | null;
  if (cached) return cached;

  const resp = await fetch(`${ZITADEL_ISSUER}/.well-known/jwks.json`);
  const jwks = await resp.json();
  await env.CACHE.put("jwks:zitadel", JSON.stringify(jwks), { expirationTtl: 1800 });
  return jwks;
}
```

### 7.3 HTML Streaming / SSR

```typescript
// src/handlers/ssr.ts
export async function renderDashboard(request: Request, env: Env, auth: AuthContext): Promise<Response> {
  // Parallel data fetching
  const [studentCount, feeSummary, attendanceRate] = await Promise.all([
    callGoService(env, "student", "/api/v1/students/count"),
    callGoService(env, "fees", `/api/v1/fees/summary?tenant=${auth.tenantId}`),
    callGoService(env, "attendance", `/api/v1/attendance/rate?tenant=${auth.tenantId}`),
  ]);

  // Stream HTML response
  const stream = new ReadableStream({
    async start(controller) {
      // Send head immediately
      controller.enqueue(`
        <!DOCTYPE html>
        <html><head>
          <title>Dashboard</title>
          <script src="/hydrate.js" defer></script>
        </head><body>
        <div id="app">
      `);

      // Stream components as data arrives
      controller.enqueue(renderHeader(auth));
      controller.enqueue(renderStats(studentCount, feeSummary, attendanceRate));
      controller.enqueue(renderStudentTable(await fetchStudents(env, auth)));
      
      controller.enqueue("</div></body></html>");
      controller.close();
    }
  });

  return new Response(stream, {
    headers: { "Content-Type": "text/html; charset=utf-8" }
  });
}
```

---

## 8. Security

### 8.1 Headers

```typescript
function addSecurityHeaders(response: Response): Response {
  const headers = new Headers(response.headers);
  headers.set("X-Content-Type-Options", "nosniff");
  headers.set("X-Frame-Options", "DENY");
  headers.set("Referrer-Policy", "strict-origin-when-cross-origin");
  headers.set("Permissions-Policy", "camera=(), microphone=(), geolocation=()");
  headers.set("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload");
  
  // CSP - adjust for your app
  headers.set("Content-Security-Policy", `
    default-src 'self';
    script-src 'self' 'wasm-unsafe-eval';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self' data:;
    connect-src 'self' https://api.example.com wss://api.example.com;
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self';
  `.replace(/\s+/g, " ").trim());

  return new Response(response.body, {
    status: response.status,
    statusText: response.statusText,
    headers,
  });
}
```

### 8.2 WAF Rules (Terraform)

```hcl
# cloudflare_waf_rule.rate_limit
resource "cloudflare_waf_rule" "rate_limit_api" {
  zone_id = var.zone_id
  action  = "challenge"
  filter {
    expression = "(http.request.uri.path matches \"^/api/.*\" and not cf.client.bot)"
  }
  rate_limit {
    threshold = 100
    period    = 60
    mitigation_timeout = 300
  }
}

# cloudflare_waf_rule.bot_protection
resource "cloudflare_waf_rule" "block_bad_bots" {
  zone_id = var.zone_id
  action  = "block"
  filter {
    expression = "cf.bot_management.score lt 30"
  }
}
```

---

## 9. Observability

### 9.1 Logging

```typescript
// src/utils/logger.ts
export function createLogger(env: Env, requestId: string) {
  return {
    info: (msg: string, meta?: Record<string, unknown>) => log("info", msg, meta),
    warn: (msg: string, meta?: Record<string, unknown>) => log("warn", msg, meta),
    error: (msg: string, err?: Error, meta?: Record<string, unknown>) => log("error", msg, { ...meta, error: err?.stack }),
  };

  function log(level: string, msg: string, meta?: Record<string, unknown>) {
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      level,
      message: msg,
      request_id: requestId,
      worker: env.WORKER_NAME,
      ...meta,
    }));
  }
}
```

### 9.2 Tail Workers (Real-time Logs)

```bash
# Stream logs to terminal
wrangler tail --format=json | jq '. | {time: .eventTimestamp, level: .level, msg: .message, request_id: .requestId}'

# Filter by level
wrangler tail --filter-level=error
```

### 9.3 Metrics (GraphQL Analytics API)

```typescript
// Query Workers analytics
const query = `
  query($zoneTag: string!, $filter: ZoneHttpRequestsAdaptiveGroupsFilter!) {
    viewer {
      zones(filter: { zoneTag: $zoneTag }) {
        httpRequestsAdaptiveGroups(limit: 100, filter: $filter) {
          sum { requests, bytes, responseStatusMap }
          dimensions { date, cacheStatus, edgeResponseStatus }
        }
      }
    }
  }
`;
```

---

## 10. Limits & Quotas

| Resource | Free | Paid (Bundled) | Unbound |
|----------|------|----------------|---------|
| Requests/day | 100,000 | 10M/mo included | Unlimited |
| CPU time/request | 10 ms | 50 ms | 30 s |
| Memory | 128 MB | 128 MB | 128 MB |
| Script size | 1 MB | 3 MB | 10 MB |
| KV reads/day | 100,000 | 10M/mo | Unlimited |
| KV writes/day | 1,000 | 1M/mo | Unlimited |
| KV storage | 1 GB | 10 GB | 100 GB |
| D1 rows read/day | 5M | 25B/mo | Unlimited |
| D1 rows written/day | 100k | 50M/mo | Unlimited |
| D1 storage | 5 GB | 10 GB | 100 GB |
| R2 Class A ops/mo | 1M | 10M | Unlimited |
| R2 Class B ops/mo | 10M | 100M | Unlimited |
| R2 storage | 10 GB | 1 TB | Unlimited |
| Queues messages/mo | 1M | 10M | Unlimited |
| Durable Objects | 1M req/mo | 10M req/mo | Unlimited |

---

## 11. Deployment

### 11.1 Environments

```toml
# wrangler.toml - Multiple environments
[env.staging]
name = "eduplatform-api-staging"
vars = { ENVIRONMENT = "staging" }
kv_namespaces = [{ binding = "SESSIONS", id = "staging-id", preview_id = "staging-preview" }]
d1_databases = [{ binding = "DB", database_name = "eduplatform-staging" }]

[env.production]
name = "eduplatform-api"
vars = { ENVIRONMENT = "production" }
kv_namespaces = [{ binding = "SESSIONS", id = "prod-id" }]
d1_databases = [{ binding = "DB", database_name = "eduplatform-prod" }]
```

```bash
# Deploy to staging
wrangler deploy --env staging

# Deploy to production
wrangler deploy --env production

# Preview (uses preview_ids)
wrangler dev --env staging
```

### 11.2 CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy Workers

on:
  push:
    branches: [main]
    paths: ['workers/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: workers/package-lock.json
      
      - name: Install dependencies
        run: cd workers && npm ci
      
      - name: Type check
        run: cd workers && npm run typecheck
      
      - name: Test
        run: cd workers && npm test
      
      - name: Build
        run: cd workers && npm run build
      
      - name: Deploy to staging
        if: github.ref == 'refs/heads/main'
        run: cd workers && npx wrangler deploy --env staging
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
      
      - name: Deploy to production
        if: github.event_name == 'release'
        run: cd workers && npx wrangler deploy --env production
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
```

---

## 12. Local Development

```bash
# Start local dev server with all bindings
wrangler dev --local --persist-to=./.wrangler/state

# With remote bindings (uses preview namespaces)
wrangler dev --remote

# Test specific route
curl -X POST http://localhost:8787/api/v1/students \
  -H "Content-Type: application/json" \
  -d '{"admissionNo":"STU001","firstName":"John","lastName":"Doe"}'
```

### 12.1 Miniflare (Testing)

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import { miniflare } from "miniflare";

export default defineConfig({
  test: {
    environment: "miniflare",
    poolOptions: {
      miniflare: {
        wranglerConfigPath: "./wrangler.toml",
        bindings: {
          SESSIONS: "test-sessions",
          DB: "test-db",
        },
      },
    },
  },
});
```

```typescript
// test/auth.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { env } from "cloudflare:test";

describe("Auth", () => {
  beforeEach(async () => {
    await env.SESSIONS.put("session:test", JSON.stringify({ userId: "user1", tenantId: "t1" }), { expirationTtl: 60 });
  });

  it("validates session", async () => {
    const req = new Request("http://localhost/api/test", {
      headers: { Cookie: "__Host-sid=test" }
    });
    const result = await authenticate(req, env);
    expect(result.ok).toBe(true);
    expect(result.userId).toBe("user1");
  });
});
```

---

## 13. EduPlatform Integration Points

### 13.1 Replacing Envoy Gateway Functions

| Envoy Function | Workers Equivalent |
|----------------|-------------------|
| JWT validation | Worker middleware (`authenticate`) |
| Tenant resolution | Subdomain → KV lookup → header injection |
| Entitlement check | `ENTITLEMENTS` KV + `tenant-billing` DO |
| Rate limiting | `RATE_LIMITER` Durable Object |
| WAF | Cloudflare WAF rules (managed) |
| TLS termination | Automatic (Cloudflare edge) |
| mTLS to services | `cf-connecting-ip` + service mesh |

### 13.2 BFF → Workers Migration Path

```
Phase 1: Edge Auth + Rate Limit
  - Move JWT validation, tenant resolution, rate limit to Workers
  - Keep BFF for aggregation, gRPC, complex logic

Phase 2: Read Cache at Edge
  - Timetable snapshots, student briefs, fee summaries in KV
  - Cache-aside with D1 fallback

Phase 3: Edge SSR (Optional)
  - Move dashboard/server components to Workers
  - Stream HTML from edge

Phase 4: Full Edge API
  - All REST endpoints on Workers
  - Go services become internal-only (gRPC only)
```

---

## 14. Cost Optimization

```typescript
// 1. Use CPU efficiently
// BAD: await Promise.all(promises)  // sequential if not careful
// GOOD: Batch operations
const results = await Promise.all(promises.map(p => p.catch(e => e)));

// 2. Cache aggressively
// Timetable: cache until next publish (versioned key)
// Student briefs: 24h TTL
// Entitlements: 60s TTL + event invalidation

// 3. Minimize KV writes
// Batch session updates, use metadata for TTL instead of separate keys

// 4. D1: Use prepared statements, batch inserts
await env.DB.batch(statements);

// 5. R2: Use multipart for >100MB, presigned URLs for client uploads

// 6. Queues: Batch consumers (max_batch_size=100)
```

---

## 15. Troubleshooting

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| `Script exceeded CPU limit` | Heavy computation in handler | Move to Unbound / Durable Object / Queue |
| `KV put failed: quota exceeded` | Too many writes | Batch writes, increase TTL, check plan |
| `D1: database locked` | Concurrent writes | Use batch, serialize via DO |
| `Durable Object: overloaded` | Too many requests to one DO | Shard by key, use multiple DOs |
| `CORS error` | Missing OPTIONS handler | Add explicit OPTIONS route |
| `Secret not found` | Not set in env | `wrangler secret put SECRET_NAME` |

---

## 16. Useful Commands

```bash
# Development
wrangler dev --local --persist-to=./.wrangler/state
wrangler dev --remote --env staging

# Deployment
wrangler deploy --env production
wrangler deploy --dry-run --env production

# Secrets
wrangler secret put JWT_SECRET --env production
wrangler secret list --env production

# KV
wrangler kv:key put --binding=SESSIONS "session:abc" '{"userId":"1"}'
wrangler kv:key get --binding=SESSIONS "session:abc"
wrangler kv:namespace create "SESSIONS" --preview

# D1
wrangler d1 execute eduplatform-db --local --command="SELECT * FROM students"
wrangler d1 migrations apply eduplatform-db --local
wrangler d1 backup create eduplatform-db

# R2
wrangler r2 object put eduplatform-assets/path/file.pdf --file=./file.pdf
wrangler r2 object list eduplatform-assets

# Queues
wrangler queues list
wrangler queues purge email-notifications

# Durable Objects
wrangler durable-object list
wrangler durable-object delete RATE_LIMITER <id>

# Logs
wrangler tail --format=json --filter-status=error

# Types
wrangler types --env-interface Env
```

---

## 17. References

- [Workers Docs](https://developers.cloudflare.com/workers/)
- [Wrangler Config](https://developers.cloudflare.com/workers/wrangler/configuration/)
- [Runtime APIs](https://developers.cloudflare.com/workers/runtime-apis/)
- [KV API](https://developers.cloudflare.com/kv/api/)
- [D1 API](https://developers.cloudflare.com/d1/worker-api/)
- [R2 API](https://developers.cloudflare.com/r2/api/workers/)
- [Queues](https://developers.cloudflare.com/queues/)
- [Durable Objects](https://developers.cloudflare.com/durable-objects/)
- [Limits](https://developers.cloudflare.com/workers/platform/limits/)
- [Pricing](https://developers.cloudflare.com/workers/platform/pricing/)

---

## 18. EduPlatform-Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Edge auth** | Workers middleware | Removes latency, offloads Go services |
| **Session store** | KV (`SESSIONS`) | Global replication, 30m TTL matches design |
| **Entitlements** | KV (`ENTITLEMENTS`) + DO | 60s cache + compacted topic sync |
| **Timetable cache** | KV versioned keys | Write-through on publish, immutable |
| **Rate limiting** | DO (`RATE_LIMITER`) | Precise per-tenant/user, distributed |
| **File uploads** | R2 presigned URLs | Direct client→R2, no Worker bandwidth |
| **Async tasks** | Queues | Webhooks, emails, reconciliation |
| **Long-running** | DO (`TIMETABLE_SOLVER`) | 30s CPU, persistent state, WebSocket progress |
| **Logs/metrics** | Tail + GraphQL Analytics | Real-time debugging + historical dashboards |

---

*Last updated: 2026-09-11. Keep in sync with `EduPlatform-V1-Design.md` §2, §4, §6, §7, §13, §15.*