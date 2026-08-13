# # Day 7: Cache Invalidation – The Hardest Part of Caching

## The Unexpected Problem

"Adding Redis made We Connect fast. Keeping Redis correct was much harder."
In the previous article, we introduced Redis to reduce database load and make We Connect respond in milliseconds.

The results were impressive.

Faster API responses
Lower database load
Better scalability

Everything looked perfect.

Until one user changed their profile picture.

## The Unexpected Problem

Alice uploads a new profile picture.

The application saves it successfully.

User
   │
Update Profile
   │
PostgreSQL ✅ Updated
Alice refreshes her profile.

The new picture appears.

Everything seems fine.

But Bob opens Alice's profile a few seconds later.

He still sees the old picture.

How?

## The Hidden Copy

Remember our Redis architecture?

 User
             │
        API Server
             │
         Check Redis
             │
      Cache Hit → Return Data
             │
      Cache Miss → PostgreSQL
When Alice's profile was first viewed, it was stored in Redis.

Now there are two copies of the same data.

PostgreSQL → New profile picture ✅
Redis → Old profile picture ❌

Redis has become outdated.

This is called stale data.

Why Does This Happen?
Redis doesn't automatically know when PostgreSQL changes.

Imagine photocopying an important document.

Later, you edit the original.

The photocopy doesn't magically update.

Redis behaves the same way.

It stores a copy of the data.

If the original changes, the cached copy stays unchanged until we update or remove it.

Why Can't We Skip Caching?
At this point, someone might ask:

"If caching causes stale data, why use it at all?"
Because without Redis...

Every request goes back to PostgreSQL.

User
   │
API Server
   │
PostgreSQL
Thousands of users requesting the same profile would generate thousands of identical database queries.

Caching exists because databases should spend time processing new work—not repeating the same work.

The challenge isn't caching.

The challenge is keeping the cache fresh.

## What Is Cache Invalidation?

Cache invalidation simply means:

Removing or updating cached data when the original data changes.
Whenever data changes in PostgreSQL, we need to decide:

Should we delete the cached copy?
Should we update the cached copy?
Should we wait until it expires?

There isn't a single correct answer.

Different applications choose different strategies.

Strategy 1: Time-To-Live (TTL)
The simplest solution is expiration.

When data is stored in Redis, we attach a timer.

Example:

User Profile

TTL = 300 seconds
For five minutes, Redis serves the cached data.

After five minutes...

The cache disappears automatically.

The next request fetches fresh data from PostgreSQL.

Simple.

Easy.

Very common.

The Problem with TTL
Suppose Alice changes her profile picture after just 30 seconds.

The cache still has 270 seconds remaining.

Everyone continues seeing the old profile picture until the cache expires.

The application is fast.

But the data isn't accurate.

TTL improves performance, but it cannot guarantee fresh data.

Strategy 2: Cache Eviction
Instead of waiting for TTL...

We immediately remove the cached data whenever PostgreSQL changes.

Example:

Update Profile
      │
Update PostgreSQL
      │
Delete Redis Cache
The next request becomes a Cache Miss.

Fresh data is loaded from PostgreSQL and stored back in Redis.

This approach ensures users receive updated information much sooner.

Strategy 3: Cache Update
Some applications go one step further.

Instead of deleting the cache...

They update Redis immediately.

Update Database
      │
Update Redis
Now both PostgreSQL and Redis contain the latest data.

No stale information.

No Cache Miss.

This provides the fastest user experience but requires additional logic to keep both systems synchronized.

Which Strategy Should We Use?
It depends on the type of data.

User Profile
Changes occasionally.

Cache eviction or cache update works well.

Trending Posts
Small delays are acceptable.

TTL is usually enough.

Banking Transactions
Never serve stale data.

Avoid caching critical balances or ensure updates happen instantly.

Leaderboards
High read traffic with frequent updates.

Often use Redis as the primary store for real-time rankings.

Every application has different requirements.

Caching is always a trade-off between speed and consistency.

A Real Example from We Connect
Imagine Alice changes her display name.

Without cache invalidation:

Database

Alice Johnson

Redis

Alice Smith
Some users see the new name.

Others see the old one.

Eventually the cache expires.

Everything becomes consistent again.

These temporary inconsistencies are common in distributed systems.

The important part is designing how long they're acceptable.

Why Engineers Say Cache Invalidation Is Hard
There's a famous quote in software engineering:

"There are only two hard things in Computer Science: cache invalidation and naming things."
It sounds humorous...

But there's truth behind it.

Making an application fast is relatively straightforward.

Keeping fast data synchronized with the source of truth is much harder.

As systems grow larger, this challenge becomes increasingly complex.

What We Learned
Redis solved our performance problems.

But it introduced a new challenge.

Cached data can become stale.

To handle this, applications use strategies such as:

TTL (automatic expiration)
Cache eviction
Cache updates

Each strategy involves a balance between:

Performance
Freshness
Complexity

There is no universal solution.

The best strategy depends on the business requirements.

What's Next?
Now that we understand when cached data becomes stale, the next question is:

How should our application interact with the cache?

Should it read from Redis first?

Should Redis automatically fetch from the database?

Should writes go through Redis?

In the next article, we'll explore the three most common caching patterns used in production systems:

Cache-Aside (Lazy Loading)
Read-Through Cache
Write-Through Cache

These patterns are used by companies like Netflix, Amazon, and many large-scale distributed systems to balance performance, scalability, and data consistency.