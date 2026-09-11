---
title: "19. Graceful Shutdown"
tags:
  - backend
  - video-notes
  - graceful-shutdown
  - deployment
course: "[[_00 - Backend from First Principles - Index]]"
source: "https://www.youtube.com/watch?v=6rfBgphiCWM"
video_id: 6rfBgphiCWM
playlist_position: 19
duration_seconds: 2156
published: 2025-09-19
status: completed
---

# Part 19 — Graceful Shutdown

> [!info] Video reference
> - **Part 19 of 29** in the playlist [[_00 - Backend from First Principles - Index]]
> - **Watch on YouTube:** [19. Graceful Shutdown](https://www.youtube.com/watch?v=6rfBgphiCWM)
> - **Duration:** 35:56 | **Views:** 15,655 | **Likes:** 524 | **Published:** 2025-09-19
> - **Speaker/Channel:** Sriniously

> [!abstract] In this chapter
> What exactly graceful shutdown is, why we use it, and why it is so important. We start from a realistic disaster — a payment transaction cut off mid-deployment — and rebuild the solution from first principles: the **process lifecycle**, how an operating system politely talks to an application using **signals** (**SIGTERM**, **SIGINT**, **SIGKILL** and their differences), the graceful shutdown sequence (*stop accepting new connections → drain in-flight requests → clean up resources → exit*), the **drain timeout** and the trade-off in choosing it, coordination with load balancers and health checks/service discovery, cleaning up resources in reverse order of acquisition, and a live Go demo whose logs walk through the entire shutdown step by step.

---

## Part 1 — Why Graceful Shutdown Exists

### [00:00] The Realistic Scenario — A Payment Mid-Deployment

The video opens with a very realistic scenario. You are in the middle of processing a **critical payment transaction** and, suddenly, your server needs to restart for a deployment. Someone pushed something to the production environment, and your server needs to deploy itself.

```
sequenceDiagram
    participant LB as Load Balancer
    participant Old as Old server (current code)
    participant New as New server (new code)
    LB->>Old: sending traffic (a payment is mid-transaction)
    Note over New: new code is deployed and boots up
    LB->>New: health check ... passes ✓
    LB->>New: all NEW traffic now goes to the new server
    Old->>Old: stop accepting new connections, drain in-flight requests
    Old-->>LB: old server shuts down cleanly
    Note over LB: traffic fully on the new server
```

**What this diagram shows:** A zero-downtime deployment. While the old server is mid-payment, the new server (with the new code) boots up alongside it. Once the new server passes its health check, the load balancer starts sending new traffic to it, and only then does the old server need to drain its remaining requests and shut down. The dangerous window we care about is the moment in the middle: a request that is *still being processed* when the old server is told to leave.

For deployments we already have techniques called **zero downtime deployment** which make sure that the existing server does *not* go down before the new server — the server with the new code — comes up and is ready to receive traffic. Those mechanisms exist. **But** at some point, when the new server is ready to go online and ready to receive traffic, the old server *has* to shut down. It has to stop receiving traffic, and the transition happens to the new server.

### [00:43] Lost in a Digital World, or Charged Twice?

During that critical moment — you are in the middle of a transaction. Say this is an e-commerce transaction: you are buying something from **Amazon** or **Flipkart**, and the Amazon/Flipkart server needs to restart for a deployment.

The question is now very concrete:

> What exactly happens to that payment? Does it get lost in the digital world? Or does the customer get charged **twice** because of some kind of **race condition**?

These are the scenarios you have to think about as a backend engineer. And this is **not a new problem** — it has been around *since the start of servers and backends*. The good news is that we already have a solution, and that solution is called **graceful shutdown**. It is exactly as the name sounds: we want to stop our server *gracefully*. We don't want to stop abruptly or suddenly. That is the whole idea.

### [01:47] Graceful Shutdown = Teaching the Server Good Manners

There are some **surrounding concepts** to understand first, so that your foundations are clear about *why* we do this. But if we oversimplify, graceful shutdown means: **we want to teach our backend good manners**.

A backend cannot just stop abruptly when it is in the middle of a transition into a new deployment. It has to perform some steps. It cannot just "slam the door" when it is time to leave, when it is time to shut down. Instead, your application:

1. **politely finishes whatever it is doing** — finishes its ongoing conversations,
2. **says goodbye to all the guests** and **cleans up after itself**,
3. and only **then closes the door**.

Think of hosting guests: when it's 9:00 PM and it's time to go to sleep, you don't push your guests out of the door and slam the door on their face. There are steps you perform. It is exactly the same for a server.

In this video we talk about *the art and the science* of making backend applications as good-mannered and as efficient as possible. Worrying about graceful shutdown:

- gives your application a **very good user experience**, and
- **avoids issues like data corruption** — because if you're in the middle of a payment transaction, you avoid double-charging the customer, the transaction getting lost, and processing refunds.

---

## Part 2 — Process Lifecycle and OS Signals

### [03:43] Process Lifecycle Management

The first concept we need is **process lifecycle management**. We need it because the backend is an application which runs as a **process** in some kind of server, some kind of computer.

A **process** is the term for *anything that runs in an operating system*. Everything that runs in an operating system runs as a process. If you already know operating-system concepts, this will make sense; if not, you can simply learn the term: whatever you run in an OS runs as a process.

And like all living things, every process has a **life cycle**: when it starts, *how* it starts, when it ends, and *how* it ends. In a way:

- processes are **born** when they start,
- they **live** while they are executing,
- they **die** when they are terminated.

This whole arc — birth, life, death — is the **life cycle of a process**. This is very closely connected to how graceful shutdown is implemented, so we have to understand it first.

```
stateDiagram-v2
    [*] --> Born: process starts
    Born --> Running: scheduled, executing
    Running --> Running: doing work (serving requests)
    Running --> Terminated: graceful stop (SIGTERM/SIGINT handled)
    Running --> Terminated: forceful kill (SIGKILL)
    Terminated --> [*]
```

**What this diagram shows:** The three states of a process's life — born (started), running (executing the application's work), and terminated (dead). There are two very different ways to travel from "Running" to "Terminated": the graceful route (the application cooperates) and the forceful route (the operating system kills it). The whole chapter is about which route is taken and how to guarantee the graceful one.

### [05:20] The Operating System Does Not Pull the Plug

When the operating system decides it is time for your application to stop running, it does **not** just pull the plug. It does not just kill the process. Instead, it follows an **established protocol of communication** — an established protocol for *how to communicate with the process* to say "it is time for you to stop," followed by the steps to stop it.

You can imagine it as a **conversation** between your operating system and your application (which is running inside a process):

> **OS:** "Hey, it's time for you to stop."
> **Application:** "Okay, give me a few minutes" — realistically, a few seconds — "then I'll stop myself, or you can stop me."

```
sequenceDiagram
    participant OS as Operating System
    participant App as Application (process)
    OS->>App: "Hey, it's time for you to stop" (sends a signal)
    App->>App: signal handler runs: finish requests, clean up
    App-->>OS: "I'm done" → exits cleanly
```

**What this diagram shows:** The polite negotiation between the OS and the application. The OS sends a message; the application gets a window to finish its work and shut itself down *cleanly*; only then does it exit. This is what a graceful shutdown conversation looks like at the system level.

This conversation does **not** happen through text — programs don't understand text. The whole communication between an application and the operating system happens through a concept called **signals**.

### [06:00] Signals — The Unix Communication Protocol

**Signals** are an important concept in **Unix operating systems**. By "Unix", we mean all the Linux operating systems — Arch Linux, Ubuntu, whatever you're familiar with — and also **Macs**: macOS originated from a Unix core, a Unix kernel. And in the real world, whenever we talk about **servers** we almost always mean Linux: around *99% of the time*, when you deploy an application with a cloud provider, it selects a Linux operating system. You'll never see Windows except for specialized use cases like Windows Server. So for deploying servers we mostly use Linux-based OSes.

Unix operating systems have the concept of **signals**, which is used for **IPC — interprocess communication**. Simply speaking, IPC is the technique by which *two processes communicate with each other* using an established protocol (you don't have to worry about the details of that protocol).

### [07:16] Handlers — Code That Waits for the Signal

Your application runs inside a process, and it **registers handlers**. What is a handler? You can imagine it as some kind of **code which waits**, running continuously behind the scenes, waiting for some communication — some signal — from your operating system.

- handlers **detect** whenever these signals come, and then **do something**;
- handlers basically tell the operating system: *"When you want me to stop, send me this specific message, and I'll handle it appropriately — I'll stop myself using my predefined steps."*

You cannot just say "stop" — that's human-readable text. There has to be a specific message, and we'll now talk about what kind of message it is.

### [08:49] The Two Major Types of Signals

There are two major types of signals we'll talk about — plus an important third one the presenter adds a moment later:

- **SIGTERM** — the SIG means *signal*, TERM means *terminate*.
- **SIGKILL** — SIG means *signal*, KILL stands for the *kill* command.
- **SIGINT** — SIG means *signal*, INT means *interrupt* (added shortly after, when the presenter "forgot" it the first time).

The following table summarises the three — keep it handy for the rest of the chapter:

| Signal | Full meaning | Who/What sends it | Can the app catch it? | Typical use |
|---|---|---|---|---|
| **SIGTERM** | signal + terminate | Programs: deployment systems, process managers, orchestration (Kubernetes, systemd, PM2) | Yes | Production shutdown (polite) |
| **SIGINT** | signal + interrupt | A human pressing **Ctrl+C** on the keyboard | Yes | Development, user-initiated shutdown (polite) |
| **SIGKILL** | signal + kill | Anything, as a last resort | **No — cannot be caught or ignored** | Forced, instant kill (impolite) |

### [09:14] SIGTERM — The Polite Nudge

**SIGTERM** is the **polite way for the operating system to ask your application to shut down**. It is not an extreme way; it's just a nudge. Imagine standing somewhere and someone comes from behind and pokes your shoulder — *"hey."* SIGTERM is something like that: a gentle nudge meaning:

> "Excuse me, could you please finish up and leave?"

Because it is a gentle request, your application has the **opportunity to complete whatever it is already doing**. It does not have to leave that very moment — it has some kind of **window**, realistically a few seconds, to finish its work.

What *might* it be doing? Since we're talking about backends: an **HTTP backend** is primarily processing requests. Your client (an app, a web app, a Chrome extension — whatever it is) sends HTTP requests to the backend, and the backend processes them and returns responses. At a random point in time, a backend might be processing **10 or 12 requests** — and if the application is big enough, perhaps **500 or 600 requests concurrently**. When it gets a SIGTERM signal, it is time to finish processing those requests.

So what does the server actually do on SIGTERM? Three steps:

```
flowchart LR
    A["Receive SIGTERM"] --> B["1. Finish existing (in-flight) requests"]
    B --> C["2. Clean up resources"]
    C --> D["3. Exit"]
```

**What this diagram shows:** The high-level graceful shutdown procedure triggered by SIGTERM: first finish the requests already in progress, then clean up the resources the app holds, and finally exit. Steps 1 and 2 are covered in depth later in the chapter; for now this is the bird's-eye view.

### [11:44] Who Uses SIGTERM?

SIGTERM is mostly used by:

- **deployment systems**,
- **process managers**, and
- **orchestration platforms** like **Kubernetes**.

Basically, *any kind of system you have established for managing your process*. Concretely, that could be Kubernetes, **systemd**, or **PM2** (a well-known Node.js process manager). These tools use SIGTERM to properly let your application finish whatever it is doing, clean up, and leave gracefully.

### [12:20] SIGINT — The Interrupt (Ctrl+C)

The second important signal is **SIGINT** (signal + **interrupt**). When does it happen? If you are a developer, you have probably already used it — its most famous use case is **Ctrl+C**.

If you've worked with any command-line or terminal-based application, you know the pattern: some process is running, some task is executing, and you want to *abruptly close it at that moment* — you press **Ctrl+C** on your keyboard and the process is instantly stopped.

The example in the video: the presenter is running a **Go-based backend** locally. It is running, ready to accept requests, and serving clients. He presses Ctrl+C, and the app logs (it already has graceful shutdown implemented): *"A signal has been received and the signal type is an interrupt signal."*

A SIGINT signal **requires a user — a developer — to press some key** (Ctrl+C) using a keyboard, so it is mostly used **during development environments**. When process-to-process communication happens, programs normally *don't* use it, precisely because SIGINT requires a key press. That's why it is also called a **user-initiated shutdown**.

Here is the key mental model from the video:

> **SIGINT** is used by *developers* (because it requires a key press). **SIGTERM** is used by *programs*.

And yet, **in pretty much all cases, you want to handle SIGINT the same way you handle SIGTERM**. It makes sense: it does not matter whether the backend runs in a development environment on your local machine (stopped with Ctrl+C) or in a cloud provider like an AWS EC2 instance managed by PM2 (which would send SIGTERM programmatically or manually). Whether a *human* is initiating the shutdown or a *program* is, the point is the same — the **intention** is that we want to shut down. So we want to shut down in a clean, graceful way either way.

---

## Part 3 — SIGKILL, the Nuclear Option

### [15:07] SIGKILL — Instant, Unstoppable Death

The final signal is **SIGKILL** — the presenter writes it in *red*, because it is the kill signal. As before, *SIG* means signal and *KILL* is the actual command. It is exactly as it sounds: **we want to instantly kill the application**.

The interesting thing about this signal: **it cannot be caught, and it also cannot be ignored.** That means:

- We **cannot register a handler** in our application that would do cleanup tasks when SIGKILL arrives — that will never happen, because this signal *cannot be detected* by the application; the application is simply not given that permission or capability.
- It **cannot be ignored** — you cannot say "since I wasn't able to detect it, I'll just ignore it and I don't really have to stop." That doesn't happen either.

So if your application is sent a SIGKILL signal, it will not be able to detect it, and it **has to stop at that particular moment**. Nothing else happens — it just stops. That is why it is called a *kill* signal.

### [16:50] The Nuclear Option — Pulling the Plug

Think of SIGKILL as the **nuclear option**. It is the equivalent of — instead of clicking on your system icon and clicking *Shutdown* — going to your power plug and **just pulling the plug**. That's it: your computer just dies. That is exactly how SIGKILL works.

And **this is why graceful shutdown is an important concept**. If you don't respond to, don't respect, the **polite signals**:

- **SIGTERM** and **SIGINT** are the polite signals — they let you finish whatever you're doing, let you clean up, and let you exit gracefully.

If you don't respect them, then *eventually* you **will receive a kill signal**. You will have to stop — and you won't even get the opportunity to clean up after yourself.

```
flowchart LR
    START["Server running,\nprocessing requests"] -->|"politely cooperates"| P["SIGTERM / SIGINT received"]
    START -->|"ignores the polite signals"| K["eventually SIGKILL"]
    P --> DRAIN["drain in-flight requests"]
    DRAIN --> CLEAN["clean up resources"]
    CLEAN --> EXIT["exit cleanly"]
    K --> DEAD["instant death —\nno cleanup at all"]
```

**What this diagram shows:** The fork in the road. Respect the polite signals (SIGTERM/SIGINT) and the process gets to drain, clean up, and exit gracefully. Ignore them and the orchestrator escalates to SIGKILL — instant death with zero opportunity to clean up. The whole point of implementing graceful shutdown is to make sure your app always takes the top path.

---

## Part 4 — The Graceful Shutdown Sequence in Detail

### [17:44] The Two Key Steps: Finishing Requests and Cleaning Up

We earlier mentioned two important things that we now look at in depth:

1. What does **finishing existing requests** mean?
2. What does **cleaning up resources** mean?

These are the two important steps that happen during a graceful shutdown process.

The first important part is **stopping the on-the-fly requests** (also written *in-flight* requests). What are "on-the-fly" requests? Your HTTP server/backend can process **multiple requests at the same time** — *concurrently*. So when it's time to stop your server, it is entirely possible that the backend is already in the middle of processing a couple of requests: maybe 10, 12, hundreds, or even **thousands**, depending on the scale of your server. Those are the on-the-fly requests: **the requests which are already getting processed by your server at a particular time**.

### [18:54] The Restaurant Analogy

To understand the whole flow, imagine a **restaurant**. You have gone with friends, and for some reason the restaurant has to close — maybe it's closing time, like 10:30 or 11 PM. What exactly happens? The restaurant owners cannot just come up and turn all the lights off, and they certainly cannot throw you out. That's not a good idea.

Instead:

1. **They ask someone at the reception / at the gate to stop allowing new customers in.** First step — you don't want new customers that you'd have to deal with and say no to.
2. **They announce to all the existing customers** — the people already having their meal: *"It's time for us to close up. You have, let's say, 15–20 minutes to finish your meal. Take your time; that's more than enough. Then please pay your bills, pay your tips, and leave."*

### [20:19] Connection Draining

Now imagine the same situation for our backend, our application — we call that process **connection draining**. It means that when your application receives a shutdown signal (a SIGTERM sent from some process, or a SIGINT from a developer's Ctrl+C):

**The first thing your application has to do is stop accepting new connections.** Just like the restaurant stops letting in new customers, the application stops accepting new connections and new requests *from any kind of client*, so that we can deal with the existing connections — the requests already being processed — and let them finish as soon as possible.

```
flowchart LR
    C["Clients"] -->|"new requests"| B["Backend"]
    B -->|"stops accepting new connections"| STOP["🚪 reception closes"]
    B -->|"in-flight requests continue"| DRAIN["existing requests finish"]
```

**What this diagram shows:** During connection draining, the backend closes the door to new traffic (like the restaurant reception stopping new customers) while every request that already made it inside is allowed to run to completion. New clients get nothing; old clients finish what they started.

### [21:20] Connection Draining Depends on the Architecture

The *implementation* of connection draining is obviously different depending on the architecture of the application we're dealing with:

- **HTTP backend** (the core backend, the HTTP server): it has to **stop accepting new HTTP requests** from any kind of client, and **allow the in-flight / on-the-fly requests** — the existing ones — to complete.
- **Database application**: as discussed in previous videos, a database is *also* a backend — it's an application, just not in the HTTP sense. For a database, connection draining means it has to **finish all existing queries / transactions**, and **stop taking new transactions or new queries into execution**, before it closes its connection.
- **WebSocket-based connections**: it has to **first notify the clients that it is closing**, and *then* close the socket. It cannot just close the socket abruptly. Once again: in-flight requests continue until the WebSocket is intentionally and ceremoniously ended.

So depending on the architecture — HTTP backend, database, WebSocket — the technical steps differ, but **the high-level idea is the same**:

1. **Stop accepting new connections.**
2. **Stop accepting new requests.**
3. **Finish the existing ones.**
4. **Then close the connection.**

A three-step process (counting stop-new-traffic as the first step).

### [23:00] The Challenge: Timing and the Timeout

The challenge with connection draining is **the timing of it**: you want to give the existing connections *enough time* to complete their work, but you cannot really wait *as long as they need*. There has to be some kind of limit.

That's why **most production systems / most backend systems implement a timeout mechanism** — they'll have some kind of **timeout**. For example, **30 seconds** or, say, **60 seconds** — it depends on you. The most common value is **30 seconds**.

The timeout is the **maximum duration your system will wait for you**:

> "We'll give you 30 seconds to finish whatever requests you're processing. After that, we'll just stop."

Usually this is fine: if you are not accepting new requests or new connections, then 30 seconds should be more than enough to finish all the existing requests. But if — for some reason, some kind of **blocking operation** — you cannot finish within this window, then you will be **forcefully stopped**. We do have a backup plan: we cannot just wait for our backend to process all requests while taking as long as it needs. There has to be a limit, and **the timeout is that hard limit** — you have exactly this amount of time to finish whatever you are doing.

```
flowchart LR
    T0["Shutdown signal received →\nstop accepting new connections"] --> T1["in-flight requests drain"]
    T1 --> T2{"30s timeout reached?\n(hard limit)"}
    T2 -->|"work finished in time"| CLEAN["clean up resources → exit cleanly"]
    T2 -->|"work still running (blocking op)"| FORCE["forceful stop"]
```

**What this diagram shows:** The drain window. The process gets the shutdown signal, stops accepting new connections, and must finish its in-flight requests before the timeout (commonly 30 seconds). Finish in time → graceful exit. Exceed the hard limit → the system stops waiting and force-stops the process. The timeout is what keeps a graceful shutdown from becoming an infinite hang.

### [24:11] Choosing the Timeout — A Design Trade-off

Choosing this timeout creates a **very interesting design consideration**: *how long exactly should you wait?*

- **If it is too short**, you risk interrupting **actual, legitimate operations** — real requests get killed mid-flight.
- **If it is too long**, your whole shutdown process becomes **sluggish**, and that eventually impacts your **deployment speed** and your **system responsiveness**.

So the right amount depends on:

- your application's **typical request duration**, and
- your **operational requirements**.

This is **not a hard and fast rule** — 30 seconds / 60 seconds are common starting points, but you have to *understand your systems* and the *kind of requests* you are processing:

- For a **traditional, normal backend**, 30 seconds should be more than enough.
- For **WebSockets** or **other more complicated architectures**, you have to understand your system and **decide a timeout which suits you and your system accordingly**.

```
flowchart LR
    TOOSHORT["Timeout too short"] --> BAD1["interrupts legitimate in-flight operations"]
    TOOLONG["Timeout too long"] --> BAD2["sluggish shutdown → slower deploys,\nworse responsiveness"]
    BALANCED["Timeout matched to typical\nrequest duration + ops needs"] --> GOOD["fast and safe shutdown"]
```

**What this diagram shows:** The two failure modes when tuning the drain timeout. Too short and you chop off legitimate work in progress; too long and every deployment crawls. The sweet spot is a timeout sized from your own typical request duration and operational requirements — not a magic number copied from a blog post.

### [25:25] Coordination: Load Balancers and Service Discovery

Connection draining also requires **coordination between your load balancers and your service discovery systems**. It has to work with:

- your **health check systems**, and
- **registering and deregistering with your service discovery** mechanism.

**Service discovery** is a slightly advanced concept: basically, if you have deployed a set of applications — say your backend, your database, your Elasticsearch instance — then *how they connect with each other and communicate with each other after deployment* is the responsibility of your **service discovery tool / mechanism**. Connection draining plugs directly into this: the instance that is shutting down must be deregistered (or flipped to "unhealthy") so the load balancer stops sending it new traffic before it starts draining.

```
sequenceDiagram
    participant LB as Load Balancer
    participant App as Backend instance (draining)
    participant SD as Service discovery / health checks
    App->>App: receives shutdown signal
    App->>SD: deregister / mark unhealthy
    SD-->>LB: instance unavailable → stop sending traffic
    LB->>LB: no new requests to this instance
    App->>App: drains in-flight requests, cleans up, exits
```

**What this diagram shows:** The coordination dance that sits *around* connection draining. The moment the instance decides to shut down, it deregisters from service discovery (or fails its health check), so the load balancer stops routing new traffic to it. Only then does the drain begin — otherwise the load balancer would keep sprinkling new requests into a server that is trying to leave.

### [26:03] Resource Cleanup

The second important step is **cleanup — resource cleanup**.

Think about your desk: you have a working area, and when it's time to leave the house or go to sleep, you do some kind of cleanup before leaving — you take your coffee cup and put it in the sink, or you tidy your cables. We all have some tiny cleanup ritual before we leave our desks. Same thing for a backend application.

When we say **resources** in the context of the backend, we mean things like:

- **file handles**,
- **network connections**,
- **database connections**,
- **temporary files**,
- **caches**,
- or **any other system resources** the application acquired during its execution.

The application has to **let go of all of them**.

```
flowchart TD
    R["Resources an application must clean up\nduring graceful shutdown"] --> F["file handles"]
    R --> N["network connections"]
    R --> D["database connections"]
    R --> T["temporary files"]
    R --> C["caches"]
    R --> O["other system resources"]
```

**What this diagram shows:** A checklist of everything a backend typically holds onto while running. "Cleaning up resources" during a graceful shutdown means deliberately releasing each of these — otherwise the process dies with them still open, creating leaks and inconsistent state on the way out.

**File handles.** The way file-system access works with programs: your application sends a request to the operating system, and the OS provides you a **handle** to that file. At a high level, you get a handle you must let go of — you must clean it up at some point. Otherwise that handle keeps running, you acquire more and more **memory (RAM, random access memory)**, and eventually you run out of it. Cleaning up after your file handles is important.

**Network connections.** The most common kind of resource cleanup is cleaning up network connections. The OS is the *mediator*: all requests come from the Internet through your operating system (it receives them from your **network card** and hands them to your application). The OS therefore knows all about your network connections. Typically, operating systems **limit the number of file handles** *and* **network connections** a particular process can have open simultaneously. Just like file handles: if you don't give up a network connection after dealing with it, you eventually run out of memory or face performance issues.

**Database connections.** Before your backend process is shut down, the **database transactions** your backend was dealing with must either be **committed or rolled back — explicitly, by your application**. If you don't, those transactions might end up in an **inconsistent state**, which can lead to things like **deadlocks** or **data corruption** and all different kinds of issues. These — file handles, database connections, network connections, etc. — are what we mean when we say *resource cleanup*.

---

## Part 5 — Thinking in Reverse Order, and the Working Code

### [29:31] Clean Up in Reverse Order of Acquisition

One more thing to keep in mind while cleaning up: in the whole graceful-shutdown workflow, we want to **clean up resources in the reverse order of the way we acquired them**.

For example, suppose you first established a **Redis connection**, then you established your **database connection**, and so on. When you give up your resources, you should go in the **reverse order** (database first, then Redis).

Why do we need to do this? **To prevent situations where we clean up a resource / an operation that depends on a previous operation.** If you tear down a dependency before the thing that depends on it, the dependent operation breaks mid-cleanup. Releasing in reverse acquisition order ensures each resource is released only after everything that relies on it has already been released.

```
flowchart LR
    subgraph Acquired["Acquisition order (startup)"]
        direction TB
        A1["1. Redis connection"] --> A2["2. Database connection"] --> A3["3. HTTP server"]
    end
    subgraph Released["Release order (shutdown)"]
        direction TB
        R1["1. HTTP server"] --> R2["2. Database connection"] --> R3["3. Redis connection"]
    end
    Acquired -.->|"reverse order"| Released
```

**What this diagram shows:** Startup acquires resources in one order (here: Redis, then database, then the HTTP server). Shutdown releases them in the exact reverse order (HTTP server, then database, then Redis). This prevents you from ever destroying a resource that another, still-active resource depends on.

### [30:14] A Practical Example — Graceful Shutdown in Code

This series typically avoids looking at code, but here the presenter shows a quick, practical, realistic example so we don't end up with just a hollow understanding of the graceful shutdown procedure. This is **Go code** — you don't have to understand it; you just follow the narrative (which is more than enough).

**Step 1 — Register a handler that waits for signals.** The first thing you do is register some kind of handler which will wait for signals from your operating system. In the code this is done using a **context** (a Go concept) — the process waits for *interrupts* from the OS. When an interrupt is received, it calls the **shutdown function** — the graceful shutdown function.

**Step 2 — What the graceful shutdown function does.** Let's go inside it. The first thing you see: **we shut down our HTTP server**. Since this is a backend application, the core part of it is the HTTP engine. The code calls a method provided by the framework/library used for the HTTP server (whatever framework/library you use typically provides such a function). When you call it, the library internally:

- **stops receiving additional connections**, and
- **finishes whatever existing connections** it has.

Then, when that function finishes, **we close our database**. Same idea as before: the database stops receiving additional queries and transactions, finishes the existing ones, and gives up whatever handles — whatever **database connections** — it has.

### [31:45] Closing the Database — TCP and Connection Pools

Recall how a backend and a database connect: through **TCP connections**. For the backend to talk to the database, it must have an **active TCP connection** running — that's the only way they communicate.

When we talk about **database pooling**, we mean the backend holds a number of these active TCP connections to the database, and uses a *particular* connection from that **pool of connections** to communicate at any given moment.

So when you **close the database**:

1. it stops **accepting new queries** through those pooled connections,
2. it **finishes whatever existing queries / transactions** are already being processed,
3. and then it **starts closing each connection in the pool, one by one**.

That is exactly what "cleaning up database resources" means.

### [33:09] Cleaning Up Background Workers and Redis

In the end, the code is also **cleaning up the background job processing server** — the server that processes background jobs and runs on top of a Redis instance (the transcript's captions render these names ambiguously as "asynq"/"async Q" and "radius"; ⇢ *inferred* the library is **asynq** and the store is **Redis**, matching the background-jobs material from [[14 - Task Queues and Background Jobs]]). Internally, this function also **closes the Redis connection**. After all this, the application has **successfully gracefully shut down**.

```
sequenceDiagram
    participant App as Backend process
    participant Srv as HTTP server
    participant DB as Database
    participant W as Background workers (asynq/Redis)
    App->>App: receives shutdown signal (Ctrl+C → SIGINT)
    App->>Srv: call stop-accepting method
    Srv-->>App: finishes in-flight requests, closes
    App->>DB: close database
    DB-->>App: finishes queries/transactions, closes pooled TCP connections
    App->>W: stop background job processing server
    W-->>App: waits for workers to finish, closes Redis connection
    App-->>App: logs "server has exited properly" → process exits
```

**What this diagram shows:** The exact order of operations in the demo's graceful shutdown: stop the HTTP server (drain in-flight requests), then close the database (finish transactions, close pooled TCP connections), then stop the background-job workers and close their Redis connection, then exit. Notice it follows the reverse-order principle — the components started latest (workers/Redis after the server came up) are torn down first.

### [33:24] The Live Demo — Ctrl+C in Action

The presenter then runs the demo. He clears the terminal and **starts the server**. Then he presses **Ctrl+C**, which sends an **interrupt signal** to the backend and initiates the graceful shutdown procedure.

It "took some time" — but since there are **no in-flight requests** (it's a locally-run backend), it was able to shut down *instantly*; still, the logs show it took around **1 second** to properly shut down because it had to run through the shutdown sequence.

Looking at the logs:

- **Start logs** (in startup order): *connected to the database* → *starting our background job server* → *started our server*. That's the **starting phase**.
- After pressing Ctrl+C: the first thing done is closing the **database connection** — the log shows we closed the database connection. Then the **background job processing server** was stopped. The logs for background-job shutdown come from **asynq** (the background job processing library used), and when it received the shutdown signal (via the method call that shuts down all the Redis connections), it logged messages like:
  - *starting graceful shutdown*
  - *waiting for all workers to finish*
  - *all workers have finished*
  - *exiting*
- Finally, the server logs a message that **the server has exited properly**. And that's how you stop your server.

### [34:57] The Whole Picture — Process Lifecycle and Why It Matters

This whole workflow is called the **process lifecycle**:

- the **starting phase**,
- the **running phase**, and
- the **stopping phase**.

And *this part* — the stopping phase — is called the **graceful shutdown procedure**.

It is very important, especially **when we are deploying our applications**: we don't want to risk corrupting our in-flight requests and corrupting our workflows — any kind of workflows — and we want to provide a **very good, delightful user experience** to our customers and users.

### [35:24] What You Actually Need to Remember

That's pretty much all you need to understand about graceful shutdown. You don't really have to understand the code, because most probably you'll be using some kind of library or framework — **Node.js, Go, Rust, Python** — and most frameworks have graceful-shutdown code available which you can **just copy and paste** to implement the workflow.

But it *is* important that you understand **what exactly happens** when you do a graceful shutdown, and **why it is important**. That's why the video spends so much time on it.

---

## Key Takeaways

- **Graceful shutdown is how you stop a server without hurting anyone.**: In the middle of a deployment (or any restart), in-flight requests — payments, e-commerce checkouts — must be allowed to finish, or you risk lost transactions, double charges, refunds, and corrupted workflows.
- **Every application is a process**, and every process has a life cycle: born (started), running (executing), and terminated (stopped). The stopping phase is where graceful shutdown lives.
- **The OS communicates with memory-safe apps through *signals*** — an IPC (interprocess communication) concept from Unix/Linux (which is what servers run ~99% of the time). Applications **register handlers** that wait for specific signals and respond with predefined shutdown steps.
- **SIGTERM is the polite nudge** used by programs (Kubernetes, systemd, PM2, deployment systems). It asks the app to finish up; the app gets a window to comply.
- **SIGINT is Ctrl+C** — the *user-initiated shutdown* used by developers in development. Always handle it exactly the same as SIGTERM: the intention (shut down cleanly) is identical whether a human or a program asked.
- **SIGKILL is the nuclear option.** It cannot be caught *or* ignored — no handler runs, no cleanup happens, the process just dies (like pulling the power plug). If you ignore SIGTERM/SIGINT, you will eventually get SIGKILL and lose your chance to clean up.
- **Connection draining is the first step**: stop accepting new connections/requests (like a restaurant closing its reception), let the in-flight / on-the-fly requests finish, then close the connection. The exact mechanics vary by architecture (HTTP server vs. database vs. WebSockets) but the high-level three-step idea is the same.
- **The drain timeout is a hard limit** (commonly 30 seconds). Too short → you kill legitimate operations; too long → sluggish shutdowns that hurt deployment speed and responsiveness. Tune it to your typical request duration and operational requirements.
- **Draining must be coordinated**: deregister / fail health checks in service discovery so load balancers stop routing new traffic to the instance that is leaving.
- **Resource cleanup is the second step**: release file handles, network connections, database connections (transactions must be committed or rolled back explicitly, or you risk deadlocks and data corruption), temporary files, caches. Clean up **in the reverse order of acquisition** so you never destroy a dependency while something still depends on it.
- **Real-world implementation is often one call**: frameworks in Go, Node.js, Rust, and Python ship copy-paste shutdown helpers, but the value is in understanding the sequence — stop accepting → drain → clean → exit.

---

## Related Notes

- MOC: [[_00 - Backend from First Principles - Index]]
- Previous: [[18 - Logging, Monitoring and Observability]] — the demo's shutdown logs are the observability twins of this chapter.
- Next: [[20 - Backend Security - Everything You Need to Know]]
- Building blocks referenced here: [[12 - Mastering Databases with Postgres]] (transactions, commit/rollback), [[13 - Caching, The Secret Behind It All]] (Redis), [[14 - Task Queues and Background Jobs]] (asynq/background workers), [[05 - Understanding HTTP for Backend Engineers]] (HTTP request/response)

---

> [!note] Source fidelity
> This note is written from the video transcript (video_id `6rfBgphiCWM`, published 2025-09-19). Where the auto-generated captions were unclear, the meaning was inferred and marked with ⇢ *inferred* — notably the background-job library name (**asynq**, captioned "async Q") and the in-memory store (**Redis**, captioned "radius"), both consistent with earlier chapters of the series.