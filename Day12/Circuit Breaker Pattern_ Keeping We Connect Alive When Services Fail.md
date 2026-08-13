# Circuit Breaker Pattern: Keeping We Connect Alive When Services Fail

## We Connect HLD Series

In our previous articles, we explored how **Kafka** helps services communicate asynchronously and how an **API Gateway** provides a single entry point to our microservices.

But there is another important question in a distributed system:

**What happens when one service fails?**

Imagine a normal day on **We Connect**.

Thousands of users are browsing feeds, creating posts, liking content, sending messages, and receiving notifications.

Everything is working perfectly.

Then suddenly...

**Recommendation Service goes down.**

Should the entire We Connect application become slow or unavailable?

**Absolutely not.**

This is where the **Circuit Breaker Pattern** becomes important.

---

# The Problem: Failure Can Spread

Let's imagine the following architecture:

```text
                 User
                  │
                  ▼
             API Gateway
                  │
                  ▼
             Feed Service
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
   Post Service  Search  Recommendation
```

The Feed Service needs data from the Recommendation Service.

Normally:

```text
Feed Service
     │
     │ Request
     ▼
Recommendation Service
     │
     ▼
   Response
```

Everything works.

But what happens when Recommendation Service becomes unavailable?

```text
Feed Service
     │
     │ Request
     ▼
Recommendation Service
     X
   FAILED
```

The Feed Service waits.

Then it retries.

The request fails again.

It retries again.

Soon hundreds or thousands of requests are waiting.

---

# The Real Problem Is Not One Failure

The biggest danger isn't that **Recommendation Service is down**.

The bigger problem is that its failure can spread to other services.

Imagine this:

```text
Recommendation Service
          ❌
          │
          ▼
     Feed Service
          │
          ▼
     API Gateway
          │
          ▼
         Users
```

A small failure can become a **system-wide failure**.

This is called a:

## Cascading Failure

And this is exactly what we want to prevent.

---

# What Is a Circuit Breaker?

Think about the electrical circuit breaker in your house.

If there is an electrical problem...

The breaker trips.

Why?

To prevent the problem from damaging the rest of the system.

The same idea applies to microservices.

A Circuit Breaker monitors calls to a downstream service.

If too many requests fail...

**It opens the circuit.**

Instead of continuing to call the unhealthy service, requests fail fast or receive a fallback response.

---

# How It Works

A Circuit Breaker typically has three states:

```text
             Failures exceed threshold
                    │
                    ▼
             ┌─────────────┐
             │    CLOSED   │
             │ Normal flow │
             └──────┬──────┘
                    │
                    ▼
             ┌─────────────┐
             │     OPEN    │
             │ Fail fast   │
             └──────┬──────┘
                    │
              After timeout
                    │
                    ▼
             ┌─────────────┐
             │ HALF-OPEN   │
             │ Test request│
             └──────┬──────┘
                    │
              ┌─────┴─────┐
              ▼           ▼
           Success       Failure
              │           │
              ▼           ▼
           CLOSED        OPEN
```

Let's understand each state.

---

# 1. CLOSED

Everything is working normally.

Requests flow through the circuit.

```text
Feed Service
     │
     ▼
Circuit Breaker
     │
     ▼
Recommendation Service
     │
     ▼
   Success
```

The Circuit Breaker monitors:

- Success rate
- Failure rate
- Response time
- Timeouts

---

# 2. OPEN

Now imagine Recommendation Service starts failing.

After reaching a configured threshold:

```text
Circuit Breaker
      │
      ▼
     OPEN
```

The Circuit Breaker stops sending requests to Recommendation Service.

Instead:

```text
Feed Service
     │
     ▼
Circuit Breaker
     │
     X
     │
     ▼
Fallback Response
```

The request fails immediately instead of waiting for a timeout.

This is called:

## Fail Fast

---

# Why Fail Fast?

Imagine 10,000 users requesting their feed.

Without a Circuit Breaker:

```text
10,000 requests
       │
       ▼
Recommendation Service
       │
       X
       │
Long timeouts
       │
       ▼
Threads consumed
       │
       ▼
Feed Service becomes slow
```

Eventually, Feed Service itself may run out of resources.

With a Circuit Breaker:

```text
10,000 requests
       │
       ▼
Circuit Breaker
       │
       X
       │
       ▼
Fallback / Default Response
```

The unhealthy service is isolated.

The rest of the system continues working.

---

# 3. HALF-OPEN

But we don't want the circuit to remain open forever.

After some time, the Circuit Breaker allows a small number of test requests.

This is the **HALF-OPEN** state.

```text
Circuit OPEN
     │
     │ Wait
     ▼
 HALF-OPEN
     │
     ▼
Test Request
```

If the Recommendation Service has recovered:

```text
Test Request
     │
     ▼
 Success
     │
     ▼
 CLOSED
```

Normal traffic resumes.

If it is still failing:

```text
Test Request
     │
     ▼
 Failure
     │
     ▼
 OPEN
```

The system continues protecting itself.

---

# Let's Apply This to We Connect

Imagine Alice opens her feed.

The Feed Service normally requests recommendations.

```text
Alice
 │
 ▼
API Gateway
 │
 ▼
Feed Service
 │
 ▼
Circuit Breaker
 │
 ▼
Recommendation Service
```

Everything works.

Alice receives:

```text
Recommended for you

• Follow John
• Follow Sarah
• Follow Tech Community
```

Now Recommendation Service crashes.

The Circuit Breaker detects repeated failures.

```text
Feed Service
     │
     ▼
Circuit Breaker
     │
     X
Recommendation Service
```

Instead of failing the entire feed...

We provide a fallback.

```text
Feed Service
     │
     ▼
Fallback
     │
     ▼
Show chronological posts
```

Alice can still use We Connect.

She might not get personalized recommendations...

But the **core application continues working**.

That's the real value of resilience.

---

# Graceful Degradation

This leads to another important concept:

## Graceful Degradation

Not every feature has the same importance.

For We Connect:

### Critical

- Login
- Creating posts
- Reading posts
- Messaging

### Less Critical

- Recommendations
- Trending content
- Analytics
- Personalized suggestions

If Recommendations fail...

We shouldn't take down the entire platform.

Instead:

```text
Recommendation unavailable
           ↓
Use fallback
           ↓
Show normal feed
           ↓
User continues using We Connect
```

This is graceful degradation.

---

# Circuit Breaker + Retry

A common mistake is to think:

> "If the service fails, let's retry."

Retries can help with temporary failures.

But unlimited retries can make things worse.

Imagine:

```text
Request
  ↓
Failure
  ↓
Retry
  ↓
Failure
  ↓
Retry
  ↓
Failure
  ↓
Retry
```

Now thousands of clients are retrying simultaneously.

The already-unhealthy service receives even more traffic.

This can make the outage worse.

That's why production systems often combine:

**Timeout + Retry + Circuit Breaker + Backoff**

For example:

```text
Request
   │
   ▼
Timeout
   │
   ▼
Retry with Exponential Backoff
   │
   ▼
Repeated failures
   │
   ▼
Circuit Opens
   │
   ▼
Fail Fast
```

---

# Circuit Breaker + Kafka

Our previous article introduced Kafka.

The two patterns solve different problems.

### API Gateway

Handles:

**Client → Backend**

### Circuit Breaker

Handles:

**Service → Service**

### Kafka

Handles:

**Asynchronous Service → Service communication**

For example:

```text
             API Gateway
                  │
                  ▼
             Post Service
                  │
                  ▼
                Kafka
          ┌───────┼────────┐
          ▼       ▼        ▼
        Feed   Search   Analytics
```

If Feed Service synchronously calls Recommendation Service, a Circuit Breaker can protect that dependency.

Meanwhile, analytics and other asynchronous processing can continue through Kafka.

---

# Where Should the Circuit Breaker Live?

Usually, it sits around the call to the downstream dependency.

```text
Feed Service
     │
     ▼
┌───────────────┐
│Circuit Breaker│
└───────┬───────┘
        │
        ▼
Recommendation
Service
```

The important idea is:

**Protect the caller from an unhealthy dependency.**

---

# What Should We Configure?

A production Circuit Breaker needs sensible thresholds.

For example:

```text
Failure Threshold
        ↓
50%

Timeout
        ↓
2 seconds

Open State Duration
        ↓
30 seconds

Half-Open Test Requests
        ↓
5
```

These are only examples.

Real values should be determined using:

- Traffic patterns
- Service latency
- Error rates
- Business requirements
- Capacity testing

---

# Circuit Breaker Isn't a Magic Solution

There are trade-offs.

Poor configuration can cause:

- Too many false positives
- Unnecessary fallback responses
- Requests being rejected too aggressively
- Difficult debugging

We also need good observability.

We should monitor:

- Circuit state
- Failure rate
- Timeout rate
- Latency
- Number of rejected requests
- Fallback usage

---

# Popular Implementations

In a Java/Spring Boot ecosystem, one popular option is:

**Resilience4j**

It provides several resilience patterns:

- Circuit Breaker
- Retry
- Rate Limiter
- Bulkhead
- Time Limiter

These patterns can be combined to build more resilient microservices.

---

# The Bigger Lesson

Distributed systems will fail.

Networks fail.

Servers crash.

Databases become slow.

Third-party APIs become unavailable.

The goal isn't to build a system where **nothing ever fails**.

That's unrealistic.

The goal is to build a system where:

> **One failure doesn't bring down everything else.**

That's what the Circuit Breaker Pattern gives us.

---

# We Connect Architecture

Our architecture is now becoming more resilient:

```text
                    Users
                      │
                      ▼
                API Gateway
                      │
                      ▼
                 Feed Service
                      │
              ┌───────┴────────┐
              ▼                ▼
       Circuit Breaker       Kafka
              │                │
              ▼          ┌─────┼─────┐
     Recommendation      ▼     ▼     ▼
         Service        Feed Search Analytics
              │
              X
        Service Failure
              │
              ▼
          Fail Fast
              │
              ▼
        Fallback Feed
```

The system doesn't pretend failures won't happen.

It **expects them and contains them**.

---

# Final Thoughts

As We Connect grows from a simple application into a distributed platform, reliability becomes just as important as scalability.

Kafka helps us handle asynchronous communication.

API Gateway gives us a controlled entry point.

And the Circuit Breaker protects our services from cascading failures.

Together, these patterns help us build systems that don't just scale...

**They survive failures.**

---

### Coming Next

**Service Discovery: How Microservices Find Each Other**

When We Connect has hundreds of service instances running across multiple servers or containers, how does one service know where another service is?

We'll explore **Service Discovery, Service Registry, Load Balancing, and how Kubernetes changes the way microservices discover each other.**