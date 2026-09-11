---
title: "Object Storage - Everything You Need to Know (Part 1)"
tags: [backend, video-notes, object-storage]
course: "[[_00 - Backend from First Principles - Index]]"
source: "https://www.youtube.com/watch?v=Ie0TjKI9cDI"
video_id: "Ie0TjKI9cDI"
playlist_position: 24
duration_seconds: 5368
published: "2026-08-26"
status: completed
---

# Part 24 — Object Storage - Everything You Need to Know (Part 1)

> [!info] Video Reference
> **Title:** 23. Object Storage - Everything You Need to Know: Part -1
> **Channel:** Sriniously
> **URL:** https://www.youtube.com/watch?v=Ie0TjKI9cDI
> **Playlist Position:** 24 of 29
> **Duration:** 1 hour 29 minutes 28 seconds (5368 seconds)
> **Published:** 2026-08-26

> [!abstract] In This Chapter
> We begin with the naive file-upload design (save to local disk, store the path in the database) and systematically dismantle it, exposing six fundamental problems that emerge at production scale: memory pressure from large files, ephemeral container filesystems, horizontal-scaling incompatibility, fixed disk ceilings, durability risks, and the server becoming an expensive, slow CDN. We then survey the three storage paradigms—block, file, and object—and show why object storage’s minimal four-operation interface (PUT, GET, DELETE, LIST) is the deliberate trade-off that buys unlimited horizontal scale, eleven-nines durability via erasure coding, and direct browser-to-bucket data paths via pre-signed URLs. We dissect the anatomy of an object (key, value, system metadata, user metadata), explain why folders are an illusion created by prefix grouping, and derive key-design rules that avoid hot partitions and encode ownership for zero-hop authorization. We peel back the two-plane architecture (data plane vs. metadata plane), showing how a distributed B-tree index delivers strong read-after-write consistency since December 2020. We cover conditional writes (`If-None-Match: *` and `If-Match: <ETag>`) that enable compare-and-swap semantics—critical for distributed locks, leader election, and write-ahead logs on S3. Finally, we walk through the production-grade two-phase upload flow: the backend generates the key, writes a `pending` database row, returns a pre-signed POST URL with a policy (enforcing size, content-type, and key prefix), the browser uploads directly to the bucket, the client calls a completion endpoint, the server verifies via `HEAD`, and a lifecycle rule plus background job clean up abandoned multipart uploads and stale `pending` rows. CORS configuration for direct browser uploads closes the chapter.

## [00:51] The Naive Design: Save the File, Store the Path

The intuitive, "naive" way to handle a file upload in a backend is exactly what comes to mind first: the browser sends the file as `multipart/form-data`, the server extracts it from the request body, writes it to the local filesystem (e.g., `/var/www/uploads/avatar.png` on an ext4 or XFS volume), and records that absolute path in a database column on the user’s row. When the profile page loads, the backend reads the path, streams the file back, and the browser renders it. This works perfectly in the happy path—small profile pictures (300 KB – 20 MB), a single server instance, no traffic spikes.

> **Why this feels right**: The server runs on a physical (or virtual) machine that has a CPU, RAM, and a filesystem. Files belong on filesystems. The database stores structured data (JSON, rows). The mapping feels natural.

### [03:54] Where It Breaks: Size, Memory, and the 4 GB Video

The first crack appears when the requirement shifts from profile pictures to **videos**—files that range from a few megabytes to several gigabytes. Suppose a user uploads a **4 GB video** to a mid-sized startup’s backend where each container instance is provisioned with **4 GB of total memory**. Most web frameworks (Node.js, Go, Python, Java) "helpfully" buffer the entire request body into RAM before handing it to your handler. The moment the 4 GB payload arrives, the process tries to allocate 4 GB+ of heap, exceeds the container’s memory limit, and the OOM killer terminates the container instantly.

**Consequence**: The container restarts (assuming you have auto-healing via Kubernetes or a PaaS), but **every file ever stored on that instance’s local disk is gone**—profile pictures, videos, everything. Container filesystems are **ephemeral**: they only persist until the container restarts.

### [06:51] Ephemeral Disks and Three Instances

Even if you solve the memory issue by streaming (we’ll come to that), horizontal scaling introduces the second fatal flaw. Assume you’ve scaled to **three instances behind a round-robin load balancer**.

1.  User uploads `avatar.png` → load balancer routes to **Instance 1** → file saved to Instance 1’s disk, path stored in DB.
2.  Two minutes later, user requests the file → load balancer routes to **Instance 3** (round-robin) → Instance 3 has no knowledge of the file → **404 Not Found**.

With *N* instances, the probability of hitting the correct instance is **1/N**. This violates the **single most important property of a horizontally scalable server: statelessness**. Your server must never hold state that other servers don’t have.

> **Mermaid: Naive Design vs. Horizontal Scaling**
> ```mermaid
> flowchart LR
>     subgraph Naive["Naive Design (Single Instance)"]
>         Client1[Browser] --> LB1[Load Balancer]
>         LB1 --> App1[App Instance 1]
>         App1 --> Disk1[(Local Disk)]
>         App1 --> DB[(Database)]
>         DB -.->|stores path| Disk1
>     end
>     
>     subgraph Scaled["Horizontal Scaling (3 Instances)"]
>         Client2[Browser] --> LB2[Load Balancer]
>         LB2 --> AppA[Instance 1]
>         LB2 --> AppB[Instance 2]
>         LB2 --> AppC[Instance 3]
>         AppA --> DiskA[(Disk 1)]
>         AppB --> DiskB[(Disk 2)]
>         AppC --> DiskC[(Disk 3)]
>         AppA --> DB
>         AppB --> DB
>         AppC --> DB
>         DB -.->|stores path| DiskA
>         DB -.->|stores path| DiskB
>         DB -.->|stores path| DiskC
>     end
> ```
> **What this diagram shows:** In the naive single-instance design, the file path stored in the database points to a file on that instance’s local disk. When scaled to multiple instances behind a load balancer, a request for the file can land on any instance. Only the instance that originally received the upload has the file on its disk; the other instances return 404. The database path becomes ambiguous—it doesn’t encode *which* instance holds the file.

### [09:54] Why Object Storage Exists: The Six Problems

Object storage exists to solve **six distinct problems** that the naive design cannot:

| # | Problem | Naive Design Failure |
|---|---------|----------------------|
| 1 | **Ephemeral Disk** | Container restarts wipe all local files. |
| 2 | **Horizontal Scaling** | Files stuck on one instance; 1/N chance of 404. |
| 3 | **Fixed Disk Ceiling** | Disk grows vertically (resize = downtime); hard max per cloud provider. User-uploaded files are the fastest-growing data in most apps (e.g., 1,000 users × 100 MB = 100 GB/month forever). |
| 4 | **Durability** | Single disk = single point of failure. Disk death = total data loss. Durability ≠ availability: you can wait out an availability issue, but **you cannot recover from a durability loss by waiting**—the data is gone. The only fix is replication (multiple copies), which introduces checksums, background repair, and a whole new engineering domain. |
| 5 | **Downloads Through Your Server** | Serving a 50 MB file to a mobile client at 2 Mbps ties up a worker (goroutine / event-loop thread) for ~3.5 minutes just copying bytes. Your server becomes an **expensive, slow CDN** with a ~100 concurrent connection limit. |
| 6 | **File–Database Transaction Gap** | No atomic transaction spans "write file to disk" + "insert row in DB". Either order fails: write file first → DB insert fails → orphan file on disk; write DB first → file write fails → DB row points to non-existent path. Object storage doesn’t *directly* solve this, but makes it easier to manage via the two-phase flow. |

---

## [25:03] Three Kinds of Storage: Block, File, Object

Before diving into object storage, we must understand the landscape. There are **three fundamental storage paradigms**, each a layer of abstraction atop the previous.

### 1. Block Storage (The Raw Array)
- **Interface**: Fixed-size blocks (typically 512 B or 4 KB), addressed by block number (LBA).
- **Example**: An NVMe SSD, Amazon EBS (Elastic Block Store).
- **Pros**: Lowest-level interface → **microsecond latency**. Databases (PostgreSQL, MySQL) sit directly on block storage to preserve this latency.
- **Cons**: No concept of files, names, or folders. **A block device can attach to exactly ONE machine at a time**—no sharing across instances.

### 2. File Storage / Network File System (The POSIX Abstraction)
- **Interface**: Hierarchical directories, files, paths, permissions, inodes, POSIX semantics (atomic rename, partial write consistency, file locking, `seek`).
- **Example**: ext4, XFS, NFS, EFS (Amazon Elastic File System).
- **Pros**: Rich, familiar interface. `rename` is atomic; editors safely write to temp files then swap.
- **Cons**: **Coordination doesn’t scale across network boundaries**. Locking, tree rebalancing, and metadata consistency become distributed-systems nightmares at planet scale.

### 3. Object Storage (The Minimal Interface)
- **Core Question**: *What is the absolute minimum interface that still lets us store and retrieve data, so we can scale to exabytes / "planet scale"?*
- **Interface**: **Four operations** — `PUT` (upload object with key), `GET` (fetch by key), `DELETE` (remove by key), `LIST` (list keys with prefix/delimiter).
- **Trade-offs** (what you give up):
  - ❌ **No in-place modification** — objects are **immutable**; to change one byte, download the whole object, modify, re-upload with same key (replace).
  - ❌ **No real directories/folders** — flat namespace; "folders" are an illusion created by key prefixes.
  - ❌ **No atomic rename** — rename = copy to new key + delete old key (costs full object size).
  - ❌ **No file locking** — no POSIX-style advisory/mandatory locks.
  - ❌ **Higher latency** — every operation is an HTTP request crossing a network boundary (milliseconds to tens of ms vs. microseconds).
- **What you get back**:
  - ✅ **No coordination** → entire class of distributed-systems problems eliminated.
  - ✅ **No hierarchy** → no tree to lock, rebalance, or maintain.
  - ✅ **Self-contained HTTP requests** → any of 10,000+ servers worldwide can serve any request → **unlimited horizontal scale**.
  - ✅ **Infinite capacity** — never provision, never resize, just keep uploading.
  - ✅ **Durability guarantees** — e.g., AWS S3’s **11 nines (99.999999999%)** via erasure coding.
  - ✅ **Direct HTTP addressability** → browsers, video players, mobile apps, IoT devices can fetch objects **without your server in the middle**.

> **Mermaid: Storage Paradigm Comparison**
> ```mermaid
> flowchart TB
>     subgraph Block["Block Storage (EBS, Raw SSD)"]
>         B1[Block 0] --- B2[Block 1] --- B3[Block 2] --- B4["... Block N"]
>         B1 -.->|512 B / 4 KB| B2
>         style B1 fill:#f9f,stroke:#333
>     end
>     subgraph File["File Storage (ext4, NFS, EFS)"]
>         F1[/] --> F2[home] --> F3[user] --> F4[file.txt]
>         F1 --> F5[var] --> F6[log]
>         style F1 fill:#bbf,stroke:#333
>     end
>     subgraph Object["Object Storage (S3, GCS, R2)"]
>         O1[Bucket] --> O2["user/123/abc.png"]
>         O1 --> O3["video/456/xyz.mp4"]
>         O1 --> O4["logs/2026/01/01.gz"]
>         style O1 fill:#bfb,stroke:#333
>     end
>     Block -->|"format + mount"| File
>     File -->|"abstraction layer"| Object
> ```
> **What this diagram shows:** Block storage is a raw addressed array of fixed-size blocks. File storage adds a hierarchical namespace (directories, files, POSIX semantics) on top of blocks. Object storage deliberately strips away hierarchy, locking, and in-place mutation, keeping only a flat key–value namespace with four HTTP verbs. This minimalism is what enables planet-scale horizontal scaling.

---

## [32:29] The Trade-offs, and What You Get Back

The trade-off is summed up in one sentence: **To achieve infinite horizontal scale, you must give up mutability.** In distributed systems, every time you push for "infinite scale," the price is **immutability**—you cannot modify state in place. Object storage is one of the purest examples of this principle.

| Given Up | Gained |
|----------|--------|
| In-place modification (update/append/seek) | No coordination → linear scale-out |
| Hierarchical directories / rename / locking | No metadata tree → no hot metadata partitions |
| Microsecond latency (local syscall) | Millisecond latency (HTTP) but **any server anywhere** can serve |
| Provisioned capacity | **Effectively infinite** capacity |
| Single-copy durability | **11 nines** via erasure coding (see below) |
| Server-mediated downloads | **Direct browser-to-bucket** via pre-signed URLs |

---

## [38:06] What an Object Actually Is

An object has **four components**:

1.  **Key** — a string, the **complete and entire identity** of the object. Globally unique within the bucket (bucket name + key must be globally unique across the cloud platform, e.g., all of AWS).
2.  **Value** — an opaque **blob of bytes** (binary). No internal structure imposed.
3.  **System Metadata** — set by the storage service: `Content-Length`, `Last-Modified`, `Content-Type`, `ETag` (usually an MD5 hash for single-part uploads; different for multipart).
4.  **User Metadata** — arbitrary key-value pairs **you** attach at upload time (e.g., `x-amz-meta-uploaded-by: user-42`). Stored at the object level, returned on `HEAD`/`GET`.

All objects live inside a **Bucket** — a named container one level above objects. Bucket names are globally unique per provider (e.g., across all of AWS).

---

## [39:46] Folders Are an Illusion

When you see `uploads/2026/cat.png` in the AWS Console or Cloudflare dashboard, it *looks* like a folder hierarchy. **It is not.** There is exactly **one object** whose key is the literal string `"uploads/2026/cat.png"`. The slashes are ordinary characters. The console runs a **group-by algorithm** on the fly:

- `LIST` with `prefix=uploads/` and `delimiter=/` → server finds all keys starting with `uploads/`, chops each at the next `/`, and returns unique prefixes as `CommonPrefixes` (the "folders").
- **No folder object exists**. Moving a "folder" means copying *every* object to a new key prefix and deleting the olds—hence slow and costly.
- **Empty folders cannot exist** (no object = no prefix). Workaround: create a zero-byte object with key ending in `/` (e.g., `uploads/empty/`); the delimiter algorithm will surface it as a "folder".

> **Key Insight**: The "folder" UI is a **client-side rendering convenience**, not a server-side reality.

---

## [43:52] Designing the Key

The key is the object’s entire identity—choose it carefully. **Three non-negotiable rules**:

### Rule 1: Never Use the User-Supplied Filename
- **Collisions**: Two users upload `resume.pdf` → second `PUT` silently overwrites first (PUT = replace).
- **Path Traversal**: Filenames can contain `../`, null bytes, RTL overrides, emojis → can escape your intended prefix.
- **Sanitization Nightmare**: You cannot reliably sanitize arbitrary user input.

**Solution**: Generate the key yourself (random UUID, ULID, hash prefix). Store the original filename in your database **and/or** in user metadata (`x-amz-meta-original-filename`).

### Rule 2: Put High-Entropy (Random) Prefix First
Object storage indexes are **partitioned by key prefix** (sorted index). If every key in the next hour starts with `2026-08-26/` (timestamp prefix), all writes hit **one partition** → **hot partition** → 503 SlowDown errors.

**Bad**: `uploads/2026-08-26/user-123/abc.png`  
**Good**: `user-123/uploads/abc.png` or `a1b2c3d4/uploads/abc.png` (random/ULID/user-ID first).

> AWS has improved auto-partitioning, but **it costs nothing to do it right from day one**.

### Rule 3: Encode Ownership in the Key (Personal Preference)
Structure: `tenant-id/user-id/object-id`.  
**Benefit**: Your authorization middleware can validate access with a **string comparison** before ever calling S3. You can also scope IAM policies / pre-signed URL conditions to a single prefix (`tenant-id/user-id/*`), skipping a database hop entirely.

---

## [49:12] Behind the Interface: The Two Planes

When you `PUT` a 100 MB file to S3, the request hits one of **millions of stateless front-end nodes**. After authentication, the system splits into **two completely separate planes**:

| Plane | Responsibility | Characteristics |
|-------|----------------|-----------------|
| **Data Plane** | Store the 100 MB of bytes durably (erasure coding, replication across drives/racks/power domains). | Huge volume, **simple operations**, eventual consistency acceptable for background repair. |
| **Metadata Plane** | Maintain the **strongly consistent, globally distributed, sorted index** mapping `(bucket, key)` → `(shard locations, size, ETag, content-type, ...)`. | Tiny volume per entry, **massive operation count**, **must be strongly consistent**, single-digit-millisecond latency. |

> **Why metadata plane is the harder problem**: Storing exabytes on drives is a solved problem (decades of RAID/erasure coding). Maintaining a **strongly consistent distributed B-tree over trillions of keys** with millisecond lookups is *the* hard engineering challenge. Countless production systems at AWS scale depend on this single property.

### Strong Read-After-Write Consistency (Since Dec 2020)
Historically, S3 was **eventually consistent**: `PUT` → immediate `GET` could return 404 because the read hit a stale index replica. This caused massive pain (retry loops, sleeps). **Since December 2020, S3 guarantees strong read-after-write consistency**: a successful `PUT` (200) means *any* subsequent `GET`/`HEAD` will see the object.

> **Definitions**:
> - **Strongly consistent**: Write at *t* → read at *t+ε* guaranteed to see the write.
> - **Eventually consistent**: Write at *t* → read at *t+ε* may miss the write for some bounded but unbounded-in-practice delay.

---

## [56:34] Eleven Nines and Erasure Coding

**Durability** = "does the data still exist *somewhere*?" (vs. availability = "can I reach it *right now*?").  
AWS S3 promises **11 nines (99.999999999%)** annual durability: if you store 10 million objects, you statistically lose **one object every 10,000 years**.

### Naive Replication (3×)
- Store 3 full copies on 3 machines in 3 failure domains.
- Cost: **200% overhead** (3 PB raw for 1 PB data).
- Tolerates 2 simultaneous failures.

### Erasure Coding (e.g., 14+6 = 20 shards)
- Split object into **k=14 data shards**.
- Compute **m=6 parity shards** via **Reed–Solomon** coding.
- Scatter all 20 shards across 20 drives in different racks, power domains, buildings.
- **Magic property**: **Any 14 of the 20 shards** can reconstruct the original object.
- Overhead: **20/14 ≈ 1.43×** (vs. 3×) → **~half the raw storage cost**.
- Tolerates **6 simultaneous failures** (vs. 2).
- Trade-off: CPU for encoding/decoding + extra reads on recovery. Drives are more expensive than CPU → **net win**.

> **⚠️ Durability ≠ Backup**: 11 nines protects against *hardware failure*. It does **not** protect against `DELETE` (wrong prefix), application bugs that overwrite objects, or credential compromise.  
> **Mitigations**: Enable **Bucket Versioning** (every overwrite/delete creates a new version, old preserved). Replicate to a **second bucket in a different AWS account** (cross-account replication) so compromised credentials can’t nuke both.

---

## [59:37] The Metadata Plane: Distributed B-Tree Index

Conceptually, the metadata plane is a **giant sorted key-value index** (distributed B-tree / B+ tree) mapping `(bucket, key)` → `{shard locations, size, ETag, content-type, ...}`.

- **Partitioned by key range** → explains the hot-partition issue (Rule 2 above).
- **`LIST` is a range scan** over this distributed index → fundamentally slower than point lookup. **Never use `LIST` in the request path**. Your database should be the index of what files exist; the bucket just holds the bytes.
- **No atomic rename**: Renaming would require changing the key in the index while data stays put—but the key *determines the partition*, so it’s a cross-partition move = hard.
- **Instant `HEAD` size/ETag**: Metadata plane knows size/ETag without touching a single drive.

---

## [1:03:09] Conditional Writes: `If-None-Match` and `If-Match`

For nearly 20 years, object storage had **no compare-and-swap**. `PUT` always overwrote silently. If two processes wrote the same key concurrently, one silently won, the other silently lost—no way to detect or prevent.

**S3 added conditional writes in 2024**:

| Header | Value | Semantics |
|--------|-------|-----------|
| `If-None-Match` | `*` | **Create only if key does NOT exist**. Returns **412 Precondition Failed** if object exists. Prevents silent overwrites / duplicate-key bugs. |
| `If-Match` | `<ETag>` | **Replace only if current ETag matches**. Implements **compare-and-swap (CAS)**. Read object → get ETag → modify locally → `PUT` with `If-Match: <original-ETag>`. If another writer changed it first, ETag differs → **412** → your write fails, theirs survives. |

### Demo: Distributed Lock via `If-None-Match: *`
```bash
# First PUT succeeds (200) + returns ETag
aws s3api put-object --bucket my-bucket --key lock/leader --if-none-match "*" --body /dev/null

# Second PUT with same key + If-None-Match:* fails (412)
aws s3api put-object --bucket my-bucket --key lock/leader --if-none-match "*" --body /dev/null
# -> 412 Precondition Failed
```
This **is a distributed lock primitive**. Combined with strong consistency, S3 now supports leader election, write-ahead logs, and distributed databases (e.g., SQLite-on-S3, rqlite-style systems).

> **⚠️ S3-Compatible ≠ Conditional Writes**: Many "S3-compatible" providers (e.g., Backblaze B2) do **not** support `If-Match`/`If-None-Match`. Check documentation before depending on it.

---

## [1:10:32] Getting the File In: Two Architectures

### Architecture 1: Proxy Through Your Server (The Buffering Trap)
Browser → `POST /upload` (multipart/form-data) → Your Backend → `PUT` to S3 → DB insert.

**Pros**: Server sees every byte → can validate, scan, hash, reject. Auth is standard middleware. Fine for **small files** (< few MB).

**Cons**:
- **Buffering**: Default parsers (`parseMultipartForm` in Go, `bodyParser` in Express) load **entire file into RAM**. 20 concurrent 100 MB uploads = 2 GB RAM → OOM kill.
- **Local dev hides this**: Fast LAN, 16+ GB RAM, small test files.
- **Fix**: Stream — read request in chunks, pipe directly to S3 SDK uploader (memory = one chunk, ~KB). Wrap request body in a **size-limit reader** (reject > limit before reading).
- **Hard ceilings**: Load balancer idle timeout (default 60 s), Nginx `client_max_body_size` (413 error), API Gateway payload cap (few MB, unchangeable). **Not production-grade for large files**.

> **Mermaid: Proxy Upload (Buffering vs. Streaming)**
> ```mermaid
> sequenceDiagram
>     participant Browser
>     participant LB[Load Balancer]
>     participant Server[Your Backend]
>     participant S3[Object Storage]
>     
>     Note over Browser,Server: Buffered (Bad)
>     Browser->>LB: POST /upload (4 GB)
>     LB->>Server: POST /upload
>     Server->>Server: Buffer ENTIRE 4 GB in RAM 💥 OOM
>     Server->>S3: PUT object
>     
>     Note over Browser,Server: Streaming (Better)
>     Browser->>LB: POST /upload (4 GB)
>     LB->>Server: POST /upload (stream)
>     Server->>S3: PUT object (stream, chunk by chunk)
>     Server-->>Browser: 200 OK
> ```
> **What this diagram shows:** In the buffered approach, the server reads the entire request body into memory before forwarding to S3, causing OOM on large files. In the streaming approach, the server acts as a pipe: it reads a chunk from the client and immediately writes it to the S3 SDK, keeping memory usage constant (one chunk) regardless of file size. However, both variants still route all bytes through your server, hitting load-balancer timeouts and proxy body limits.

### Architecture 2: Pre-Signed URLs (The Production Pattern)
**Core Idea**: Let the **browser talk directly to the bucket**. Bytes never touch your server.

**Constraints**:
- Bucket must be **private** (public = security risk).
- Cannot give browser AWS credentials.
- Browser needs **exactly one operation** (PUT/POST) on **exactly one key** for **~5 minutes** with **zero credential leakage**.

**Solution**: **Pre-Signed URL** — a normal HTTPS URL with signed query parameters:
```
https://bucket.s3.region.amazonaws.com/key?
  X-Amz-Algorithm=AWS4-HMAC-SHA256&
  X-Amz-Credential=AKIA.../20260826/region/s3/aws4_request&
  X-Amz-Date=20260826T120000Z&
  X-Amz-Expires=300&
  X-Amz-SignedHeaders=host&
  X-Amz-Signature=<HMAC-SHA256(secret, canonical-request-string)>
```
- Server computes HMAC with its **secret key** over a canonical request string (method, bucket, key, expiry, headers).
- Browser `PUT`s to that URL. S3 recomputes the same string, verifies HMAC → authorizes **without ever seeing the secret**.
- Changing **one character** of key/method/expiry/headers → signature mismatch → **403**.
- Expiry passed → signature mismatch → **403**.

> **Mermaid: Pre-Signed URL Flow**
> ```mermaid
> sequenceDiagram
>     participant Browser
>     participant Backend[Your Backend]
>     participant S3[Object Storage]
>     
>     Browser->>Backend: GET /presign-upload?filename=avatar.png
>     Backend->>Backend: Generate key, sign URL (HMAC with secret)
>     Backend-->>Browser: { uploadUrl, fields?, key, expiry }
>     Browser->>S3: PUT /key (file bytes) + signature in query
>     S3->>S3: Recompute signature, verify HMAC
>     S3-->>Browser: 200 OK
>     Browser->>Backend: POST /upload-complete { uploadId, key }
>     Backend->>S3: HEAD /key (verify size, type, ETag)
>     Backend->>DB: UPDATE status='completed'
> ```
> **What this diagram shows:** The backend never sees the file bytes. It only generates a time-limited, key-scoped, cryptographically signed URL. The browser uploads directly to S3. After upload, the browser notifies the backend, which verifies the object exists and matches expectations via a cheap `HEAD` request, then marks the database record complete.

---

## [1:16:42] Pre-Signed URLs: Deep Dive

### The Hole in a Signed URL: No Body Size Enforcement
A pre-signed **PUT URL** signs the *method, bucket, key, headers, expiry*—**not the body size**. A malicious (or buggy) client could `PUT` a 5 GB file to a URL you signed for a 200 KB profile picture. You pay the storage/ingress cost; your quota fills.

### Fix: Pre-Signed **POST** with a **Policy Document**
Instead of signing a URL, you sign a **JSON policy** and return:
- `url` — the bucket endpoint (e.g., `https://bucket.s3.region.amazonaws.com/`)
- `fields` — form fields the browser **must** include in the `multipart/form-data` POST:
  - `policy` — base64-encoded policy JSON
  - `X-Amz-Algorithm`, `X-Amz-Credential`, `X-Amz-Date`, `X-Amz-Signature`
  - `key` — the exact object key (or `${filename}` with `key-starts-with` condition)
  - ...other conditions

**Policy Conditions Example**:
```json
{
  "expiration": "2026-08-26T12:05:00Z",
  "conditions": [
    ["content-length-range", 1024, 5242880],   // 1 KB – 5 MB
    ["starts-with", "$Content-Type", "image/"],
    ["starts-with", "$key", "tenant-7/user-42/"]
  ]
}
```
- **Enforced by S3 itself** — your server never sees the bytes.
- Violations → **400 Entity Too Large** / **403** with condition name in error body.
- **Client gotchas**: `file` field **must be last** in FormData; don’t set `Content-Type` on `fetch` — let browser set `multipart/form-data; boundary=...`.

---

## [1:24:42] The Two-Phase Flow: Telling Your Database

Since the browser uploads directly, your backend **never sees the upload happen**. How does the DB learn the file exists?

### Phase 1: Request Upload URL
1.  Browser → `POST /api/upload/start` `{ filename, contentType, size }`
2.  Backend: Authorize → **generate key** (`tenant/user/ULID.ext`) → **insert DB row** `status='pending', key=..., expected_size=..., expected_type=...` → generate **pre-signed POST URL + policy** → return to browser.

### Phase 2: Complete
3.  Browser → `POST` to S3 with policy fields + file (direct).
4.  Browser → `POST /api/upload/complete` `{ uploadId }` (or just `key`).
5.  Backend → `HEAD` object in S3 → verify `Content-Length`, `Content-Type`, `ETag` match expectations.
6.  Backend → `UPDATE` DB row `status='completed'`.

**Why `HEAD`?** Cheap metadata-only call; confirms object exists and matches policy *after* the fact (defense in depth).

---

## [1:26:50] Cleanup: The Uploads That Never Finish

A percentage of uploads **will be abandoned**: network drop, tab close, battery death, user cancels. Two cleanup axes:

1.  **S3 Lifecycle Rule**: Delete objects under `pending/` prefix after **24 hours** (or `multipart/uploads` for multipart).
2.  **Background Job** (cron / scheduled worker): Scan DB for `status='pending'` older than 24 h → mark `expired`. Keeps DB and bucket from **drifting apart** (two sources of truth: DB = index of files; Bucket = actual bytes).

---

## [1:28:21] CORS: The Final Piece

When a browser uploads directly to `bucket.s3.region.amazonaws.com`, it’s a **cross-origin request** (your frontend origin ≠ S3 origin). The bucket **must have a CORS configuration** allowing:
- `AllowedOrigins`: your frontend domain(s) (e.g., `https://app.example.com`)
- `AllowedMethods`: `PUT`, `POST` (and `GET`/`HEAD` for downloads)
- `AllowedHeaders`: `*` (or specific: `Content-Type`, `x-amz-*`)
- `ExposeHeaders`: **`ETag`** (required for multipart upload completion in Part 2)
- `MaxAgeSeconds`: cache preflight (e.g., 3600)

> **Debugging tip**: If `curl`/Postman works but browser fails → **it’s CORS**. Check bucket CORS policy first.

---

## [1:30:00] Four Implementation Notes & Resumable Uploads (Preview of Part 2)

The video closes with four practical notes that bridge into Part 2 (Multipart Uploads):

1.  **Single PUT limit**: 5 GB max object size for a single `PUT`. Larger files **require multipart upload**.
2.  **Multipart upload mechanics**: Initiate → upload parts (5 MB – 5 GB each, last part can be smaller) → complete. Max **10,000 parts**.
3.  **ETag on multipart**: Not an MD5 of the whole file. It’s `MD5(part1) + MD5(part2) + ...` + `-<part-count>` (e.g., `"d41d8cd98f00b204e9800998ecf8427e-125"`). You **cannot** verify integrity by hashing the downloaded file and comparing to ETag.
4.  **Abandoned multipart uploads**: Parts consume storage until `CompleteMultipartUpload` or `AbortMultipartUpload`. Use **Lifecycle Rule → AbortIncompleteMultipartUpload after N days**.
5.  **Resumable uploads**: Browser uploads parts in parallel (e.g., 5 at a time), tracks completed parts in `localStorage`/IndexedDB. On resume, `LIST parts` → upload missing parts → complete. Demo: 2 GB file → 125 parts × 16 MB, 5 concurrent → survives network interruptions.

> **Part 2 will cover**: Multipart upload deep dive, presigned URLs for each part, parallel upload coordination, resumable client logic, download range requests, video seeking, and cost optimization.

## Key Takeaways

1.  **The naive design (local disk + DB path) fails at scale** due to six independent problems: ephemeral disks, horizontal scaling incompatibility, fixed disk ceilings, single-copy durability risk, server-mediated downloads wasting compute, and the file–DB transaction gap.
2.  **Object storage is a deliberate minimalism**: four HTTP verbs (PUT, GET, DELETE, LIST), immutable objects, flat namespace, no locking, millisecond latency. This buys unlimited horizontal scale, infinite capacity, 11-nines durability, and direct browser-to-bucket data paths.
3.  **Folders are an illusion** created by prefix grouping (`LIST` with `delimiter=/`). No folder objects exist; moving/renaming "folders" copies every object.
4.  **Key design is critical**: Never use user filenames. Generate keys yourself with high-entropy prefix first (ULID, UUID, user-ID) to avoid hot partitions. Encode ownership (`tenant/user/id`) for zero-hop authorization.
5.  **Two-plane architecture**: Data plane (bytes, erasure coding, eventual consistency for repair) + Metadata plane (distributed B-tree index, strong consistency, millisecond lookups). The metadata plane is the harder engineering problem.
6.  **Strong read-after-write consistency since Dec 2020**: `PUT` 200 → immediate `GET`/`HEAD` guaranteed to see the object. No more retry loops.
7.  **Erasure coding (e.g., 14+6) beats 3× replication**: ~1.43× overhead vs. 3×, tolerates 6 failures vs. 2. CPU cost < drive cost.
8.  **Durability ≠ backup**: 11 nines protects against hardware failure, not `DELETE`, bugs, or credential compromise. Enable **versioning** + **cross-account replication**.
9.  **Conditional writes (`If-None-Match: *`, `If-Match: <ETag>`) enable CAS**: Prevents silent overwrites, implements distributed locks, enables write-ahead logs on S3. Not all "S3-compatible" providers support this.
10. **Pre-signed POST URLs with policies are the production upload pattern**: Browser uploads directly; policy enforces size, content-type, key prefix at the storage layer. Server never sees bytes.
11. **Two-phase flow + cleanup**: Phase 1 = generate key, write `pending` DB row, return signed URL. Phase 2 = browser uploads, calls completion endpoint, server `HEAD` verifies, marks `completed`. Lifecycle rules + background job clean abandoned uploads and stale `pending` rows.
12. **CORS is mandatory for direct browser uploads**: Configure `AllowedOrigins`, `PUT`/`POST`, `ExposeHeaders: ETag`.

## Related Notes

- **MOC**: [[_00 - Backend from First Principles - Index]]
- **Previous**: [[23 - Concurrency & Parallelism - IO Bound vs CPU Bound]]
- **Next**: [[25 - Object Storage - Everything You Need to Know (Part 2)]]
- **Concepts**: [[Object Storage]], [[Pre-Signed URLs]], [[Erasure Coding]], [[Conditional Writes]], [[Multipart Upload]], [[CORS]], [[Distributed Systems]], [[Horizontal Scaling]]

> [!note] Source Fidelity
> This chapter is a comprehensive, timestamp-faithful transcription of **Sriniously "Backend from First Principles" Episode 24 (playlist position 24, video ID `Ie0TjKI9cDI`)**. Every concept, example, demo, and diagram originates from the video transcript. Caption transcription errors have been silently corrected (e.g., "PZIX" → POSIX, "mgabytes" → megabytes, "xabyte" → exabyte, "UYU ID" → UUID/ULID). Ambiguous or inferred passages are marked with ⇢ *inferred*. Mermaid diagrams were constructed from verbal descriptions; they do not appear in the source video. All timestamps reference the video's chapter markers and transcript cues.