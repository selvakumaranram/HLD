# Database Replication: Scaling Reads Without Overloading PostgreSQL

## The System Design Diaries – Building We Connect from Scratch

When we first launched We Connect, a single PostgreSQL database was more than enough.

It handled user registrations, profile updates, posts, comments, and notifications without breaking a sweat.

As our user base grew, we added more application servers behind a load balancer.

The backend scaled beautifully.

But one day, we noticed something strange.

The application servers were healthy.

CPU usage was low.

Memory was stable.

Yet the application was becoming slower.

The culprit wasn't the application layer.

It was the database.

## The Hidden Bottleneck

Every action inside We Connect required database access.

When users opened the home page, the application had to fetch:

User profile
Timeline posts
Comments
Likes
Friend suggestions
Notifications

A single page refresh could generate dozens of SQL queries.

Although users were writing data occasionally—creating posts, liking content, updating profiles—the majority of traffic consisted of read operations.

In most social networking applications,

80–90% of database traffic is reads.
Our PostgreSQL server was handling both reads and writes.

Application Servers
        │
        ▼
  PostgreSQL
(Read + Write)
As the number of users increased, the database became the bottleneck.

Adding more application servers didn't help.

They all depended on the same database.

Why Bigger Servers Aren't Always the Answer
Our first thought was simple.

"Let's upgrade the database server."

More CPU.

More RAM.

Faster SSD.

While this helps temporarily, it doesn't solve the architectural problem.

Eventually, even the largest server reaches its limit.

This approach is called Vertical Scaling.

Small Server
      │
Upgrade
      ▼
Bigger Server
It's expensive.

It has hardware limits.

And it usually requires downtime.

We needed something better.

## The Idea Behind Database Replication

Instead of making one database more powerful...

Why not create multiple copies?

This is called Database Replication.

One database becomes the Primary.

Additional databases become Read Replicas.

 Primary Database
             (Read + Write)
            /       |        \
           ▼        ▼         ▼
      Replica    Replica   Replica
       (Read)     (Read)    (Read)
Now the workload can be shared.

How Replication Works
The Primary database handles every write operation.

INSERT
UPDATE
DELETE

Whenever data changes, PostgreSQL copies those changes to every replica.

The replicas continuously stay synchronized with the primary.

Application servers now follow a simple rule:

Write Requests

Application
      │
      ▼
Primary Database
Read Requests

Application
      │
      ▼
Read Replica
Instead of one database answering thousands of SELECT queries...

Multiple databases share the load.

We Connect's Architecture
Our updated architecture looked like this.

 Load Balancer
                          │
         ┌────────────────┴───────────────┐
         │                                │
    Application Server              Application Server
         │                                │
         └──────────────┬─────────────────┘
                        │
                 Primary PostgreSQL
                (Writes Only)
                 │      │      │
                 ▼      ▼      ▼
            Read Replica 1
            Read Replica 2
            Read Replica 3
Every application server could send:

INSERT → Primary
UPDATE → Primary
DELETE → Primary
SELECT → Any Replica

This dramatically reduced the workload on the primary database.

The Benefits
After introducing replication, we noticed immediate improvements.

Faster Response Times
Read queries were distributed across multiple databases.

Users experienced quicker page loads.

Lower Database CPU Usage
Instead of one server processing every request,

the workload was shared.

Better Scalability
When traffic doubled,

we didn't need a larger database.

We simply added another replica.

Improved Reliability
If one read replica became unavailable,

traffic could be routed to another.

The application remained online.

## Replication Isn't Perfect

Database replication introduces a new challenge.

Imagine this scenario.

A user updates their profile.

The update is saved successfully.

Immediately after that, the user refreshes the page.

The application reads from a replica.

But...

The replica hasn't received the latest update yet.

The user briefly sees old information.

This happens because replication is usually asynchronous.

Changes take a small amount of time to reach every replica.

This delay is called Replication Lag.

Eventual Consistency
Large distributed systems often accept this trade-off.

Instead of guaranteeing every database is updated instantly,

they guarantee that all replicas will become consistent eventually.

This concept is known as Eventual Consistency.

For most applications, a delay of a few milliseconds—or even a second—is perfectly acceptable.

However, for critical operations like:

Payment confirmation
Password changes
Profile updates
Order placement

it's better to read directly from the Primary database immediately after a write.

When Should You Use Replication?
Database replication is an excellent solution when:

✅ Read traffic is significantly higher than write traffic

✅ Your database has become the application's bottleneck

✅ You need better scalability without changing application logic

✅ High availability is important

It's commonly used in:

Social Media Platforms
E-commerce Websites
Banking Dashboards
News Portals
SaaS Applications

Lessons We Learned
One of the biggest mistakes engineers make is assuming that adding more application servers automatically improves performance.

It doesn't.

If every application server depends on a single database, that database eventually becomes the bottleneck.

Database replication shifts the focus from scaling servers to scaling data access.

Sometimes the smartest optimization isn't making one server stronger.

It's allowing multiple servers to share the work.

Key Takeaway
As your application grows, database reads often become the first scalability challenge.

Instead of continuously upgrading your PostgreSQL server, distribute read traffic across replicas.

Scale your reads before you scale your hardware.

Coming Next
In the next article of the We Connect series, we'll explore Database Sharding—the next step when even a replicated database can no longer handle your application's growth.

#SystemDesign #PostgreSQL #DatabaseReplication #BackendEngineering #Java #SpringBoot #DistributedSystems #Scalability #CloudComputing #SoftwareArchitecture #WeConnect