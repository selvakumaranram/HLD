🔗 **We Connect #19 — Saga Pattern**

### 🔄 How Distributed Transactions Work Without a Traditional Transaction

Imagine you're booking a flight online.

Behind the scenes, several services might be involved:

```text
Booking Service
      ↓
Payment Service
      ↓
Seat Reservation Service
      ↓
Notification Service
```

Now imagine this:

✅ Payment succeeds
✅ Seat reservation succeeds
❌ Flight booking fails

What happens now?

We can't simply use a traditional database transaction across all these services.

That's where the **Saga Pattern** comes in.

---

### 🧩 What is the Saga Pattern?

The **Saga Pattern** manages a business transaction that spans multiple services by breaking it into a sequence of **local transactions**.

Each service completes its own transaction.

If something fails, previously completed transactions are **compensated**.

For example:

```text
Create Order
     ↓
Process Payment
     ↓
Reserve Inventory
     ↓
Ship Order
```

If everything succeeds:

```text
Order ✅
Payment ✅
Inventory ✅
Shipping ✅
```

But what if inventory reservation fails?

```text
Order ✅
Payment ✅
Inventory ❌
```

We need to undo the previous business operations.

So:

```text
Inventory Reservation ❌
        ↓
Refund Payment
        ↓
Cancel Order
```

These are called **compensating transactions**.

---

### 🔙 Compensation is NOT a database rollback

This is an important distinction.

With a traditional database transaction:

```text
BEGIN TRANSACTION

Update A
Update B
Update C

COMMIT
```

If something fails:

```text
ROLLBACK
```

The database can undo the changes atomically.

But with microservices:

```text
Order DB
    +
Payment DB
    +
Inventory DB
```

There may be **different databases owned by different services**.

A single traditional transaction across all of them is usually undesirable or impractical.

Saga instead says:

> "Let each service commit its own transaction, and define what business action should compensate it if a later step fails."

---

### 🏗️ Two common ways to implement Saga

There are two popular approaches:

**1️⃣ Choreography**

Services communicate through events.

```text
Order Service
     │
     │ OrderCreated
     ↓
Payment Service
     │
     │ PaymentCompleted
     ↓
Inventory Service
     │
     │ InventoryReserved
     ↓
Shipping Service
```

There is no central coordinator.

Each service listens for events and decides what to do next.

### 👍 Advantage

Less central orchestration.

### ⚠️ Challenge

As the system grows, the flow can become difficult to understand.

---

**2️⃣ Orchestration**

A central **Saga Orchestrator** controls the workflow.

```text
             Saga Orchestrator
             /      |       \
            ↓       ↓        ↓
        Order    Payment   Inventory
        Service   Service    Service
```

The orchestrator tells each service what to do.

For example:

```text
1. Create Order
2. Charge Payment
3. Reserve Inventory
4. Ship Order
```

If step 3 fails:

```text
Reserve Inventory ❌
        ↓
Orchestrator
        ↓
Refund Payment
        ↓
Cancel Order
```

### 👍 Advantage

The business workflow is easier to visualize and control.

### ⚠️ Challenge

The orchestrator becomes an important component that needs to be designed carefully.

---

### 💳 A Payment Example

Consider an e-commerce checkout:

```text
Customer
   ↓
Create Order
   ↓
Payment
   ↓
Reserve Inventory
   ↓
Confirm Order
```

Suppose payment succeeds:

```text
Payment → SUCCESS ✅
```

But inventory is unavailable:

```text
Inventory → FAILED ❌
```

We can't simply pretend nothing happened.

Instead:

```text
Inventory Failed
       ↓
Refund Payment
       ↓
Cancel Order
```

The final business state becomes:

```text
Order     → CANCELLED
Payment   → REFUNDED
Inventory → NOT RESERVED
```

This is **Saga + Compensation**.

---

### 🚨 Saga introduces a new challenge

A Saga does **not** give us the same immediate atomic consistency as a single database transaction.

There can be intermediate states.

For example:

```text
Payment → SUCCESS
Inventory → WAITING
Order → PROCESSING
```

For a short period, the system may be in an intermediate state.

This is why Saga is often associated with **eventual consistency**.

The goal is not:

> "Everything changes at exactly the same moment."

The goal is:

> "The system eventually reaches a valid business state."

---

### 🧠 Saga vs Traditional Transaction

```text
Traditional Transaction

A → B → C
     ↓
  COMMIT
     ↓
Everything succeeds together
```

Saga:

```text
A → B → C
        ↓
       FAIL
        ↓
Compensate B
        ↓
Compensate A
```

So instead of a technical rollback, we use **business-level compensation**.

---

### 🌍 Where is Saga useful?

Saga is particularly useful when a business workflow spans multiple independent services:

• 🛒 E-commerce orders
• 💳 Payments
• ✈️ Travel booking
• 🏨 Hotel reservations
• 🚚 Logistics
• 📦 Inventory management
• 💰 Financial workflows

---

### 🔗 The Bigger Distributed Systems Picture

Saga doesn't work in isolation.

In a real distributed system, we often combine it with:

```text
Saga
  +
Events
  +
Message Broker
  +
Idempotency
  +
Retries
  +
Outbox Pattern
  +
Dead Letter Queue
```

For example, if an event is delivered twice, **Idempotency** prevents duplicate processing.

If a service temporarily fails, **Retries** help.

If publishing an event must be reliable, the **Outbox Pattern** can help.

All these patterns start connecting together.

---

### ⭐ The key takeaway

A traditional transaction gives us:

> **Atomicity across operations.**

A Saga gives us:

> **A way to coordinate distributed business transactions using local transactions and compensating actions.**

And the most important idea to remember:

> **Saga doesn't roll back the past.
> It performs a new action to compensate for it.**

That's how distributed systems can coordinate complex workflows **without requiring one giant database transaction.**

---

🔗 **We Connect #19 — Saga Pattern**

In the next topic, we'll connect this with another important pattern:

**20. Outbox Pattern — How do we reliably publish an event after a database transaction?**

#SystemDesign #DistributedSystems #SagaPattern #Microservices #SoftwareArchitecture #EventDrivenArchitecture #BackendEngineering #Java #SystemDesignInterview #WeConnect
