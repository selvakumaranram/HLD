# Distributed Cache: Designing a Cache That Works at Scale

Modern applications often depend on databases for almost every
operation. As traffic grows, repeatedly reading the same data from a
database becomes expensive.

A distributed cache helps us keep frequently accessed data in memory and
serve it with very low latency.

For **We Connect**, where users may repeatedly request profiles, feeds,
posts, recommendations, and session-related data, a distributed cache
can significantly reduce database load and improve response times.

------------------------------------------------------------------------

## What Is a Distributed Cache?

A distributed cache is a shared caching layer that runs across multiple
application instances.

Instead of:

``` text
Client
  |
  v
API
  |
  v
Database
```

we introduce a cache:

``` text
Client
  |
  v
API
  |
  v
Cache
  |
  +---- Cache Hit ----> Return data
  |
  +---- Cache Miss ---> Database
                         |
                         v
                       Cache
```

The cache stores frequently accessed data closer to the application.

A popular technology for this is **Redis**.

------------------------------------------------------------------------

# Why Do We Need a Distributed Cache?

Consider a social networking application.

Suppose:

``` text
GET /users/123
```

is requested 10,000 times per second.

Without caching:

``` text
10,000 requests/sec
        |
        v
     Database
```

The database must process all 10,000 requests.

With caching:

``` text
10,000 requests/sec
        |
        v
      Redis
        |
        +---- 9,900 Cache Hits
        |
        +---- 100 Cache Misses
                     |
                     v
                  Database
```

The database now handles only a fraction of the traffic.

This improves:

-   Response latency
-   Database performance
-   Application scalability
-   Infrastructure efficiency
-   Availability during traffic spikes

------------------------------------------------------------------------

# Cache vs Distributed Cache

A local cache lives inside one application instance.

``` text
              Load Balancer
             /             \
            v               v
       Server A          Server B
       Local Cache       Local Cache
```

The problem is that Server A and Server B have different cache contents.

A distributed cache provides shared state:

``` text
              Load Balancer
             /             \
            v               v
       Server A          Server B
            \               /
             \             /
              v           v
                Redis
```

Now all application instances can access the same cached data.

------------------------------------------------------------------------

# Why Redis?

Redis is commonly used as a distributed cache because it provides:

-   In-memory data access
-   Very low latency
-   High throughput
-   TTL support
-   Atomic operations
-   Rich data structures
-   Replication
-   High availability
-   Redis Cluster for horizontal scaling

A typical architecture is:

``` text
                 Clients
                    |
                    v
              API Gateway
                    |
                    v
              Load Balancer
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Service A Service B Service C
          |         |         |
          +---------+---------+
                    |
                    v
                 Redis
                    |
                    v
                Database
```

------------------------------------------------------------------------

# Cache-Aside Pattern

**Cache-aside** is one of the most commonly used caching patterns.

The application is responsible for reading from and writing to the
cache.

The flow is:

``` text
Application
    |
    v
Check Cache
    |
    +---- Hit ----> Return
    |
    +---- Miss
          |
          v
       Database
          |
          v
      Update Cache
          |
          v
       Return
```

------------------------------------------------------------------------

## Cache-Aside Read Flow

Suppose we need:

``` text
GET /users/123
```

Step 1:

``` text
GET user:123
```

Check Redis.

If the key exists:

``` text
Cache Hit
    |
    v
Return cached user
```

If it does not exist:

``` text
Cache Miss
    |
    v
Query Database
    |
    v
Store result in Redis
    |
    v
Return result
```

Example:

``` text
Application
     |
     | GET user:123
     v
   Redis
     |
     | MISS
     v
 Database
     |
     | User data
     v
   Redis
     |
     v
Application
```

------------------------------------------------------------------------

# Why Cache-Aside Is Popular

The biggest advantage is control.

The application decides:

-   What should be cached
-   When to cache it
-   How long to cache it
-   What should be invalidated

It also avoids requiring every database write to go through the cache.

------------------------------------------------------------------------

# Cache-Aside Write Flow

Suppose a user changes their name.

``` text
UPDATE /users/123
```

The application updates the database first:

``` text
Application
    |
    v
Database
    |
    v
Update successful
    |
    v
Invalidate Cache
```

Then:

``` text
DEL user:123
```

The next read becomes a cache miss and loads the latest data from the
database.

------------------------------------------------------------------------

# Write-Through Cache

In a **write-through** architecture, the application writes to the
cache, and the cache is responsible for persisting the data to the
database.

Conceptually:

``` text
Application
    |
    v
Cache
    |
    v
Database
```

When the application writes:

``` text
UPDATE user:123
```

the cache is updated and the data is also written to the persistent
store.

The advantage is that the cache remains warm with recently written data.

------------------------------------------------------------------------

# Cache-Aside vs Write-Through

  -----------------------------------------------------------------------
  Feature                 Cache-Aside             Write-Through
  ----------------------- ----------------------- -----------------------
  Read miss               Application loads DB    Cache handles read path
                                                  depending on
                                                  implementation

  Write responsibility    Application             Cache layer

  Cache freshness         Application controlled  More automatic

  Complexity              Lower                   Higher

  Common use              Read-heavy workloads    Workloads requiring
                                                  cache consistency

  Cache warming           Usually on read         Usually on write
  -----------------------------------------------------------------------

For many microservices, **cache-aside is a practical default** because
it gives the application explicit control over caching behavior.

------------------------------------------------------------------------

# Cache Invalidation

One of the hardest problems in caching is invalidation.

Suppose the database contains:

``` text
user:123
name = Selva
```

Redis contains:

``` text
user:123
name = Selva
```

The user changes the name:

``` text
Database:
name = Selvakumaran
```

But Redis still contains:

``` text
name = Selva
```

Now the application returns stale data.

This is why cache invalidation matters.

------------------------------------------------------------------------

# Invalidation Strategy 1: Delete the Cache Key

The simplest strategy is:

``` text
Update Database
       |
       v
Delete Cache Key
```

Example:

``` text
UPDATE users
SET name = 'Selvakumaran'
WHERE id = 123;

DEL user:123;
```

The next read:

``` text
Cache Miss
    |
    v
Database
    |
    v
Refresh Cache
```

This is simple and reliable for many use cases.

------------------------------------------------------------------------

# Invalidation Strategy 2: Update the Cache

Instead of deleting the key:

``` text
Database Update
      |
      v
Cache Update
```

For example:

``` text
Database:
user:123 → name = Selvakumaran

Redis:
user:123 → name = Selvakumaran
```

This avoids a cache miss on the next request.

However, updating both systems introduces consistency concerns.

------------------------------------------------------------------------

# Invalidation Strategy 3: Event-Based Invalidation

In a microservice architecture, an event can notify other components
that data has changed.

``` text
User Service
    |
    v
Database
    |
    v
UserUpdated Event
    |
    v
Message Broker
    |
    v
Cache Consumer
    |
    v
Invalidate Redis
```

For example:

``` text
UserUpdated {
    userId: 123
}
```

The cache consumer executes:

``` text
DEL user:123
```

This is useful when multiple services maintain related cached data.

------------------------------------------------------------------------

# TTL --- Time To Live

A cache entry should usually not live forever.

**TTL** defines how long a cached item remains valid.

Example:

``` text
user:123
TTL = 300 seconds
```

After five minutes, Redis automatically expires the key.

``` text
Create
  |
  v
Cache Entry
  |
  | 300 seconds
  v
Expired
```

TTL provides an important safety mechanism against stale data.

------------------------------------------------------------------------

# Choosing TTL

TTL depends on how frequently the underlying data changes.

For example:

  Data                             Example TTL
  ----------------- --------------------------
  User profile                   5--30 minutes
  Feed                Seconds to a few minutes
  Trending topics                      Seconds
  Product catalog             Minutes to hours
  Configuration               Minutes to hours
  Session data         Based on session policy

These are starting points, not universal values.

The correct TTL should come from actual freshness requirements and
traffic patterns.

------------------------------------------------------------------------

# TTL and Stale Data

A longer TTL:

``` text
Long TTL
   |
   +--> Better cache hit rate
   |
   +--> Higher chance of stale data
```

A shorter TTL:

``` text
Short TTL
   |
   +--> Fresher data
   |
   +--> More database/cache misses
```

So TTL is a trade-off between:

**Freshness vs Performance**

------------------------------------------------------------------------

# Cache Eviction

Redis memory is not infinite.

Eventually, the cache may run out of available memory.

An eviction policy determines what happens when memory is full.

Common strategies include:

### LRU --- Least Recently Used

Remove data that has not been accessed recently.

``` text
A → accessed recently
B → accessed recently
C → not accessed
D → not accessed for a long time

Evict D
```

### LFU --- Least Frequently Used

Remove data that is accessed least often.

``` text
A → 10,000 accesses
B → 5,000 accesses
C → 20 accesses

Evict C
```

### TTL-Based Expiration

Entries naturally disappear when their TTL expires.

### No Eviction

Redis can also be configured to reject new writes when memory is
exhausted.

------------------------------------------------------------------------

# Cache Stampede

A cache stampede occurs when a popular cache entry expires and many
requests simultaneously try to rebuild it.

Suppose:

``` text
Popular key
user:123
```

expires.

Suddenly:

``` text
1,000 requests
      |
      v
Redis
      |
      X
Cache Miss
      |
      v
1,000 Database Queries
```

The database can become overloaded.

This is also called the **thundering herd problem**.

------------------------------------------------------------------------

# Preventing Cache Stampede

Several techniques can help.

## 1. Locking

Allow only one request to rebuild the cache.

``` text
Request A → Cache Miss → Acquire Lock → DB
Request B → Cache Miss → Wait
Request C → Cache Miss → Wait
```

After Request A updates Redis:

``` text
Redis
  |
  v
Requests B and C
  |
  v
Cache Hit
```

------------------------------------------------------------------------

## 2. TTL Jitter

Instead of giving every cache entry exactly the same TTL:

``` text
TTL = 300 seconds
```

add a small random variation:

``` text
TTL = 285–315 seconds
```

This prevents thousands of keys from expiring simultaneously.

------------------------------------------------------------------------

## 3. Background Refresh

Refresh frequently accessed data before it expires.

``` text
Cache
  |
  | nearing expiration
  v
Background Refresh
  |
  v
Database
  |
  v
Updated Cache
```

This is useful for highly popular data.

------------------------------------------------------------------------

# Cache Penetration

Cache penetration happens when requests repeatedly ask for data that
does not exist.

Example:

``` text
GET /users/999999999
```

If the user doesn't exist:

``` text
Redis → MISS
Database → MISS
```

Every request goes to the database.

An attacker could intentionally send many such requests.

------------------------------------------------------------------------

# Preventing Cache Penetration

One approach is to cache negative results.

``` text
user:999999999
value = NOT_FOUND
TTL = 60 seconds
```

Now repeated requests return from the cache.

Another technique is a **Bloom Filter**.

``` text
Request
   |
   v
Bloom Filter
   |
   +---- Definitely doesn't exist → Reject
   |
   +---- Might exist → Cache / DB
```

------------------------------------------------------------------------

# Cache Breakdown

Cache breakdown happens when a highly popular key expires and suddenly
many requests hit the database.

This overlaps with cache stampede, but the focus is often on a single
very hot key.

For example:

``` text
Trending Post
      |
      v
Popular cache key expires
      |
      v
Thousands of requests
      |
      v
Database
```

Distributed locking, background refresh, and carefully managed TTLs can
help.

------------------------------------------------------------------------

# Redis Clustering

A single Redis instance eventually becomes a bottleneck.

For example:

``` text
Application
     |
     v
Single Redis
     |
     X
Memory / CPU / Network limit
```

Redis Cluster allows data to be distributed across multiple nodes.

``` text
                 Redis Cluster
        +-----------+-----------+
        |           |           |
        v           v           v
      Node A      Node B      Node C
      Slots       Slots       Slots
```

Redis Cluster partitions keys using hash slots.

There are **16,384 hash slots**.

Keys are mapped to slots, and slots are distributed among cluster nodes.

Conceptually:

``` text
Key
 |
 v
Hash
 |
 v
Hash Slot
 |
 +----> Node A
 +----> Node B
 +----> Node C
```

This allows the cache to scale horizontally.

------------------------------------------------------------------------

# Redis Cluster and Replication

For high availability, Redis Cluster can use replicas.

``` text
           Primary A
          /          \
         v            v
    Replica A1    Replica A2

           Primary B
               |
               v
          Replica B1
```

If a primary fails, a replica can take over.

This provides:

-   Horizontal partitioning
-   High availability
-   Failover
-   Larger aggregate memory capacity

------------------------------------------------------------------------

# Redis Cluster Key Design

Key naming becomes very important.

A consistent naming convention helps:

``` text
user:{id}
post:{id}
feed:{userId}
session:{sessionId}
recommendations:{userId}
```

For example:

``` text
user:123
user:456
post:1001
post:1002
feed:123
```

Clear key naming makes debugging and cache management easier.

------------------------------------------------------------------------

# Hot Keys

A hot key is a cache key that receives an unusually large amount of
traffic.

For example:

``` text
post:123
```

might represent a viral post.

Millions of requests could target the same key:

``` text
1,000,000 requests
        |
        v
    post:123
        |
        v
 Single Redis shard
```

Even though Redis Cluster distributes keys, all requests for the same
key can still concentrate on one shard.

Possible strategies include:

-   Local caching for extremely hot data
-   Key replication
-   Request coalescing
-   Application-level caching
-   Splitting data where appropriate

------------------------------------------------------------------------

# Cache Consistency

Caching introduces another important question:

**How consistent must the cache be with the database?**

There are different consistency models.

### Stronger Consistency

The application prioritizes returning the latest data.

Useful for:

-   Payments
-   Account balances
-   Security permissions

### Eventual Consistency

The cache may temporarily contain stale data.

Useful for:

-   Social feeds
-   Trending content
-   Recommendations
-   View counts

The correct strategy depends on the business requirement.

------------------------------------------------------------------------

# Cache Architecture for We Connect

For our **We Connect** system, Redis can sit between microservices and
persistent databases.

``` text
                         Clients
                            |
                            v
                     +-------------+
                     | API Gateway |
                     +-------------+
                            |
                     Load Balancer
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
        User Service   Post Service   Feed Service
              |             |             |
              +-------------+-------------+
                            |
                            v
                    +---------------+
                    | Redis Cluster |
                    +---------------+
                            |
                            v
                       Databases
```

Example cached data:

``` text
user:{userId}
post:{postId}
feed:{userId}
trending
recommendations:{userId}
```

------------------------------------------------------------------------

# Example: User Profile Request

Without cache:

``` text
Client
  |
  v
User Service
  |
  v
Database
  |
  v
Client
```

With cache-aside:

``` text
Client
  |
  v
User Service
  |
  v
Redis
  |
  +---- HIT ----> User Service
  |
  +---- MISS
        |
        v
     Database
        |
        v
      Redis
        |
        v
   User Service
        |
        v
      Client
```

This simple change can dramatically reduce database reads for frequently
accessed profiles.

------------------------------------------------------------------------

# Cache Failure Strategy

What happens if Redis is unavailable?

The application should have a deliberate strategy.

One option is:

``` text
Redis unavailable
       |
       v
Fallback to Database
```

This is called **fail open** for the cache layer.

But there is a danger:

``` text
Redis failure
    |
    v
All requests hit DB
    |
    v
Database overload
```

Therefore, cache failure handling should often be combined with:

-   Database connection limits
-   Circuit breakers
-   Rate limiting
-   Timeouts
-   Load shedding

Caching should reduce load, but the database should still have
protection when the cache disappears.

------------------------------------------------------------------------

# Cache Metrics

A cache should be observable.

Important metrics include:

### Cache Hit Ratio

``` text
Cache Hits
-------------------------
Cache Hits + Cache Misses
```

For example:

``` text
Hits   = 950
Misses = 50

Hit Ratio = 95%
```

A high hit ratio generally means the cache is serving its purpose.

### Other Metrics

Monitor:

-   Cache hit rate
-   Cache miss rate
-   Redis latency
-   Memory utilization
-   Eviction count
-   Key expiration rate
-   Connection count
-   Command throughput
-   Hot keys
-   Replication lag
-   Cluster health

------------------------------------------------------------------------

# Cache Design Checklist

Before introducing a distributed cache, ask:

``` text
1. What data should be cached?
2. How stale can the data be?
3. What should the TTL be?
4. What is the cache invalidation strategy?
5. What happens on cache miss?
6. What happens when Redis fails?
7. What eviction policy should be used?
8. Can a cache stampede occur?
9. Are there hot keys?
10. Do we need Redis Cluster?
11. How will cache consistency be maintained?
12. Which metrics will we monitor?
```

------------------------------------------------------------------------

# Putting It All Together

A production-oriented distributed cache architecture can look like:

``` text
                           Client
                              |
                              v
                       API Gateway
                              |
                              v
                        Microservices
                              |
                    +---------+---------+
                    |                   |
                    v                   v
              Redis Cluster          Database
                    |
          +---------+---------+
          |         |         |
          v         v         v
       Primary   Primary   Primary
          |         |         |
       Replica   Replica   Replica
```

The request flow:

``` text
Request
   |
   v
Check Redis
   |
   +---- HIT --------------------> Return
   |
   +---- MISS
          |
          v
       Database
          |
          v
      Populate Redis
          |
          v
        Return
```

And when data changes:

``` text
Write
  |
  v
Database
  |
  v
Invalidate / Update Cache
```

------------------------------------------------------------------------

# Final Takeaway

A distributed cache is not simply:

``` text
"Put Redis in front of the database."
```

A production-ready cache requires careful decisions around:

-   **Cache-aside vs write-through**
-   **Invalidation**
-   **TTL**
-   **Eviction**
-   **Cache stampede**
-   **Cache penetration**
-   **Hot keys**
-   **Consistency**
-   **Failure handling**
-   **Redis clustering**
-   **Observability**

For **We Connect**, a strong starting architecture is:

``` text
Client
  ↓
API Gateway
  ↓
Microservices
  ↓
Redis Cluster
  ↓
Database
```

With:

``` text
Cache-Aside
     +
TTL
     +
Explicit Invalidation
     +
Redis Cluster
     +
Monitoring
```

The most important lesson is:

> **A cache improves performance, but a well-designed cache improves the
> scalability and resilience of the entire system.**

When traffic grows, the goal is not simply to make the database faster.

The goal is to **avoid unnecessary database work in the first place.**
