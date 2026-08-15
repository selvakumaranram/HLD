                    WE CONNECT HLD SERIES

                         ┌──────────────┐
                         │ Fundamentals │
                         └──────┬───────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
        DB Replication       Redis          CAP Theorem
              │                 │                 │
              └─────────────────┼─────────────────┘
                                ▼
                         Kafka / Messaging
                                │
                                ▼
                    Event-Driven Architecture
                                │
                                ▼
                         API Gateway
                                │
                                ▼
                     Circuit Breaker
                                │
                                ▼
                      Service Discovery
                                │
                                ▼
                     Distributed Tracing
                                │
                                ▼
                        Database Sharding
                                │
                                ▼
                         Elasticsearch
                                │
                                ▼
                       Rate Limiting
                                │
                                ▼
                      Distributed Systems
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
            Saga              CQRS            Idempotency
              │                 │                 │
              └─────────────────┼─────────────────┘
                                ▼
                       Complete System Design
                                │
        ┌──────────┬───────────┼───────────┬──────────┐
        ▼          ▼           ▼           ▼          ▼
      Uber     Instagram    Netflix     YouTube    LinkedIn


1	Database Replication: Scaling Reads Without Overloading PostgreSQL	
2	Redis: Making Applications Lightning Fast
3	Event-Driven Architecture with Kafka	
4	API Gateway: One Entry Point for Hundreds of Microservices


5. Circuit Breaker Pattern

Preventing Cascading Failures in Microservices

How to keep We Connect running when one service becomes unavailable.

6. Service Discovery

How Microservices Find Each Other

Eureka, Kubernetes Service Discovery, DNS, and dynamic service registration.

7. Distributed Tracing

Finding a Slow Request Across 20 Microservices

OpenTelemetry, trace IDs, spans, and tools such as Jaeger.

8. CAP Theorem

Consistency vs Availability: The Distributed Systems Trade-off

Understanding what happens when network partitions occur.

9. Database Sharding

Scaling Databases Beyond a Single Machine

Horizontal partitioning, shard keys, routing, and rebalancing.

10. NoSQL Internals

How NoSQL Databases Scale to Millions of Requests

Document stores, key-value databases, partitioning, replication, and consistency.

11. Message Queues

Kafka vs RabbitMQ: Choosing the Right Messaging System

When to use event streaming versus traditional message queues.

12. Kafka Deep Dive

Partitions, Consumer Groups, Offsets & Exactly-Once Processing

Going deeper into Kafka after introducing Event-Driven Architecture.

13. Elasticsearch

Building Lightning-Fast Search for We Connect

Indexes, inverted indexes, shards, replicas, and distributed search.

14. Typeahead Search

How LinkedIn-Style Search Suggestions Work

Prefix search, caching, Elasticsearch/Trie approaches, and scaling.

15. Rate Limiting

Protecting APIs from Traffic Spikes and Abuse

Token Bucket, Leaky Bucket, Fixed Window, Sliding Window, and distributed rate limiting.

16. Distributed Cache

Designing a Cache That Works at Scale

Cache-aside, write-through, invalidation, TTL, eviction, and Redis clustering.

17. Distributed Locking

Preventing Multiple Services from Doing the Same Work

Redis-based locks, database locks, leases, and failure scenarios.

18. Idempotency

How Payment Systems Avoid Duplicate Transactions

Idempotency keys, retries, duplicate requests, and distributed systems.

19. Saga Pattern

Managing Distributed Transactions Across Microservices

Choreography vs orchestration and compensating transactions.

20. CQRS

Separating Reads and Writes for Massive Scale

Command Query Responsibility Segregation and when it actually makes sense.

21. Design WhatsApp

Messaging, WebSockets, message delivery, presence, and offline messages.

22. Design Instagram

Posts, feeds, media storage, likes, comments, and notifications.

23. Design Twitter/X

Timeline generation, fan-out, trending topics, and massive read traffic.

24. Design Uber

Location tracking, driver matching, trip management, and geospatial indexing.

25. Design YouTube

Video upload, transcoding, CDN, storage, recommendations, and streaming.

26. Design Netflix

Video streaming, CDN, recommendations, content metadata, and availability.

27. Design Hotstar

Live streaming, massive traffic spikes, CDN, buffering, and concurrency.

28. Design LinkedIn

Feed generation, connections, search, messaging, notifications, and recommendations.

29. Design an E-Commerce Platform

Product catalog, inventory, cart, orders, payments, and fulfillment.

30. Design a Notification System

Push notifications, email, SMS, fan-out, retries, and delivery guarantees.


