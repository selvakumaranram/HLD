# Service Discovery: How Microservices Find Each Other

## We Connect HLD Series

In our previous articles, we explored how:

- **Kafka** enables asynchronous communication.
- **API Gateway** provides a single entry point for clients.
- **Circuit Breaker** prevents failures from spreading across services.

But as **We Connect** grows, another problem appears.

Imagine we now have hundreds of microservice instances running across multiple servers and containers.

The Feed Service needs to communicate with the Recommendation Service.

But...

**Where is the Recommendation Service?**

---

# The Problem

In a simple application, we might configure the service URL directly:

```text
http://recommendation-service:8080
```

But production systems aren't that simple.

Imagine Recommendation Service has multiple instances:

```text
Recommendation Service

Instance 1 → 10.0.1.10:8080
Instance 2 → 10.0.1.11:8080
Instance 3 → 10.0.1.12:8080
Instance 4 → 10.0.1.13:8080
```

Now consider what happens when:

- An instance crashes.
- A new instance is created.
- Traffic increases.
- An instance moves to another machine.
- Kubernetes creates a new pod.
- An instance is removed during deployment.

Should the Feed Service keep a list of all these IP addresses?

**Definitely not.**

This is where **Service Discovery** comes in.

---

# What Is Service Discovery?

Service Discovery allows microservices to dynamically find other services without knowing their exact IP addresses.

Instead of:

```text
Feed Service
      │
      ├── 10.0.1.10
      ├── 10.0.1.11
      ├── 10.0.1.12
      └── 10.0.1.13
```

We can simply say:

```text
Feed Service
      │
      ▼
Recommendation Service
```

The infrastructure handles the actual location.

---

# Think of It Like a Phone Book

Imagine you want to call a company.

You don't memorize which phone number belongs to every employee.

You search for:

> "Customer Support"

The directory gives you the current number.

Service Discovery works similarly.

```text
Service Name
     │
     ▼
Service Registry
     │
     ▼
Available Instances
```

The service name stays stable.

The underlying instances can change.

---

# How It Works

A typical Service Discovery architecture looks like this:

```text
                    Service Registry
                  ┌──────────────────┐
                  │                  │
                  │ User Service     │
                  │ Post Service     │
                  │ Feed Service     │
                  │ Search Service   │
                  │ Recommendation   │
                  │                  │
                  └────────┬─────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         Feed Service   Post Service  Search Service
```

Services register themselves.

Other services discover them when needed.

---

# Step 1: Service Registration

When Recommendation Service starts:

```text
Recommendation Service
          │
          ▼
   Register itself
          │
          ▼
   Service Registry
```

The registry stores information such as:

```text
Service:
Recommendation Service

Host:
10.0.1.12

Port:
8080

Status:
Healthy
```

---

# Step 2: Service Discovery

Now Feed Service needs Recommendation Service.

Instead of knowing the IP address:

```text
Feed Service
      │
      ▼
"Where is Recommendation Service?"
      │
      ▼
Service Registry
      │
      ▼
10.0.1.12:8080
```

The Feed Service can now send the request.

---

# Step 3: Health Checking

What happens if the Recommendation Service crashes?

The registry shouldn't continue returning a dead instance.

Health checks help solve this.

```text
Service Registry
       │
       ▼
Health Check
       │
   ┌───┴────┐
   ▼        ▼
Healthy   Unhealthy
   │        │
   ▼        ▼
Available  Remove
```

An unhealthy instance is removed from the available service list.

---

# Why Static IP Addresses Don't Work

Let's say Feed Service is configured with:

```text
10.0.1.12:8080
```

Then that server crashes.

Now:

```text
Feed Service
     │
     ▼

10.0.1.12
     X
   DOWN
```

The application fails.

With Service Discovery:

```text
Feed Service
     │
     ▼
Service Registry
     │
     ▼
10.0.1.13
```

The request can be routed to another healthy instance.

No code change required.

---

# Scaling Becomes Easier

Suppose We Connect suddenly gets a huge traffic spike.

We scale Recommendation Service:

```text
Before:

Recommendation
      │
      └── Instance 1
```

After scaling:

```text
Recommendation
      │
 ┌────┼────┬────┐
 ▼    ▼    ▼    ▼
 I1   I2   I3   I4
```

The new instances register automatically.

The service consumers don't need to know that new instances were created.

This is one of the major benefits of dynamic discovery.

---

# Client-Side vs Server-Side Discovery

There are two common approaches.

## Client-Side Discovery

The client asks the registry for available instances.

```text
Feed Service
     │
     ▼
Service Registry
     │
     ▼
Instance List
     │
     ▼
Load Balancer
     │
     ▼
Recommendation Service
```

The client is responsible for choosing an instance.

---

## Server-Side Discovery

The client sends the request to a load balancer.

```text
Feed Service
     │
     ▼
Load Balancer
     │
     ▼
Service Discovery
     │
     ▼
Recommendation Instance
```

The infrastructure handles instance selection.

This approach is common in cloud-native environments.

---

# Service Discovery in Kubernetes

Modern applications increasingly use Kubernetes.

Kubernetes provides built-in service discovery.

Instead of calling a pod's IP address:

```text
10.0.1.12
```

We can call a Kubernetes Service:

```text
recommendation-service
```

Kubernetes handles the mapping between the service name and the available pods.

For example:

```text
Feed Service
      │
      ▼
recommendation-service
      │
      ▼
 Kubernetes Service
      │
 ┌────┼────┐
 ▼    ▼    ▼
Pod  Pod  Pod
```

If a pod disappears...

Kubernetes updates the available endpoints.

---

# DNS-Based Discovery

Kubernetes commonly uses DNS for service discovery.

A service might be accessible through a stable DNS name such as:

```text
recommendation-service
```

The actual pod IP addresses can change.

The application doesn't need to care.

This gives us an important principle:

> **Use stable service names instead of depending on ephemeral instance addresses.**

---

# Service Discovery + Load Balancing

Service Discovery tells us:

> **Where are the available instances?**

Load Balancing decides:

> **Which instance should receive this request?**

For example:

```text
                 Feed Service
                      │
                      ▼
            Recommendation Service
                      │
                ┌─────┼─────┐
                ▼     ▼     ▼
              Pod 1  Pod 2  Pod 3
                │     │     │
                └─────┼─────┘
                      ▼
                Load Balancer
```

Together they allow traffic to be distributed across healthy instances.

---

# Service Discovery + Circuit Breaker

Our previous article introduced Circuit Breaker.

Now we can combine both patterns.

```text
Feed Service
     │
     ▼
Circuit Breaker
     │
     ▼
Service Discovery
     │
     ▼
Load Balancer
     │
 ┌───┼────┐
 ▼   ▼    ▼
 I1  I2   I3
```

If one instance becomes unhealthy:

```text
Instance 2
    X
  DOWN
```

Service Discovery removes it from the available instances.

The Circuit Breaker protects the caller from repeated failures.

These patterns solve different problems:

**Service Discovery → Find healthy services**

**Load Balancing → Distribute traffic**

**Circuit Breaker → Contain failures**

---

# What Happens During Deployment?

Imagine we are deploying version 2 of Recommendation Service.

We have:

```text
Version 1
├── Instance 1
├── Instance 2
└── Instance 3
```

We deploy:

```text
Version 2
├── Instance 4
└── Instance 5
```

The new instances register themselves.

Traffic can gradually move toward the new instances.

Old instances can then be removed.

This supports safer deployment strategies such as:

- Rolling deployments
- Blue-green deployments
- Canary deployments

Service Discovery becomes an important part of these workflows.

---

# What If the Registry Fails?

The Service Registry itself becomes a critical component.

If every service depends on one registry and that registry goes down...

We could have another failure point.

Therefore, production systems need:

- High availability
- Multiple registry instances
- Health checks
- Replication
- Failure recovery
- Appropriate caching

The service discovery infrastructure must itself be highly reliable.

---

# Popular Service Discovery Approaches

Different ecosystems use different solutions.

### Spring Cloud

**Eureka** is a popular service registry in Spring-based architectures.

### Kubernetes

Kubernetes provides built-in service discovery through Services and DNS.

### Consul

Provides service discovery, health checking, and configuration capabilities.

### Cloud Platforms

AWS, Azure, and other cloud platforms provide managed service discovery mechanisms.

The right solution depends on the deployment environment.

---

# We Connect Example

Let's put everything together.

Imagine We Connect has:

```text
                API Gateway
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       User       Post        Feed
      Service    Service     Service
                                │
                                ▼
                        Service Discovery
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
                   Rec 1       Rec 2       Rec 3
                    │           │           │
                    └───────────┼───────────┘
                                ▼
                         Circuit Breaker
```

And asynchronous processing:

```text
Post Service
     │
     ▼
   Kafka
     │
 ┌───┼───────────┐
 ▼   ▼           ▼
Feed Search   Analytics
```

Now our architecture has several important building blocks:

**API Gateway**

→ Controls incoming client traffic.

**Service Discovery**

→ Helps services find each other.

**Load Balancer**

→ Distributes traffic across instances.

**Circuit Breaker**

→ Prevents cascading failures.

**Kafka**

→ Handles asynchronous communication.

Each pattern has a specific responsibility.

---

# The Bigger Lesson

Microservices aren't simply about creating many small services.

The difficult part is managing communication between those services.

Once we have hundreds of instances:

- IP addresses change.
- Containers restart.
- Pods move.
- Services scale.
- Instances become unhealthy.
- Deployments happen continuously.

Hardcoding service locations doesn't scale.

**Service Discovery allows our architecture to become dynamic.**

---

# Final Thoughts

Imagine We Connect five years ago:

```text
Service A → Fixed IP → Service B
```

Simple.

But imagine We Connect today:

```text
Hundreds of services
Thousands of instances
Containers
Kubernetes
Auto-scaling
Continuous deployments
Dynamic infrastructure
```

We cannot depend on fixed addresses anymore.

We need infrastructure that understands:

> **Where are my services, which instances are healthy, and where should I send the request?**

That's the role of **Service Discovery**.

Combined with API Gateway, Load Balancing, Circuit Breaker, and Kafka, it becomes another important building block for a resilient and scalable distributed system.

---

### Coming Next

**Distributed Tracing: Finding One Slow Request Across Dozens of Microservices**

A user reports:

> "My feed is taking 5 seconds to load."

But the request travels through the API Gateway, Feed Service, Recommendation Service, Database, and several other dependencies.

**Which service is actually slow?**

Next, we'll explore **Distributed Tracing, Trace IDs, Spans, OpenTelemetry, and how to debug requests across a distributed architecture.**
