# The System Design Diaries — Building We Connect from Scratch (Day 4)

## When One Server Wasn't Enough: Scaling We Connect with Load Balancers

"Every startup dreams of millions of users. Every backend engineer fears the day they actually arrive."

In the previous article, We Connect had become a real social network.

Users could:

Create posts
Like posts
Comment
View their home feed

Everything worked beautifully.

Until one day...

It didn't.

The issue wasn't a bug.

The issue was growth.

## Yesterday vs Today

A few weeks ago, our application looked like this:

500 users
3,000 requests per hour
One backend server
One database

Life was simple.

Then one of our users posted a viral article.

The next morning...

100,000 active users
Millions of API requests
Thousands of concurrent connections

Suddenly, users started reporting problems.

Login was slow.
Posts took several seconds to load.
Comments sometimes failed.
The feed kept spinning.

Our single backend server had become the bottleneck.

## Our Original Architecture

When we first built We Connect, the architecture was intentionally simple.

 Users
                │
           Internet
                │
        Spring Boot API
                │
          PostgreSQL
Every request followed the same path.

A user opened the app.

The request reached our Spring Boot application.

The application queried PostgreSQL.

The response was returned.

This worked perfectly...

Until thousands of users arrived simultaneously.

## Why Did the Server Become Slow?

Imagine a restaurant with only one chef.

If ten customers arrive...

Everything is fine.

If one thousand customers arrive at the same time...

The chef doesn't suddenly become faster.

The queue keeps growing.

Eventually, customers leave.

A backend server behaves the same way.

Every incoming request consumes:

CPU
Memory
Threads
Network bandwidth

Once these resources are exhausted, response times increase dramatically.

Eventually, requests start timing out.

## The First Solution Everyone Thinks About

Our first thought was simple.

"Let's buy a bigger server."
Instead of:

4 CPU cores
8 GB RAM

We upgrade to:

32 CPU cores
64 GB RAM

This approach is called Vertical Scaling (Scaling Up).

Small Server
      │
      ▼
Bigger Server
      │
      ▼
Even Bigger Server
It works.

For a while.

## The Problem with Vertical Scaling

No matter how powerful your machine is...

Eventually you'll reach its limit.

What happens after buying the biggest available server?

You can't keep upgrading forever.

Even worse...

Everything still depends on one machine.

If that server crashes...

Your entire application goes offline.

One server means one single point of failure.

That isn't acceptable for a production system.

## A Better Idea

Instead of buying one powerful server...

What if we use several ordinary servers?

Server A

Server B

Server C

Server D
Each server handles part of the traffic.

Together, they can process far more requests than one expensive machine.

This approach is called Horizontal Scaling (Scaling Out).

Instead of making one employee work overtime...

You hire more employees.

## But There's a Problem

Now we have four backend servers.

When a user sends a request...

How do they know which server to contact?

Should they choose randomly?

Should the mobile app remember?

Should the browser decide?

No.

Users shouldn't know anything about our infrastructure.

We need something smarter.

## Meet the Load Balancer

A Load Balancer sits between users and backend servers.

Every request first reaches the Load Balancer.

It decides where the request should go.

 Users
                    │
             Load Balancer
           ┌─────┼─────┐
           │     │     │
        API-1  API-2 API-3
           │     │     │
           └─────┼─────┘
             PostgreSQL
From the user's perspective...

Nothing changes.

The application still works exactly the same.

Behind the scenes...

Traffic is now distributed across multiple servers.

## How Does the Load Balancer Decide?

There isn't just one strategy.

Different systems use different algorithms.

1. Round Robin
The simplest approach.

Request 1 → Server A

Request 2 → Server B

Request 3 → Server C

Request 4 → Server A

Request 5 → Server B

Everyone gets an equal share.

Simple.

Fast.

Perfect when every server has identical hardware.

2. Least Connections
Suppose:

Server A is processing 300 requests.

Server B is processing 50 requests.

Instead of sending more traffic to Server A...

The Load Balancer chooses Server B.

This keeps the workload balanced.

3. Weighted Round Robin
Not every server is equally powerful.

Imagine:

Server A → 32 CPU
Server B → 8 CPU
Server C → 8 CPU
Server A should receive more requests.

Weighted routing lets us distribute traffic according to server capacity.

## What If a Server Crashes?

Failures are inevitable.

Servers restart.

Applications crash.

Operating systems fail.

Hardware dies.

Without protection:

Load Balancer
      │
      ▼
 Dead Server
Users receive errors.

Modern Load Balancers constantly perform health checks.

Every few seconds they ask:

"Are you alive?"
If a server stops responding...

It is automatically removed from rotation.

New requests are sent only to healthy servers.

Users often never notice the failure.

## Scaling Without Downtime

Another huge advantage of multiple servers is deployment.

Imagine deploying a new version.

With one server:

Stop application
Deploy new version
Start application

Users experience downtime.

With multiple servers:

API-1 → Update

API-2 → Serving Users

API-3 → Serving Users
Once API-1 finishes updating:

API-2 → Update

API-1 → Serving Users

API-3 → Serving Users
Then API-3.

Users continue using the application while deployments happen in the background.

This is called Rolling Deployment.

It's one of the biggest reasons production systems run multiple application instances.

## Our New Architecture

After introducing a Load Balancer, We Connect looked like this:

 Users
                      │
               Load Balancer
          ┌────────┼────────┐
          │        │        │
      Spring Boot Spring Boot Spring Boot
         API-1      API-2      API-3
             │         │         │
             └─────────┼─────────┘
                   PostgreSQL
Now we could handle far more traffic.

Adding capacity became simple.

Need more performance?

Launch another server.

Register it with the Load Balancer.

Done.

## But Then Something Strange Happened...

A user logged in successfully.

The next API request failed with:

401 Unauthorized
They refreshed.

Sometimes it worked.

Sometimes it didn't.

Why?

Because the login request reached API-1.

The next request reached API-3.

The authentication session existed only on API-1.

The second server had no idea who the user was.

Ironically...

Scaling our application introduced a brand-new problem.

One that every distributed system eventually faces.

Key Takeaways
A single server eventually becomes a bottleneck.
Vertical Scaling means upgrading one server.
Horizontal Scaling means adding more servers.
A Load Balancer distributes traffic across multiple servers.
Health checks automatically remove failed servers.
Multiple servers enable high availability and zero-downtime deployments.
Scaling the application introduces session management challenges.

## Key Takeaways