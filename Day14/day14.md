# Distributed Tracing: Understanding What Happens Inside We Connect

As We Connect grows from a simple application into a microservices-based system, a new problem appears.

A user sends **one request**.

But internally, that request may travel through **10 or 20 different components**.

For example:

```text
Mobile App
    |
    ▼
API Gateway
    |
    ▼
Feed Service
    |
    ├──> User Service
    |
    ├──> Post Service
    |
    ├──> Redis
    |
    └──> Recommendation Service
              |
              ▼
          Database
```

The user sees one request.

The system sees many requests.

So when something becomes slow or fails, we need to answer:

> **Where exactly did the problem happen?**

This is the problem **Distributed Tracing** solves.

---

# What Is Distributed Tracing?

Distributed tracing is a way to track a request as it travels across multiple services.

Instead of looking at each service independently, we can see the **complete journey of a request**.

For example:

```text
Request
  |
  ▼
API Gateway       20ms
  |
  ▼
Feed Service      80ms
  |
  ├── User Service       15ms
  |
  ├── Post Service       30ms
  |
  └── Recommendation     250ms
          |
          ▼
       Database           220ms
```

Immediately we can see that the Recommendation Service is responsible for most of the latency.

Without distributed tracing, finding this might require checking logs from several different services manually.

---

# Why Do We Need It?

In a monolithic application, debugging a request is relatively straightforward.

```text
Client
   |
   ▼
Monolith
   |
   ▼
Database
```

We can inspect the application's logs and follow the request.

But in We Connect:

```text
Client
   |
   ▼
API Gateway
   |
   ▼
Feed Service
   |
   ├── User Service
   ├── Post Service
   ├── Like Service
   ├── Comment Service
   └── Recommendation Service
```

Each service may have:

- Its own logs
- Its own database
- Its own deployment
- Its own instances
- Its own monitoring

Now debugging becomes much harder.

---

# The Problem Without Distributed Tracing

Imagine a user reports:

> "My home feed takes 3 seconds to load."

We check the Feed Service.

It says:

```text
Feed Service response time: 3000ms
```

But that doesn't tell us **why**.

We then check other services.

```text
User Service       → 20ms
Post Service       → 40ms
Like Service       → 30ms
Recommendation     → 2800ms
```

Then we discover:

```text
Recommendation Service
        |
        ▼
Database Query
        |
        ▼
2800ms
```

The problem was actually a slow database query.

Distributed tracing allows us to see this entire chain much faster.

---

# Trace and Span

Two concepts are fundamental to distributed tracing:

### Trace

A **trace** represents the complete journey of one request.

For example:

```text
Trace ID: abc123

Client
  ↓
API Gateway
  ↓
Feed Service
  ↓
Recommendation Service
  ↓
Database
```

The entire journey belongs to one trace.

---

### Span

A **span** represents one operation inside that trace.

For example:

```text
Trace
│
├── API Gateway              20ms
│
├── Feed Service             80ms
│
├── User Service             15ms
│
├── Post Service             30ms
│
└── Recommendation Service  250ms
```

Each individual operation is a span.

So:

> **Trace = complete request journey**

> **Span = individual operation within that journey**

---

# Trace ID

To connect all these operations together, we need an identifier.

That's where the **Trace ID** comes in.

Suppose a request enters We Connect:

```text
Trace ID:
4bf92f3577b34da6a3ce929d0e0e4736
```

The same trace ID is propagated across services.

```text
API Gateway
Trace ID = ABC123
       |
       ▼
Feed Service
Trace ID = ABC123
       |
       ├── User Service
       │   Trace ID = ABC123
       │
       └── Post Service
           Trace ID = ABC123
```

Now the tracing system can reconstruct the entire request.

---

# Span IDs

Every operation also gets its own Span ID.

For example:

```text
Trace ID: ABC123

API Gateway
Span ID: 001

    ↓

Feed Service
Span ID: 002

    ↓

Post Service
Span ID: 003
```

The Trace ID connects everything together.

The Span ID identifies the individual operation.

---

# Trace Context Propagation

This is one of the most important concepts.

How does the Trace ID travel from one service to another?

The tracing context is propagated through the request.

For HTTP communication, this is commonly done using tracing headers.

Conceptually:

```text
API Gateway
    |
    | Trace Context
    ▼
Feed Service
    |
    | Trace Context
    ▼
Post Service
```

The downstream service extracts the context and creates its own span under the same trace.

This allows the tracing system to understand the relationship between operations.

---

# Example: Loading the We Connect Home Feed

Let's take a real example.

A user opens the We Connect home page.

The client sends:

```text
GET /api/feed
```

The request travels through:

```text
Mobile App
     |
     ▼
API Gateway
     |
     ▼
Feed Service
     |
     ├──> User Service
     |
     ├──> Post Service
     |
     └──> Recommendation Service
```

The trace might look like:

```text
Trace: ABC123

API Gateway
├── 15ms
│
└── Feed Service
    ├── 80ms
    │
    ├── User Service
    │   └── 20ms
    │
    ├── Post Service
    │   └── 40ms
    │
    └── Recommendation Service
        └── 500ms
```

Now we know:

```text
Total latency ≈ 595ms

Recommendation Service = major contributor
```

This is much more useful than simply knowing:

```text
Feed Service = 595ms
```

---

# Distributed Tracing + Logs

Tracing becomes even more powerful when combined with logs.

Consider this log:

```text
ERROR Payment processing failed
```

It doesn't tell us which request caused the problem.

Now imagine:

```text
Trace ID: ABC123
Span ID: XYZ789

ERROR Payment processing failed
```

We can search for:

```text
Trace ID = ABC123
```

and find all related logs across services.

```text
API Gateway
    |
    ├── Log
    |
    ▼
Order Service
    |
    ├── Log
    |
    ▼
Payment Service
    |
    ├── Error
    |
    ▼
Database
```

This creates a much stronger debugging workflow.

---

# Distributed Tracing + Metrics

Metrics tell us:

> **Something is wrong.**

Logs tell us:

> **What happened.**

Traces tell us:

> **Where the request went and where it became slow or failed.**

For example:

```text
Metrics
   ↓
Feed API latency increased
   ↓
Tracing
   ↓
Recommendation Service is slow
   ↓
Logs
   ↓
Database query timeout
```

Together, they form a powerful observability system.

---

# Distributed Tracing + Microservices

As We Connect adds more services, tracing becomes increasingly valuable.

Imagine:

```text
                  API Gateway
                       |
                       ▼
                  Feed Service
                       |
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       User         Post         Like
      Service      Service      Service
                       |
                       ▼
                Recommendation
                    Service
                       |
                       ▼
                   Database
```

Without tracing:

```text
"Something is slow."
```

With tracing:

```text
Feed Service
      ↓
Recommendation Service
      ↓
PostgreSQL
      ↓
Slow Query
      ↓
850ms
```

We can quickly identify the bottleneck.

---

# Finding Cascading Failures

Distributed tracing is also useful for understanding cascading failures.

Suppose:

```text
Feed Service
      |
      ▼
Recommendation Service
      |
      ▼
ML Service
      |
      ▼
Database
```

The database becomes slow.

That causes:

```text
Database
   ↓
ML Service becomes slow
   ↓
Recommendation Service becomes slow
   ↓
Feed Service becomes slow
   ↓
Users experience slow feed
```

The user only sees:

> "We Connect is slow."

Tracing allows engineers to follow the chain backwards and identify the original source.

---

# Sampling

At We Connect scale, we may receive millions or billions of requests.

Storing every single trace can become expensive.

So tracing systems often use **sampling**.

For example:

```text
1,000,000 requests
        |
        ▼
    Sampling
        |
        ▼
  10,000 traces stored
```

We might choose different strategies.

### Head-based sampling

Decide whether to keep a trace when the request starts.

### Tail-based sampling

Wait until the trace finishes and then decide.

For example, we may want to keep:

- Errors
- Slow requests
- Important transactions

while sampling normal successful requests at a lower rate.

---

# What Should We Trace?

We don't necessarily need to trace every internal operation.

For We Connect, useful trace points could include:

```text
API Gateway
    ↓
Feed Service
    ↓
User Service
    ↓
Post Service
    ↓
Recommendation Service
    ↓
Database
```

We can also trace important asynchronous operations such as:

```text
Post Created
     ↓
Message Queue
     ↓
Notification Service
     ↓
Push Notification
```

This becomes especially important once We Connect adopts event-driven architecture.

---

# Distributed Tracing and Asynchronous Communication

Consider:

```text
User
 |
 ▼
Post Service
 |
 ▼
Message Queue
 |
 ├──> Notification Service
 |
 ├──> Feed Service
 |
 └──> Analytics Service
```

The request doesn't end when the message is published.

The processing continues asynchronously.

Tracing can help connect the producer operation with downstream processing, provided trace context is propagated through the messaging system.

This allows engineers to understand:

```text
Post Created
      ↓
Event Published
      ↓
Notification Processed
      ↓
Feed Updated
```

rather than debugging each service independently.

---

# Distributed Tracing Architecture

A simplified architecture looks like this:

```text
                    We Connect
                        |
                        ▼
                 ┌──────────────┐
                 │ API Gateway  │
                 └──────┬───────┘
                        |
              ┌─────────┼─────────┐
              ▼         ▼         ▼
           Feed       Post       User
          Service    Service    Service
              \         |         /
               \        |        /
                └───────┼───────┘
                        |
                        ▼
                Trace Instrumentation
                        |
                        ▼
                ┌───────────────┐
                │ Trace Backend │
                └───────┬───────┘
                        |
                        ▼
                 Trace Visualization
```

A commonly used open standard for this ecosystem is **OpenTelemetry**.

OpenTelemetry provides APIs, SDKs, and instrumentation approaches for collecting telemetry such as:

- Traces
- Metrics
- Logs

The collected data can then be sent to an observability backend.

---

# The Observability Stack

Distributed tracing shouldn't exist in isolation.

A mature We Connect platform can have:

```text
                 We Connect Services
                         |
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        Logs          Metrics         Traces
          |              |              |
          └──────────────┼──────────────┘
                         ▼
                  Observability
                     Platform
```

This gives engineers three different perspectives:

### Metrics

**What is happening?**

Example:

```text
Feed latency = 850ms
Error rate = 4%
```

### Logs

**What happened?**

```text
Database timeout
```

### Traces

**Where did it happen?**

```text
Feed
  ↓
Recommendation
  ↓
PostgreSQL
  ↓
Slow Query
```

---

# A Real We Connect Debugging Scenario

Imagine users complain:

> "The feed is slow."

### Step 1 — Metrics

We discover:

```text
Feed API P95 latency
100ms → 900ms
```

Something changed.

### Step 2 — Distributed Trace

We inspect a slow request:

```text
API Gateway       10ms
Feed Service      50ms
Post Service      40ms
User Service      20ms
Recommendation   750ms
```

The bottleneck is obvious.

### Step 3 — Trace Deeper

```text
Recommendation Service
        |
        ▼
Database Query
        |
        ▼
750ms
```

### Step 4 — Logs

We search the Trace ID in logs.

We discover:

```text
Database query timeout/retry
```

### Step 5 — Fix

We investigate:

- Missing index
- Inefficient query
- Database contention
- Connection pool saturation
- Increased traffic

The important point is that tracing dramatically reduces the time needed to move from:

**"Users say the system is slow"**

to:

**"This specific database operation inside Recommendation Service is causing the latency."**

---

# Common Mistakes

### Mistake 1: Only tracing the API Gateway

If tracing stops at the gateway, we still don't know what happens downstream.

### Mistake 2: Not propagating trace context

Each service creates a new trace.

Now the complete request cannot be reconstructed.

### Mistake 3: Storing too much data

Tracing every request and every operation can become expensive.

Sampling and retention strategies become important.

### Mistake 4: Treating tracing as logging

Logs and traces solve different problems.

They work best together.

### Mistake 5: Ignoring asynchronous flows

Events and message queues need trace-context propagation too.

---

# Where Distributed Tracing Fits in We Connect

Our architecture is becoming more mature:

```text
                         Clients
                            |
                            ▼
                     Load Balancer
                            |
                            ▼
                      API Gateway
                            |
                            ▼
                   ┌─────────────────┐
                   │  Microservices  │
                   └────────┬────────┘
                            |
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
      Redis              Databases         Message Queue
        |                   |                   |
        └───────────────────┼───────────────────┘
                            |
                            ▼
                    Observability
                    ┌─────────────┐
                    │   Metrics   │
                    │    Logs     │
                    │   Traces   │
                    └─────────────┘
```

Now we are not just building a system that **works**.

We are building a system that we can **understand, monitor, debug, and operate at scale**.

---

# Key Takeaways

Distributed tracing helps us:

1. Follow a request across microservices
2. Identify latency bottlenecks
3. Debug failures across services
4. Correlate logs using Trace IDs
5. Understand cascading failures
6. Trace asynchronous workflows
7. Improve incident investigation
8. Measure service-level latency
9. Reduce debugging time
10. Build a strong observability platform

The most important mental model is:

```text
Metrics → Something is wrong

Logs → What happened?

Traces → Where did it happen?
```

And that is why distributed tracing becomes essential as We Connect evolves from a simple application into a distributed system.

---

## What's Next for We Connect?

We now have multiple services communicating through APIs and asynchronous events.

But another major problem appears:

**How do we make sure our system continues working when traffic suddenly increases?**

The next challenge is **Scalability and Auto Scaling** — understanding how We Connect can automatically add or remove application instances based on traffic and system load.