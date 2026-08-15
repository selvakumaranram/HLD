# We Connect — SQL vs NoSQL: When Should We Move from PostgreSQL to MongoDB?

As We Connect grows, our data grows with it.

At the beginning, PostgreSQL can handle almost everything we need.

Users.\
Posts.\
Comments.\
Likes.\
Relationships.\
Transactions.

So why would we ever introduce MongoDB?

A common system-design question is:

> **When should we move from SQL to NoSQL?**

And an even better question is:

> **Do we actually need to move completely from SQL to NoSQL?**

The answer is usually **no**.

For a large system like We Connect, we may use **both PostgreSQL and MongoDB**, with each database solving a different problem.

---

# 1. SQL vs NoSQL — The Basic Difference

Let's start with the fundamental difference.

### SQL Database

PostgreSQL is a relational database.

Data is organized into:

```text
Database
   │
   ├── Tables
   │     ├── Rows
   │     └── Columns
   │
   └── Relationships
```

For example:

```text
Users
--------------------------------
id
name
email
created_at
```

```text
Posts
--------------------------------
id
user_id
content
created_at
```

The relationship is explicit:

```text
User
  │
  └── Posts
```

PostgreSQL is excellent when our data has strong relationships and we need transactional consistency.

---

# 2. What About MongoDB?

MongoDB follows a document-oriented model.

Instead of rows and columns, we store documents.

For example:

```json
{
  "userId": "1001",
  "name": "Selva",
  "preferences": {
    "theme": "dark",
    "language": "en"
  },
  "skills": [
    "Java",
    "Spring Boot",
    "Kafka",
    "MongoDB"
  ]
}
```

The document can contain nested objects and arrays.

Conceptually:

```text
MongoDB
   │
   └── Collection
          │
          ├── Document
          ├── Document
          └── Document
```

This is useful when our data naturally looks like a document.

---

# 3. The Biggest Misconception

One of the most common mistakes in system design is saying:

> "Our application has complex data, so let's move from PostgreSQL to MongoDB."

**Complexity alone is not a reason to choose NoSQL.**

PostgreSQL can handle extremely complex applications.

The real question is:

> **What kind of data do we have, and how do we access it?**

We should evaluate:

- Data relationships
- Transaction requirements
- Query patterns
- Schema flexibility
- Scale
- Read/write patterns
- Consistency requirements
- Data ownership
- Operational complexity

Only then should we decide.

---

# 4. Where PostgreSQL Is Strong

Let's take a simple We Connect example.

A user creates a post.

```text
User
  │
  ▼
Post
  │
  ├── Comments
  ├── Likes
  └── Shares
```

There are relationships everywhere.

We might need queries such as:

```text
Find all posts created by a user

Find all comments for a post

Find users who follow another user

Find mutual connections

Find posts liked by a user
```

This is where PostgreSQL is extremely strong.

---

# 5. Transactions

Suppose a user transfers ownership of something or we have a business operation involving multiple records.

We might need:

```text
Operation A → SUCCESS
Operation B → SUCCESS
Operation C → SUCCESS
```

Or:

```text
Operation A → SUCCESS
Operation B → FAILURE
```

In that case, we may want the entire transaction rolled back.

PostgreSQL provides strong ACID transaction support.

Conceptually:

```text
BEGIN TRANSACTION
       │
       ├── Update User
       ├── Update Post
       ├── Update Relationship
       │
       ▼
     COMMIT
```

If something fails:

```text
ROLLBACK
```

This is one of the major reasons PostgreSQL remains valuable.

---

# 6. Relationships Are PostgreSQL's Strength

Consider We Connect.

A user can:

- Follow another user
- Like a post
- Comment on a post
- Share a post
- Join a group
- Create a post

This produces relationships such as:

```text
User ─── follows ───> User

User ─── creates ───> Post

User ─── likes ─────> Post

User ─── comments ──> Post

User ─── joins ─────> Group
```

Relational databases are naturally designed for this kind of structured relationship.

With PostgreSQL, we can use:

- Primary keys
- Foreign keys
- Joins
- Constraints
- Transactions
- Indexes

This gives us strong data integrity.

---

# 7. So Where Does MongoDB Help?

Now let's look at a different type of data.

Suppose We Connect allows users to create highly customizable profiles.

One user may have:

```json
{
  "name": "Selva",
  "skills": ["Java", "Spring Boot"],
  "socialLinks": {
    "linkedin": "...",
    "github": "..."
  }
}
```

Another user might have:

```json
{
  "name": "John",
  "skills": ["Python"],
  "certifications": [...],
  "projects": [...],
  "preferences": {...},
  "devices": [...]
}
```

The structure can evolve frequently.

Trying to maintain dozens of nullable columns in a relational table could become cumbersome.

A document model can represent this naturally.

---

# 8. Schema Flexibility

This is one of MongoDB's major advantages.

In PostgreSQL, we might have:

```text
users
--------------------------------
id
name
email
phone
address
...
```

As requirements evolve, we may need migrations:

```text
ALTER TABLE users
ADD COLUMN preferences JSONB;
```

Then another feature arrives:

```text
ALTER TABLE users
ADD COLUMN social_links JSONB;
```

Then another:

```text
ALTER TABLE users
ADD COLUMN devices JSONB;
```

PostgreSQL can absolutely support JSON/JSONB.

But if the data is fundamentally document-oriented and changes frequently, MongoDB can provide a more natural model.

---

# 9. MongoDB Is Not "Better" Than PostgreSQL

This is important.

The comparison isn't:

```text
PostgreSQL ❌
MongoDB    ✅
```

It is:

```text
PostgreSQL → Best for certain workloads

MongoDB    → Best for other workloads
```

We choose based on the problem.

---

# 10. Where Should We Use PostgreSQL in We Connect?

Let's define the core data.

### Users

```text
users
```

### Relationships

```text
followers
following
connections
```

### Posts

```text
posts
```

### Comments

```text
comments
```

### Likes

```text
likes
```

These entities have strong relationships and data integrity requirements.

So PostgreSQL is a natural choice.

Our architecture might look like:

```text
                    PostgreSQL
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
      Users           Posts         Relationships
                                        │
                             ┌──────────┼──────────┐
                             ▼          ▼          ▼
                           Likes    Comments     Follows
```

This becomes our **transactional source of truth**.

---

# 11. Where Could We Use MongoDB?

Now let's identify data that naturally behaves like documents.

For We Connect, MongoDB could be useful for areas such as:

### User Preferences

```json
{
  "userId": "1001",
  "theme": "dark",
  "notifications": {
    "email": true,
    "push": false
  },
  "contentPreferences": {
    "technology": true,
    "sports": false
  }
}
```

The structure can evolve independently.

---

### Activity / Event Documents

Imagine we want to maintain user activity:

```json
{
  "userId": "1001",
  "date": "2026-08-14",
  "activities": [
    {
      "type": "LIKE",
      "postId": "5001",
      "timestamp": "10:15"
    },
    {
      "type": "COMMENT",
      "postId": "5002",
      "timestamp": "10:20"
    }
  ]
}
```

This is naturally document-shaped.

MongoDB can be a good fit depending on our read/write patterns.

---

### Flexible Content

Suppose We Connect introduces different types of content:

```text
Text Post
Image Post
Poll
Question
Event
Article
Video
```

Each type might have different fields.

Instead of forcing every possible field into one relational table, a document model can represent these structures more naturally.

For example:

```json
{
  "type": "POLL",
  "content": "Which database do you prefer?",
  "options": [
    "PostgreSQL",
    "MongoDB",
    "MySQL"
  ],
  "metadata": {
    "expiresAt": "2026-08-20"
  }
}
```

This is a strong candidate for a document-oriented model.

---

# 12. But Wait — Couldn't PostgreSQL Store This Too?

**Absolutely.**

This is where the architecture discussion becomes interesting.

PostgreSQL supports JSONB.

We could store:

```json
{
  "preferences": {
    "theme": "dark",
    "language": "en"
  }
}
```

inside a PostgreSQL JSONB column.

So the question isn't:

> "Can PostgreSQL store JSON?"

It can.

The question is:

> **Is this data primarily relational, or primarily document-oriented?**

If most of our application depends on relationships and transactions, keeping it in PostgreSQL may be simpler.

If a particular workload is naturally document-oriented and has different scaling/access requirements, MongoDB becomes more attractive.

---

# 13. The Right Time to Introduce MongoDB

So when should we actually introduce MongoDB?

Not when:

```text
"PostgreSQL feels old."
```

Not when:

```text
"Our JSON is getting bigger."
```

And definitely not when:

```text
"Everyone is using MongoDB."
```

Instead, consider MongoDB when we have one or more of these conditions:

### 1. Schema changes frequently

The structure changes often and different documents legitimately have different fields.

### 2. Data is naturally document-shaped

The application usually reads and writes the document as a whole.

### 3. Joins are not central to the workload

The data doesn't require many relational joins to answer common queries.

### 4. Horizontal scaling becomes an important requirement

We need to distribute a large document workload across multiple nodes.

### 5. Read/write patterns are different from our relational workload

A particular workload has very different access characteristics from our transactional database.

### 6. Independent scaling is valuable

We want to scale this workload separately from PostgreSQL.

---

# 14. A Very Important HLD Principle: Don't Migrate Everything

Suppose We Connect has:

```text
PostgreSQL
  │
  ├── Users
  ├── Posts
  ├── Comments
  ├── Likes
  └── Relationships
```

We discover that user activity has become difficult to manage.

We don't need to do this:

```text
PostgreSQL
      ↓
      ❌
MongoDB
```

Instead:

```text
             We Connect
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
   PostgreSQL           MongoDB
        │                   │
        │                   │
  Core relational     Document-oriented
       data              workloads
```

This is called **polyglot persistence**.

Different databases can coexist in the same system.

---

# 15. PostgreSQL + MongoDB + Elasticsearch

Now our We Connect architecture is becoming much more realistic.

We already introduced Elasticsearch in the previous article.

Now we have:

```text
                       We Connect
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         PostgreSQL     MongoDB    Elasticsearch
              │            │            │
              │            │            │
        Transactions   Documents       Search
        Relationships  Flexible Data   Full-text
        Source Truth   Activity        Ranking
```

And Kafka can connect asynchronous workflows:

```text
                     PostgreSQL
                         │
                         ▼
                       Kafka
                    /          \
                   /            \
                  ▼              ▼
             MongoDB       Elasticsearch
```

Now each system has a specific responsibility.

---

# 16. Example: Creating a Post

Let's walk through a real We Connect operation.

A user creates:

> "My journey learning Java and Spring Boot"

The core post is stored in PostgreSQL:

```text
Post
 ├── id
 ├── user_id
 ├── content
 └── created_at
```

Then an event is published:

```text
PostCreated
```

Kafka distributes the event.

The search indexer updates Elasticsearch:

```text
PostCreated
      │
      ▼
Elasticsearch
      │
      ▼
Searchable post
```

Another consumer could update a document-oriented activity or analytics workload:

```text
PostCreated
      │
      ▼
Kafka
      │
      ▼
MongoDB
      │
      ▼
User activity document
```

So one business event can feed multiple specialized systems.

---

# 17. What Happens During Search?

The user searches:

```text
Java Spring Boot
```

We don't ask PostgreSQL to perform a massive text scan.

Instead:

```text
User
 │
 ▼
Search API
 │
 ▼
Elasticsearch
 │
 ▼
Relevant Posts
```

PostgreSQL still owns the actual post data.

Elasticsearch owns the search representation.

---

# 18. What Happens During a Transaction?

Suppose the user changes their account information.

This is core business data.

We need:

- Strong consistency
- Constraints
- Transactions
- Relationships

So:

```text
User
 │
 ▼
User Service
 │
 ▼
PostgreSQL
```

There is no reason to introduce MongoDB just because we have NoSQL available.

---

# 19. The Decision Matrix

A useful way to think about the decision is:

| Requirement                 | PostgreSQL | MongoDB |
| --------------------------- | ---------: | ------: |
| Strong transactions         |      ⭐⭐⭐⭐⭐ |    ⭐⭐⭐⭐ |
| Complex relationships       |      ⭐⭐⭐⭐⭐ |      ⭐⭐ |
| SQL queries                 |      ⭐⭐⭐⭐⭐ |       ⭐ |
| Joins                       |      ⭐⭐⭐⭐⭐ |      ⭐⭐ |
| Referential integrity       |      ⭐⭐⭐⭐⭐ |      ⭐⭐ |
| Flexible document structure |        ⭐⭐⭐ |   ⭐⭐⭐⭐⭐ |
| Nested documents            |       ⭐⭐⭐⭐ |   ⭐⭐⭐⭐⭐ |
| Frequently changing schema  |        ⭐⭐⭐ |   ⭐⭐⭐⭐⭐ |
| Document-oriented workload  |        ⭐⭐⭐ |   ⭐⭐⭐⭐⭐ |
| Horizontal scaling          |       ⭐⭐⭐⭐ |   ⭐⭐⭐⭐⭐ |

These aren't absolute scores.

The actual choice depends on workload, scale, and access patterns.

---

# 20. What We Should NOT Do

A common architectural mistake is using unnecessary databases and systems.

For example:

```text
PostgreSQL
MongoDB
Redis
Elasticsearch
Cassandra
Neo4j
Kafka
...
```

just because each technology is popular.

This creates:

- More infrastructure
- More operational complexity
- More monitoring
- More failure modes
- More developers needing specialized knowledge
- More data synchronization problems

The best architecture isn't the one with the most technologies.

> **The best architecture is the simplest architecture that satisfies the requirements.**

---

# 21. Our We Connect Decision

For our current We Connect design, we don't need to replace PostgreSQL.

Instead:

### PostgreSQL

Use it for:

```text
Users
Posts
Comments
Likes
Relationships
Core transactional data
```

Why?

Because these entities have strong relationships and consistency requirements.

---

### MongoDB

Introduce it selectively for:

```text
Flexible user preferences
Document-oriented content
Activity/event documents
Workloads requiring independent document scaling
```

Why?

Because these workloads can benefit from flexible schemas and document-oriented access.

---

### Elasticsearch

Use it for:

```text
Full-text search
Search ranking
Filtering
Fast retrieval
Search suggestions
```

Why?

Because search is a specialized workload.

---

### Kafka

Use it for:

```text
Asynchronous events
Decoupling services
Data propagation
Search indexing
Document updates
```

---

# 22. The Final We Connect Architecture

Now our architecture looks like:

```text
                              Users
                                │
                                ▼
                         ┌─────────────┐
                         │ API Gateway │
                         └──────┬──────┘
                                │
                                ▼
                         ┌─────────────┐
                         │   Services  │
                         └──────┬──────┘
                                │
                ┌───────────────┼────────────────┐
                │               │                │
                ▼               ▼                ▼
         ┌────────────┐   ┌────────────┐   ┌──────────────┐
         │ PostgreSQL │   │  MongoDB   │   │ Elasticsearch│
         └─────┬──────┘   └─────┬──────┘   └──────────────┘
               │                │
               │                │
               └────────┬───────┘
                        ▼
                     ┌───────┐
                     │ Kafka │
                     └───┬───┘
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
        Search Indexer        Document Consumer
              │                     │
              ▼                     ▼
       Elasticsearch             MongoDB
```

The important point is not the number of databases.

The important point is the **separation of responsibilities**.

---

# 23. The Real Architecture Question

When someone asks:

> **"Should we use SQL or NoSQL?"**

Don't immediately answer:

> "MongoDB."

Instead ask:

### What is the data?

Is it relational or document-oriented?

### How is it accessed?

Are we performing joins or retrieving whole documents?

### What consistency do we need?

Do we need strong transactional guarantees?

### How does the schema evolve?

Is the structure stable or constantly changing?

### How will it scale?

Do we need independent horizontal scaling?

### Can our existing database already solve the problem?

If PostgreSQL can solve it efficiently, **keep PostgreSQL**.

---

In We Connect, I would introduce MongoDB only for specific microservices where its document model and scaling characteristics make sense.

## Where we can use MongoDB in We Connect

Microservice | Why MongoDB fits
--- | ---
User Preferences Service | Flexible structure — notification settings, themes, privacy settings, interests can evolve frequently
Activity Service | Huge volume of user activities such as likes, views, clicks, follows; document-based activity records
Notification Service | Notifications have different structures/types and can be stored as documents
Content Service | Different content types — text, poll, article, media, event — can have different fields
Feed Service | Pre-computed/user-specific feed documents can be represented naturally and scaled independently
Chat/Messaging Service | Messages are naturally document-oriented and can grow to very large volumes
Audit/Event Service | Flexible event payloads where different event types contain different fields
Media Metadata Service | Images/videos can have different metadata, tags, resolutions, thumbnails, processing information
Recommendation/Profile Enrichment Service | User interests, recommendations, behavioral attributes and dynamically generated profiles can change frequently

## Why MongoDB for these services?

The common characteristics are:

- Flexible/evolving schema
- Nested JSON/document data
- High write volume
- Large-scale data
- Less dependency on complex joins
- Read/write whole document
- Independent horizontal scaling
- Different document structures for different records
- Microservice owns its data independently

## Where PostgreSQL stays in We Connect

PostgreSQL should remain the primary source of truth for core relational data:

- User identity/account
- Users
- Posts
- Comments
- Likes
- Followers/following
- Groups
- Relationships
- Core transactions
- Data requiring strong referential integrity

## So our architecture becomes

                    We Connect
                         │
              ┌──────────┴──────────┐
              │                     │
        Relational Data       Document Workloads
              │                     │
              ▼                     ▼
        PostgreSQL               MongoDB
              │                     │
       Source of Truth       Service-specific data
              │                     │
              └──────────┬──────────┘
                         │
                       Kafka
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
       Elasticsearch             Other Services
          Search

## The key HLD principle

MongoDB shouldn't replace PostgreSQL. A specific microservice should choose MongoDB when its workload naturally fits a document database.

And one more important point: high volume alone is not enough reason to choose MongoDB. We should consider the data model, access pattern, consistency requirements, query patterns, and scaling requirements together.


# Final Thought

The goal of system design isn't to replace SQL with NoSQL.

The goal is to understand the workload and choose the right storage model.

For We Connect:

> **PostgreSQL is our source of truth.**

> **MongoDB handles workloads that naturally fit the document model.**

> **Elasticsearch handles search.**

> **Kafka connects these systems through events.**

This is **polyglot persistence**.

And the most important lesson is:

> **Don't move from SQL to NoSQL because your system is getting bigger. Move a specific workload to NoSQL when its data model and access patterns justify it.**

That's how we scale We Connect without turning our architecture into unnecessary complexity.

---

### What's Next?

We now have:

**PostgreSQL → Transactions**\
**MongoDB → Document workloads**\
**Elasticsearch → Search**\
**Kafka → Event streaming**

But there is another major problem.

Millions of users are constantly generating:

- Likes
- Comments
- Follows
- Post views
- Notifications
- Feed requests

We cannot keep hitting PostgreSQL for every single request.

So the next question becomes:

> **How do we reduce database load and make We Connect extremely fast?**

That takes us to the next We Connect architecture topic:

**Redis — Caching, Distributed Cache & Making We Connect Fast.**
