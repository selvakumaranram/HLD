# CAP Theorem: Choosing the Right Trade-Off in We Connect

When we start designing a distributed system, one question appears again and again:

> **What happens when our servers cannot communicate with each other?**

Imagine We Connect is running across multiple servers or regions.

```text
                    We Connect
                         |
              ┌──────────┴──────────┐
              ▼                     ▼
         Region A               Region B
              │                     │
          Database              Database
              │                     │
              └─────── Network ─────┘
                         ❌
```

What happens if the network connection between Region A and Region B fails?

Should both regions continue accepting requests?

Or should one region stop accepting writes until consistency can be guaranteed?

This is where the **CAP Theorem** becomes important.

---

# What Is CAP Theorem?

CAP stands for:

### C — Consistency

Every read receives the most recent successful write, or an error.

In simple terms:

> **All clients see the same latest data.**

Example:

```text
User updates profile:

Name = "Selva"

Region A → Selva
Region B → Selva
```

A client shouldn't read an older value after the write has been successfully acknowledged, assuming the system's consistency guarantee requires that behavior.

---

### A — Availability

Every request to a non-failing node receives a response, even if that response may not contain the latest data.

In simple terms:

> **The system continues responding.**

For example:

```text
Region A ❌
       |
       X
       |
Region B
       |
       ▼
   Request succeeds
```

The system remains available from the client's perspective.

---

### P — Partition Tolerance

The system continues operating despite a network partition between nodes.

For example:

```text
Region A                 Region B

Service A                Service B
Database A               Database B

     XXXXXXXXXXXXXXXXX
       Network Failure
     XXXXXXXXXXXXXXXXX
```

The system must tolerate this communication failure.

---

# The Important Part of CAP

The common explanation is:

> "You can choose only two of C, A, and P."

This is useful as an introduction, but it is incomplete.

The more precise interpretation is:

> **When a network partition occurs, a distributed system cannot simultaneously guarantee both strong consistency and availability.**

Why?

Because the nodes cannot communicate.

Suppose:

```text
Region A
Balance = ₹1,000

        X
    NETWORK
    PARTITION
        X

Region B
Balance = ₹1,000
```

A customer performs:

```text
Withdraw ₹800
```

in Region A.

Now another customer attempts:

```text
Withdraw ₹800
```

in Region B.

If both regions continue accepting writes without coordination:

```text
Region A → ₹200
Region B → ₹200
```

The system may have accepted withdrawals that should not both have been allowed.

To preserve strong consistency, one side may need to stop accepting certain operations until communication is restored.

That's the fundamental CAP trade-off.

---

# CAP During a Network Partition

The real decision looks like this:

```text
             Network Partition
                    |
          ┌─────────┴─────────┐
          ▼                   ▼
     Choose C              Choose A
          |                   |
          ▼                   ▼
   Reject/limit some      Continue serving
   operations             requests
          |                   |
          ▼                   ▼
   Stronger consistency   Higher availability
```

You cannot guarantee both **strong consistency and availability** during the partition.

---

# CP Systems

A **CP system** prioritizes:

- Consistency
- Partition tolerance

During a partition, it may sacrifice availability for some operations.

```text
Partition occurs
       |
       ▼
Can we guarantee consistency?
       |
   ┌───┴───┐
   │       │
  YES      NO
   │       │
   ▼       ▼
 Process   Reject/
 request   delay request
```

The system would rather return an error than return potentially conflicting or stale data.

### When is CP useful?

CP is appropriate when incorrect data is worse than temporarily rejecting a request.

Examples include:

- Financial transactions
- Inventory reservation
- Account ownership
- Certain authentication/security operations
- Distributed locks
- Systems where violating an invariant is unacceptable

---

# Banking Example

Consider a bank account:

```text
Account Balance = ₹10,000
```

A customer withdraws:

```text
₹8,000
```

At the same time, another transaction attempts:

```text
₹5,000
```

If two disconnected regions independently accept both operations, the system could temporarily or permanently violate the account's balance invariant.

For this type of operation, correctness is more important than continuing to accept every request.

A simplified design might therefore prefer:

```text
Consistency
      +
Partition Tolerance
      =
CP behavior
```

During a partition, some transactions may be rejected or delayed rather than risking inconsistent financial state.

**Important:** real banking systems are more nuanced than simply labeling the whole banking system "CP." Different subsystems can use different consistency models.

---

# What About Availability?

Suppose we are building a social media application.

A user creates a post:

```text
"Having a great day!"
```

The post is replicated to multiple regions.

A temporary network partition occurs.

Should we completely stop the user from posting?

Probably not.

For a social network, it can often be better to:

```text
Accept the post
      ↓
Store the event
      ↓
Replicate later
      ↓
Other regions catch up
```

This is where availability and eventual consistency become useful.

---

# Eventual Consistency

Eventual consistency means:

> If updates stop, replicas will eventually converge to the same value.

For example:

```text
             User creates post
                    |
                    ▼
                Region A
                    |
                    | asynchronous replication
                    |
                    ▼
                Region B
                    |
                    ▼
              Data catches up
```

For a short period:

```text
Region A → Post visible
Region B → Post not visible yet
```

Later:

```text
Region A → Post visible
Region B → Post visible
```

The system has converged.

---

# Is We Connect Eventually Consistent?

**Partially — yes.**

I would not describe the entire We Connect system as simply "eventually consistent."

Instead, we should choose the consistency model **per feature**.

That's a much more realistic system-design decision.

---

# We Connect: Where Can We Use Eventual Consistency?

Consider the We Connect home feed.

A user creates a post:

```text
User
  |
  ▼
Post Service
  |
  ▼
Post Database
  |
  ▼
Event / Message Queue
  |
  ├── Feed Service
  ├── Notification Service
  ├── Search Service
  └── Analytics Service
```

We don't necessarily need every component to update synchronously before responding to the user.

We can:

```text
Create Post
    ↓
Persist Post
    ↓
Return Success
    ↓
Publish Event
    ↓
Update Feed
    ↓
Send Notifications
    ↓
Update Analytics
```

This improves availability and reduces latency.

---

# Example: Likes

Suppose a post currently has:

```text
1,000 likes
```

A user clicks Like.

We don't necessarily need every replica, cache, feed view, analytics system, and counter to update at exactly the same moment.

For a short period, different users might see:

```text
User A → 1,001 likes
User B → 1,000 likes
```

After replication:

```text
Everyone → 1,001 likes
```

For a social platform, this is usually an acceptable trade-off.

---

# Example: Notifications

Imagine:

```text
Selva posts something
       |
       ▼
Post Service
       |
       ▼
Event
       |
       ▼
Notification Service
```

The notification might arrive a few seconds later.

We don't need to make the entire post-creation request wait for notification delivery.

Therefore:

**Eventual consistency + asynchronous processing**

is a natural fit.

---

# Example: Follower Count

Suppose a user has:

```text
10,000 followers
```

Five users follow them at nearly the same time.

Different replicas might temporarily show:

```text
10,003
10,004
10,005
```

Eventually they converge.

For most social-media experiences, this is acceptable.

We gain:

- Higher availability
- Better scalability
- Lower latency
- Less cross-region coordination

---

# Where We Connect Needs Stronger Consistency

Not everything should be eventually consistent.

Consider:

### User Identity

```text
User ID
Email
Account ownership
Authentication
```

We don't want different regions making contradictory decisions about who owns an account.

### Security

Permissions and security-sensitive state may require stronger guarantees.

### Critical Account Operations

If We Connect eventually adds:

```text
Payments
Subscriptions
Purchases
```

those operations need stronger transactional guarantees than a like counter.

So our architecture should be:

```text
                 We Connect
                     |
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   Stronger       Eventual       Stronger
 Consistency    Consistency    Consistency
       |             |             |
 Authentication    Likes        Payments
 Account data      Feed         Transactions
 Security          Views        Critical state
```

This is one of the most important lessons from CAP:

> **Don't choose one consistency model for the entire application. Choose it based on business requirements.**

---

# CAP Trade-Offs in We Connect

Let's make the decision explicit.

| We Connect Feature | Preferred Approach | Why |
|---|---|---|
| User authentication | Stronger consistency | Security and correctness |
| Account ownership | Stronger consistency | Must avoid conflicting state |
| Creating posts | Durable write + async propagation | Availability and low latency |
| Home feed | Eventual consistency | Small delays are acceptable |
| Likes | Eventual consistency | High write volume |
| Comments | Eventual consistency | Can tolerate propagation delay |
| Follower count | Eventual consistency | Approximate/temporary lag acceptable |
| Notifications | Asynchronous/eventual | Delivery can happen later |
| Recommendations | Eventual | Freshness matters more than immediate consistency |
| Analytics | Eventual | Processing can happen asynchronously |
| Payments | Strong consistency | Financial correctness |
| Inventory | Strong consistency | Avoid overselling |

This is how CAP should actually influence system design.

---

# CAP and Redis in We Connect

We already introduced Redis in the We Connect architecture.

Suppose:

```text
Post Database
      |
      ▼
    Redis
      |
      ▼
    Feed
```

Redis may contain a cached version of feed data.

The database might already contain:

```text
New Post
```

while Redis temporarily contains:

```text
Older Feed
```

That's a consistency trade-off.

We accept a short period of staleness because the benefit is:

```text
Lower latency
       +
Reduced database load
       +
Higher scalability
```

But for security-sensitive or correctness-critical data, we should not blindly rely on stale cache data.

---

# CAP and Service Discovery

We previously discussed Service Discovery.

Imagine:

```text
Service A
   |
   X
Service Registry
   X
Service B
```

If communication with the registry fails, we need to decide how the system behaves.

Should services continue using their last-known service information?

Or should they stop making calls because they cannot verify the current topology?

This is another example of distributed systems requiring explicit availability and consistency decisions.

---

# CAP and Circuit Breaker

Our previous **Circuit Breaker** discussion also connects here.

Suppose:

```text
Feed Service
      |
      ▼
Recommendation Service
      |
      X
Network Partition
```

A circuit breaker may stop sending requests to the unhealthy dependency.

Instead of waiting indefinitely:

```text
Feed
 ↓
Recommendation ❌
 ↓
Fallback response
```

We preserve the availability of the overall user experience.

For example:

> "Recommended posts are temporarily unavailable."

while the main feed continues working.

CAP doesn't directly define circuit breakers, but CAP trade-offs influence how we design failure behavior.

---

# CAP and Message Queues

Our future We Connect architecture will also use asynchronous messaging.

```text
Post Service
     |
     ▼
Message Queue
     |
     ├── Feed Service
     ├── Notification Service
     ├── Analytics Service
     └── Search Service
```

Suppose Notification Service is temporarily unavailable.

Should Post Service fail?

Probably not.

Instead:

```text
Post Created
     ↓
Event persisted
     ↓
Post request succeeds
     ↓
Notification processing waits
     ↓
Service recovers
     ↓
Message processed
```

This improves availability by decoupling the services.

The trade-off is that notifications are not instantaneous.

Again:

**Availability + eventual processing**

is often more valuable for this use case.

---

# CAP Is Not Just About Databases

This is an important interview point.

CAP is often introduced using databases, but the underlying problem is broader:

> **Distributed systems communicating over an unreliable network.**

It affects decisions around:

- Distributed databases
- Replication
- Microservices
- Service discovery
- Message queues
- Caches
- Multi-region systems
- Distributed coordination

Whenever independent nodes need to coordinate across a network, partition behavior matters.

---

# CAP Theorem Limitation #1: It Focuses on Partitions

CAP becomes relevant specifically when a partition occurs.

During normal operation, a system can potentially provide both:

```text
Strong consistency
        +
High availability
```

The trade-off becomes unavoidable when the network partition prevents coordination.

Therefore, saying:

> "Our database is CP, so it is always unavailable during failures"

would be an oversimplification.

---

# CAP Theorem Limitation #2: CAP Does Not Tell You Everything

CAP doesn't directly answer:

- How fast should the system respond?
- How much stale data is acceptable?
- How many replicas should we have?
- How should retries work?
- What happens to queued messages?
- How should conflicts be resolved?
- What is the recovery strategy?

These require additional architectural decisions.

---

# CAP Theorem Limitation #3: Real Systems Have More Than Two Choices

Modern distributed databases often provide **tunable or multiple consistency models** rather than forcing one global choice.

For example, Amazon DynamoDB supports both eventually consistent and strongly consistent reads for supported operations, while global tables also offer different multi-Region consistency modes.

MongoDB similarly exposes read and write concerns that let applications choose stronger or weaker guarantees depending on the operation.

So the real-world question becomes:

> **What consistency guarantee does this particular operation require?**

---

# CAP Limitation #4: PACELC

There is another useful concept called **PACELC**.

CAP talks about what happens:

```text
IF Partition occurs
```

PACELC extends the discussion:

```text
IF Partition:
    choose Availability or Consistency

ELSE:
    choose Latency or Consistency
```

In other words, even when there is **no partition**, distributed systems still make trade-offs between consistency and latency.

This is highly relevant for global applications.

For example:

```text
User in India
       |
       ▼
Asia Region
       |
       ▼
Local replica
```

is usually faster than:

```text
User in India
       |
       ▼
US Region
       |
       ▼
Strongly coordinated write
       |
       ▼
Other regions
```

Stronger cross-region coordination can increase latency.

---

# How Other Large-Scale Systems Make These Trade-Offs

There isn't one "correct" CAP choice for every company.

The workload determines the decision.

---

## Amazon DynamoDB

DynamoDB supports both eventually consistent and strongly consistent reads for supported data paths. Eventual consistency is the default for reads, while strong reads provide the latest committed value; global tables also support multi-Region eventual and multi-Region strong consistency modes.

**Lesson:** consistency can be selected according to the operation and deployment requirement rather than treating the entire application as one consistency model.

---

## Google Cloud Spanner

Spanner is designed for workloads requiring very strong transactional guarantees across distributed infrastructure. Its default serializable transactions provide external consistency, and strong reads see committed data up to the start of the read.

**Lesson:** when correctness and globally consistent transactions are critical, an architecture can deliberately pay the coordination cost to obtain stronger guarantees.

---

## Azure Cosmos DB

Cosmos DB exposes multiple consistency levels, from strong to eventual, with intermediate choices such as bounded staleness, session, and consistent prefix. This allows applications to make a more granular trade-off among consistency, latency, and throughput.

**Lesson:** real-world distributed databases often provide a spectrum of consistency guarantees rather than a simple binary choice.

---

## MongoDB

MongoDB allows applications to control read consistency and isolation through read concerns, while write concerns control how writes are acknowledged. Stronger guarantees can come with additional latency or availability implications.

**Lesson:** consistency decisions can be made at the operation level instead of treating every read and write identically.

---

# A Simple Comparison

Think about four different systems:

```text
Social Media
     ↓
Availability matters heavily
     ↓
Eventual consistency is acceptable
```

```text
Banking
     ↓
Incorrect balance is unacceptable
     ↓
Strong consistency for critical transactions
```

```text
Global Database
     ↓
Users are distributed worldwide
     ↓
Trade-off between consistency and latency
```

```text
Analytics
     ↓
Data can arrive later
     ↓
Eventual consistency is usually acceptable
```

The architecture follows the **business requirement**, not the other way around.

---

# Our We Connect CAP Decision

For We Connect, our high-level decision is:

```text
                    We Connect
                         |
          ┌──────────────┴──────────────┐
          ▼                             ▼
  Correctness-critical            User experience
       operations                   operations
          |                             |
          ▼                             ▼
 Stronger consistency            Eventual consistency
          |                             |
 Authentication                    Feed
 Account ownership                 Likes
 Security                          Comments
 Critical state                    Notifications
                                   Analytics
                                   Recommendations
```

And our architecture can look like:

```text
                         Client
                            |
                            ▼
                     API Gateway
                            |
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
       User              Post              Feed
      Service            Service           Service
          |                 |                 |
          ▼                 ▼                 ▼
     Stronger            Durable           Redis /
     consistency           write          Feed Store
                              |
                              ▼
                       Message Queue
                              |
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
       Notification      Analytics       Recommendation
          Service          Service            Service
             |
             ▼
       Eventual Processing
```

This gives us a practical balance:

**Strong where correctness matters.**

**Eventual where availability, scale, and latency matter more.**

---

# The Most Important Lesson

CAP is not a checklist where we simply write:

```text
We Connect = AP
```

That would be too simplistic.

Instead, ask for every important piece of data:

> **What happens if the network is partitioned?**

Then ask:

> **Can we accept stale data?**

If yes:

```text
Prefer availability
        +
Eventual consistency
```

If no:

```text
Prefer consistency
        +
Reject/delay conflicting operations
```

That is how CAP becomes a real system-design tool rather than just an interview definition.

---

# Interview-Friendly Summary

If someone asks:

**"What is CAP Theorem?"**

A strong answer is:

> CAP states that during a network partition, a distributed system cannot simultaneously guarantee strong consistency and availability. Partition tolerance is generally required in distributed systems, so the practical design decision is what behavior we prefer during a partition. Systems that prioritize correctness may choose consistency and reject or delay some requests, while systems that prioritize availability may continue serving requests and reconcile data later.

Then give the We Connect example:

> In We Connect, we would not choose one consistency model for the entire platform. Authentication and critical account state require stronger guarantees, while feeds, likes, notifications, recommendations, and analytics can generally tolerate eventual consistency. This lets us achieve better availability and scalability without sacrificing correctness where it matters.

---

# Final Mental Model

Remember CAP like this:

```text
              NETWORK PARTITION
                     |
          ┌──────────┴──────────┐
          ▼                     ▼
    Protect Data            Keep Serving
          |                     |
          ▼                     ▼
    Consistency              Availability
          |                     |
          ▼                     ▼
    CP behavior            AP-style behavior
```

And for We Connect:

```text
        "What does this feature need?"
                    |
          ┌─────────┴─────────┐
          ▼                   ▼
   Correctness first    Availability first
          |                   |
          ▼                   ▼
 Stronger consistency   Eventual consistency
          |                   |
   Auth / security      Feed / likes /
   critical state       notifications /
                        recommendations
```

**The best distributed system isn't the one that chooses consistency everywhere or availability everywhere.**

**It is the one that makes the right trade-off for each business requirement.**

That is the real value of CAP Theorem in **We Connect HLD**.