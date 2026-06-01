---
layout: post
title: "Idempotency — A Deep Dive for Engineers"
date: 2026-01-01
categories: [system-design, backend, distributed-systems]
tags: [idempotency, api-design, redis, kafka, outbox-pattern, distributed-systems]
description: "From the first double-charge bug to Kafka exactly-once semantics — understand idempotency from first principles and never fear retries again."
---

# Idempotency — A Deep Dive for Engineers

There is a specific kind of bug that shows up in production systems and causes genuine panic in engineering teams. A user gets charged twice. Or an email goes out twice to every customer. Or an order is created twice in the database. And the support tickets start coming in.

These bugs almost always share the same root cause — someone assumed that a failed HTTP request meant the operation never happened. It usually did.

---

## Start with the Problem

You are building a payment API. A user taps "Pay ₹10,000." Your mobile app sends a `POST /payments` request to your server. The server processes the payment, charges the card, and is about to respond with `200 OK`. At that exact moment — the network drops.

```
Client  →  POST /payments  →  Server
                               ↓
                           payment processed
                           card charged ✓
                               ↓
                           200 OK  →  ✗ network drops
```

The client never receives the response. From the client's perspective, the request timed out. Now the client has to make a decision.

**Choice 1: Give up.** The user sees "payment failed." But the payment actually went through. The user calls support, confused and frustrated. You now have a support incident for a payment that succeeded.

**Choice 2: Retry.** The client sends `POST /payments` again. The server processes it again. The card gets charged a second time. You now have a double-charge incident and a furious user.

Both choices are wrong. This is the core problem idempotency was designed to solve — and once you understand this problem deeply, everything else about idempotency becomes obvious.

---

## What Idempotency Actually Means

Idempotency means: **performing the same operation multiple times produces the same result as performing it once.**

In mathematical terms: `f(f(x)) = f(x)`

In API terms: sending the same request N times has the same outcome as sending it once. No duplicate side effects. No duplicate charges. No duplicate records.

The key word in that definition is **side effects**. An idempotent operation may return different responses at different times — but it must produce the same *state change in the world*. This distinction matters enormously and is where most engineers first get confused.

Consider a `DELETE /orders/123` request. Call it once — order deleted, response is `200 OK`. Call it again — order is already gone, response is `404 Not Found`. Two different responses. But the state of the world is identical in both cases: the order does not exist. That makes DELETE idempotent. The definition is about state, not response codes.

---

## HTTP Methods and Idempotency

Before talking about solutions, understand which HTTP methods are already idempotent and which are not. There is also a related concept worth knowing: **safety**. A safe method has no side effects at all — it only reads data and never modifies state. All safe methods are also idempotent, but not all idempotent methods are safe.

| Method | Safe | Idempotent | Why |
|---|---|---|---|
| GET | ✓ | ✓ | Reads only, no side effects |
| HEAD | ✓ | ✓ | Same as GET, no body returned |
| OPTIONS | ✓ | ✓ | Metadata only |
| DELETE | ✗ | ✓ | First call deletes; repeat calls find it already gone — same final state |
| PUT | ✗ | ✓ | Replaces resource entirely; calling twice = same final state |
| PATCH | ✗ | Sometimes | Depends entirely on the operation (see below) |
| POST | ✗ | ✗ | Creates a new resource each time by default |

**The PATCH nuance deserves special attention.** Whether PATCH is idempotent depends entirely on what the operation does:

```
PATCH /accounts/123  { "balance": 5000 }    → idempotent
                                               Sets balance TO 5000.
                                               Repeat = same final state.

PATCH /accounts/123  { "balance": "+500" }  → NOT idempotent
                                               Adds 500 each time.
                                               Repeat = different final state.
```

This is why API design matters. `SET` operations are idempotent. `INCREMENT` operations are not.

POST is the real challenge. You need POST for payments, order creation, email sending — all operations where duplicates are unacceptable. POST is not idempotent by default, but the business requires it to behave as if it is. This is where the idempotency key pattern comes in.

---

## The Idempotency Key Pattern

The solution is conceptually simple: the client generates a unique identifier — typically a UUID — before sending the first request. This identifier is sent as a request header. The server uses this identifier to detect and deduplicate retries.

```http
POST /payments HTTP/1.1
Idempotency-Key: a3f8b2c1-9d4e-4f7a-b6c2-1e8d3f5a9b2c
Content-Type: application/json

{
  "amount": 10000,
  "currency": "INR",
  "card_token": "tok_visa_4242"
}
```

The server behavior works like this:

1. A request arrives with an `Idempotency-Key` header.
2. The server checks its store (typically Redis) for this key.
3. **If key is found with status `completed`**: return the stored response immediately. Do not process again.
4. **If key is found with status `in_progress`**: return `409 Conflict`. Another request with this key is already being processed.
5. **If key is not found**: process the operation, store the result, return the response.

Now the client can retry safely any number of times. The payment only ever happens once. The key insight is that **the idempotency key must be generated by the client, before the first request is sent.** If the server generated it and returned it in the response — and that response was lost — the client would have no key and could not safely retry. The key must survive network failures, which means it must exist before any network activity begins.

---

## What to Store in Redis

When a request is processed, the server stores a record in Redis keyed by the idempotency key:

```json
{
  "idempotency_key": "a3f8b2c1-9d4e-4f7a-b6c2-1e8d3f5a9b2c",
  "request_hash": "sha256 of request body",
  "status": "completed",
  "response_status_code": 200,
  "response_body": { "payment_id": "pay_xyz", "status": "charged" },
  "created_at": "2024-01-15T10:30:00Z",
  "completed_at": "2024-01-15T10:30:01Z",
  "expires_at": "2024-01-16T10:30:00Z"
}
```

Two fields here deserve particular attention.

The `status: in_progress` field is set immediately when the lock is acquired, before any actual processing begins. If a second request arrives for the same key while processing is ongoing, it reads `in_progress` and returns `409` immediately — without ever competing for the lock or waiting for the first request to finish.

The `request_hash` stores a SHA-256 hash of the original request body. If someone sends the same idempotency key with a different request body — say, the same key but with `amount: 5000` instead of `amount: 10000` — the server detects the mismatch and returns `422 Unprocessable Entity`. This prevents a class of attacks where someone tries to reuse an existing idempotency key with a different payload to get a different operation processed.

The TTL is typically 24 hours (Stripe's standard) or up to 7 days for stricter financial systems.

---

## The Two-Layer Check — Fast Path and Lock Path

Here is where the implementation gets interesting. A naive implementation would acquire a distributed lock for every request and check Redis inside the lock. This works but is unnecessarily slow for the common case — returning a cached result for a completed operation.

The correct implementation uses two layers:

```
Request arrives
     ↓
READ Redis (no lock required)
     ↓
┌─────────────────────────────────────┐
│ in_progress? → 409 immediately      │
│ completed?   → return stored result │
│ not found?   → proceed to lock      │
└─────────────────────────────────────┘
     ↓ (only if not found)
Acquire distributed lock
     ↓
READ Redis AGAIN inside lock  ← double-checked locking
     ↓
┌─────────────────────────────────────┐
│ still not found? → process → write  │
│ found now?       → return result    │
└─────────────────────────────────────┘
```

The first read — outside the lock — is an optimistic fast path. The overwhelming majority of retry requests will hit a `completed` entry and return immediately without touching any lock. The lock only comes into play for genuinely new operations.

The second read — inside the lock — is called **double-checked locking**. Between the first read returning "not found" and the lock being acquired, another thread may have completed the operation. You must check once more after acquiring the lock. This is not paranoia; it is the correct handling of a real race condition.

---

## The Race Condition That Locks Prevent

Without the distributed lock, two concurrent requests with the same key can both slip through:

```
Time 0ms: Request A arrives with key abc-123
Time 1ms: Request B arrives with key abc-123

Thread A: reads Redis → key not found
Thread B: reads Redis → key not found  (A hasn't written yet)
Thread A: processes payment → charges card
Thread B: processes payment → charges card AGAIN ← double charge
```

Both threads saw "not found" before either one wrote. The lock prevents this:

```
Thread A: SET lock:abc-123 1 NX EX 30  → lock acquired ✓
Thread B: SET lock:abc-123 1 NX EX 30  → failed, waits

Thread A: processes, writes result, releases lock
Thread B: acquires lock, reads Redis → found, returns stored result
Thread B: never processes the payment
```

`NX` (only set if the key does not exist) is atomic in Redis. Only one thread wins the lock. This atomicity is guaranteed by Redis's single-threaded execution model — there is no race between the check and the set.

---

## The Lock Expiry Problem

There is one more scenario that trips up even experienced engineers: what happens when processing takes longer than the lock's TTL?

```
Thread A: acquires lock with EX 30 (expires in 30 seconds)
          starts processing payment
          external card network is slow...

30 seconds pass → Redis automatically expires the lock

Thread B: tries to acquire lock → lock is gone (expired)
          acquires it successfully
          starts processing same payment

Thread A: finishes, writes result
Thread B: finishes, overwrites result
→ Payment charged twice
```

This is a genuine production failure mode. Three solutions exist:

**Option 1 — Fencing Tokens (most correct).** When a lock is acquired, the store returns a monotonically increasing token. Every write operation must include this token. The store rejects writes from outdated token holders.

```
Thread A acquires lock → token: 42
Thread B acquires lock → token: 43  (after expiry)

Thread A tries to write with token 42
Store sees current token is 43 → rejects Thread A's write ✓
```

**Option 2 — Heartbeat.** Thread A sends a heartbeat every few seconds to extend the lock TTL while processing. If the process crashes, the heartbeat stops, and the lock expires naturally.

**Option 3 — Conservative TTL.** Set the TTL to something like 10 minutes for payment operations. Simple and pragmatic. Most teams start here, then add fencing tokens after an actual incident.

---

## HTTP Status Codes in Context

The idempotency pattern introduces specific semantics for a few status codes that are worth understanding precisely.

**409 Conflict** — In the idempotency context, this means "A request with this key is already being processed. Do not send another one. Wait and retry after a delay." This is not an error. It is the server protecting against race conditions at the HTTP layer.

**422 Unprocessable Entity** — Often confused with 400. The distinction:
- `400 Bad Request` — malformed JSON, missing required field. The server cannot even parse the request.
- `422 Unprocessable Entity` — the server understood the request perfectly, but the data violates a business rule or is semantically incorrect. In the idempotency context: the same key was sent with a different request body.

**Retry behavior by status code:**

| Status Code | Action |
|---|---|
| 409 | Retry after delay — timing conflict, will resolve |
| 422 | Do NOT retry — semantic violation, will never resolve |
| 429 | Retry after delay — rate limited |
| 500 | Retry — server error, may resolve |
| 503 | Retry — service unavailable, temporary |

---

## Idempotency vs Exactly-Once Semantics

These two terms are frequently used interchangeably. They are different guarantees.

**Idempotency** means the operation can be retried safely. The server handles deduplication. The client sends; the server detects duplicates.

**Exactly-once delivery** means an operation is guaranteed to execute precisely one time, end to end. This is a much stronger guarantee — and fundamentally harder to achieve, because of the Two Generals Problem. Two systems can never be completely certain they agree on whether something happened, because the acknowledgment message itself can always be lost.

In practice, most production systems settle for **at-least-once delivery with idempotent consumers**:

- The producer sends events and retries until acknowledged. It may send duplicate events if an acknowledgment is lost.
- The consumer processes events idempotently. It deduplicates using an event ID. It is safe to receive the same event multiple times.

The combination gives you the practical equivalent of exactly-once processing, without the theoretical impossibility.

---

## Kafka and the Exactly-Once Question

Kafka, by default, gives you at-least-once delivery:

```
Producer sends message → Kafka acknowledges → stored ✓

Producer sends message → no acknowledgment (network blip)
Producer retries → message stored TWICE
Consumer receives it twice
```

Kafka transactions enable exactly-once semantics (EOS) at the broker level, using a producer ID plus sequence number for deduplication. The consumer only sees committed messages.

The tradeoff is real: EOS adds roughly 20-30% throughput reduction and 5-10ms additional latency per message due to transaction coordination and two-phase commit between producer and broker. At-least-once has essentially no overhead.

**Practical guidance**: Most production systems use at-least-once Kafka with idempotent consumers. Pay the EOS overhead only when the downstream system genuinely cannot be made idempotent, or when regulatory requirements mandate it. There is a third option — the Outbox Pattern — that often makes the question moot entirely.

---

## The Outbox Pattern — Solving the Dual Write Problem

Consider a typical order placement scenario:

```
1. Save order to database     (OrderService DB)
2. Publish "order created" event  (Kafka)
```

You need both to happen atomically — or neither. But these are two separate systems. You cannot wrap them in a single ACID transaction.

Without a pattern to handle this, you get exactly two failure modes:

- **Scenario A**: Save to DB succeeds, publish to Kafka fails (Kafka is down). Order exists in DB. Downstream services — inventory, fulfillment, notifications — never find out.
- **Scenario B**: Save to DB succeeds, publish to Kafka succeeds, app crashes before confirming. On retry: DB save creates a duplicate order, Kafka publish creates a duplicate event.

The Outbox Pattern solves this by treating the database itself as the reliable message bus:

```
Step 1: Single DB transaction
        → Write order to orders table
        → Write event to outbox table (same DB, same transaction)
        Either both commit or neither does.

Step 2: Outbox worker (runs continuously)
        → Reads unpublished events from outbox table
        → Publishes to Kafka
        → Marks event as published

Step 3: Kafka consumer (idempotent)
        → Checks event_id before processing
        → Already seen? Skip.
        → Not seen? Process and mark seen.
```

The outbox worker may publish the same event twice — if it crashes after publishing but before marking as published. This is why the consumer must be idempotent. The combination gives you effectively-exactly-once processing without Kafka transaction overhead.

---

## Propagating Idempotency Across Service Boundaries

Your API receives a request with an idempotency key and calls multiple downstream services. Each downstream call must also be idempotent — and the keys must be derived deterministically from the parent key:

```
Client → POST /checkout  (key: abc-123)
              ↓
         OrderService.createOrder    (key: abc-123:order)
              ↓
         PaymentService.charge       (key: abc-123:payment)
              ↓
         NotificationService.notify  (key: abc-123:notify)
```

Each service independently checks its own Redis store using its derived key. If the checkout handler retries, all downstream services see the same derived keys and deduplicate independently. No service needs to know that it is participating in a retry — the derived key carries that information.

---

## A Full End-to-End Example — Food Delivery

User taps "Place Order." Request reaches the server. Order is saved, payment is charged, restaurant is notified. The response never reaches the mobile app. The app retries.

**One important detail first:** the operation order matters for business correctness. You must charge the payment before notifying the restaurant. Never tell a restaurant to start cooking until money is confirmed. If payment fails after notifying the restaurant, you now have a restaurant preparing food for a cancelled order — a complex and expensive compensation flow.

The correct order: save order → charge payment → notify restaurant.

Here is the full idempotent flow:

```
Step 1: Mobile client generates UUID, persists it locally.
        Sends: POST /checkout  Idempotency-Key: abc-123

Step 2: HTTP layer reads Redis (no lock).
        completed   → return stored response immediately
        in_progress → 409, client waits and retries
        not found   → proceed to lock

Step 3: Acquire Redis lock.
        SET lock:abc-123 1 NX EX 300
        Double-check Redis inside lock.
        Set status = in_progress in Redis.

Step 4: OrderService.createOrder (key: abc-123:order)
        Checks own Redis.
        Not found → creates order, stores result.
        Found     → returns existing order ID, skips creation.

Step 5: PaymentService.charge (key: abc-123:payment)
        Checks own Redis.
        Not found → charges payment.
                    In single DB transaction:
                      UPDATE payments SET status = charged
                      INSERT INTO outbox (event: payment_confirmed)
        Found     → returns existing payment result, skips charge.

Step 6: Outbox worker publishes payment_confirmed to Kafka.
        Kafka consumer deduplicates by event_id.

Step 7: NotificationService (key: abc-123:notify)
        Receives Kafka event.
        Checks own Redis.
        Not found → sends restaurant notification, stores result.
        Found     → skips, restaurant already knows.

Step 8: Write final result to Redis.
        status = completed
        response body = { order_id, payment_id, status }
        Release lock.

Step 9: Return response to mobile client.
        All future retries return this stored response immediately.
```

On any retry, at any point in this flow, every layer independently deduplicates. No duplicate order. No duplicate charge. No duplicate restaurant notification.

---

## How Stripe and Razorpay Implement This

**Stripe** uses exactly the pattern described above. The `Idempotency-Key` header is client-generated, any string up to 255 characters (UUID recommended), scoped per API key and per endpoint. The same key on different endpoints is treated as different operations. TTL is 24 hours. Concurrent requests with the same in-flight key return `409 Conflict`.

**Razorpay** adds one important refinement: it ties the idempotency key to a hash of the request body. If you send the same key with a different request body — same key, different amount — it returns `422`. This closes a potential attack vector where someone reuses a key with a modified payload.

---

## Quick Reference

**Idempotent HTTP methods**: GET, HEAD, OPTIONS, PUT, DELETE

**Not idempotent by default**: POST, PATCH (depends on whether the operation is a SET or an INCREMENT)

**Making POST idempotent**: Client-generated UUID key → Redis store with TTL → `SET NX EX` distributed lock → double-checked locking inside lock

**Status codes**:
- `200` with stored response → completed, cached result returned
- `409 Conflict` → in-progress, wait and retry
- `422 Unprocessable` → same key, different body — semantic violation, do not retry

**Dual write solution**: Outbox Pattern — DB transaction writes to both orders table and outbox table atomically; outbox worker publishes to Kafka; Kafka consumer deduplicates by event ID

**Cross-service propagation**: Derived child keys (`parent-key:service-name`); each service deduplicates independently

**Key ownership**: Always client-generated, before first request, persisted across retries

---

## Architecture and Flow Diagrams

<style>
.arch-diagram-container {
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  background: #0f1117;
  border-radius: 12px;
  padding: 2rem;
  margin: 2rem 0;
  overflow-x: auto;
  border: 1px solid #2a2d3e;
}

.arch-diagram-container h3 {
  color: #7dd3fc;
  font-size: 0.85rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin-bottom: 1.5rem;
  border-bottom: 1px solid #2a2d3e;
  padding-bottom: 0.75rem;
}

.flow-svg {
  width: 100%;
  overflow: visible;
}

.quick-ref-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.25rem;
  margin: 2rem 0;
}

.qr-card {
  background: #0f1117;
  border: 1px solid #2a2d3e;
  border-radius: 10px;
  padding: 1.25rem 1.5rem;
  position: relative;
  overflow: hidden;
}

.qr-card::before {
  content: '';
  position: absolute;
  top: 0; left: 0; right: 0;
  height: 3px;
  background: var(--card-accent, #7dd3fc);
  border-radius: 10px 10px 0 0;
}

.qr-card h4 {
  color: #e2e8f0;
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin: 0 0 1rem 0;
  font-family: 'JetBrains Mono', monospace;
}

.qr-card ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.qr-card ul li {
  font-size: 0.82rem;
  color: #94a3b8;
  padding: 0.35rem 0;
  border-bottom: 1px solid #1e2130;
  font-family: 'JetBrains Mono', monospace;
  display: flex;
  align-items: flex-start;
  gap: 0.5rem;
}

.qr-card ul li:last-child { border-bottom: none; }

.tag {
  display: inline-block;
  padding: 0.2rem 0.6rem;
  border-radius: 4px;
  font-size: 0.72rem;
  font-weight: 600;
  font-family: 'JetBrains Mono', monospace;
  white-space: nowrap;
}

.tag-green { background: #052e16; color: #4ade80; }
.tag-red { background: #2d0c0c; color: #f87171; }
.tag-yellow { background: #2d1f00; color: #fbbf24; }
.tag-blue { background: #0c1a2d; color: #7dd3fc; }
.tag-purple { background: #1a0c2d; color: #c084fc; }

.status-table {
  width: 100%;
  border-collapse: collapse;
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.82rem;
  margin: 1rem 0;
}

.status-table th {
  background: #1e2130;
  color: #7dd3fc;
  padding: 0.6rem 1rem;
  text-align: left;
  font-weight: 600;
  text-transform: uppercase;
  font-size: 0.72rem;
  letter-spacing: 0.05em;
}

.status-table td {
  padding: 0.6rem 1rem;
  border-bottom: 1px solid #1e2130;
  color: #94a3b8;
  vertical-align: middle;
}

.status-table tr:hover td {
  background: #1a1d2e;
}

.flow-step {
  display: flex;
  align-items: flex-start;
  gap: 1rem;
  padding: 0.75rem 0;
  border-bottom: 1px solid #1e2130;
}

.flow-step:last-child { border-bottom: none; }

.step-num {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  font-weight: 700;
  flex-shrink: 0;
  margin-top: 2px;
}

.step-content {
  flex: 1;
}

.step-title {
  color: #e2e8f0;
  font-size: 0.82rem;
  font-weight: 600;
  margin-bottom: 0.2rem;
}

.step-desc {
  color: #64748b;
  font-size: 0.78rem;
  line-height: 1.5;
}

.diagram-wrapper {
  background: #0f1117;
  border-radius: 12px;
  padding: 2rem;
  margin: 2rem 0;
  border: 1px solid #2a2d3e;
  overflow-x: auto;
}

.diagram-wrapper h3 {
  color: #7dd3fc;
  font-size: 0.85rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin-bottom: 1.5rem;
  padding-bottom: 0.75rem;
  border-bottom: 1px solid #2a2d3e;
  font-family: 'JetBrains Mono', monospace;
}
</style>

<div class="diagram-wrapper">
<h3>Full Architecture — End-to-End Idempotent Order Flow</h3>

<svg viewBox="0 0 900 620" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:900px;display:block;margin:0 auto;">
  <defs>
    <marker id="arrow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#475569"/>
    </marker>
    <marker id="arrow-blue" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#7dd3fc"/>
    </marker>
    <marker id="arrow-green" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#4ade80"/>
    </marker>
    <marker id="arrow-yellow" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#fbbf24"/>
    </marker>
  </defs>

  <!-- Background -->
  <rect width="900" height="620" fill="#0f1117" rx="8"/>

  <!-- Mobile Client -->
  <rect x="30" y="40" width="110" height="60" rx="8" fill="#1e2130" stroke="#7dd3fc" stroke-width="1.5"/>
  <text x="85" y="66" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="11" font-weight="600">Mobile</text>
  <text x="85" y="82" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="11" font-weight="600">Client</text>
  <text x="85" y="115" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">UUID key persisted</text>

  <!-- Arrow client to HTTP layer -->
  <line x1="140" y1="70" x2="200" y2="70" stroke="#7dd3fc" stroke-width="1.5" marker-end="url(#arrow-blue)"/>
  <text x="170" y="62" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="9">POST</text>
  <text x="170" y="54" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="9">+ Idempotency-Key</text>

  <!-- HTTP Layer / API Gateway -->
  <rect x="200" y="30" width="130" height="80" rx="8" fill="#1e2130" stroke="#c084fc" stroke-width="1.5"/>
  <text x="265" y="60" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="11" font-weight="600">HTTP Layer</text>
  <text x="265" y="76" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="11" font-weight="600">/ API Gateway</text>
  <text x="265" y="95" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">reads Redis first</text>

  <!-- Redis -->
  <rect x="200" y="170" width="130" height="60" rx="8" fill="#1e2130" stroke="#f87171" stroke-width="1.5"/>
  <text x="265" y="196" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="11" font-weight="600">Redis Store</text>
  <text x="265" y="212" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">key → {status, response}</text>
  <text x="265" y="224" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">TTL: 24h</text>

  <!-- Arrow HTTP to Redis -->
  <line x1="265" y1="110" x2="265" y2="168" stroke="#f87171" stroke-width="1.5" stroke-dasharray="5,3" marker-end="url(#arrow-yellow)"/>
  <text x="285" y="145" fill="#f87171" font-family="monospace" font-size="9">fast read</text>

  <!-- Lock box -->
  <rect x="200" y="270" width="130" height="60" rx="8" fill="#1e2130" stroke="#fbbf24" stroke-width="1.5"/>
  <text x="265" y="296" text-anchor="middle" fill="#fbbf24" font-family="monospace" font-size="11" font-weight="600">Distributed</text>
  <text x="265" y="312" text-anchor="middle" fill="#fbbf24" font-family="monospace" font-size="11" font-weight="600">Lock (SET NX)</text>

  <!-- Arrow Redis to Lock -->
  <line x1="265" y1="232" x2="265" y2="268" stroke="#fbbf24" stroke-width="1.5" marker-end="url(#arrow-yellow)"/>
  <text x="285" y="254" fill="#fbbf24" font-family="monospace" font-size="9">not found</text>

  <!-- Order Service -->
  <rect x="400" y="30" width="120" height="60" rx="8" fill="#1e2130" stroke="#4ade80" stroke-width="1.5"/>
  <text x="460" y="56" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="11" font-weight="600">Order</text>
  <text x="460" y="72" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="11" font-weight="600">Service</text>

  <!-- Payment Service -->
  <rect x="400" y="150" width="120" height="60" rx="8" fill="#1e2130" stroke="#4ade80" stroke-width="1.5"/>
  <text x="460" y="176" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="11" font-weight="600">Payment</text>
  <text x="460" y="192" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="11" font-weight="600">Service</text>

  <!-- Notification Service -->
  <rect x="400" y="270" width="120" height="60" rx="8" fill="#1e2130" stroke="#4ade80" stroke-width="1.5"/>
  <text x="460" y="296" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="11" font-weight="600">Notification</text>
  <text x="460" y="312" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="11" font-weight="600">Service</text>

  <!-- Arrows HTTP to services -->
  <line x1="330" y1="60" x2="398" y2="60" stroke="#4ade80" stroke-width="1.5" marker-end="url(#arrow-green)"/>
  <text x="363" y="53" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="8">:order</text>
  <line x1="330" y1="70" x2="398" y2="180" stroke="#4ade80" stroke-width="1.5" marker-end="url(#arrow-green)"/>
  <text x="363" y="128" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="8">:payment</text>
  <line x1="330" y1="80" x2="398" y2="300" stroke="#4ade80" stroke-width="1.5" marker-end="url(#arrow-green)"/>
  <text x="335" y="195" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="8">:notify</text>

  <!-- DB -->
  <rect x="550" y="130" width="100" height="55" rx="8" fill="#1e2130" stroke="#94a3b8" stroke-width="1.5"/>
  <text x="600" y="156" text-anchor="middle" fill="#94a3b8" font-family="monospace" font-size="11" font-weight="600">Database</text>
  <text x="600" y="172" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">orders + outbox</text>

  <!-- Outbox Worker -->
  <rect x="550" y="250" width="100" height="55" rx="8" fill="#1e2130" stroke="#fbbf24" stroke-width="1.5"/>
  <text x="600" y="274" text-anchor="middle" fill="#fbbf24" font-family="monospace" font-size="10" font-weight="600">Outbox</text>
  <text x="600" y="290" text-anchor="middle" fill="#fbbf24" font-family="monospace" font-size="10" font-weight="600">Worker</text>

  <!-- Payment to DB -->
  <line x1="520" y1="180" x2="548" y2="170" stroke="#94a3b8" stroke-width="1.5" marker-end="url(#arrow)"/>
  <text x="535" y="163" fill="#64748b" font-family="monospace" font-size="8">tx</text>

  <!-- DB to Outbox Worker -->
  <line x1="600" y1="185" x2="600" y2="248" stroke="#fbbf24" stroke-width="1.5" stroke-dasharray="4,3" marker-end="url(#arrow-yellow)"/>
  <text x="615" y="222" fill="#fbbf24" font-family="monospace" font-size="8">poll</text>

  <!-- Kafka -->
  <rect x="680" y="230" width="100" height="55" rx="8" fill="#1e2130" stroke="#7dd3fc" stroke-width="1.5"/>
  <text x="730" y="256" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="11" font-weight="600">Kafka</text>
  <text x="730" y="272" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">at-least-once</text>

  <!-- Outbox to Kafka -->
  <line x1="650" y1="272" x2="678" y2="262" stroke="#7dd3fc" stroke-width="1.5" marker-end="url(#arrow-blue)"/>
  <text x="663" y="258" fill="#7dd3fc" font-family="monospace" font-size="8">publish</text>

  <!-- Kafka to Notification -->
  <line x1="730" y1="285" x2="730" y2="350" stroke="#7dd3fc" stroke-width="1.5"/>
  <line x1="730" y1="350" x2="522" y2="350" stroke="#7dd3fc" stroke-width="1.5"/>
  <line x1="522" y1="350" x2="522" y2="315" stroke="#7dd3fc" stroke-width="1.5" marker-end="url(#arrow-blue)"/>
  <text x="626" y="365" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="8">event consumed</text>

  <!-- Per-service Redis stores (small) -->
  <rect x="400" y="90" width="70" height="30" rx="4" fill="#1a0c0c" stroke="#f87171" stroke-width="1"/>
  <text x="435" y="109" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="8">Redis :order</text>

  <rect x="400" y="210" width="70" height="30" rx="4" fill="#1a0c0c" stroke="#f87171" stroke-width="1"/>
  <text x="435" y="229" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="8">Redis :pay</text>

  <rect x="400" y="330" width="70" height="30" rx="4" fill="#1a0c0c" stroke="#f87171" stroke-width="1"/>
  <text x="435" y="349" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="8">Redis :notify</text>

  <!-- Service to own Redis -->
  <line x1="460" y1="90" x2="450" y2="90" stroke="#f87171" stroke-width="1" stroke-dasharray="3,2"/>
  <line x1="460" y1="210" x2="450" y2="210" stroke="#f87171" stroke-width="1" stroke-dasharray="3,2"/>
  <line x1="460" y1="330" x2="450" y2="330" stroke="#f87171" stroke-width="1" stroke-dasharray="3,2"/>

  <!-- Legend -->
  <rect x="30" y="400" width="840" height="190" rx="8" fill="#0a0c14" stroke="#2a2d3e" stroke-width="1"/>
  <text x="50" y="425" fill="#7dd3fc" font-family="monospace" font-size="10" font-weight="600" text-transform="uppercase">LEGEND &amp; DECISION FLOW</text>

  <!-- Decision flow: Redis check outcomes -->
  <text x="50" y="450" fill="#94a3b8" font-family="monospace" font-size="10" font-weight="600">Redis Check Outcomes (HTTP Layer)</text>

  <rect x="50" y="460" width="110" height="28" rx="4" fill="#052e16" stroke="#4ade80" stroke-width="1"/>
  <text x="105" y="479" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="9">completed → 200</text>

  <rect x="175" y="460" width="120" height="28" rx="4" fill="#2d0c0c" stroke="#f87171" stroke-width="1"/>
  <text x="235" y="479" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="9">in_progress → 409</text>

  <rect x="310" y="460" width="130" height="28" rx="4" fill="#1a0c2d" stroke="#c084fc" stroke-width="1"/>
  <text x="375" y="479" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="9">not found → acquire lock</text>

  <!-- Retry guidance -->
  <text x="50" y="520" fill="#94a3b8" font-family="monospace" font-size="10" font-weight="600">Client Retry Guide</text>
  <text x="50" y="540" fill="#4ade80" font-family="monospace" font-size="9">200 → done, stop retrying</text>
  <text x="50" y="556" fill="#f87171" font-family="monospace" font-size="9">409 → wait + retry (will resolve)</text>
  <text x="50" y="572" fill="#f87171" font-family="monospace" font-size="9">422 → never retry (semantic violation)</text>

  <text x="280" y="520" fill="#94a3b8" font-family="monospace" font-size="10" font-weight="600">Derived Key Pattern</text>
  <text x="280" y="540" fill="#64748b" font-family="monospace" font-size="9">parent key: abc-123</text>
  <text x="280" y="556" fill="#7dd3fc" font-family="monospace" font-size="9">→ abc-123:order</text>
  <text x="280" y="572" fill="#7dd3fc" font-family="monospace" font-size="9">→ abc-123:payment</text>
  <text x="280" y="588" fill="#7dd3fc" font-family="monospace" font-size="9">→ abc-123:notify</text>

  <text x="500" y="520" fill="#94a3b8" font-family="monospace" font-size="10" font-weight="600">Outbox Pattern</text>
  <text x="500" y="540" fill="#64748b" font-family="monospace" font-size="9">1. DB txn: orders + outbox (atomic)</text>
  <text x="500" y="556" fill="#fbbf24" font-family="monospace" font-size="9">2. Worker: poll → publish to Kafka</text>
  <text x="500" y="572" fill="#7dd3fc" font-family="monospace" font-size="9">3. Consumer: dedupe by event_id</text>
  <text x="500" y="588" fill="#4ade80" font-family="monospace" font-size="9">Result: effectively exactly-once</text>
</svg>
</div>

<div class="diagram-wrapper">
<h3>Two-Layer Check — Decision Flow</h3>

<svg viewBox="0 0 700 400" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:700px;display:block;margin:0 auto;">
  <defs>
    <marker id="arr2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#475569"/>
    </marker>
    <marker id="arr-b" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#7dd3fc"/>
    </marker>
    <marker id="arr-g" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#4ade80"/>
    </marker>
    <marker id="arr-r" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#f87171"/>
    </marker>
    <marker id="arr-y" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto">
      <path d="M0,0 L0,6 L8,3 z" fill="#fbbf24"/>
    </marker>
  </defs>
  <rect width="700" height="400" fill="#0f1117" rx="8"/>

  <!-- Start -->
  <rect x="280" y="20" width="140" height="40" rx="6" fill="#1e2130" stroke="#7dd3fc" stroke-width="1.5"/>
  <text x="350" y="45" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="11" font-weight="600">Request Arrives</text>

  <!-- Read Redis -->
  <line x1="350" y1="60" x2="350" y2="90" stroke="#7dd3fc" stroke-width="1.5" marker-end="url(#arr-b)"/>
  <rect x="250" y="90" width="200" height="40" rx="6" fill="#1a0c2d" stroke="#c084fc" stroke-width="1.5"/>
  <text x="350" y="115" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="11">READ Redis (no lock)</text>

  <!-- Three branches -->
  <line x1="250" y1="110" x2="100" y2="160" stroke="#4ade80" stroke-width="1.5" marker-end="url(#arr-g)"/>
  <text x="155" y="140" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="9">completed</text>

  <line x1="350" y1="130" x2="350" y2="165" stroke="#f87171" stroke-width="1.5" marker-end="url(#arr-r)"/>
  <text x="375" y="152" fill="#f87171" font-family="monospace" font-size="9">in_progress</text>

  <line x1="450" y1="110" x2="600" y2="160" stroke="#fbbf24" stroke-width="1.5" marker-end="url(#arr-y)"/>
  <text x="545" y="140" text-anchor="middle" fill="#fbbf24" font-family="monospace" font-size="9">not found</text>

  <!-- Completed outcome -->
  <rect x="30" y="160" width="140" height="40" rx="6" fill="#052e16" stroke="#4ade80" stroke-width="1.5"/>
  <text x="100" y="184" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="10" font-weight="600">Return 200</text>
  <text x="100" y="196" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="9">(cached response)</text>

  <!-- In-progress outcome -->
  <rect x="270" y="165" width="160" height="40" rx="6" fill="#2d0c0c" stroke="#f87171" stroke-width="1.5"/>
  <text x="350" y="188" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="10" font-weight="600">Return 409 Conflict</text>
  <text x="350" y="200" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="9">(client waits + retries)</text>

  <!-- Lock acquire -->
  <rect x="520" y="160" width="150" height="40" rx="6" fill="#2d1f00" stroke="#fbbf24" stroke-width="1.5"/>
  <text x="595" y="184" text-anchor="middle" fill="#fbbf24" font-family="monospace" font-size="10" font-weight="600">Acquire Lock</text>
  <text x="595" y="196" text-anchor="middle" fill="#fbbf24" font-family="monospace" font-size="9">SET NX EX</text>

  <!-- Second check inside lock -->
  <line x1="595" y1="200" x2="595" y2="235" stroke="#fbbf24" stroke-width="1.5" marker-end="url(#arr-y)"/>
  <rect x="490" y="235" width="210" height="40" rx="6" fill="#1a0c2d" stroke="#c084fc" stroke-width="1.5"/>
  <text x="595" y="259" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="10" font-weight="600">READ Redis Again</text>
  <text x="595" y="271" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="9">(double-checked locking)</text>

  <!-- Inner branches -->
  <line x1="490" y1="255" x2="400" y2="300" stroke="#4ade80" stroke-width="1.5" marker-end="url(#arr-g)"/>
  <text x="430" y="285" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="9">found (race won)</text>

  <line x1="595" y1="275" x2="595" y2="305" stroke="#fbbf24" stroke-width="1.5" marker-end="url(#arr-y)"/>
  <text x="620" y="295" fill="#fbbf24" font-family="monospace" font-size="9">not found</text>

  <!-- Inner found -->
  <rect x="295" y="300" width="160" height="40" rx="6" fill="#052e16" stroke="#4ade80" stroke-width="1.5"/>
  <text x="375" y="324" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="10" font-weight="600">Return cached</text>
  <text x="375" y="336" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="9">release lock</text>

  <!-- Process -->
  <rect x="510" y="305" width="170" height="40" rx="6" fill="#1e2130" stroke="#7dd3fc" stroke-width="1.5"/>
  <text x="595" y="328" text-anchor="middle" fill="#7dd3fc" font-family="monospace" font-size="10" font-weight="600">Process → Write → Release</text>
  <text x="595" y="340" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">store result in Redis</text>

  <!-- Label: fast path -->
  <text x="50" y="240" fill="#64748b" font-family="monospace" font-size="9">← FAST PATH (no lock needed)</text>
  <text x="480" y="400" fill="#64748b" font-family="monospace" font-size="9">LOCK PATH →</text>
</svg>
</div>

---

<div class="quick-ref-grid">

<div class="qr-card" style="--card-accent: #7dd3fc;">
<h4>HTTP Method Idempotency</h4>
<ul>
  <li><span class="tag tag-green">GET</span> safe + idempotent</li>
  <li><span class="tag tag-green">HEAD</span> safe + idempotent</li>
  <li><span class="tag tag-green">OPTIONS</span> safe + idempotent</li>
  <li><span class="tag tag-green">PUT</span> idempotent (replaces entirely)</li>
  <li><span class="tag tag-green">DELETE</span> idempotent (state = absent)</li>
  <li><span class="tag tag-yellow">PATCH</span> depends on operation type</li>
  <li><span class="tag tag-red">POST</span> not idempotent by default</li>
</ul>
</div>

<div class="qr-card" style="--card-accent: #f87171;">
<h4>Status Code Reference</h4>
<ul>
  <li><span class="tag tag-green">200</span> completed — return cached</li>
  <li><span class="tag tag-red">409</span> in-progress — retry after delay</li>
  <li><span class="tag tag-red">422</span> bad key reuse — do NOT retry</li>
  <li><span class="tag tag-yellow">429</span> rate limited — retry after delay</li>
  <li><span class="tag tag-blue">500</span> server error — safe to retry</li>
  <li><span class="tag tag-blue">503</span> unavailable — safe to retry</li>
</ul>
</div>

<div class="qr-card" style="--card-accent: #4ade80;">
<h4>Idempotency Key Rules</h4>
<ul>
  <li>Always generated by the <strong style="color:#4ade80">client</strong></li>
  <li>Generated <em>before</em> the first request</li>
  <li>Persisted across all retries</li>
  <li>Scoped to: endpoint + API key</li>
  <li>Format: UUID (recommended)</li>
  <li>TTL: 24h (Stripe) or 7d (financial)</li>
</ul>
</div>

<div class="qr-card" style="--card-accent: #fbbf24;">
<h4>Redis Lock Pattern</h4>
<ul>
  <li>Use <code style="color:#fbbf24">SET key 1 NX EX 30</code></li>
  <li>NX = only set if not exists (atomic)</li>
  <li>Double-check after acquiring lock</li>
  <li>Set <code>in_progress</code> before processing</li>
  <li>Use fencing tokens for long ops</li>
  <li>Heartbeat to extend TTL on slow ops</li>
</ul>
</div>

<div class="qr-card" style="--card-accent: #c084fc;">
<h4>Cross-Service Key Propagation</h4>
<ul>
  <li><code style="color:#c084fc">abc-123</code> → parent key</li>
  <li><code style="color:#c084fc">abc-123:order</code> → order service</li>
  <li><code style="color:#c084fc">abc-123:payment</code> → payment service</li>
  <li><code style="color:#c084fc">abc-123:notify</code> → notification service</li>
  <li>Each service deduplicates independently</li>
</ul>
</div>

<div class="qr-card" style="--card-accent: #fb923c;">
<h4>Outbox Pattern Steps</h4>
<ul>
  <li>1. DB transaction: table + outbox (atomic)</li>
  <li>2. Outbox worker: poll → publish to Kafka</li>
  <li>3. Consumer: check event_id → skip or process</li>
  <li>Worker may publish twice (crash recovery)</li>
  <li>Consumer must be idempotent</li>
  <li>Result: effectively exactly-once</li>
</ul>
</div>

</div>

<div class="diagram-wrapper">
<h3>Idempotency vs Exactly-Once — Key Differences</h3>

<svg viewBox="0 0 700 180" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:700px;display:block;margin:0 auto;">
  <rect width="700" height="180" fill="#0f1117" rx="8"/>

  <!-- Left: Idempotency -->
  <rect x="20" y="20" width="310" height="140" rx="8" fill="#052e16" stroke="#4ade80" stroke-width="1.5"/>
  <text x="175" y="45" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="12" font-weight="700">Idempotency</text>
  <text x="175" y="65" text-anchor="middle" fill="#86efac" font-family="monospace" font-size="10">Operation safe to retry</text>
  <text x="175" y="83" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">Server handles deduplication</text>
  <text x="175" y="99" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">Client sends → server detects dup</text>
  <text x="175" y="115" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">Practical, widely implemented</text>
  <text x="175" y="131" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="9" font-weight="600">at-least-once + idempotent consumer</text>
  <text x="175" y="149" text-anchor="middle" fill="#4ade80" font-family="monospace" font-size="9">= effectively exactly-once ✓</text>

  <!-- Right: Exactly Once -->
  <rect x="370" y="20" width="310" height="140" rx="8" fill="#1a0c2d" stroke="#c084fc" stroke-width="1.5"/>
  <text x="525" y="45" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="12" font-weight="700">Exactly-Once Semantics</text>
  <text x="525" y="65" text-anchor="middle" fill="#d8b4fe" font-family="monospace" font-size="10">Guaranteed one execution end-to-end</text>
  <text x="525" y="83" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">Requires distributed coordination</text>
  <text x="525" y="99" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">Two Generals Problem limits this</text>
  <text x="525" y="115" text-anchor="middle" fill="#64748b" font-family="monospace" font-size="9">Kafka EOS: +20-30% latency overhead</text>
  <text x="525" y="131" text-anchor="middle" fill="#f87171" font-family="monospace" font-size="9" font-weight="600">True EOS is theoretically impossible</text>
  <text x="525" y="149" text-anchor="middle" fill="#c084fc" font-family="monospace" font-size="9">Use only when truly required ⚠</text>
</svg>
</div>