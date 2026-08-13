# # The System Design Diaries — Building We Connect from Scratch Redis: Making Applications Lightning Fast

## The Database That Never Got a Break

"The fastest database query is the one you never have to execute."
In the previous article, we solved one of We Connect's biggest challenges.

We made our application stateless using JWT authentication.

Users could log in through any application server.

Everything was finally scalable.

Or so we thought.

As more users joined We Connect, another problem slowly emerged.

Not in the application servers.

Not in the Load Balancer.

The real bottleneck was hiding somewhere else.

Our database.

## The Database That Never Got a Break

Every user interaction required a database query.

Opening the Home Feed?

➡ Read from PostgreSQL.

Viewing a profile?

➡ Read from PostgreSQL.

Loading notifications?

➡ Read from PostgreSQL.

Checking followers?

➡ Read from PostgreSQL.

Every request looked like this:

User
   │
API Server
   │
PostgreSQL
Initially, this wasn't a problem.

With hundreds of users, PostgreSQL handled everything comfortably.

But We Connect wasn't serving hundreds anymore.

It was serving thousands.

Soon...

Hundreds of thousands.

## The Hidden Problem

One afternoon we checked our database metrics.

Something strange stood out.

The same queries kept appearing again and again.

For example:

SELECT * FROM users WHERE id = 1001;

SELECT * FROM users WHERE id = 1001;

SELECT * FROM users WHERE id = 1001;
The user hadn't updated their profile.

Nothing had changed.

Yet the application kept asking the database for exactly the same information.

Imagine asking your friend the same question every five seconds.

Eventually...

They're going to get tired.

Databases feel the same way.

Why Is Reading from a Database Expensive?
Even modern databases are incredibly fast.

But compared to computer memory...

Disk access is slow.

Every database request involves:

Parsing the query
Finding the required data
Reading from storage
Sending the response back

Now imagine millions of users performing those operations simultaneously.

Even powerful databases become overloaded.

The issue wasn't PostgreSQL.

The issue was unnecessary work.

## A Better Question

Instead of asking:

"How can we make the database faster?"
We asked:

"How can we avoid asking the database at all?"
That simple question introduced us to Redis.

## Meet Redis

Redis is an in-memory data store.

Unlike traditional databases that primarily store data on disk, Redis keeps data in RAM.

Because memory is dramatically faster than disk, Redis can return data in milliseconds.

Think of Redis as your application's short-term memory.

Frequently used information lives there.

Everything else remains safely stored in PostgreSQL.

Life Before Redis
Without caching, every request followed the same path.

User
   │
API Server
   │
PostgreSQL
Even if the requested data had been fetched just one second earlier...

The application still queried the database again.

The database became the busiest component in the entire system.

Life After Redis
Now the request flow changed.

 User
               │
          API Server
               │
          Check Redis
         /           \
 Cache Hit         Cache Miss
     │                 │
 Return Data     PostgreSQL
                     │
              Save to Redis
                     │
              Return Response
The application always checks Redis first.

If the data is already available...

The database is never contacted.

If the data isn't found...

The application fetches it from PostgreSQL, stores it in Redis, and returns it to the user.

The next request becomes incredibly fast.

Cache Hit vs Cache Miss
Understanding these two terms is essential.

Cache Hit
The requested data already exists in Redis.

User requests profile

↓

Redis has it

↓

Return immediately
No database query.

Fast response.

Happy users.

Cache Miss
The requested data doesn't exist in Redis.

User requests profile

↓

Redis doesn't have it

↓

Read from PostgreSQL

↓

Store in Redis

↓

Return response
Future requests become Cache Hits.

What Should We Cache?
Not everything belongs in Redis.

Caching works best for data that:

Is read frequently
Doesn't change every second
Is expensive to calculate
Is expensive to retrieve

Examples in We Connect include:

User profiles
Home feed metadata
Trending posts
Notification counts
Popular hashtags
Application settings

These values are requested thousands of times but updated relatively infrequently.

Perfect candidates for caching.

What Shouldn't Be Cached?
Some data changes constantly.

For example:

Bank account balances
Payment transactions
Live inventory
Critical financial records

Serving stale data in these cases could lead to serious problems.

Choosing what to cache is just as important as choosing what not to cache.

Redis Is More Than a Cache
Although Redis is famous for caching, that's only one of its strengths.

Production systems also use Redis for:

Session Storage
Instead of storing sessions inside application memory, every server reads shared session information from Redis.

Rate Limiting
Protect APIs from abuse.

User
↓

100 Requests

↓

Redis Counter

↓

Allow or Reject
Leaderboards
Gaming applications update rankings in real time.

Redis makes these operations extremely efficient.

OTP Storage
Verification codes usually expire within a few minutes.

Redis is perfect for temporary data like this.

Real-Time Counters
Likes.

Views.

Shares.

Downloads.

Redis can update these counters thousands of times per second.

Distributed Locks
When multiple application servers modify the same resource, Redis helps coordinate access and prevents conflicts.

Why Developers Love Redis
After introducing Redis into We Connect, we immediately noticed improvements.

API responses became much faster.
Database CPU usage dropped significantly.
PostgreSQL handled fewer requests.
Users experienced quicker page loads.
The application scaled more efficiently.

Most importantly...

The database could now focus on work that actually required the database.

One Important Question
If Redis stores data in memory...

What happens when the application updates that data?

Suppose a user changes their profile picture.

Redis still contains the old profile.

Should we return outdated information?

Of course not.

This introduces one of the hardest problems in distributed systems:

Cache Invalidation.

As computer scientist Phil Karlton famously said:

"There are only two hard things in Computer Science: cache invalidation and naming things."
We'll explore cache invalidation, TTL (Time-To-Live), Cache Hit, Cache Miss, and different caching strategies in the next article.

Key Takeaways
Databases become bottlenecks when every request reads the same data repeatedly.
Redis stores frequently accessed data in memory, making retrieval much faster.
The application checks Redis before querying PostgreSQL.
A Cache Hit avoids a database query.
A Cache Miss fetches data from PostgreSQL and stores it in Redis.
Redis is widely used for caching, sessions, rate limiting, leaderboards, OTP storage, distributed locks, and real-time counters.
Caching improves response time, reduces database load, and helps applications scale.

What's Next?
Redis made We Connect dramatically faster.

But speed introduces another challenge.

How do we ensure cached data stays fresh?

Should it expire automatically?

Should we update it immediately after every database change?

Should we remove it manually?

In the next article, we'll dive into Cache Invalidation, TTL (Time-To-Live), Cache Hit vs Cache Miss, and popular caching strategies that power modern applications.

