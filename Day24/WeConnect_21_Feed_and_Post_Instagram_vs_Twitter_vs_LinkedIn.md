# WeConnect #21 — Designing the Social Feed: Instagram vs Twitter vs LinkedIn

> **Core Social Platform — Feed & Post**

Every social platform looks different on the surface.

Instagram is visual.  
Twitter/X is conversation and real-time information.  
LinkedIn is professional content and networking.

But underneath, all three need to solve the same fundamental problem:

> **A user opens the application. How do we decide which posts to show them, in what order, and at what scale?**

This article compares the feed and post architecture of Instagram, Twitter/X, and LinkedIn, then uses those lessons to design the **WeConnect Feed & Post system**.

---

## 1. The Problem We Are Solving

Imagine a user follows 2,000 people.

Every day, those people can create thousands of posts.

We cannot simply execute:

```text
SELECT * FROM posts
WHERE author_id IN (all_following_users)
ORDER BY created_at DESC;
```

That approach becomes expensive very quickly.

We need to answer:

- How are posts created?
- Where are posts stored?
- How does a feed know which posts belong to a user?
- Should posts be pushed into feeds when created?
- Should feeds be generated when the user opens the application?
- How do we rank posts?
- How do we handle viral users?
- How do we paginate?
- How do we keep the feed fast?
- How do likes, comments and shares affect ranking?

This is the core social-feed problem.

---

# 2. Instagram vs Twitter/X vs LinkedIn

At a high level:

| Platform | Content | Feed Characteristics | Important Signals |
|---|---|---|---|
| Instagram | Photos, videos, captions | Highly personalized and visual | Interest, engagement, relationships |
| Twitter/X | Short posts, media, conversations | Fast-moving and conversation-oriented | Recency, relevance, engagement |
| LinkedIn | Professional posts, articles, media | Network + professional relevance | Relationships, professional context, engagement |

The important observation is:

> **The user experience is different, but the underlying problem is remarkably similar.**

All three need:

```text
Users
  ↓
Relationships
  ↓
Posts
  ↓
Candidate Posts
  ↓
Ranking
  ↓
Personalized Feed
```

---

# 3. Instagram — A Visual Feed

Instagram's feed is heavily optimized around visual content and personal relevance.

A simplified flow looks like:

```text
User creates post
       ↓
Post Service
       ↓
Media Storage
       ↓
Event
       ↓
Feed / Ranking System
       ↓
Followers' candidate feeds
       ↓
User opens Instagram
       ↓
Personalized Feed
```

A post can contain:

- Image
- Video
- Caption
- Hashtags
- Location
- Author information
- Engagement information

The important architectural challenge is that media can be much larger than the post metadata.

Therefore we should separate:

```text
Post Metadata
      +
Media Object
```

For example:

```text
Post DB
---------
post_id
author_id
caption
created_at
visibility
media_id

Object Storage
--------------
media_id
original file
processed files
thumbnails
```

The database does not need to store the actual image or video bytes.

---

# 4. Twitter/X — A High-Velocity Feed

Twitter/X has a different content pattern.

A post can be:

```text
Text
Text + Image
Text + Video
Link
Reply
Quote
Repost
```

The feed also has a stronger real-time characteristic.

Imagine a major event happening.

Thousands of posts may be created within minutes.

The architecture therefore needs to handle:

```text
Very high write velocity
        +
Very high read volume
        +
Real-time ranking
```

A simplified flow:

```text
User
 ↓
Tweet/Post API
 ↓
Post Service
 ↓
Event Bus
 ↓
Feed / Ranking
 ↓
Timeline
```

Twitter-like systems also introduce an important problem:

### The viral user problem

Suppose a user has 50 million followers.

If we synchronously insert the new post into 50 million user feeds:

```text
1 Post
 ↓
50,000,000 Feed Updates
```

That is clearly expensive.

This is one reason social-feed systems often use **hybrid feed generation** rather than blindly pushing every post to every follower.

---

# 5. LinkedIn — A Professional Feed

LinkedIn has another dimension:

> **Professional relevance.**

A user's feed can contain posts from:

- Connections
- People they follow
- Companies
- Industry professionals
- Communities
- Recommended content

The ranking problem therefore becomes more contextual.

For example:

```text
Post A
Author: Close connection
Topic: Software Architecture

Post B
Author: Unknown person
Topic: Random entertainment

Post C
Author: Industry expert
Topic: Distributed Systems
```

A professional network may consider:

```text
Relationship
+
Professional relevance
+
Engagement
+
Content quality
+
Recency
```

This shows us an important design principle:

> **The feed should not be tightly coupled to one ranking algorithm.**

The ranking system should be replaceable.

---

# 6. What Is Common Across All Three?

Despite the differences, the architecture can be generalized.

```text
                    Social Platform
                          │
             ┌────────────┴────────────┐
             │                         │
          Users                       Posts
             │                         │
       Social Graph               Post Service
             │                         │
             └────────────┬────────────┘
                          ↓
                    Feed Service
                          ↓
                   Candidate Posts
                          ↓
                  Ranking Service
                          ↓
                  Personalized Feed
```

This is exactly where **WeConnect** comes in.

We don't want to build:

```text
Instagram Feed
Twitter Feed
LinkedIn Feed
```

as three completely separate systems.

Instead, we want:

```text
                 WeConnect Feed
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Social        Content      Ranking
       Graph          Store       Engine
          │            │            │
          └────────────┼────────────┘
                       ↓
                Personalized Feed
```

---

# 7. Designing WeConnect Posts

Let's start with the simplest part: creating a post.

A user sends:

```http
POST /posts
```

Example:

```json
{
  "text": "Designing a scalable social feed",
  "media": [
    "media-123"
  ],
  "visibility": "PUBLIC"
}
```

The API should not perform every operation synchronously.

A better architecture is:

```text
Client
  ↓
API Gateway
  ↓
Post Service
  ↓
Post Database
  ↓
Event Bus
```

The response can be returned after the post is safely persisted.

Then asynchronous consumers process the event.

```text
PostCreated
     │
     ├──→ Feed Service
     ├──→ Notification Service
     ├──→ Search Indexer
     ├──→ Analytics
     └──→ Moderation
```

This gives us loose coupling.

---

# 8. Post Service

The Post Service owns post lifecycle.

Responsibilities:

- Create post
- Update post
- Delete post
- Get post
- Validate visibility
- Store metadata
- Publish post events

A simplified model:

```text
Post
----
post_id
author_id
content
media_ids
visibility
created_at
updated_at
status
```

We should avoid putting unrelated responsibilities into the Post Service.

For example:

```text
Post Service
      ❌ Send push notification
      ❌ Calculate feed ranking
      ❌ Resize images
      ❌ Build search index
```

Instead:

```text
PostCreated Event
       │
       ├──→ Notification Service
       ├──→ Feed Service
       ├──→ Media Service
       └──→ Search Service
```

---

# 9. Feed Generation — The Big Design Decision

There are two classic approaches.

## Approach 1 — Fan-out on Read

When the user opens the feed:

```text
User opens app
      ↓
Find following users
      ↓
Fetch their latest posts
      ↓
Merge
      ↓
Rank
      ↓
Return feed
```

Advantages:

- Simple writes
- No need to update millions of feeds
- Good for users with small networks

Disadvantages:

- Expensive reads
- Difficult to guarantee low latency
- Ranking can become expensive

---

# 10. Fan-out on Write

When a user creates a post:

```text
Post Created
     ↓
Find Followers
     ↓
Push Post ID
into follower feeds
```

For example:

```text
Alice posts
     ↓
Feed Service
     ↓
Bob's Feed
Carol's Feed
David's Feed
Eve's Feed
...
```

Advantages:

- Fast reads
- Feed is already prepared
- Good for read-heavy systems

Disadvantages:

- Expensive writes
- Large fan-out
- Viral users become problematic

---

# 11. The Hybrid Approach

For WeConnect, a hybrid model is a strong starting point.

```text
Normal User
     ↓
Fan-out on Write
     ↓
Follower Feed Cache
```

But for a very large account:

```text
Celebrity / High-Follower User
     ↓
Do NOT fan out to everyone
     ↓
Fetch dynamically during read
```

Then:

```text
                 Feed Request
                      │
            ┌─────────┴─────────┐
            ↓                   ↓
      Precomputed Feed      Dynamic Posts
            │                   │
            └─────────┬─────────┘
                      ↓
                 Merge
                      ↓
                   Rank
                      ↓
                  Response
```

This avoids the worst-case behavior of pure fan-out-on-write.

---

# 12. Candidate Generation

A feed should not rank every post in the entire system.

First we create a candidate set.

Possible sources:

```text
Following
Followers
Connections
Groups
Companies
Topics
Recommended creators
Trending content
```

Example:

```text
                Candidate Generator
                       │
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
    Following       Trending        Recommended
       │               │                │
       └───────────────┼────────────────┘
                       ↓
                 Candidate Set
```

Suppose we have:

```text
10,000,000 posts
```

We might only select:

```text
500 candidate posts
```

for a particular user.

Then ranking becomes much more manageable.

---

# 13. Ranking

Now comes the intelligence.

A simplified ranking score might be:

```text
Score =
    Relationship Score
  + Content Relevance
  + Engagement Score
  + Recency Score
  + Quality Score
```

For example:

```text
Post A
Relationship     0.9
Relevance        0.8
Engagement       0.7
Recency          0.9
                 ----
Final Score      0.83
```

The exact algorithm is product-specific.

And that is the important architectural point:

> **Feed generation should not own ranking logic.**

Instead:

```text
Feed Service
     ↓
Candidate Posts
     ↓
Ranking Service
     ↓
Ranked Posts
```

This allows WeConnect to evolve its ranking algorithm independently.

---

# 14. Feed Storage

We need a fast way to retrieve a user's feed.

We don't necessarily need to store complete post objects in the feed.

Instead, we can store post IDs.

Example:

```text
Feed: user-123

post-901
post-782
post-651
post-432
post-201
```

Then:

```text
Feed Store
    ↓
Post IDs
    ↓
Post Cache
    ↓
Post Database
```

This avoids duplicating large post objects.

Redis or another low-latency store can be useful for hot feed data.

---

# 15. Pagination

A social feed can never be returned all at once.

We need pagination.

Avoid relying only on:

```text
OFFSET 100000
```

For large datasets, cursor-based pagination is generally more scalable.

Example:

```http
GET /feed?cursor=eyJ0cyI6MT...
```

Conceptually:

```text
First Request
     ↓
Posts 100 → 81
     ↓
Cursor
     ↓
Posts 80 → 61
     ↓
Cursor
     ↓
Posts 60 → 41
```

This works better for continuously changing feeds.

---

# 16. What Happens When a Post Is Created?

Let's put everything together.

```text
                    User
                      │
                      ↓
                API Gateway
                      │
                      ↓
                 Post Service
                      │
               ┌──────┴──────┐
               ↓             ↓
           Post DB       Media Service
               │
               ↓
          PostCreated Event
               │
       ┌───────┼────────┬──────────┐
       ↓       ↓        ↓          ↓
     Feed   Search   Notify    Analytics
   Service  Indexer  Service
       │
       ↓
  Feed Storage
```

The user gets a response without waiting for every downstream system.

---

# 17. What Happens When a User Opens the Feed?

```text
User
 ↓
GET /feed
 ↓
Feed Service
 ↓
Retrieve candidate post IDs
 ↓
Fetch post metadata
 ↓
Apply visibility rules
 ↓
Ranking Service
 ↓
Personalization
 ↓
Pagination
 ↓
Response
```

A more complete version:

```text
                     Feed Request
                          │
                          ↓
                  Candidate Generator
                          │
          ┌───────────────┼────────────────┐
          ↓               ↓                ↓
       Following       Trending       Recommended
          │               │                │
          └───────────────┼────────────────┘
                          ↓
                  Candidate Posts
                          ↓
                  Filtering Layer
                          ↓
                   Ranking Service
                          ↓
                  Personalized Feed
                          ↓
                      Response
```

---

# 18. Instagram vs Twitter/X vs LinkedIn — What Should We Learn?

The comparison gives us three important lessons.

## Instagram teaches us

**Media-first social content**

WeConnect should support:

- Images
- Videos
- Captions
- Visual ranking
- Media processing

---

## Twitter/X teaches us

**High-velocity content and real-time conversations**

WeConnect should support:

- Fast publishing
- Real-time feed updates
- Replies
- Reposts/shares
- Trending content

---

## LinkedIn teaches us

**Context and professional relevance**

WeConnect should support:

- Different relationship types
- Professional context
- Network relevance
- Content recommendations

---

# 19. WeConnect — Unified Architecture

Putting the lessons together:

```text
                         WeConnect
                              │
                        API Gateway
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ↓               ↓                ↓
         User Service     Post Service    Relationship
                                              Service
              │               │                │
              └───────────────┼────────────────┘
                              ↓
                         Event Bus
                              │
          ┌───────────────────┼────────────────────┐
          ↓                   ↓                    ↓
     Feed Service       Notification          Search
                              │
                              ↓
                       Ranking Service
                              │
                              ↓
                       Feed Storage
                              │
                              ↓
                            Redis
                              │
                              ↓
                         Post Cache
                              │
                              ↓
                         Post Store
```

This is the foundation of our social platform.

---

# 20. The Most Important Design Decision

WeConnect should **not hard-code Instagram, Twitter/X, or LinkedIn behavior into the feed service**.

Instead, create configurable components:

```text
Feed Service
     │
     ├── Candidate Strategy
     │
     ├── Ranking Strategy
     │
     ├── Relationship Strategy
     │
     └── Content Strategy
```

This gives us flexibility.

For example:

```text
Instagram-like feed
→ Visual relevance + engagement

Twitter-like feed
→ Recency + conversation + relevance

LinkedIn-like feed
→ Professional relevance + network
```

The infrastructure stays largely the same.

The **business rules and ranking strategy change**.

---

# 21. Key Trade-offs

| Decision | Option A | Option B |
|---|---|---|
| Feed generation | Fan-out on read | Fan-out on write |
| Best for | Lower write volume | Read-heavy systems |
| Feed strategy | Chronological | Ranked |
| Pagination | Offset | Cursor |
| Feed storage | Database | Cache + database |
| Ranking | Synchronous | Dedicated service |
| Post processing | Synchronous | Event-driven |
| Viral users | Fan-out | Hybrid |
| Media | Database | Object storage |

There is no single architecture that is perfect for every platform.

The right design depends on:

- Read/write ratio
- Number of followers
- Content velocity
- Ranking complexity
- Latency requirements
- Consistency requirements
- Cost

---

# 22. Failure Scenarios

A production system must also assume that things fail.

### Ranking Service is down

The feed should still work.

Fallback:

```text
Personalized Ranking
       ↓
     Failed
       ↓
Chronological Ranking
```

### Redis is unavailable

Fall back to the persistent feed store.

### Event processing is delayed

The post should still exist.

The feed may temporarily lag behind.

This is an example of **eventual consistency**.

### Duplicate events

Use idempotent processing.

```text
PostCreated(post-123)
PostCreated(post-123)
```

should not result in two copies of the same post in a user's feed.

---

# 23. What WeConnect Is Really Building

At first glance, we are building:

> **A social media feed.**

But architecturally, we are actually building several systems:

```text
              Social Graph
                   +
               Content
                   +
            Event Streaming
                   +
             Candidate Gen
                   +
               Ranking
                   +
              Personalization
                   +
              Feed Storage
                   +
                 Cache
```

The feed is simply the place where all of these systems meet.

---

# 24. Final Architecture

The complete high-level design:

```text
                         CLIENT
                            │
                            ↓
                       API GATEWAY
                            │
             ┌──────────────┼───────────────┐
             │              │               │
             ↓              ↓               ↓
          Users           Posts        Relationships
             │              │               │
             └──────────────┼───────────────┘
                            ↓
                       EVENT BUS
                            │
       ┌────────────────────┼────────────────────┐
       ↓                    ↓                    ↓
   Feed Service       Notification Service   Search Service
       │
       ↓
Candidate Generation
       │
       ├── Following
       ├── Connections
       ├── Trending
       └── Recommendations
       │
       ↓
 Filtering / Policy
       │
       ↓
 Ranking Service
       │
       ↓
 Personalized Feed
       │
       ↓
   Feed Cache
       │
       ↓
   Post Cache
       │
       ↓
   Post Store
```

---

# 25. Final Takeaway

Instagram, Twitter/X, and LinkedIn may look like completely different products.

But from a system-design perspective, they repeatedly solve the same fundamental problems:

> **Who should see this post?**

> **When should they see it?**

> **Where should we store it?**

> **How do we rank it?**

> **How do we deliver it quickly?**

> **How do we handle millions of users and billions of interactions?**

That is the real lesson for **WeConnect**.

We don't need three separate feed architectures.

We need **one extensible feed platform** with configurable:

```text
Candidate Generation
        +
Ranking
        +
Relationship Model
        +
Content Model
        +
Personalization
```

And that gives us the foundation for everything that comes next:

**Likes → Comments → Notifications → Search → Recommendations → Reels → Stories → Chat.**

---

## WeConnect Series

**Core Social Platform**

**21. Feed & Post — Instagram vs Twitter vs LinkedIn** ← *This article*

**22. Likes & Comments — Designing Social Engagement at Scale**

**23. Follow & Connections — Instagram vs Twitter vs LinkedIn**

**24. Notifications — Designing Event-Driven Social Notifications**

**25. Search & Explore — Designing Social Search**

**26. Feed Ranking — How Do We Decide What Users See?**

Then we move into the media platform:

**27. Media Storage**

**28. Image Processing**

**29. Reels / Short Video**

**30. Video Streaming & CDN**

> **WeConnect is not an Instagram clone, Twitter clone, or LinkedIn clone. It is a system-design journey where we combine the best architectural lessons from all of them.**
