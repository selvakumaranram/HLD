# WeConnect #22 — Likes & Comments: Designing Social Engagement at Scale

> **Core Social Platform — Engagement**

In our previous WeConnect article, we designed the **Feed & Post system** by comparing Instagram, Twitter/X, and LinkedIn.

But creating a post is only the beginning.

The real activity starts after the post is published.

Users:

- Like
- Comment
- Reply
- Share
- Repost
- React
- Remove their reaction
- Interact with other users

A post with millions of views can generate millions of interactions.

So the next question is:

> **How do we design a Like & Comment system that can handle massive traffic without slowing down the feed?**

---

# 1. The Problem

Imagine a popular post receives:

```text
10 million views
2 million likes
500,000 comments
100,000 shares
```

A naive implementation might do this:

```text
User clicks Like
       ↓
UPDATE Posts
SET like_count = like_count + 1
WHERE post_id = ?
```

At small scale, this looks fine.

At massive scale, it becomes a problem.

Thousands or millions of users could be updating the same row simultaneously.

Now we have:

- Database contention
- Hot rows
- High write traffic
- Locking
- Increased latency
- Counter inconsistencies
- Duplicate requests

The architecture needs to separate **user actions** from **aggregated counters and downstream processing**.

---

# 2. Instagram vs Twitter/X vs LinkedIn

The interaction model differs slightly across platforms.

| Platform | Primary Engagement | Conversation Model | Important Signals |
|---|---|---|---|
| Instagram | Likes, comments, shares | Comments and replies | Engagement, relationships |
| Twitter/X | Likes, replies, reposts, quotes | Conversation-heavy | Recency, engagement, conversation |
| LinkedIn | Reactions, comments, reposts | Professional discussion | Relevance, relationships, engagement |
| WeConnect | Like, reaction, comment, reply, share | Configurable | Engagement, relevance, relationships |

The important architectural observation is:

> **The user experience differs, but the underlying engagement infrastructure can be shared.**

---

# 3. What Happens When a User Likes a Post?

Let's start with a simple example.

```text
Alice
  ↓
Likes Post-123
```

We need to answer:

1. Has Alice already liked the post?
2. Should the like be stored?
3. Should the counter increase?
4. Should the post author's notification be created?
5. Should the engagement score change?
6. Should analytics record the event?
7. Should the feed ranking change?

These are different responsibilities.

We should not put all of them into the API request.

---

# 4. Naive Architecture

A simple implementation might look like:

```text
Client
   ↓
Like API
   ↓
Database
   ↓
Update Like
   ↓
Update Counter
   ↓
Create Notification
   ↓
Update Analytics
```

This creates a long synchronous request.

If Notification Service is slow:

```text
Like
 ↓
Database
 ↓
Notification Service
 ↓
Timeout
```

The user may think the like failed even though the database update succeeded.

This is not ideal.

---

# 5. Event-Driven Engagement

A better architecture is:

```text
                    Like Request
                         │
                         ↓
                  Engagement API
                         │
                         ↓
                  Engagement Store
                         │
                         ↓
                    LikeCreated
                         │
                         ↓
                     Event Bus
                         │
        ┌────────────────┼─────────────────┐
        ↓                ↓                 ↓
 Notification       Analytics        Ranking
   Service             Service         Service
```

The request handles the critical operation.

Other work happens asynchronously.

This gives us:

- Lower API latency
- Better scalability
- Loose coupling
- Independent consumers
- Easier retries

---

# 6. Designing the Like Model

A simple Like table could look like:

```text
Like
----
like_id
post_id
user_id
created_at
```

The important constraint is:

```text
UNIQUE(post_id, user_id)
```

This prevents the same user from creating multiple likes for the same post.

For example:

```text
Alice → Like → Post-123
Alice → Like → Post-123
```

The second request should not create another like.

This is where **idempotency** becomes important.

---

# 7. Idempotency

Mobile applications can retry requests.

For example:

```text
POST /posts/123/like
```

The client sends the request.

The network fails.

The client retries.

Now the server receives:

```text
Like Request #1
Like Request #2
```

Without protection:

```text
like_count = 2
```

even though Alice only liked once.

With an idempotent operation:

```text
Like #1 → Created
Like #2 → Already exists
```

The final state remains:

```text
Alice likes Post-123
```

This is one of the most important properties of a production engagement system.

---

# 8. Like vs Unlike

A like system has two primary operations:

```text
LIKE
UNLIKE
```

Conceptually:

```text
Like
 ↓
Create relationship

Unlike
 ↓
Remove relationship
```

We can model it as:

```text
User ──LIKES──> Post
```

When the user unlikes:

```text
User ──X──> Post
```

The system should also generate events:

```text
LikeCreated
LikeRemoved
```

Downstream services can react accordingly.

---

# 9. Comments Are Different

A Like is relatively simple:

```text
User → Post
```

A Comment creates content.

```text
User
  ↓
Comment
  ↓
Post
```

A comment might contain:

```text
comment_id
post_id
author_id
parent_comment_id
content
created_at
status
```

The `parent_comment_id` allows us to support replies.

Example:

```text
Post
 ├── Comment A
 │    ├── Reply A1
 │    └── Reply A2
 │
 ├── Comment B
 │    └── Reply B1
 │
 └── Comment C
```

This creates a comment tree.

---

# 10. Comments Should Be Their Own Domain

We should avoid putting all comments inside the Post Service.

Instead:

```text
Post Service
      │
      │ Post
      ↓
Comment Service
      │
      ├── Create Comment
      ├── Delete Comment
      ├── Reply
      ├── Moderate
      └── Retrieve Comments
```

This allows the Comment Service to evolve independently.

For example, later we may introduce:

- Comment ranking
- Spam detection
- Toxicity detection
- Mention detection
- Comment search

---

# 11. Comment Creation Flow

A simplified architecture:

```text
User
 ↓
API Gateway
 ↓
Comment Service
 ↓
Comment DB
 ↓
CommentCreated Event
 ↓
Event Bus
```

Consumers:

```text
                    CommentCreated
                          │
            ┌─────────────┼─────────────┐
            ↓             ↓             ↓
       Notification     Analytics    Moderation
          Service         Service       Service
```

This keeps the core comment operation independent of downstream processing.

---

# 12. The Counter Problem

Now we reach one of the interesting distributed-system problems.

Suppose a post has:

```text
Likes: 9,999,999
```

Thousands of users like it at the same time.

Should we update one database row for every like?

```text
UPDATE post
SET like_count = like_count + 1
```

Potentially millions of times?

This creates a **hot counter**.

A single popular post can become a hotspot.

---

# 13. Don't Treat Counters as the Source of Truth

We can separate:

```text
Interaction Data
       +
Aggregated Counter
```

The interaction record answers:

> **Who liked this post?**

The counter answers:

> **Approximately/how many likes does this post have?**

These are different concerns.

For example:

```text
Like Store
-----------
Post-123 → Alice
Post-123 → Bob
Post-123 → Charlie
```

and:

```text
Counter Store
-------------
Post-123 → 3 likes
```

The counter can be updated asynchronously.

---

# 14. Counter Aggregation

Instead of updating the primary database for every interaction:

```text
Like
Like
Like
Like
Like
 ↓
Event Stream
 ↓
Counter Aggregator
 ↓
Batch / Atomic Increment
 ↓
Counter Store
```

This reduces database pressure.

A possible flow:

```text
             Like Events
                  │
                  ↓
              Kafka
                  │
                  ↓
         Counter Aggregator
                  │
            ┌─────┴─────┐
            ↓           ↓
          Redis       Database
```

Redis can provide very fast counter operations.

The persistent store can periodically receive durable aggregates.

---

# 15. Eventual Consistency

This means the displayed counter may temporarily be slightly behind reality.

For example:

```text
Actual likes:       1,000,245
Displayed likes:    1,000,241
```

A moment later:

```text
Displayed likes:    1,000,245
```

For social engagement counters, this is often an acceptable trade-off.

The key question is:

> **Does the business require the counter to be perfectly accurate at every millisecond?**

Usually, no.

---

# 16. What About "Liked by You"?

There is another important distinction.

The user needs to know:

```text
❤️ You liked this post
```

This is not the same as:

```text
1,000,245 likes
```

We can retrieve the user's interaction separately:

```text
GET /posts/123/engagement

{
  "likedByMe": true,
  "likeCount": 1000245
}
```

The two pieces of information can come from different stores.

---

# 17. Engagement Service

This suggests a common service:

```text
                 Engagement Service
                         │
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
       Like          Reaction          Share
        │                │                │
        └────────────────┼────────────────┘
                         ↓
                     Event Bus
```

Comments can remain a separate content service because they have substantially different storage and lifecycle requirements.

A broader architecture becomes:

```text
                    Social Content
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
      Post Service   Engagement      Comment
                       Service        Service
```

---

# 18. Engagement Events

Every important action can become an event.

```text
LikeCreated
LikeRemoved
CommentCreated
CommentDeleted
ReplyCreated
ShareCreated
ReactionCreated
```

These events can feed multiple systems.

```text
                    Engagement Event
                           │
       ┌───────────────────┼───────────────────┐
       ↓                   ↓                   ↓
 Notifications         Analytics           Ranking
       │                   │                   │
       ↓                   ↓                   ↓
   User alerts        Metrics/Trends      Feed scoring
```

This is where our previous **event-driven architecture and Kafka discussions** become directly useful.

---

# 19. Likes Affect the Feed

Remember our previous Feed article?

We had:

```text
Candidate Generation
        ↓
Ranking
        ↓
Personalized Feed
```

Engagement becomes one of the ranking signals.

For example:

```text
Post A
Likes       50
Comments    10

Post B
Likes       5000
Comments    800
```

The ranking system may determine that Post B has stronger engagement.

But we should be careful.

If ranking only uses engagement:

```text
More likes = higher ranking
```

then popular content becomes increasingly popular.

This can create a feedback loop.

A real ranking system should combine multiple signals:

```text
Relationship
+
Relevance
+
Engagement
+
Freshness
+
Quality
+
Personalization
```

---

# 20. Notifications

A like can trigger:

```text
Alice likes Bob's post
       ↓
LikeCreated
       ↓
Notification Service
       ↓
Bob receives:
"Alice liked your post"
```

But imagine:

```text
10,000 users like the same post
```

Sending 10,000 individual notifications could become noisy.

A notification system might aggregate:

```text
Alice, Bob and 2,341 others liked your post.
```

This is another reason notifications should be decoupled from the Like API.

---

# 21. Comments and Notifications

Comments can produce multiple notification scenarios.

```text
Alice comments on Bob's post
       ↓
Bob gets notification
```

If Charlie replies to Alice's comment:

```text
Charlie
   ↓
Reply to Alice
   ↓
Alice gets notification
```

If Alice mentions David:

```text
"Amazing architecture @David"
              ↓
        MentionDetected
              ↓
       David notification
```

This suggests another asynchronous pipeline:

```text
CommentCreated
      ↓
Mention Detection
      ↓
Notification
```

---

# 22. Handling Viral Posts

Now let's consider a post receiving:

```text
10 million likes
```

We should not allow one database row to become a bottleneck.

A scalable strategy could be:

```text
                 Like Requests
                       │
                       ↓
                  Load Balancer
                       │
                ┌──────┼──────┐
                ↓      ↓      ↓
              API-1  API-2  API-3
                │      │      │
                └──────┼──────┘
                       ↓
                    Events
                       │
                       ↓
                  Partitioned
                    Stream
                       │
             ┌─────────┼─────────┐
             ↓         ↓         ↓
          Counter   Analytics  Ranking
```

The stream partitions allow the system to process large volumes horizontally.

---

# 23. Database Partitioning

At large scale, engagement records may become enormous.

Instead of one giant table:

```text
Likes
```

we can partition data.

Possible partition strategies:

### By post

```text
Partition(post_id)
```

Good when we frequently ask:

> Who liked this post?

### By user

```text
Partition(user_id)
```

Good when we frequently ask:

> Which posts has this user liked?

The correct strategy depends on the dominant access pattern.

In some systems, we may maintain different read models for different queries.

This is where **CQRS** can become useful.

---

# 24. CQRS for Engagement

We can separate:

```text
Write Model
     ↓
Interaction Events
     ↓
Read Models
```

For example:

```text
              Like Request
                   ↓
             Write Store
                   ↓
              LikeCreated
                   ↓
              Event Stream
             /      |       \
            /       |        \
           ↓        ↓         ↓
      User Likes  Post Count  Analytics
       Read Model Read Model
```

The write model represents the actual interaction.

Read models are optimized for different queries.

---

# 25. Caching

Popular posts are read frequently.

Caching can help:

```text
Post
 ↓
Redis
 ↓
Like Count
Comment Count
Reaction Summary
```

For example:

```text
post:123:engagement

{
  "likes": 1250345,
  "comments": 8432
}
```

But cache should not automatically become the permanent source of truth.

A common approach is:

```text
Persistent Store
      +
Cache
```

with appropriate consistency rules.

---

# 26. Race Conditions

Consider:

```text
User clicks Like
User clicks Unlike
```

very quickly.

Requests may arrive in this order:

```text
Like
Unlike
```

or:

```text
Unlike
Like
```

The system needs deterministic state handling.

We can use:

- Idempotency keys
- Version numbers
- Timestamps
- Ordered event processing
- Conditional writes

The exact choice depends on the storage technology and consistency requirements.

---

# 27. Moderation

Comments introduce another production concern:

> **Not every comment should immediately become visible.**

A possible flow:

```text
Comment
   ↓
Comment Service
   ↓
Moderation
   ↓
┌───────────────┐
│               │
Safe          Suspicious
│               │
↓               ↓
Publish      Review / Block
```

Moderation can include:

- Spam detection
- Abuse detection
- Toxicity detection
- Duplicate content detection
- Policy checks

This should not necessarily block every comment request synchronously.

Asynchronous moderation can improve scalability.

---

# 28. Complete WeConnect Engagement Architecture

Now we can combine everything.

```text
                         CLIENT
                            │
                            ↓
                       API GATEWAY
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
         Engagement       Comment        Post
          Service         Service       Service
              │             │             │
              ↓             ↓             ↓
         Interaction      Comment       Post DB
            Store           DB
              │             │
              └──────┬──────┘
                     ↓
                  Event Bus
                     │
       ┌─────────────┼───────────────────┐
       ↓             ↓                   ↓
 Notification    Analytics            Ranking
   Service        Pipeline             Service
       │             │                   │
       ↓             ↓                   ↓
    Push/In-App   Metrics          Feed Signals
       │
       ↓
      User
```

Counters can have their own path:

```text
Interaction Events
       ↓
Counter Aggregator
       ↓
Redis / Counter Store
       ↓
Feed API
```

---

# 29. Request Flow — Like

Let's walk through the complete flow.

```text
1. User clicks Like

2. Client
   ↓
   POST /posts/123/like

3. API Gateway
   ↓

4. Engagement Service
   ↓

5. Check idempotency / existing relationship
   ↓

6. Store Like
   ↓

7. Publish LikeCreated
   ↓

8. Return success to user
```

Then asynchronously:

```text
LikeCreated
    │
    ├──→ Counter Aggregator
    │
    ├──→ Notification Service
    │
    ├──→ Ranking Service
    │
    ├──→ Analytics
    │
    └──→ Recommendation System
```

The user does not need to wait for all six operations.

---

# 30. Request Flow — Comment

```text
User
 ↓
POST /posts/123/comments
 ↓
API Gateway
 ↓
Comment Service
 ↓
Validate
 ↓
Persist Comment
 ↓
CommentCreated
 ↓
Event Bus
```

Then:

```text
CommentCreated
     │
     ├──→ Notification
     ├──→ Moderation
     ├──→ Counter
     ├──→ Analytics
     └──→ Ranking
```

Again:

> **One action, many downstream consumers.**

This is the power of event-driven architecture.

---

# 31. Important Design Decisions

| Problem | Possible Solution |
|---|---|
| Duplicate likes | Unique constraint + idempotency |
| High like volume | Event-driven processing |
| Hot counters | Counter aggregation |
| Fast reads | Redis/cache |
| Large interaction data | Partitioning |
| Different read patterns | CQRS/read models |
| Notifications | Async events |
| Feed ranking | Engagement events |
| Comment moderation | Async moderation pipeline |
| Viral posts | Horizontal scaling + partitioned events |
| Feed consistency | Eventual consistency where acceptable |

---

# 32. What WeConnect Learns from Instagram, Twitter/X and LinkedIn

### Instagram

Teaches us:

> **Visual engagement can become extremely high-volume.**

Likes, comments, shares and media interactions must scale independently from the post itself.

### Twitter/X

Teaches us:

> **Conversation creates enormous event volume.**

Replies, reposts, quotes and trending conversations can generate rapid bursts of activity.

### LinkedIn

Teaches us:

> **Engagement is also a relevance signal.**

A reaction or comment is not just a counter.

It can help determine what content is valuable to a professional network.

### WeConnect

We combine these ideas:

```text
High-volume engagement
        +
Conversation
        +
Relevance
        +
Event-driven processing
        ↓
      WeConnect
```

---

# 33. The Bigger Lesson

A Like button looks simple.

```text
❤️ Like
```

But at scale, it becomes:

```text
User Interaction
       ↓
Idempotency
       ↓
Persistent State
       ↓
Event
       ↓
Counter
       ↓
Notification
       ↓
Analytics
       ↓
Ranking
       ↓
Recommendations
```

That is the difference between:

> **Building a feature**

and

> **Designing a scalable system.**

---

# 34. Final WeConnect Architecture

```text
                         USER
                          │
                          ↓
                     API GATEWAY
                          │
        ┌─────────────────┼──────────────────┐
        ↓                 ↓                  ↓
      POSTS          ENGAGEMENT          COMMENTS
        │                 │                  │
        │            Like/React/Share        │
        │                 │                  │
        └─────────────────┼──────────────────┘
                          ↓
                      EVENT BUS
                          │
       ┌──────────────────┼─────────────────────┐
       ↓                  ↓                     ↓
 Notification         Counter               Analytics
   Service            Aggregator              Service
       │                  │                     │
       ↓                  ↓                     ↓
 Push / In-App         Redis / DB           Data Platform
                          │
                          ↓
                    Ranking Service
                          │
                          ↓
                     Feed Service
                          │
                          ↓
                    Personalized Feed
```

---

# 35. Final Takeaway

The biggest lesson from designing social engagement is simple:

> **Don't make the Like button responsible for everything that happens after a Like.**

Keep the critical path small:

```text
Validate
   ↓
Persist
   ↓
Publish Event
   ↓
Respond
```

Then let specialized services process the event:

```text
Notification
Counter
Analytics
Ranking
Recommendation
Moderation
```

This gives WeConnect:

- Scalability
- Loose coupling
- Resilience
- Independent service evolution
- Lower request latency
- Better handling of viral content

And most importantly:

> **A single user action can become a powerful event that drives the entire social platform.**

---

## WeConnect Series

### Core Social Platform

**21. Feed & Post — Instagram vs Twitter vs LinkedIn**

**22. Likes & Comments — Designing Social Engagement at Scale** ← *This article*

**23. Follow & Connections — Instagram vs Twitter vs LinkedIn**

**24. Notifications — Designing Event-Driven Social Notifications**

**25. Search & Explore — Designing Social Search**

**26. Feed Ranking — How Do We Decide What Users See?**

### Media Platform

**27. Media Storage**

**28. Image Processing**

**29. Reels / Short Video**

**30. Video Streaming & CDN**

---

> **WeConnect — Learn from the best. Build for what's next.**