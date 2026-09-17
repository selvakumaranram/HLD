Idempotency
How Payment Systems Avoid Duplicate Transactions

Imagine you are making an online payment.

You click Pay ₹5,000.

The payment request reaches the server, but the network connection suddenly breaks.

You don't know whether the payment succeeded.

So what do you do?

You click Pay again.

Without proper handling, the system might process:

₹5,000 + ₹5,000 = ₹10,000

You intended to pay once.

The system received the request twice.

This is where Idempotency becomes extremely important.

What is Idempotency?

Idempotency means that performing the same operation multiple times produces the same result as performing it once.

For example:

Request 1 → Create Payment → ₹5,000 charged
Request 2 → Same request → Existing result returned
Request 3 → Same request → Existing result returned

The customer is charged only once.

🚨 Why Do Duplicate Requests Happen?

In distributed systems, failures are normal.

For example:

Client
   ↓
Payment API
   ↓
Payment Service
   ↓
Bank

Suppose the Payment Service sends the transaction to the bank.

The bank successfully processes it.

But before the response reaches the Payment Service:

Network timeout ❌

Now the Payment Service doesn't know what happened.

It may retry.

Payment Service
      ↓
   Request #1
      ↓
     Bank
      ↓
Transaction SUCCESS
      X
   Response lost

The service thinks:

"Maybe the transaction failed."

So it sends:

Request #2
      ↓
     Bank

Without idempotency, the bank may process it again.

🆔 Enter the Idempotency Key

The client generates a unique identifier for the operation.

For example:

Idempotency-Key:
8f7a2c91-4b2d-4f8e-9c21-123456789abc

The client sends it with the payment request.

POST /payments

Idempotency-Key: 8f7a2c91-4b2d-4f8e-9c21-123456789abc

{
    "amount": 5000,
    "currency": "INR",
    "orderId": "ORDER-123"
}

The payment service stores this key along with the result.

Something like:

┌──────────────────────────────────────┐
│ Idempotency Store                    │
├──────────────────┬───────────────────┤
│ Key              │ Result            │
├──────────────────┼───────────────────┤
│ 8f7a2c91...      │ Payment SUCCESS   │
└──────────────────┴───────────────────┘

When the same request arrives again:

Same Idempotency-Key
        ↓
   Already exists?
        ↓
       YES
        ↓
Return previous result

The payment is not processed again.

🔄 What Happens During a Retry?

Let's follow the complete flow.

First request
Client
  │
  │ Payment ₹5,000
  │ Key = ABC123
  ↓
Payment Service
  │
  │ Check ABC123
  │ Not found
  ↓
Process Payment
  │
  ↓
Bank
  │
  │ SUCCESS
  ↓
Store ABC123 → SUCCESS
  │
  ↓
Client

Now imagine the response is lost.

The client retries.

Client
  │
  │ Payment ₹5,000
  │ Key = ABC123
  ↓
Payment Service
  │
  │ Check ABC123
  │
  │ Already processed!
  ↓
Return previous result
  │
  ↓
SUCCESS

No second transaction.

🧠 Idempotency + Retries

This is one of the most important connections in distributed systems.

Retries without idempotency can be dangerous.

Failure
  ↓
Retry
  ↓
Failure
  ↓
Retry
  ↓
Retry
  ↓
Duplicate operation 😱

But:

Failure
  ↓
Retry
  ↓
Idempotency Check
  ↓
Already processed
  ↓
Return existing result

This makes retries much safer.

💳 Where Is Idempotency Used?

It's not only for payments.

It is useful whenever an operation should happen once, even if the request is repeated.

Examples:

Payment processing
Order creation
Money transfers
Ticket booking
Subscription creation
Account creation
Sending certain notifications
Inventory reservations
Payment refunds

For example:

POST /orders

Idempotency-Key: ORDER-ABC123

If the client sends the request five times:

Request 1 → Create order
Request 2 → Return existing order
Request 3 → Return existing order
Request 4 → Return existing order
Request 5 → Return existing order

Only one order is created.

⚠️ Is POST Normally Idempotent?

HTTP semantics matter here.

Generally:

GET     → Idempotent
PUT     → Idempotent
DELETE  → Idempotent
POST    → Not necessarily idempotent

For example:

POST /orders

could create a new order every time.

So applications often add their own idempotency mechanism.

🏗️ How Would We Implement It?

A simplified implementation could look like:

Client
   ↓
API Gateway
   ↓
Payment Service
   ↓
Idempotency Store
   ↓
Payment Processor

The service does:

1. Receive request
2. Read Idempotency-Key
3. Check key
4. If key exists → return stored result
5. If key doesn't exist → process transaction
6. Store result
7. Return response

The important part is that checking and creating the idempotency record must itself be safe under concurrency.

Because two identical requests can arrive at exactly the same time.

Request A ──┐
            ├──→ Payment Service
Request B ──┘

Both could ask:

"Does ABC123 exist?"

Both might receive:

NO

And both could process the payment.

So we need atomic operations, such as a database unique constraint, conditional insert, or another concurrency-safe mechanism.

🔥 The Real Distributed-System Problem

Idempotency becomes especially interesting when multiple services are involved.

Order Service
     ↓
Payment Service
     ↓
Bank
     ↓
Notification Service

What happens if:

Payment succeeds
       ↓
Order update fails

Or:

Payment succeeds
       ↓
Notification fails
       ↓
Retry notification

This is where idempotency works together with other distributed-system patterns:

Idempotency
     +
Retries
     +
Timeouts
     +
Circuit Breaker
     +
Message Queues
     +
Outbox Pattern
     +
Distributed Transactions

They are all connected.

🎯 The Key Takeaway

A distributed system must assume:

Requests can be duplicated. Responses can be lost. Networks can fail.

Therefore, don't design your system assuming:

Request → One execution → One response

Design for:

Request
   ↓
Maybe processed
   ↓
Maybe response lost
   ↓
Retry
   ↓
Same request again
   ↓
Idempotency
   ↓
Same result
One line to remember:

Retries make systems resilient; idempotency makes those retries safe.

That is why Idempotency is one of the most important concepts behind reliable payment and distributed systems.