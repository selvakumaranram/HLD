# Day 10 – Database Sharding: When One Database Is No Longer Enough

## The System Design Diaries – Building We Connect from Scratch

As We Connect continued to grow, we celebrated an important milestone.

Millions of users were actively posting updates, sending messages, liking content, commenting on posts, and updating their profiles every second.

Our application servers scaled effortlessly.

We had introduced database replication, and read performance was excellent.

Pages loaded quickly.

The primary database wasn't overloaded with SELECT queries anymore.

Everything seemed perfect.

Until one day...

The primary database started struggling again.

But this time, the problem wasn't reads.

It was writes.

## When Replication Isn't Enough

Database replication solved one major problem.

It distributed read traffic across multiple replicas.

However, every write operation still had to go through a single Primary PostgreSQL database.

Every second, users were performing operations like:

Creating new posts
Sending messages
Following new users
Updating profiles
Writing comments
Recording notifications

All of these operations required:

INSERT
UPDATE
DELETE

No matter how many read replicas we added...

Every write request still reached the same database server.

Eventually, it became the bottleneck.

The New Bottleneck
Our architecture looked like this:

 Primary Database
            (Writes + Replication)
             /        |        \
            ▼         ▼         ▼
      Read Replica Read Replica Read Replica
Read traffic was distributed.

Write traffic wasn't.

As more users joined We Connect, the Primary database's CPU usage kept increasing.

Transactions became slower.

Locks increased.

Response times started growing.

Adding another replica didn't help.

Because replicas don't process writes.

They only receive replicated data.

We needed an entirely different strategy.

Thinking Differently
Instead of asking:

"How do we build a bigger database?"
We asked:

"Why should one database store everyone's data?"
That simple question changed our architecture.

Instead of one enormous database...

We split the data across multiple databases.

This technique is called Database Sharding.

## What Is Database Sharding?

Database Sharding is the process of horizontally partitioning data across multiple independent databases called shards.

Each shard stores only a portion of the total data.

Instead of one database handling every user, the workload is divided.

For example:

Shard 1 → Users 1 – 1,000,000

Shard 2 → Users 1,000,001 – 2,000,000

Shard 3 → Users 2,000,001 – 3,000,000

...
Each shard handles:

Reads
Writes
Indexes
Storage

independently.

Now multiple databases process write operations simultaneously.

How We Connect Uses Sharding
Whenever a request reaches our backend:

User Request
      │
      ▼
Application Server
      │
      ▼
Shard Resolver
      │
      ▼
Correct Database
The application first determines:

Which shard contains this user's data?
Only then does it connect to the appropriate database.

For example:

User ID 356782

↓

Hash(User ID)

↓

Shard 2
The request never touches the other databases.

Only one shard handles it.

## Choosing the Right Shard Key

One of the most important design decisions is selecting the Shard Key.

A shard key determines where data will live.

Common choices include:

User ID
Customer ID
Geographic Region
Organization ID
Tenant ID

For We Connect, User ID was the natural choice because almost every operation is user-centric.

A good shard key should:

Distribute data evenly
Prevent hotspots
Minimize cross-shard queries
Scale as the application grows

A poor shard key can create one overloaded shard while others remain mostly idle.

The Benefits
After introducing sharding, the improvements were significant.

Higher Write Throughput
Instead of one database processing all writes, multiple databases shared the workload.

Horizontal Scalability
Need more capacity?

Add another shard.

No need to replace the existing infrastructure.

Better Fault Isolation
If one shard experiences issues, only a subset of users is affected.

The rest of the platform continues operating normally.

Improved Performance
Each database stores less data.

Indexes become smaller.

Queries execute faster.

Maintenance becomes easier.

Sharding Isn't Free
Like every architectural decision, sharding introduces complexity.

Some of the challenges include:

Cross-Shard Queries
Generating reports across all users now requires querying multiple databases.

Simple SQL queries become more complicated.

Data Rebalancing
As one shard grows faster than others, data may need to be redistributed.

Moving millions of records safely requires careful planning.

Transactions
Transactions spanning multiple shards are much harder to manage than transactions within a single database.

Many distributed systems avoid cross-shard transactions whenever possible.

Operational Complexity
Monitoring, backups, migrations, and deployments now happen across several databases instead of one.

This requires better tooling and automation.

Replication vs. Sharding
These two techniques solve different problems.

Database ReplicationDatabase ShardingScales ReadsScales WritesMultiple Copies of Same DataDifferent Data in Different DatabasesImproves Read PerformanceImproves Write ThroughputHigh AvailabilityHorizontal Data Growth

Large-scale systems often use both.

Each shard can have its own read replicas, providing scalability for both reads and writes.

Lessons We Learned
One of the biggest lessons while building We Connect was understanding that every scaling technique has a purpose.

Replication helped us distribute read traffic.

Sharding helped us distribute write traffic.

Neither replaces the other.

Together, they form the foundation of a database architecture capable of supporting millions of users.

Key Takeaway
As applications grow, there comes a point where a single database—no matter how powerful—can no longer handle the workload.

Instead of continuously upgrading hardware, distribute your data intelligently.

Replication scales reads.

Sharding scales writes.

Knowing when to use each is what separates a scalable architecture from one that eventually reaches its limits.