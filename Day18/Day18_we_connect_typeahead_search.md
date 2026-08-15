# We Connect — Typeahead Search: How LinkedIn-Style Search Suggestions Work

## 1. What Is Typeahead Search?

Typeahead means showing relevant suggestions while the user is typing.

```text
User types: jav

Java
JavaScript
Java Developer
Java Spring Boot
Java 17
```

The key requirement is **very low latency**.

---

## 2. Why Normal Search Is Not Enough

If we send every keystroke to Elasticsearch:

```text
m
mi
mic
micr
micro
```

one search can generate many requests.

At large scale, this creates enormous traffic. We therefore need a dedicated typeahead strategy.

---

## 3. Basic Architecture

```text
                    User
                      │
                      ▼
                 Search Box
                      │
                     "mic"
                      │
                      ▼
                 Search API
                      │
                      ▼
                Typeahead Service
                      │
                      ▼
                    Redis
                  /       \
               HIT         MISS
                │            │
                ▼            ▼
            Results     Elasticsearch
```

We don't want every request to reach Elasticsearch. Redis should absorb frequently requested prefixes.

---

## 4. Prefix Search

The simplest approach is **prefix matching**.

```text
m
 ├── microservices
 ├── Microsoft
 └── Michael

mi
 ├── microservices
 ├── Microsoft
 └── Michael

micro
 ├── microservices
 └── microservice architecture
```

When the user types `micro`, we only need suggestions beginning with `micro...`.

---

## 5. Trie — A Classic Solution

A Trie stores characters as nodes.

```text
                root
                 │
                 m
                 │
                 i
                 │
                 c
                 │
                 r
                 │
                 o
```

Words with the same prefix share the same path, making prefix lookup extremely fast.

---

## 6. Ranking Suggestions

We don't just need matching words. We need the **best suggestions first**.

Example:

```text
Java Spring Boot       980
Java Developer         850
Java Microservices     720
JavaScript              650
Java                    600
```

Ranking can consider:

- Popularity
- Search frequency
- Recent searches
- User history
- Location
- Language
- Trending topics

---

## 7. Trie With Top-K Suggestions

Instead of storing only:

```text
prefix → children
```

we can store:

```text
prefix → top 5 suggestions
```

For example:

```text
Prefix: "jav"

1. Java Spring Boot
2. Java Developer
3. Java Microservices
4. Java 17
5. Java Tutorial
```

This makes lookup extremely fast.

---

## 8. The Problem With a Trie

At We Connect scale we may have:

- Millions of users
- Millions of posts
- Millions of searchable terms
- Multiple languages
- Frequently changing popularity

A large Trie can consume significant memory and becomes harder to distribute.

This is where Elasticsearch becomes useful.

---

## 9. Elasticsearch for Typeahead

Elasticsearch provides several autocomplete approaches:

- Prefix queries
- `search_as_you_type`
- Completion suggester
- Edge n-grams

We can create a dedicated suggestion index:

```text
weconnect-suggestions
```

Example document:

```json
{
  "text": "Java Spring Boot",
  "category": "TOPIC",
  "popularity": 980
}
```

---

## 10. Edge N-Grams

An edge n-gram approach indexes prefixes.

For:

```text
microservices
```

we can generate:

```text
m
mi
mic
micr
micro
micros
microse
...
microservices
```

Searching for `micro` can then directly match the indexed prefix.

Trade-off:

> More indexed terms → more storage and indexing work.

---

## 11. Completion Suggester

Elasticsearch's completion suggester is specifically optimized for autocomplete.

```text
User
 │
 │ "mic"
 ▼
Elasticsearch Completion Index
 │
 ▼
Top Suggestions
```

This is useful when the primary requirement is:

> Give me the best suggestions for this prefix.

---

## 12. Elasticsearch vs Trie

| Approach | Strength | Weakness |
|---|---|---|
| **Trie** | Extremely fast prefix lookup | Memory-heavy and harder to distribute |
| **Elasticsearch** | Distributed, scalable, powerful ranking/search | More infrastructure and network overhead |
| **Redis + Trie/Data Structure** | Very fast and cache-friendly | Memory cost and data-management complexity |
| **Database prefix query** | Simple initially | Doesn't scale well for heavy typeahead traffic |

For a large We Connect system, Elasticsearch is a strong candidate for the underlying suggestion index.

---

## 13. Redis + Elasticsearch

Suppose thousands of users search:

```text
java
```

We don't want thousands of identical requests reaching Elasticsearch.

```text
                  User
                    │
                    ▼
               Search API
                    │
                    ▼
                  Redis
                    │
              ┌─────┴─────┐
              │           │
             HIT         MISS
              │           │
              ▼           ▼
          Suggestions  Elasticsearch
                           │
                           ▼
                       Suggestions
                           │
                           ▼
                         Redis
```

The next request for `java` can be served directly from Redis.

---

## 14. Cache Keys

Examples:

```text
typeahead:java
typeahead:jav
typeahead:micro
```

The value can contain:

```json
[
  "Java Spring Boot",
  "Java Developer",
  "Java Microservices",
  "Java 17",
  "Java Tutorial"
]
```

---

## 15. Should We Cache Every Prefix?

No.

Caching every possible prefix wastes memory.

Use:

- LRU caching
- TTL
- Popularity-based caching
- Cache only frequently requested prefixes

Example:

```text
java       → CACHE
micro      → CACHE
spring     → CACHE
kafka      → CACHE

xqzv       → DON'T CACHE
```

---

## 16. Client-Side Debouncing

Without debouncing:

```text
m       → API
mi      → API
mic     → API
micr    → API
micro   → API
```

With debouncing:

```text
User typing
     │
     ▼
Wait ~100–300ms
     │
     ▼
User stops typing
     │
     ▼
Send request
```

This can dramatically reduce request volume.

The exact duration should be tuned based on UX and traffic.

---

## 17. Minimum Prefix Length

Avoid searching for extremely short prefixes.

For example:

```text
m     → no request
mi    → request
mic   → request
```

A minimum prefix of 2 or 3 characters can reduce unnecessary work. The exact threshold depends on the dataset.

---

## 18. Ranking Suggestions

Example:

```text
java
```

may match:

```text
Java
Java Developer
Java Spring Boot
JavaScript
Java Programming
```

A ranking score can combine:

```text
Score =
    Popularity
  + Recent Search Frequency
  + User Personalization
  + Trending Score
  + Exact Prefix Match
```

---

## 19. Personalized Typeahead

Two users can receive different suggestions for the same prefix.

### User A

```text
Java Spring Boot
Java Microservices
Java Kafka
```

### User B

```text
Java Developer
Java Jobs
Java Resume
```

Ranking can combine:

```text
Global popularity
        +
User history
        +
Current context
```

---

## 20. Where Does Kafka Come In?

Kafka can capture search activity and help continuously update popularity.

```text
User searches "java"
        │
        ▼
Search Service
        │
        ▼
Kafka
        │
        ▼
Search Analytics Consumer
        │
        ▼
Update popularity
        │
        ▼
Suggestion Index
```

This creates a feedback loop:

```text
User Searches
      │
      ▼
Kafka
      │
      ▼
Analytics
      │
      ▼
Ranking
      │
      ▼
Suggestion Index
      │
      ▼
Better Suggestions
```

---

## 21. Failure Handling

If Elasticsearch is temporarily unavailable, we should not necessarily break the search box.

```text
Redis
  │
  ├── Cache HIT → Return results
  │
  └── Cache MISS
          │
          ▼
      Elasticsearch
          │
          ├── UP → Results
          │
          └── DOWN
                │
                ▼
        Popular/static suggestions
```

For example:

```text
Showing popular searches:

• Java
• Spring Boot
• Kafka
• Microservices
```

---

## 22. Scaling Typeahead

At large scale, each layer can scale independently.

```text
                     Load Balancer
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
        Typeahead      Typeahead    Typeahead
        Service 1     Service 2     Service 3
             │            │            │
             └────────────┼────────────┘
                          ▼
                       Redis
                          │
                    Cache Miss
                          │
                          ▼
                  Elasticsearch
                 ┌────┬────┬────┐
                 ▼    ▼    ▼    ▼
               Shard Shard Shard Shard
```

---

## 23. Complete We Connect Typeahead Architecture

```text
                         User
                          │
                          ▼
                    Search Box
                          │
                    Debounce
                          │
                          ▼
                    API Gateway
                          │
                          ▼
                  Typeahead Service
                          │
                          ▼
                       Redis
                     /       \
                  HIT         MISS
                   │            │
                   │            ▼
                   │      Elasticsearch
                   │      Suggestion Index
                   │            │
                   │            ▼
                   │         Results
                   │            │
                   │            ▼
                   │           Redis
                   │
                   └────────────┐
                                ▼
                         Ranked Suggestions
                                │
                                ▼
                              User
```

Behind the scenes:

```text
User Search
     │
     ▼
Kafka
     │
     ▼
Search Analytics
     │
     ▼
Popularity / Ranking
     │
     ▼
Elasticsearch
```

---

## 24. Typeahead vs Full Search

### Typeahead

```text
User types: mic

microservices
Microsoft
Michael
```

Goal:

> **Fast suggestions**

### Full Search

```text
microservices architecture for Java applications
```

Returns:

```text
Posts
Articles
Users
Groups
```

Goal:

> **Relevant search results**

Therefore We Connect can have:

```text
Typeahead Service
        │
        └── Optimized for suggestions

Search Service
        │
        └── Optimized for full search
```

They can share Elasticsearch infrastructure, but their query and ranking strategies do not have to be identical.

---

## 25. What Should We Connect Choose?

### Client

Use **debouncing** to reduce unnecessary requests.

### API

Use a dedicated **Typeahead Service**.

### Cache

Use **Redis** for hot/popular prefixes.

### Search Index

Use **Elasticsearch** for scalable prefix search and ranking.

### Data Structure

Use Elasticsearch's autocomplete capabilities rather than maintaining a custom distributed Trie initially.

### Ranking

Combine:

```text
Prefix relevance
+
Popularity
+
Trending score
+
User personalization
```

### Event Processing

Use **Kafka** to capture search activity and update popularity/ranking asynchronously.

---

## 26. Why Not Build a Trie Immediately?

A Trie can be extremely fast.

But building a distributed production Trie introduces additional complexity:

- Memory management
- Replication
- Updates
- Failover
- Synchronization
- Deployment
- Horizontal scaling

If Elasticsearch already solves our requirements, we shouldn't introduce another specialized system without a clear need.

> **The best architecture is the simplest architecture that satisfies the requirements.**

We can start with:

```text
Redis + Elasticsearch
```

and introduce a dedicated Trie-based solution later if scale and latency requirements justify it.

---

## 27. Key Takeaways

1. **Typeahead is different from normal search** — its primary requirement is very low latency.
2. **Prefix search is the foundation** — search based on what the user has typed so far.
3. **Trie is a classic solution** — extremely fast but potentially memory-intensive at large scale.
4. **Elasticsearch is a practical scalable solution** — distributed autocomplete and search capabilities.
5. **Redis reduces Elasticsearch traffic** — popular prefixes can be served directly from cache.
6. **Debouncing reduces requests** — don't send an API request for every keystroke.
7. **Ranking matters** — the best matching suggestion isn't necessarily the first alphabetical result.
8. **Kafka can improve suggestions** — search events can update popularity and ranking asynchronously.
9. **Personalization improves UX** — user history and behavior can influence suggestions.
10. **Don't over-engineer** — start with Redis + Elasticsearch and introduce a custom Trie only when clearly justified.

---

# Final Thought

A search box looks simple.

But at scale:

```text
User types "mic"
        ↓
Millions of possible suggestions
        ↓
Need relevant results
        ↓
Need extremely low latency
        ↓
Need caching
        ↓
Need ranking
        ↓
Need horizontal scaling
```

For We Connect:

> **Debouncing reduces unnecessary requests.**

> **Redis serves hot prefixes quickly.**

> **Elasticsearch provides scalable autocomplete.**

> **Kafka keeps popularity and ranking signals up to date.**

> **A ranking layer makes suggestions relevant.**

The result is a search experience that feels almost instantaneous to the user—even when We Connect is serving millions of users.

---

## What's Next in We Connect?

We now have:

**PostgreSQL → Transactions**  
**MongoDB → Document workloads**  
**Elasticsearch → Full Search**  
**Typeahead → Search Suggestions**  
**Redis → Caching**  
**Kafka → Event Streaming**

But search is only one part of We Connect.

The next major challenge is:

> **How do we build the Home Feed so that millions of users can see personalized posts without querying thousands of relationships and posts for every request?**

That takes us to:

# **We Connect — Designing the Home Feed at Scale: Fan-out on Write vs Fan-out on Read**
