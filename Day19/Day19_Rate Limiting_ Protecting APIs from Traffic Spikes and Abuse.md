# Rate Limiting: Protecting APIs from Traffic Spikes and Abuse

Modern applications expose APIs to thousands or millions of users. But what happens when traffic suddenly increases?

A single user can accidentally send hundreds of requests per second. A bot can intentionally abuse an API. A traffic spike can overload downstream services.

This is where **Rate Limiting** becomes an important part of distributed system design.

---

## What Is Rate Limiting?

**Rate limiting controls how many requests a client can make within a specific period of time.**

For example:

```text
100 requests / minute / user
```

If a user sends:

```text
Request 1
Request 2
Request 3
...
Request 100
```

the requests are accepted.

When request 101 arrives before the limit resets:

```text
HTTP 429 Too Many Requests
```

The goal isn't necessarily to reject users. The goal is to **protect the system while allowing fair access to resources**.

---

# Why Do We Need Rate Limiting?

Consider an API:

```text
Client
   |
   v
API Gateway
   |
   v
User Service
   |
   v
Database
```

Suppose the API normally receives:

```text
1,000 requests/sec
```

Suddenly traffic increases to:

```text
50,000 requests/sec
```

Without rate limiting:

```text
50K requests
      |
      v
API Gateway
      |
      v
Application Servers
      |
      v
Database
      |
      X
   Overloaded
```

This can cause:

- High CPU utilization
- Memory exhaustion
- Database connection exhaustion
- Increased latency
- Thread pool exhaustion
- Cascading failures
- Increased infrastructure cost
- Service downtime

Rate limiting provides a controlled boundary:

```text
50,000 requests/sec
        |
        v
+-------------------+
|   Rate Limiter    |
|                   |
|  Allow: 1,000/s   |
|  Reject: 49,000   |
+-------------------+
        |
        v
Application
```

---

# Rate Limiting vs Throttling

These terms are often used interchangeably, but there is a useful distinction.

### Rate Limiting

Controls the maximum number of requests allowed.

```text
100 requests/minute
```

### Throttling

Controls the rate at which requests are processed.

For example:

```text
Incoming:
100 requests/sec

Processing:
20 requests/sec
```

The excess requests may be delayed, queued, or rejected depending on the implementation.

---

# Where Should Rate Limiting Happen?

Rate limiting can be implemented at different layers.

```text
                    Internet
                       |
                       v
                +-------------+
                | API Gateway |
                +-------------+
                       |
                Rate Limiting
                       |
                       v
              +----------------+
              | Load Balancer  |
              +----------------+
                       |
             +---------+---------+
             |         |         |
             v         v         v
          Service A Service B Service C
             |
             v
           Redis
```

For most distributed systems, **API Gateway / Edge Layer** is a good first line of defense.

But service-level rate limiting can also be useful for protecting individual downstream services.

---

# Common Rate Limiting Algorithms

There are several popular approaches:

1. Fixed Window
2. Sliding Window
3. Token Bucket
4. Leaky Bucket
5. Distributed Rate Limiting

Each has different trade-offs.

---

# 1. Fixed Window Counter

This is one of the simplest algorithms.

Suppose the rule is:

```text
100 requests / minute
```

We divide time into fixed windows:

```text
12:00:00 ───────── 12:00:59
12:01:00 ───────── 12:01:59
12:02:00 ───────── 12:02:59
```

Maintain a counter for each window.

```text
Window: 12:00

Requests = 75

Limit = 100

75 < 100
=> Allow
```

When the counter reaches 100:

```text
Requests = 100
Limit    = 100

=> Reject next request
```

At the beginning of the next minute:

```text
Counter = 0
```

and requests are accepted again.

### Example

```text
12:00:50
   |
   | 100 requests
   v
12:00:59
   |
   X
12:01:00
   |
   | counter resets
   v
100 more requests
```

This creates a potential burst problem.

A client could send:

```text
100 requests at 12:00:59
+
100 requests at 12:01:00
```

That's:

```text
200 requests in approximately 2 seconds
```

even though the configured limit is:

```text
100 requests/minute
```

### Advantages

- Very simple
- Easy to implement
- Low memory usage
- Fast

### Disadvantages

- Boundary burst problem
- Less accurate traffic control

---

# 2. Sliding Window

Sliding Window improves the boundary problem of Fixed Window.

Instead of looking at a fixed calendar window:

```text
12:00:00 ───────── 12:00:59
```

we continuously look backward from the current request.

For example:

```text
Limit = 100 requests
Window = previous 60 seconds
```

At:

```text
12:00:45
```

we examine requests from:

```text
11:59:45 → 12:00:45
```

At:

```text
12:00:50
```

we examine:

```text
11:59:50 → 12:00:50
```

The window moves with every request.

### Concept

```text
Time ------------------------------------------------>

        <------ 60 seconds ------>
                    |
                    v
              Current Request
                    |
                    v
        [ Requests in window ]

        1  2  3  4  5  6  7
```

If there are already 100 requests inside the window:

```text
100 requests
+
new request
=
101

=> Reject
```

### Advantages

- More accurate than Fixed Window
- Reduces boundary bursts
- Better traffic control

### Disadvantages

- More memory
- More computation
- Need to track request timestamps or aggregated buckets

---

# 3. Token Bucket

Token Bucket is one of the most popular algorithms for APIs.

Imagine a bucket containing tokens.

```text
             Token Bucket
        +-------------------+
        | 🟢 🟢 🟢 🟢 🟢 🟢 |
        | 🟢 🟢 🟢           |
        +-------------------+
                 |
                 v
              Request
```

Each API request consumes one token.

Suppose:

```text
Bucket capacity = 10 tokens
Refill rate     = 2 tokens/sec
```

If a request arrives:

```text
Token available?
      |
     Yes
      |
      v
Consume 1 token
      |
      v
Allow request
```

If there are no tokens:

```text
Request
   |
   v
No token
   |
   v
Reject / Wait
```

---

## Token Refill

Tokens are continuously added at a fixed rate.

Example:

```text
Capacity = 10
Refill   = 2 tokens/sec
```

Initially:

```text
10 tokens
```

Five requests arrive:

```text
10 → 9 → 8 → 7 → 6 → 5
```

After two seconds:

```text
5 + 4 = 9 tokens
```

The bucket cannot exceed its maximum capacity:

```text
maximum = 10
```

---

## Why Token Bucket Is Useful

One major advantage is that it allows **controlled bursts**.

Suppose the bucket has accumulated:

```text
10 tokens
```

The client can immediately send:

```text
10 requests
```

After that, it must follow the refill rate.

This is useful for APIs where short bursts are acceptable but sustained traffic must be controlled.

---

# 4. Leaky Bucket

The Leaky Bucket algorithm works differently.

Imagine requests entering a bucket and leaving at a fixed rate.

```text
Requests
   |
   v
+---------+
|         |
| Queue   |
|         |
+---------+
     |
     | fixed rate
     v
   API
```

Suppose:

```text
Processing rate = 10 requests/sec
```

Even if 100 requests arrive suddenly:

```text
100 requests
     |
     v
+-----------+
|   Queue   |
+-----------+
     |
     | 10/sec
     v
Application
```

Requests are processed at a controlled rate.

If the queue becomes full:

```text
Queue full
   |
   v
Reject new requests
```

### Advantages

- Smooths traffic
- Protects downstream services
- Controls processing rate

### Disadvantages

- Can introduce latency
- Queue memory is required
- Bursts may be delayed rather than immediately accepted

---

# Token Bucket vs Leaky Bucket

The key difference is:

```text
Token Bucket
-------------
Controls request permission

Allows bursts
Then limits sustained rate
```

```text
Leaky Bucket
-------------
Controls processing/output rate

Smooths traffic
Queues bursts
```

A simple comparison:

| Algorithm | Burst Support | Main Purpose |
|---|---|---|
| Fixed Window | High at boundaries | Simple request counting |
| Sliding Window | Low | Accurate request limiting |
| Token Bucket | Yes | Control rate + allow bursts |
| Leaky Bucket | Limited | Smooth traffic |

---

# 5. Distributed Rate Limiting

This becomes particularly important in microservices.

Imagine we have three API instances:

```text
                 API Gateway
                     |
          +----------+----------+
          |          |          |
          v          v          v
       Server A   Server B   Server C
```

Suppose the limit is:

```text
100 requests/minute/user
```

If each server maintains its own counter:

```text
Server A = 100
Server B = 100
Server C = 100
```

The user could potentially send:

```text
300 requests/minute
```

instead of 100.

The problem is:

**Where is the shared state?**

---

# Redis-Based Distributed Rate Limiting

A common solution is Redis.

```text
                    API Gateway
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
          Server A    Server B    Server C
             |           |           |
             +-----------+-----------+
                         |
                         v
                      Redis
```

All instances share the same rate-limit state.

For example:

```text
rate_limit:user:123
```

Redis can maintain:

```text
count = 73
window = 60 seconds
```

When another request arrives:

```text
Current count = 73

73 < 100

=> Allow
=> Increment to 74
```

When:

```text
count = 100
```

the next request is rejected.

---

# Why Redis?

Redis is commonly used because rate limiting requires:

- Very fast reads/writes
- Atomic operations
- Shared state
- Expiration support
- Low latency

For example:

```text
INCR key
EXPIRE key 60
```

can be used for simple fixed-window implementations.

For more advanced algorithms, Lua scripts or Redis-based atomic operations can ensure that checking and updating the limit happen together.

---

# The Atomicity Problem

Consider this sequence:

```text
Request A:
READ counter = 99

Request B:
READ counter = 99

Request A:
counter < 100 → allow

Request B:
counter < 100 → allow
```

Now:

```text
99 → 101
```

Two requests were allowed even though only one slot remained.

This is a **race condition**.

The solution is to make the rate-limit operation atomic.

```text
Check + Update
      |
      v
Atomic Operation
```

Redis Lua scripts are one common approach.

---

# Rate Limiting by Different Dimensions

Rate limits don't have to be based only on users.

You can limit by:

### User

```text
user:123 → 100 requests/min
```

### IP Address

```text
ip:192.168.x.x → 50 requests/min
```

### API Key

```text
api-key:abc → 10,000 requests/hour
```

### Endpoint

```text
POST /payments → 20 requests/min
```

### Tenant

For SaaS applications:

```text
tenant:companyA → 10,000 requests/min
tenant:companyB → 5,000 requests/min
```

You can even combine multiple limits.

```text
User Limit
    +
IP Limit
    +
Endpoint Limit
    +
Global System Limit
```

---

# Example: Payment API

Imagine:

```text
POST /payments
```

We don't want one customer repeatedly calling the payment API.

We could configure:

```text
10 requests/minute/user
100 requests/minute/IP
1,000 requests/minute/global
```

The request must pass all applicable limits:

```text
                Request
                   |
        +----------+----------+
        |          |          |
        v          v          v
     User       IP Limit   Global
     Limit                  Limit
        |          |          |
        +----------+----------+
                   |
             All allowed?
              /       \
            Yes        No
             |          |
             v          v
           API        HTTP 429
```

---

# Returning HTTP 429

When a client exceeds the limit, the API should normally return:

```http
HTTP/1.1 429 Too Many Requests
```

The response can also communicate when the client should retry.

For example:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```

This is much better than simply returning:

```text
Request failed
```

The client now knows:

```text
Wait 30 seconds
then retry
```

---

# Rate Limiting in the We Connect Architecture

For **We Connect**, rate limiting can be placed at the API Gateway.

A simplified architecture:

```text
                    Client
                      |
                      v
              +---------------+
              |  API Gateway  |
              +---------------+
                      |
               Rate Limiter
                      |
             +--------+--------+
             |        |        |
             v        v        v
          User     Post      Feed
         Service  Service   Service
             |        |        |
             +--------+--------+
                      |
                     Redis
```

The API Gateway can apply global protection before requests reach the microservices.

---

# Different Limits for Different APIs

Not every API should have the same limit.

For example:

| API | Suggested Strategy |
|---|---|
| Login | Strict rate limit |
| Search | Moderate rate limit |
| User Profile | Higher limit |
| Create Post | Moderate limit |
| Feed | Higher limit |
| Send Message | Moderate limit |
| Payment | Very strict limit |

For example:

```text
POST /login
5 requests/minute/IP
```

while:

```text
GET /feed
300 requests/minute/user
```

This is much more practical than applying one global limit to every endpoint.

---

# Rate Limiting and Caching

Rate limiting and caching work very well together.

Consider:

```text
Client
   |
   v
Rate Limiter
   |
   v
Cache
   |
   v
Service
```

Suppose thousands of users request:

```text
GET /trending-posts
```

Instead of hitting the database every time:

```text
Request
   |
Rate Limiter
   |
Redis Cache
   |
   +---- Cache Hit → Return
   |
   +---- Cache Miss → Service → DB
```

This reduces downstream load significantly.

---

# Rate Limiting and Circuit Breaker

Rate Limiting and Circuit Breaker solve different problems.

### Rate Limiting

Protects the system from **too much incoming traffic**.

```text
Too many requests
       |
       v
Rate Limiter
       |
       X
```

### Circuit Breaker

Protects the system from **an unhealthy downstream dependency**.

```text
Service A
    |
    v
Service B
    |
    X
Service B failing
    |
    v
Circuit Breaker
    |
    X
Stop sending requests
```

Together:

```text
Internet
   |
   v
Rate Limiter
   |
   v
Service A
   |
   v
Circuit Breaker
   |
   v
Service B
```

This gives us two layers of protection.

---

# What Happens When the Limit Is Exceeded?

There are several possible strategies.

### Reject

Return:

```text
429 Too Many Requests
```

Best when the request is not worth delaying.

### Queue

Place the request into a queue.

```text
Request
   |
   v
Queue
   |
   v
Worker
```

Useful when the operation can tolerate asynchronous processing.

### Delay

Wait and process later.

Useful for controlled workloads, but it increases latency.

The right choice depends on the API.

---

# Choosing the Right Algorithm

A practical decision guide:

```text
Need something very simple?
        |
        v
   Fixed Window
```

```text
Need accurate request counting?
        |
        v
  Sliding Window
```

```text
Need controlled bursts?
        |
        v
   Token Bucket
```

```text
Need smooth processing?
        |
        v
   Leaky Bucket
```

```text
Multiple API instances?
        |
        v
Distributed Rate Limiting
        |
        v
     Redis / Shared Store
```

---

# A Practical We Connect Design

For We Connect, a reasonable architecture could be:

```text
                         Internet
                            |
                            v
                    +---------------+
                    | API Gateway   |
                    +---------------+
                            |
                    +---------------+
                    | Rate Limiter  |
                    +---------------+
                            |
                         Redis
                            |
          +-----------------+----------------+
          |                 |                |
          v                 v                v
      User Service      Post Service    Feed Service
          |                 |                |
          +-----------------+----------------+
                            |
                         Database
```

Example configuration:

```text
Anonymous IP
    60 requests/minute

Authenticated User
    300 requests/minute

Expensive Search API
    30 requests/minute

Login API
    5 requests/minute

Global System
    100,000 requests/minute
```

These values are examples; production limits should be based on actual traffic, capacity, and business requirements.

---

# Important Design Considerations

Rate limiting sounds simple, but distributed systems introduce several challenges.

### 1. Clock Synchronization

Sliding-window and time-based algorithms depend on timestamps.

Distributed nodes should use reliable time handling.

### 2. Atomic Operations

Checking and updating the counter must happen atomically.

### 3. Redis Failure

What happens if Redis becomes unavailable?

Possible strategies:

```text
Fail Open
    |
    v
Allow requests
```

or:

```text
Fail Closed
    |
    v
Reject requests
```

The correct choice depends on the API.

For a critical payment API, the risk calculation may be very different from a public feed API.

### 4. Hot Keys

A very popular user or tenant can create a heavily accessed Redis key.

The design must consider high-cardinality and hot-key behavior.

### 5. Fairness

A global limit alone may allow one customer to consume most of the capacity.

Per-user or per-tenant limits provide better isolation.

---

# Rate Limiting Is More Than Blocking Requests

A good rate-limiting system should answer three questions:

```text
Who?
  |
  +-- User / IP / API Key / Tenant

How much?
  |
  +-- Requests per second/minute/hour

What happens after the limit?
  |
  +-- Reject / Queue / Delay
```

And in a distributed system:

```text
Where is the shared state?
```

That last question is especially important.

---

# Final Takeaway

Rate limiting is one of the fundamental building blocks for reliable APIs.

The basic idea is simple:

```text
Too much traffic
       |
       v
 Rate Limiter
       |
       +---- Allowed ----> Service
       |
       +---- Rejected ---> HTTP 429
```

But the implementation becomes more interesting at scale.

We need to choose between:

- **Fixed Window** for simplicity
- **Sliding Window** for more accurate limiting
- **Token Bucket** for controlled bursts
- **Leaky Bucket** for smooth traffic
- **Distributed Rate Limiting** when multiple servers share the workload

For a microservice architecture like **We Connect**, a common approach is:

```text
Client
  ↓
API Gateway
  ↓
Distributed Rate Limiter
  ↓
Redis
  ↓
Microservices
```

The key lesson is:

> **Rate limiting doesn't just protect your API. It protects everything behind your API.**

When designed correctly, it prevents traffic spikes, limits abuse, protects expensive resources, improves fairness between users, and helps keep the entire distributed system stable under pressure.