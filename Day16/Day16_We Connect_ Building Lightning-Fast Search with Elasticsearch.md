# We Connect — Building Lightning-Fast Search with Elasticsearch

Imagine **We Connect** has grown from a small social platform into an application with millions of users and posts.

A user opens the search box and types:

> `java developer`

They expect results immediately.

Not after 3 seconds.  
Not after 5 seconds.

**Almost instantly.**

At first, our database handled search perfectly well.

But as We Connect grew, we started asking a difficult question:

> **Should our primary database really be responsible for complex, high-speed search?**

That is where **Elasticsearch** enters the architecture.

---

## 1. The Problem: Searching Millions of Posts

Suppose We Connect stores posts in PostgreSQL.

A simplified table might look like:

```text
posts
--------------------------------
id
user_id
content
created_at
likes
comments
```

A simple search could be:

```sql
SELECT *
FROM posts
WHERE content ILIKE '%java%';
```

This works.

But imagine:

- 10 million posts
- 100 million posts
- 1 billion posts

Now users want more than just a simple text match.

They want:

- Search by keywords
- Search multiple words
- Partial matches
- Typo tolerance
- Ranking by relevance
- Filtering
- Sorting
- Autocomplete
- Highlighting
- Fast response times

Our SQL database wasn't designed primarily for this type of search workload.

So we introduce a dedicated search engine.

---

# 2. Introducing Elasticsearch

Elasticsearch is a **distributed search and analytics engine** built on top of Apache Lucene.

Instead of asking our primary database to perform every search operation, we create a separate search-optimized representation of our data.

The architecture becomes:

```text
                 ┌───────────────┐
                 │     User      │
                 └───────┬───────┘
                         │
                         ▼
                 ┌───────────────┐
                 │  We Connect   │
                 │   Backend     │
                 └───────┬───────┘
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
       ┌──────────────┐      ┌──────────────┐
       │  PostgreSQL  │      │ Elasticsearch│
       │  Source DB   │      │ Search Index │
       └──────────────┘      └──────────────┘
```

The important idea is:

> **PostgreSQL remains the source of truth. Elasticsearch becomes the search-optimized copy.**

---

# 3. Why Not Just Use Database Indexes?

This is an important architecture question.

Databases already support indexes.

For example:

```sql
CREATE INDEX idx_posts_created_at
ON posts(created_at);
```

This is excellent for queries such as:

```sql
SELECT *
FROM posts
ORDER BY created_at DESC;
```

But full-text search is a different problem.

Consider:

```text
"Java developer working with Spring Boot and Kafka"
```

Now the user searches:

```text
spring kafka
```

We want the search engine to understand:

- Individual words
- Their frequency
- Their relevance
- Similar terms
- Language processing
- Ranking

Elasticsearch is designed specifically for this.

---

# 4. How Elasticsearch Finds Text So Fast

The secret is an **Inverted Index**.

Instead of storing:

```text
Post 1 → Java Spring Kafka
Post 2 → Java Developer
Post 3 → Kafka Developer
```

Elasticsearch creates something conceptually similar to:

```text
java      → Post 1, Post 2
spring    → Post 1
kafka     → Post 1, Post 3
developer → Post 2, Post 3
```

Now when the user searches:

```text
java
```

Elasticsearch doesn't scan every post.

It can directly find the documents associated with the term.

This is one of the fundamental reasons search engines can perform text searches efficiently at large scale.

---

# 5. What Happens When a Post Is Created?

Let's say Selva creates this post:

> "Building scalable microservices with Java, Spring Boot and Kafka"

Our application first stores it in PostgreSQL.

Then the searchable representation is indexed into Elasticsearch.

```text
User
 │
 ▼
Create Post
 │
 ▼
We Connect API
 │
 ├──────────────► PostgreSQL
 │                  │
 │                  ▼
 │               Post saved
 │
 └──────────────► Elasticsearch
                    │
                    ▼
                Search indexed
```

Now a search for:

```text
Spring Boot
```

can return this post quickly.

---

# 6. But Should We Write to Both Databases?

This creates an interesting distributed-system problem.

What happens if:

```text
PostgreSQL → SUCCESS
Elasticsearch → FAILURE
```

The post exists in our database but cannot immediately be found through search.

Should we fail the entire request?

Not necessarily.

This is where our previous **Event-Driven Architecture with Kafka** becomes useful.

Instead of making Elasticsearch indexing part of the critical request path:

```text
Create Post
     │
     ▼
PostgreSQL
     │
     ▼
Kafka Event
     │
     ▼
Elasticsearch Consumer
     │
     ▼
Index Post
```

For example:

```text
PostCreated
{
    "postId": "12345",
    "userId": "987",
    "content": "Building scalable microservices..."
}
```

Kafka allows the indexing operation to happen asynchronously.

---

# 7. Elasticsearch Is Not Our Source of Truth

This is one of the most important design decisions.

We should not think:

> "We have two databases containing the same data."

Instead:

```text
PostgreSQL
    ↓
Source of Truth

Elasticsearch
    ↓
Search Projection
```

If Elasticsearch loses an index, we should be able to rebuild it from our primary data source.

This gives us a much safer architecture.

---

# 8. Understanding Elasticsearch Terminology

If you're coming from SQL databases, Elasticsearch terminology can initially feel confusing.

### SQL

```text
Database
  └── Table
       └── Row
```

### Elasticsearch

```text
Cluster
  └── Index
       └── Document
            └── Fields
```

For We Connect:

```text
Elasticsearch Cluster
        │
        └── posts index
                │
                ├── Document 1
                ├── Document 2
                ├── Document 3
                └── ...
```

A document might look like:

```json
{
  "id": "12345",
  "userId": "1001",
  "content": "Building scalable microservices with Java",
  "createdAt": "2026-08-13T06:00:00",
  "likes": 120
}
```

---

# 9. Indexing Is More Than Storing Text

Elasticsearch analyzes text before indexing it.

For example:

```text
"Building Microservices with Spring Boot"
```

can be analyzed into terms such as:

```text
building
microservices
spring
boot
```

This process allows Elasticsearch to perform sophisticated text searches.

We can also configure analyzers depending on our requirements.

For example:

```text
Lowercase
     ↓
Tokenization
     ↓
Stop-word handling
     ↓
Stemming
     ↓
Indexed terms
```

This is where Elasticsearch becomes much more powerful than a simple `LIKE '%keyword%'`.

---

# 10. Relevance: Which Result Should Come First?

Suppose the user searches:

```text
Java Spring Boot
```

We might have thousands of matching posts.

The user doesn't want random results.

They want the **most relevant results first**.

Elasticsearch calculates relevance scores and ranks documents.

Conceptually:

```text
Search Query
     │
     ▼
Matching Documents
     │
     ▼
Relevance Scoring
     │
     ▼
Sort Results
     │
     ▼
Top Results
```

So the result isn't simply:

```text
Post 1
Post 2
Post 3
```

It can be:

```text
Post 87   → Highly relevant
Post 142  → Very relevant
Post 29   → Relevant
Post 501  → Less relevant
```

This dramatically improves the search experience.

---

# 11. Searching We Connect

Now imagine the We Connect search experience.

The user enters:

```text
microservices
```

Our API sends a query to Elasticsearch.

```text
GET /posts/_search
```

Elasticsearch returns matching documents.

Our backend can then return:

```json
{
  "results": [
    {
      "id": "101",
      "content": "Microservices architecture..."
    },
    {
      "id": "205",
      "content": "Building scalable microservices..."
    }
  ]
}
```

The user gets the results without our PostgreSQL database scanning millions of rows.

---

# 12. Filtering + Search

This is where Elasticsearch becomes even more useful.

Suppose the user searches:

```text
Java
```

and wants:

```text
Posts
created after 2026
with more than 100 likes
```

Conceptually:

```text
Text Search
      +
Filters
      +
Sorting
      +
Pagination
```

This is a common pattern for modern applications.

For We Connect, we might eventually support:

```text
Search: Java

Filters:
 ├── Author
 ├── Date
 ├── Category
 └── Language

Sort:
 ├── Relevance
 ├── Recent
 └── Popular
```

---

# 13. Elasticsearch and Eventual Consistency

Remember our architecture:

```text
PostgreSQL
     ↓
Kafka
     ↓
Elasticsearch
```

There can be a small delay.

For example:

```text
10:00:00 → Post created
10:00:01 → Event published
10:00:02 → Elasticsearch indexed
```

During that short period:

```text
Post exists in PostgreSQL
        ↓
Post may not yet appear in search
```

This is **eventual consistency**.

For a social platform like We Connect, this can be an acceptable trade-off.

We prioritize:

- Fast writes
- High availability
- Scalable search

rather than requiring every search result to reflect a post at the exact millisecond it was created.

---

# 14. Elasticsearch Scaling

What happens when We Connect reaches billions of documents?

We cannot simply keep everything on one server.

Elasticsearch distributes data across **shards**.

Conceptually:

```text
                 Elasticsearch Cluster
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Shard 1        Shard 2        Shard 3
          │              │              │
       Posts          Posts          Posts
```

A search can be executed across multiple shards.

```text
                 Search Query
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Shard 1     Shard 2     Shard 3
          │           │           │
          ▼           ▼           ▼
       Results      Results      Results
          └───────────┼───────────┘
                      ▼
                  Merge Results
                      │
                      ▼
                 Final Response
```

This allows Elasticsearch to scale horizontally.

---

# 15. Replication for High Availability

Sharding helps us distribute data.

But what happens if a node fails?

We use replicas.

```text
Primary Shard
     │
     ├── Replica 1
     └── Replica 2
```

If the node containing the primary shard fails, Elasticsearch can promote a replica.

So we get:

```text
Horizontal Scaling
        +
Replication
        =
Scalable + Highly Available Search
```

---

# 16. What About Updates and Deletes?

Suppose a user edits their post.

```text
Post updated
     │
     ▼
PostgreSQL updated
     │
     ▼
PostUpdated event
     │
     ▼
Kafka
     │
     ▼
Elasticsearch consumer
     │
     ▼
Search document updated
```

Similarly:

```text
Post deleted
     │
     ▼
PostDeleted event
     │
     ▼
Kafka
     │
     ▼
Elasticsearch
     │
     ▼
Document removed
```

This keeps our search index synchronized with the source database.

---

# 17. What If Elasticsearch Goes Down?

This is an important architecture question.

Suppose:

```text
PostgreSQL → UP
Kafka       → UP
Elasticsearch → DOWN
```

Should We Connect stop allowing users to create posts?

**No.**

Our architecture should allow the core application to continue operating.

Events can remain in Kafka while Elasticsearch is unavailable.

Once Elasticsearch recovers:

```text
Kafka
  │
  ▼
Consumer
  │
  ▼
Elasticsearch
  │
  ▼
Index catches up
```

This is another reason why asynchronous event-driven indexing is valuable.

---

# 18. Elasticsearch Is Not a Replacement for PostgreSQL

This distinction is critical.

We don't replace:

```text
PostgreSQL
```

with:

```text
Elasticsearch
```

We use both for different responsibilities.

| Requirement | Technology |
|---|---|
| Source of truth | PostgreSQL |
| Transactions | PostgreSQL |
| Relationships | PostgreSQL |
| Complex search | Elasticsearch |
| Full-text search | Elasticsearch |
| Relevance ranking | Elasticsearch |
| Search filtering | Elasticsearch |
| Analytics/search workloads | Elasticsearch |

The architecture is about choosing the right tool for the right workload.

---

# 19. The We Connect Architecture

Putting everything we've discussed together:

```text
                         ┌───────────────┐
                         │     Users     │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │  API Gateway  │
                         └───────┬───────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
             ┌──────────────┐         ┌───────────────┐
             │ Post Service  │         │ Search Service│
             └──────┬───────┘         └───────┬───────┘
                    │                         │
                    ▼                         ▼
             ┌──────────────┐         ┌───────────────┐
             │  PostgreSQL  │         │ Elasticsearch │
             └──────┬───────┘         └───────────────┘
                    │
                    ▼
                 ┌───────┐
                 │ Kafka │
                 └───┬───┘
                     │
                     ▼
             ┌───────────────┐
             │ Search Indexer│
             └───────┬───────┘
                     │
                     ▼
             ┌───────────────┐
             │ Elasticsearch │
             └───────────────┘
```

Now our architecture has a clear separation of responsibilities.

---

# 20. The Architecture Decision

The important lesson isn't:

> "Use Elasticsearch because it is fast."

The real architectural decision is:

> **Separate transactional workloads from search workloads.**

PostgreSQL is optimized for transactional consistency and structured data.

Elasticsearch is optimized for large-scale search and retrieval.

Kafka connects these systems asynchronously.

So our architecture becomes:

```text
PostgreSQL
    │
    │ Source of Truth
    ▼
  Kafka
    │
    │ Events
    ▼
Elasticsearch
    │
    │ Search
    ▼
   User
```

Each component does what it is good at.

---

# 21. Key Takeaways

### 1. Elasticsearch is a search engine

It is designed for fast and powerful text search.

### 2. PostgreSQL remains the source of truth

Elasticsearch should generally be treated as a search projection.

### 3. Inverted indexes make text search efficient

Instead of scanning every document, Elasticsearch can efficiently locate matching terms.

### 4. Kafka can decouple database writes from search indexing

This improves resilience and reduces coupling.

### 5. Elasticsearch supports horizontal scaling

Shards distribute data across nodes.

### 6. Replicas provide high availability

A failed node doesn't necessarily mean losing search availability.

### 7. Eventual consistency is an architectural trade-off

A newly created post may take a short time to appear in search.

### 8. Don't use Elasticsearch for everything

Use the right storage engine for the right workload.

---

# Final Thought

When We Connect had only a few thousand posts, PostgreSQL search was enough.

When We Connect reached millions of posts, search became a separate problem.

And that's an important lesson in system design:

> **As a system grows, the solution isn't always to make one component do more. Sometimes the solution is to give the workload its own specialized system.**

For We Connect:

**PostgreSQL stores the truth.**  
**Kafka moves the change.**  
**Elasticsearch finds the information.**

And that's how we build a search system that can scale with the platform.

---

### What's Next?

We have a search engine.

But there's another problem.

A user shouldn't have to type the entire query.

When they type:

```text
micr...
```

We should immediately suggest:

```text
microservices
microservices architecture
microservices with spring boot
microservices design patterns
```

That takes us to our next HLD challenge:

> **How do we build Typeahead / Autocomplete Search at scale?**

**Next in We Connect: Typeahead Search — Making Search Feel Instant.**