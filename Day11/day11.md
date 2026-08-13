# API Gateway: One Entry Point for Hundreds of Microservices

## We Connect HLD Series

In our previous article, we explored how **Event-Driven Architecture with Kafka** allows services to communicate asynchronously and scale independently.

But another challenge appears as our platform grows.

Imagine **We Connect** now consists of dozens of microservices.

* User Service
* Authentication Service
* Post Service
* Feed Service
* Notification Service
* Search Service
* Messaging Service
* Recommendation Service
* Analytics Service

Now think about the mobile application.

Should it communicate with every service individually?

Not really.

This is where an **API Gateway** becomes an essential part of a modern microservice architecture.

---

# The Problem

Without an API Gateway, every client needs to know where every service lives.

```
                Mobile App
                     │
     ┌───────────────┼────────────────┐
     │               │                │
     ▼               ▼                ▼
User Service     Post Service     Feed Service
     │               │                │
     ▼               ▼                ▼
Search Service  Notification     Recommendation
```

Immediately, several problems appear.

* The client manages multiple URLs.
* Authentication must be implemented for every service.
* Every service becomes publicly exposed.
* Version management becomes difficult.
* Network calls increase dramatically.

As the number of services grows...

The client becomes increasingly complicated.

---

# A Better Approach

Instead of exposing every service...

Expose only one.

```
           Mobile App
                │
                ▼
        +----------------+
        |  API Gateway   |
        +----------------+
         │     │      │
         ▼     ▼      ▼
      User   Post   Feed
     Service Service Service
```

The client communicates with a single endpoint.

The gateway handles everything else.

---

# What Is an API Gateway?

Think of an API Gateway as the **front door** of your application.

Every request enters through it.

The gateway decides:

* Which service should receive the request
* Whether the user is authenticated
* Whether rate limits are exceeded
* Whether the request should be cached
* Whether the request should be logged

The backend services focus only on business logic.

---

# A Real Example

Suppose Alice opens We Connect.

The application needs:

* User profile
* Latest feed
* Friend suggestions
* Notification count

Without a gateway, the mobile app makes four separate requests.

With an API Gateway:

```
GET /dashboard
        │
        ▼
   API Gateway
        │
 ┌──────┼─────────┐
 ▼      ▼         ▼
User   Feed   Notification
Service Service Service
        │
        ▼
 Recommendation
```

The gateway aggregates all responses into a single payload.

One request.

One response.

Better performance.

---

# Core Responsibilities

## Request Routing

The gateway forwards requests to the correct microservice.

Example:

```
/users/*  → User Service

/posts/*  → Post Service

/search/* → Search Service
```

Clients don't need to know service locations.

---

## Authentication

Instead of validating JWT tokens in every service...

The gateway validates them once.

If authentication succeeds...

The request proceeds.

Otherwise...

It is rejected immediately.

---

## Authorization

The gateway can also verify permissions.

For example:

* Admin endpoints
* Premium features
* Internal APIs

Unauthorized requests never reach backend services.

---

## Rate Limiting

Imagine someone sends:

100,000 requests per minute.

Without protection...

Your servers could become overloaded.

The gateway can enforce limits such as:

```
100 requests/minute/user
```

Excess requests receive:

```
HTTP 429
Too Many Requests
```

---

## Load Balancing

Suppose Post Service has three instances.

```
Post Service

Instance 1

Instance 2

Instance 3
```

The gateway distributes incoming traffic evenly.

This improves performance and availability.

---

## Caching

Some requests don't change frequently.

Example:

Trending hashtags

Popular communities

Public user profiles

The gateway can cache these responses.

Benefits include:

* Lower latency
* Reduced database load
* Lower infrastructure costs

---

## Logging and Monitoring

Every request passes through the gateway.

This makes it the perfect place to collect:

* Response times
* Error rates
* Request volume
* Geographic traffic
* API usage

Operations teams gain a complete view of system health.

---

# API Versioning

Suppose Version 2 of your API is released.

Instead of changing every service...

The gateway routes traffic.

```
/api/v1/posts

/api/v2/posts
```

Both versions can coexist during migration.

---

# SSL Termination

Managing HTTPS certificates across dozens of services is difficult.

Instead...

The gateway handles SSL encryption.

Internal services communicate securely within the private network.

This simplifies certificate management.

---

# Benefits

✅ Single entry point

✅ Simplified client applications

✅ Improved security

✅ Easier monitoring

✅ Centralized authentication

✅ Load balancing

✅ Request aggregation

✅ API version management

---

# Challenges

An API Gateway introduces its own considerations.

* It can become a bottleneck if not scaled.
* High availability is essential.
* Poor routing rules can affect performance.
* Additional latency may be introduced.
* Configuration management becomes important.

Fortunately, modern gateways are designed to scale horizontally.

---

# Popular API Gateway Solutions

Several production-ready gateways are widely used.

* Kong
* NGINX
* Spring Cloud Gateway
* AWS API Gateway
* Azure API Management
* Apigee
* Traefik

The right choice depends on your infrastructure and operational needs.

---

# How API Gateway Works with Kafka

In our We Connect architecture:

* The API Gateway handles synchronous client requests.
* Kafka powers asynchronous communication between backend services.

For example:

1. User submits a new post through the API Gateway.
2. The gateway routes the request to the Post Service.
3. The Post Service stores the post.
4. A **PostCreated** event is published to Kafka.
5. Feed, Notification, Search, Analytics, and Recommendation services process the event independently.

The gateway and Kafka complement each other rather than replace one another.

---

# Final Thoughts

As applications evolve from a few services to hundreds of microservices, exposing every service directly becomes difficult to manage.

An API Gateway provides a secure, scalable, and centralized entry point for all client requests. It simplifies clients, improves security, enables monitoring, and keeps backend services focused on business logic.

When combined with event-driven systems like Kafka, it forms the backbone of many modern cloud-native architectures.

---

### Coming Next

**Circuit Breaker Pattern: Preventing Cascading Failures in Microservices**

We'll explore how resilient systems isolate failures, recover gracefully, and keep applications running even when downstream services become unavailable.
