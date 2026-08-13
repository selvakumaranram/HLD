# The System Design Diaries: Building We Connect from Scratch

### *A Story-Driven Journey from a Simple Monolith to Internet-Scale Architecture*

> *"Every company starts with an idea. Every scalable system starts with a simple architecture. This is the story of how we build one."*

---

## Welcome to We Connect

Imagine it's your first day as a backend engineer.

You've just joined **We Connect**, a brand-new social media startup inspired by platforms like Instagram and Facebook. The founders have a simple vision:

> **Create a platform where people can connect, share moments, build communities, and stay in touch with friends and family.**

Today, We Connect has no millions of users.

No global data centers.

No Kubernetes clusters.

No complex distributed systems.

Just a small engineering team, a limited budget, and one ambitious goal:

**Build something simple today that can scale tomorrow.**

Our CTO welcomed us with a sentence I'll never forget.

> **"Don't build for tomorrow's problems. Build for today's needs, but make tomorrow's changes easy."**

That became the guiding principle for everything we built.

---

# Before Writing Code, We Planned the Product

Like most developers, I was excited to open my IDE and start coding.

Instead, the CTO gathered everyone around a whiteboard and asked:

> **"What exactly are we building?"**

Ideas started flowing.

* Chat
* Stories
* Reels
* Notifications
* Search
* Live Streaming

The list became longer every minute.

Then the CTO smiled.

> **"If we try to build everything, we'll never launch."**

He was right.

Instead of building every feature, we decided to build an **MVP (Minimum Viable Product)**.

---

# Our First MVP

For Version 1, we agreed to build only the features that create immediate value.

### User Features

* User Registration
* Login
* User Profile

### Social Features

* Create Posts
* Upload Images
* Like Posts
* Comment on Posts
* Follow Users

### Feed

* Home Feed

Everything else would wait.

Not because those features weren't valuable...

...but because they weren't valuable **yet**.

That day I learned my first lesson as a software engineer.

> **Great products don't launch with every feature. They launch with the right features.**

---

# Thinking Like Architects

Once the MVP was finalized, our CTO asked another interesting question.

> **"When We Connect reaches one million users, which feature will receive the most traffic?"**

At first, everyone thought about creating posts.

But after discussing user behavior, the answer became obvious.

People don't spend most of their time posting.

They spend their time scrolling.

That means our **Feed** would eventually become the busiest module in the system.

Likes would grow quickly.

Comments would follow.

Registration, on the other hand, would represent only a tiny fraction of requests.

That discussion completely changed how I looked at software architecture.

> **Not every module grows at the same speed.**

Good architects identify future bottlenecks before they become production problems.

---

# Organizing the Application

Although we planned to deploy a single application, we didn't want a giant codebase where everything was mixed together.

Instead, we divided the application into independent modules.

```text
We Connect

├── Authentication
├── User
├── Feed
├── Post
├── Like
├── Comment
├── Follow
└── Media
```

Every module owns its own:

* Controller
* Service
* Repository
* DTOs
* Business Logic

Everything still runs inside **one application**.

Everything still shares **one database**.

But the boundaries are clear.

The CTO explained it perfectly.

> **"We're not building microservices today. We're building software that's easy to convert into microservices tomorrow."**

That sentence immediately made sense.

---

# Designing Our First Architecture

The next morning, we started designing the backend architecture.

I expected something complicated.

Maybe Kubernetes.

Maybe Kafka.

Maybe dozens of services.

Instead, the CTO drew something surprisingly simple.

```text
Users
   │
Internet
   │
NGINX
   │
Spring Boot Application
   │
MySQL Database
```

I stared at the whiteboard.

"That's it?"

No Redis.

No Kafka.

No Elasticsearch.

No API Gateway.

No Microservices.

Just one application.

One database.

One reverse proxy.

The CTO smiled.

> **"The best architecture isn't the most complicated one. It's the one that solves today's problems."**

---

# Why Didn't We Start with Microservices?

Someone finally asked the question everyone was thinking.

> **"Why don't we build every module as a separate microservice?"**

The CTO responded with another question.

> "How many developers do we have?"

"Four."

> "How many users do we have?"

"Almost none."

He smiled.

> **"Then why would we build an architecture designed for hundreds of developers and millions of users?"**

That answer changed my perspective.

Microservices solve real problems.

But they also introduce new challenges.

* Network communication
* Service discovery
* Distributed logging
* Monitoring
* Deployment complexity
* Authentication between services

Those are excellent solutions...

...when you actually have those problems.

Until then, simplicity wins.

---

# A Modular Monolith

Instead of premature microservices, we chose a **Modular Monolith**.

One deployment.

One database.

Clearly separated modules.

If the Feed module eventually becomes the busiest part of the system, we'll extract it into its own microservice.

Not because microservices are fashionable.

Because they'll solve a real business problem.

Architecture should evolve with the product—not ahead of it.

---

# What I Learned

By the end of my first few days at We Connect, I realized that High-Level Design isn't about choosing the latest technology.

It's about making good engineering decisions.

The best systems aren't designed for millions of users on Day One.

They're designed so they can naturally evolve as those millions arrive.

---

# What's Next?

Our product vision is clear.

Our MVP is defined.

Our architecture is ready.

Next, we'll begin implementing We Connect.

As our users grow from hundreds to thousands—and eventually millions—we'll introduce new technologies only when they're truly needed.

Together, we'll explore:

* Load Balancers
* Redis
* Database Replication
* Sharding
* Kafka
* Elasticsearch
* CAP Theorem
* Distributed Systems
* Microservices
* Designing Facebook News Feed
* Designing Uber
* Designing Hotstar

...and every architectural decision will be driven by one simple question:

> **"What problem are we solving today?"**

---

## Final Thought

Software architecture isn't about building the biggest system.

It's about building the **right system for the current stage of your product** while ensuring it can evolve gracefully as your users, team, and business grow.

That's the journey we're beginning with **We Connect**.

And this is only the first chapter.
