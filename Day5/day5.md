# The System Design Diaries — Building We Connect from Scratch (Day 5)

## Why Users Kept Getting Logged Out: From Sticky Sessions to Stateless Authentication

"Scaling an application isn't just about adding more servers. It's about making every server behave like the same server."

In the previous article, we solved one of We Connect's biggest challenges.

Our single application server couldn't handle the growing traffic.

So we introduced a Load Balancer and multiple Spring Boot application servers.

The application became faster.

It became highly available.

It could now serve thousands of users simultaneously.

Everything looked perfect...

Until users started complaining.

"I logged in successfully, but after refreshing the page, I was logged out."
At first, we thought it was a bug.

It wasn't.

It was our first lesson in distributed systems.

## Everything Worked on One Server

Originally, We Connect looked like this:

 Users
               │
        Spring Boot API
               │
         PostgreSQL
When a user logged in, the server created a session.

That session lived inside the application's memory.

Every future request reached the same server.

The server simply checked its own memory.

Everything worked perfectly.

## Then We Added More Servers

To handle increasing traffic, we scaled horizontally.

 Users
                  │
          Load Balancer
        ┌──────┼──────┐
        │      │      │
      API1   API2   API3
Now every request first reached the Load Balancer.

The Load Balancer distributed traffic across multiple servers.

From the user's perspective...

Nothing should have changed.

But something did.

## The Mystery of the Random Logout

Imagine Alice logs in.

The login request reaches API-1.

API-1 creates a session in its memory.

Alice Login

        │
        ▼
      API-1
(Session Stored)
A few seconds later...

Alice refreshes the page.

This time the Load Balancer sends the request to API-3.

Refresh

      │
      ▼
    API-3

"No Session Found"
API-3 has never seen Alice before.

The session only exists inside API-1.

From API-3's perspective...

Alice is not logged in.

The server returns:

401 Unauthorized

This is why users experienced random logouts.

Nothing was wrong with authentication.

The problem was where the session was stored.

Why Sessions Become a Problem
Each application server has its own memory.

API-1
Session A
Session B

----------------

API-2
Session C

----------------

API-3
Session D
None of these servers share memory.

A session created on one server is invisible to every other server.

As soon as multiple application servers are introduced, session management becomes much more complicated.

The Quick Fix: Sticky Sessions
One obvious solution is to tell the Load Balancer:

"Always send the same user to the same server."
This approach is called Sticky Sessions.

User A ─────► API-1

User B ─────► API-2

User C ─────► API-3
Now every request from Alice always reaches API-1.

Since the session exists there...

The problem disappears.

At least...

For a while.

Why Sticky Sessions Don't Scale Well
Sticky Sessions solve today's problem.

But they create tomorrow's problem.

Imagine API-1 suddenly crashes.

API-1 ❌

Alice's Session
Gone.
Every user connected to API-1 must log in again.

Even worse...

Suppose:

API-1 has 10,000 users
API-2 has 2,000 users
API-3 has 1,500 users

Traffic can no longer be balanced evenly.

One server becomes overloaded while others remain underutilized.

Sticky Sessions improve user experience initially, but they reduce scalability and resilience.

Modern distributed systems rarely rely on them.

A Better Approach: Stateless Authentication
Instead of storing user information inside server memory...

What if the client carried the authentication information?

This is the idea behind JWT (JSON Web Token).

Here's how it works.

User logs in.
Server validates credentials.
Server generates a signed JWT.
JWT is returned to the client.
The client includes the JWT in every request.

Client

   │
JWT Token

   │
Load Balancer

 │    │    │

API1 API2 API3
Every application server can validate the JWT independently.

No shared session.

No server memory.

No sticky routing.

Any server can process any request.

What Does a JWT Contain?
A JWT is simply a signed token that contains information such as:

User ID
Username
Roles or permissions
Expiration time

Since the token is digitally signed, servers can verify that it hasn't been modified.

There's no need to ask another server for session information.

Is JWT Always Enough?
Not entirely.

Some information still needs to be shared.

For example:

Revoked tokens
Rate limiting
One-time passwords
Frequently accessed user data
Blacklisted sessions

Where should this shared data live?

Not inside application memory.

Not inside the database for every request.

This is where Redis enters the picture.

Introducing Redis
Redis is an in-memory data store shared by every application server.

Instead of keeping temporary information inside API-1 or API-2...

All servers communicate with Redis.

 Users
                  │
          Load Balancer
        ┌──────┼──────┐
        │      │      │
      API1   API2   API3
          \     |     /
             Redis
                │
           PostgreSQL
Now every server sees the same shared information.

Redis is incredibly fast because it stores data in memory rather than on disk.

This makes it ideal for:

Caching
Shared sessions
Token blacklists
Counters
Rate limiting
Frequently accessed data

We'll explore Redis in depth in the next article.

Lessons We Learned
Scaling an application introduces new challenges.

The solution to one problem often creates another.

Adding more servers solved our performance issues.

But it exposed the hidden complexity of session management.

We discovered that:

Server-side sessions don't scale well across multiple application servers.
Sticky Sessions are useful but have limitations.
Stateless authentication with JWT allows any server to process any request.
Redis provides a fast, shared layer for data that must be accessible across all servers.

Modern distributed systems are designed around this principle:

Servers should be disposable. The application should continue working even if any one server disappears.
That's exactly what stateless architecture helps us achieve.

What's Next?
Our application is now stateless.

Authentication is no longer tied to a specific server.

But another challenge is waiting.

Every API request still hits the database.

As our users continue to grow, the database will soon become our next bottleneck.

In the next article, we'll explore Redis in depth—how caching works, why databases become slow under heavy load, and how an in-memory cache can dramatically improve application performance.