# Event-Driven Architecture with Kafka: Building Systems That React in Real Time

## We Connect HLD Series

In our previous articles, we explored how databases scale using replication, caching with Redis, and other building blocks of modern distributed systems.

Now imagine a different challenge.

A user posts a new update on **We Connect**.

What happens next?

* Followers should receive the post in their feed.
* Notifications should be sent.
* Search indexes should be updated.
* Analytics should record engagement.
* Recommendation systems should learn from the activity.
* Moderation services should scan the content.

Should the Post Service perform all of these operations?

Absolutely not.

This is where **Event-Driven Architecture (EDA)** becomes one of the most powerful patterns in modern system design.

---

# The Problem

A simple implementation might look like this:

```
User Creates Post
        │
        ▼
Post Service
   │
   ├── Update Feed
   ├── Send Notification
   ├── Update Search
   ├── Save Analytics
   ├── Run Moderation
   └── Update Recommendations
```

Initially this seems manageable.

But over time problems appear:

* Every new feature requires modifying the Post Service.
* A slow notification service delays post creation.
* If Analytics fails, the entire request may fail.
* Tight coupling makes deployments risky.
* Scaling becomes increasingly difficult.

One service becomes responsible for everything.

---

# A Better Approach

Instead of telling every service what to do...

The Post Service simply announces:

> "A new post has been created."

Anyone interested can react.

This is exactly how Event-Driven Architecture works.

---

# Enter Kafka

Apache Kafka acts as the communication backbone.

Instead of direct service-to-service communication:

```
Post Service
      │
      ▼
 Publish Event
      │
      ▼
+----------------+
|     Kafka      |
+----------------+
      │
      ├────────► Feed Service
      ├────────► Notification Service
      ├────────► Search Service
      ├────────► Analytics Service
      ├────────► Moderation Service
      └────────► Recommendation Service
```

The Post Service doesn't know who consumes the event.

It only publishes it.

This simple change dramatically improves scalability.

---

# What Is an Event?

An event describes **something that has already happened**.

Example:

```json
{
  "eventType": "PostCreated",
  "postId": "P12345",
  "userId": "U890",
  "timestamp": "2026-08-07T10:15:30Z"
}
```

Notice something important.

It doesn't say:

> "Update Feed"

Instead it says:

> "A post was created."

Consumers decide what to do.

---

# Life of a Post in We Connect

Imagine Alice publishes a post.

### Step 1

Post Service stores the post in PostgreSQL.

### Step 2

Post Service publishes:

```
PostCreated Event
```

to Kafka.

### Step 3

Multiple services consume independently.

Feed Service

* Generates followers' feeds

Notification Service

* Sends push notifications

Search Service

* Updates Elasticsearch

Analytics Service

* Records activity

Moderation Service

* Checks inappropriate content

Recommendation Service

* Learns user interests

All happen independently.

No service blocks another.

---

# Why Kafka?

Kafka provides several powerful capabilities.

## High Throughput

Millions of events can be processed every second.

Perfect for social media platforms.

---

## Durable Storage

Events are stored on disk.

Even if a consumer crashes...

The event is still available.

---

## Horizontal Scaling

Need more processing?

Simply add more consumers.

Kafka distributes the workload.

---

## Fault Tolerance

If Notification Service is down...

Feed Service continues working.

Nothing stops.

Once Notification Service recovers...

It resumes consuming missed events.

---

# Kafka Topics

Think of a Topic as a category.

```
Topics

posts

notifications

likes

comments

messages
```

Each topic stores related events.

Consumers subscribe only to topics they need.

---

# Consumer Groups

Suppose Feed Service receives too many events.

Instead of one consumer:

```
Feed Consumer
```

We create three.

```
Feed Consumer 1

Feed Consumer 2

Feed Consumer 3
```

Kafka automatically divides partitions among them.

More consumers.

More throughput.

Same topic.

---

# Partitions

A topic is divided into partitions.

```
posts

Partition 0

Partition 1

Partition 2

Partition 3
```

Partitions allow events to be processed in parallel.

This is how Kafka scales.

---

# Event Ordering

Ordering matters.

Suppose a user edits a post immediately after publishing it.

Wrong order:

```
PostUpdated

PostCreated
```

This creates inconsistent data.

Kafka guarantees ordering **within a partition**.

Using the same key (such as `postId`) ensures related events stay in order.

---

# What Happens If a Consumer Fails?

Imagine Analytics Service crashes.

Events continue accumulating in Kafka.

When Analytics comes back online...

It resumes reading from the last processed offset.

No events are lost.

No need to ask producers to resend data.

---

# Event Replay

This is one of Kafka's biggest advantages.

Need to rebuild Search?

Simply replay historical events.

Need to train a new ML model?

Replay events.

Need to migrate analytics?

Replay events.

Historical events become a valuable source of truth.

---

# Benefits of Event-Driven Architecture

✅ Loose coupling between services

✅ Independent deployments

✅ Better scalability

✅ Improved fault tolerance

✅ Easier feature additions

✅ Faster development

Adding a new service often requires **zero changes** to existing producers.

Just subscribe to the topic.

---

# Challenges

Event-driven systems aren't free of trade-offs.

You'll need to think about:

* Eventual consistency
* Duplicate event handling
* Idempotent consumers
* Schema evolution
* Monitoring event flows
* Dead Letter Queues (DLQ)
* Event versioning

Understanding these concepts is essential for production-grade systems.

---

# Real-World Examples

Many large-scale companies rely heavily on Kafka and Event-Driven Architecture.

* LinkedIn (Kafka was originally created there)
* Netflix
* Uber
* Airbnb
* Amazon
* Spotify
* Walmart

They process billions of events every day to power feeds, recommendations, payments, notifications, and analytics.

---

# Final Thoughts

As systems grow, synchronous communication becomes a bottleneck.

Event-Driven Architecture allows services to evolve independently while remaining responsive and scalable.

Instead of tightly coupling every service, publish meaningful events and let interested consumers react.

For a platform like **We Connect**, this approach enables new features—feeds, notifications, analytics, moderation, and recommendations—to grow independently without slowing down the core post creation flow.

Modern distributed systems aren't built around direct calls.

They're built around events.

---

### Coming Next

**API Gateway: One Entry Point for Hundreds of Microservices**

We'll explore how an API Gateway simplifies client communication, handles authentication, rate limiting, routing, and protects large-scale microservice architectures.
