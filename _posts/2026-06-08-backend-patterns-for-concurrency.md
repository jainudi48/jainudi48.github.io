---
layout: post
title: "Backend Patterns Built with Concurrency in Mind"
date: 2026-06-08
categories: [system-design, backend, concurrency]
tags: [system-design, concurrency, backend, interview-prep, csharp]
description: "A deep-dive into rate limiting, circuit breaking, load balancing, and retry patterns — backend constructs designed with concurrency in mind, with the reasoning that separates senior engineers from the rest."
---

# Backend Patterns Built with Concurrency in Mind

There is a specific kind of interview question that separates senior engineers from those still growing into that role. The interviewer gives you something deceptively simple — "implement a rate limiter" — and watches not just *what* you build, but *how* you reason about shared state, failure, and correctness under concurrent load.

This post is about that reasoning. We will cover four patterns that show up repeatedly in these interviews and in real systems: **rate limiting**, **circuit breaking**, **load balancing with health awareness**, and **retry with exponential backoff**. But more importantly, we will cover the *why* behind every design decision — because that is what the interviewer is actually listening for.

---

## First: The Concurrency Primitives You Cannot Skip

Before a single line of pattern code makes sense, you need fluency in four concurrency tools. These are the atomic building blocks. Skip this section and the patterns become magic incantations you memorise without understanding.

### The `lock` keyword — mutual exclusion, nothing more

```csharp
private readonly object _lock = new();

lock (_lock)
{
    // only one thread executes this block at a time
}
```

`lock` is a mutex. It ensures that when one thread is inside the block, every other thread trying to enter waits outside. The protected code becomes *effectively atomic* from other threads' perspectives.

When should you reach for it? When reads and writes are similarly frequent, or when the critical section is short. The cost of a lock is measurable — threads that wait are threads that are not doing useful work — so you only pay it when the alternative (a race condition) is worse.

### `ReaderWriterLockSlim` — for read-heavy state

Here is a pattern that trips up many engineers: you have a list of backend servers. It is read on *every single request* — potentially thousands of times per second. It is written only when a server is added or removed — maybe once an hour.

Using a plain `lock` here would serialise all reads, even though concurrent reads are completely safe. `ReaderWriterLockSlim` solves this elegantly: many threads can hold a **read lock simultaneously**, but the **write lock** is exclusive.

```csharp
// Hot path — hundreds of concurrent threads read this
_rwLock.EnterReadLock();
try { /* read the list */ }
finally { _rwLock.ExitReadLock(); }

// Rare path — one thread at a time
_rwLock.EnterWriteLock();
try { /* modify the list */ }
finally { _rwLock.ExitWriteLock(); }
```

Notice the `finally` blocks. This is not optional style. If you write `_rwLock.ExitReadLock()` *after* a `return` statement, the return exits the method first — the lock is never released. The next call to the write path waits forever for readers to finish. That is a deadlock, and it will happen in production at 3am.

### `Interlocked` — atomic operations without a lock

This is one of the most misunderstood primitives. Consider a round-robin counter that increments on every request:

```csharp
_counter++;
```

This looks like one operation. The CPU breaks it into three: READ the current value, ADD 1, WRITE the result back. Two threads can interleave these steps:

```
Thread A reads  → 5
Thread B reads  → 5   (before A wrote back)
Thread A writes → 6
Thread B writes → 6   (overwrites A's result — counter moved by 1, not 2)
```

You have lost an increment. More dangerously, two threads got the *same* index, so they both route to the same backend — breaking your round-robin distribution silently.

`Interlocked.Increment` is a single uninterruptible CPU instruction. It *physically cannot* be interleaved:

```csharp
var idx = Math.Abs(Interlocked.Increment(ref _counter)) % n;
```

Thread A and Thread B will always get different return values. The `Math.Abs()` is there because `_counter` is an `int` and will eventually overflow to a negative number. In C#, a negative number modulo `n` returns a negative result, which causes an `IndexOutOfRangeException`. `Math.Abs()` ensures the index stays non-negative.

### `volatile` — preventing the JIT from lying to you

The JIT compiler is aggressive about optimisation. For a tight loop:

```csharp
while (_healthy) { /* process requests */ }
```

The JIT might decide: "_healthy never changes inside this loop, so I'll put it in a CPU register and stop reading memory." Another thread can set `_healthy = false`, and this loop runs forever — it is reading a stale register copy.

`volatile` tells the JIT: "never cache this field in a register, always re-read it from the memory system."

A question that always comes up: "What about CPU cache? Won't L1/L2 cache be stale too?" No — modern multi-core CPUs implement a hardware protocol (MESI) that automatically invalidates a core's cached copy the moment another core writes to that memory location. Cache coherence is handled in hardware. What `volatile` prevents is purely the JIT-level register caching and instruction reordering. The performance cost is negligible — a few CPU cycles per access instead of zero.

---

## Part 1: Rate Limiting

Rate limiting controls how many requests a system processes in a given time window. It sounds simple. The implementation details are where it gets interesting.

### Why fixed window is not enough

The simplest approach: keep a counter, reset it every 60 seconds, reject requests once it hits the limit. This works until you notice the exploit: a client sends N requests at 11:59:59 and another N requests at 12:00:00. That is 2N requests in two seconds — twice the intended limit — and both batches are *technically* within their respective windows.

This "boundary burst" is why strict API rate limiters do not use fixed windows.

### Sliding window: the mental model first

Instead of resetting at fixed boundaries, the window slides with time. It always ends at *now* and starts at *now minus the window duration*. At any point, you count only the requests that happened in the last W seconds.

Picture it as a physical window you drag along a timeline:

```
←────────── 10 seconds ──────────→
[req][req][req]........[req][NEW?]
 old, falling off        still in window
```

As time passes, old requests fall off the left edge. New requests arrive on the right. The count at any moment is only what is currently inside the frame.

The algorithm in three steps:

```
1. cutoff = now - windowDuration
2. Evict all recorded timestamps older than cutoff
3. If count >= limit → REJECT
   Else             → ALLOW and record this timestamp
```

Walk through it with limit = 3 requests per 10 seconds:

```
t=1s  → timestamps: [1]       count=1 → ALLOW ✓
t=3s  → timestamps: [1,3]     count=2 → ALLOW ✓
t=5s  → timestamps: [1,3,5]   count=3 → ALLOW ✓
t=6s  → timestamps: [1,3,5]   count=3 → REJECT ✗
t=11s → cutoff=1s, evict t=1
        timestamps: [3,5]     count=2 → ALLOW ✓
```

At t=11s, the request from t=1s slides out of the window, freeing a slot. There is no cliff reset — capacity opens naturally and gradually as old requests age out.

### Why a `Queue`, not a `List`?

Timestamps are always added at the right (newest) and removed from the left (oldest). This is exactly what a queue is — O(1) enqueue at the back, O(1) dequeue from the front. A `List<T>` would shift all remaining elements on every removal, making eviction O(n).

```csharp
public class SlidingWindowLimiter
{
    private readonly object _lock = new();
    private readonly TimeSpan _window;
    private readonly int _limit;
    private readonly Queue<DateTime> _timestamps = new();

    public SlidingWindowLimiter(int limit, TimeSpan window)
    {
        _limit = limit;
        _window = window;
    }

    public bool Allow()
    {
        lock (_lock)
        {
            var cutoff = DateTime.UtcNow - _window;

            while (_timestamps.Count > 0 && _timestamps.Peek() < cutoff)
                _timestamps.Dequeue();

            if (_timestamps.Count >= _limit)
                return false;

            _timestamps.Enqueue(DateTime.UtcNow);
            return true;
        }
    }
}
```

The entire method is inside a single `lock`. The reason: evict → check → enqueue must be one atomic sequence. Without it, two threads can both pass the count check before either enqueues. Both get allowed, count exceeds the limit. The lock is the correctness guarantee.

### Token bucket: a different mental model entirely

Token bucket does not record history. Instead, it manages a bucket of tokens. Each request consumes one token. Tokens refill at a constant rate. If the bucket is empty, the request is rejected.

The elegant part: you do not need a background thread to refill tokens. Every call to `Allow()` calculates how many tokens should have accumulated since the last call and adds them:

```csharp
var elapsed = (now - _lastRefill).TotalSeconds;
_tokens = Math.Min(_maxTokens, _tokens + elapsed * _refillRate);
```

If 0.1 seconds elapsed and refill rate is 10/sec, add 1 token. If 2 seconds elapsed, add 20 tokens (capped at `maxTokens`). The math replaces the timer.

```csharp
public class TokenBucket
{
    private readonly object _lock = new();
    private double _tokens;
    private readonly double _maxTokens;
    private readonly double _refillRate; // tokens per second
    private DateTime _lastRefill = DateTime.UtcNow;

    public TokenBucket(double maxTokens, double refillRate)
    {
        _maxTokens = maxTokens;
        _tokens = maxTokens;
        _refillRate = refillRate;
    }

    public bool Allow()
    {
        lock (_lock)
        {
            var now = DateTime.UtcNow;
            var elapsed = (now - _lastRefill).TotalSeconds;
            _tokens = Math.Min(_maxTokens, _tokens + elapsed * _refillRate);
            _lastRefill = now;

            if (_tokens < 1) return false;
            _tokens--;
            return true;
        }
    }
}
```

### Sliding window vs token bucket: the real difference

With identical configurations — limit 10/sec — they behave identically under steady traffic. The difference only emerges when you deliberately set `maxTokens` higher than one second of refill:

```
System is idle for 5 seconds, then 15 requests fire at once:

Sliding window (limit=10):      allows 10, rejects 5
Token bucket  (maxTokens=10):   allows 10, rejects 5  ← same
Token bucket  (maxTokens=50):   allows 15, rejects 0  ← burst absorbed!
```

Token bucket lets you decouple burst capacity from sustained rate. A client that has been quiet can accumulate capacity for a future spike. This is the knob sliding window simply does not have.

**In an interview, this is the answer**: sliding window for strict per-second accuracy with no burst accumulation, token bucket when you want to allow controlled bursts during idle periods.

---

## Part 2: Circuit Breaker

Imagine your service calls a downstream payment API. That API goes down. Every request to your service now spawns a thread that waits 30 seconds for the payment API to time out, then fails. Under load, your thread pool exhausts waiting on a dead dependency — and your service goes down too, even though it was fine. This is a cascading failure.

A circuit breaker stops it. It is a state machine that wraps calls to a dependency. When that dependency starts failing, the circuit opens — requests are rejected immediately without ever touching the dependency. Resources are freed instantly.

### The three states

```
CLOSED    → normal operation, requests pass through
OPEN      → dependency is failing, reject immediately without calling it
HALF_OPEN → timeout has passed, allow one probe to check for recovery
```

The transitions:

- **CLOSED → OPEN**: failure count reaches threshold
- **OPEN → HALF_OPEN**: configured timeout elapses since last failure
- **HALF_OPEN → CLOSED**: the probe request succeeds (dependency recovered)
- **HALF_OPEN → OPEN**: the probe request fails (dependency still down)

```csharp
public enum CircuitState { Closed, Open, HalfOpen }

public class CircuitBreaker
{
    private readonly object _lock = new();
    private CircuitState _state = CircuitState.Closed;
    private int _failures;
    private readonly int _threshold;
    private readonly TimeSpan _timeout;
    private DateTime _lastFailTime;

    public CircuitBreaker(int threshold, TimeSpan timeout)
    {
        _threshold = threshold;
        _timeout = timeout;
    }

    public bool Allow()
    {
        lock (_lock)
        {
            return _state switch
            {
                CircuitState.Closed => true,
                CircuitState.Open when DateTime.UtcNow - _lastFailTime > _timeout =>
                    Transition(CircuitState.HalfOpen),
                CircuitState.Open => false,
                CircuitState.HalfOpen => false,
                _ => false
            };
        }
    }

    public void RecordSuccess()
    {
        lock (_lock) { _failures = 0; _state = CircuitState.Closed; }
    }

    public void RecordFailure()
    {
        lock (_lock)
        {
            _failures++;
            _lastFailTime = DateTime.UtcNow;
            if (_failures >= _threshold)
                _state = CircuitState.Open;
        }
    }

    private bool Transition(CircuitState next) { _state = next; return true; }
}
```

There is an inherent race here worth naming explicitly. The check in `Allow()` and the actual HTTP call that follows are two separate operations — they cannot be made atomic. A concurrent thread could slip through between `Allow()` returning `true` and the call being made, especially in HALF_OPEN state where you want only one probe. The circuit breaker provides *probabilistic* protection, not a hard gate. This is acceptable: the goal is to prevent avalanche failures, not to guarantee exactly one probe request with mathematical certainty.

Always use `DateTime.UtcNow`, not `DateTime.Now`. Daylight Saving Time can move `Now` backward by an hour — which would make every timeout check fail for an hour, holding the circuit open far longer than intended.

---

## Part 3: Health-Aware Load Balancer

A load balancer distributes incoming requests across multiple backend servers. A *health-aware* one skips servers that are currently down. Combined with a per-backend circuit breaker, it also skips servers whose circuit is open.

The design has three distinct pieces of shared state, each with different access patterns — and each deserves a different primitive.

### Three primitives, three reasons

**The `_healthy` flag on each backend**: written rarely (only when health checks run), read on every single request. A full `lock` here would be disproportionate overhead on the hot path. `volatile` ensures every thread re-reads from memory rather than a stale register copy, at negligible cost.

**The backends list**: read on every request, written only when servers are added or removed. `ReaderWriterLockSlim` lets all routing threads proceed in parallel (shared read lock) while briefly serialising the rare add/remove operation (exclusive write lock).

**The round-robin counter**: incremented on every request. A write lock here would serialise all requests — defeating the purpose of having multiple backends. `Interlocked.Increment` gives each thread a unique counter value with a single atomic CPU instruction, zero contention.

```csharp
public class Backend
{
    public string Address { get; }
    private volatile bool _healthy = true;
    public bool IsHealthy => _healthy;
    public void SetHealthy(bool value) => _healthy = value;
    public CircuitBreaker CircuitBreaker { get; } =
        new(threshold: 3, timeout: TimeSpan.FromSeconds(30));

    public Backend(string address) => Address = address;
}

public class LoadBalancer
{
    private readonly ReaderWriterLockSlim _rwLock = new();
    private readonly List<Backend> _backends;
    private int _counter;

    public LoadBalancer(List<Backend> backends) => _backends = backends;

    public Backend? Next()
    {
        _rwLock.EnterReadLock();
        try
        {
            var n = _backends.Count;
            for (int i = 0; i < n; i++)
            {
                var idx = Math.Abs(Interlocked.Increment(ref _counter)) % n;
                var b = _backends[idx];
                if (b.IsHealthy && b.CircuitBreaker.Allow())
                    return b;
            }
            return null;
        }
        finally { _rwLock.ExitReadLock(); }
    }
}
```

### Why mark unhealthy instead of removing?

The instinct is to remove a failing backend from the list. This is wrong for three reasons.

First, there is no recovery path. A backend that no longer exists in the list cannot be routed to when it comes back up. You would need a separate "known backends" registry, which is just the same list with extra complexity.

Second, removing an element from a list shifts all subsequent indices. Threads that computed an index and are mid-flight will read the wrong backend or throw `IndexOutOfRangeException`.

Third, health state changes frequently under load. Each change triggering a write lock momentarily blocks all request routing.

The clean solution: mark the backend unhealthy and let `Next()` skip it. The loop naturally handles sparse availability. When the server recovers, mark it healthy — it re-enters rotation immediately.

### The health checker

The health checker runs in the background, probing *all* backends — including unhealthy ones. Probing unhealthy backends is the mechanism by which recovered servers re-enter rotation.

```csharp
public class HealthChecker
{
    private readonly LoadBalancer _lb;
    private readonly TimeSpan _interval;

    public async Task StartAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            foreach (var backend in _lb.GetAllBackends())
            {
                var alive = await ProbeAsync(backend);
                backend.SetHealthy(alive);

                if (alive) backend.CircuitBreaker.RecordSuccess();
                else       backend.CircuitBreaker.RecordFailure();
            }
            await Task.Delay(_interval, ct);
        }
    }
}
```

`Task.Delay(_interval, ct)` takes the `CancellationToken` as a second argument. Without it, app shutdown would wait for the full interval before the health checker could stop. With the token, cancellation wakes `Task.Delay` immediately and throws `OperationCanceledException`, propagating cleanly out of the loop.

---

## Part 4: Retry with Exponential Backoff

When a request fails, retrying immediately is almost always wrong. The downstream service is likely still processing (or still down). Hammering it with immediate retries makes recovery harder, not easier.

Exponential backoff increases the wait between retries geometrically: 100ms, 200ms, 400ms, 800ms... giving the dependency time to recover.

### Jitter: the detail most engineers miss

Without jitter, consider 100 clients that all fail at the same moment. They will all wait 100ms and retry simultaneously. Then wait 200ms and retry simultaneously. They hit the recovering service in synchronised waves — a thundering herd that can re-break a service that was just coming back up.

Jitter adds a random offset to each delay:

```csharp
var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(
    -(int)(delay.TotalMilliseconds * 0.25),
     (int)(delay.TotalMilliseconds * 0.25)));
await Task.Delay(delay + jitter, ct);
```

At `delay = 1000ms`, each client waits somewhere between 750ms and 1250ms. Retries naturally spread across a 500ms window. The recovering service sees a gradual ramp instead of a wave.

```csharp
public static async Task<T> WithRetry<T>(
    Func<Task<T>> operation,
    int maxAttempts,
    CancellationToken ct)
{
    var delay = TimeSpan.FromMilliseconds(100);

    for (int attempt = 0; attempt < maxAttempts; attempt++)
    {
        try { return await operation(); }
        catch when (attempt == maxAttempts - 1) { throw; }
        catch
        {
            var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(
                -(int)(delay.TotalMilliseconds * 0.25),
                 (int)(delay.TotalMilliseconds * 0.25)));

            await Task.Delay(delay + jitter, ct);
            delay = TimeSpan.FromMilliseconds(
                Math.Min(delay.TotalMilliseconds * 2, 30_000));
        }
    }
    throw new InvalidOperationException("unreachable");
}
```

The cap at 30 seconds prevents delay from growing indefinitely (100ms × 2¹⁵ ≈ 54 minutes without a cap). `Random.Shared` is a thread-safe singleton in .NET 6+ — before .NET 6, you needed per-thread `Random` instances to avoid races on internal state.

The final `throw new InvalidOperationException("unreachable")` exists purely to satisfy the compiler, which does not follow the logic that the last `catch` always re-throws. It can never execute at runtime. The string is a message to the next developer reading this code.

---

## Architecture: How It All Fits Together

Here is the complete picture when all four patterns compose into a production backend pool. **Policy order matters** — get it wrong and retries hammer the same broken backend instead of finding a healthy one.

```
Incoming Request
       │
       ▼
┌─────────────────┐
│  Token Bucket   │ ← global rate limit (10 req/sec)
│  Rate Limiter   │   shed excess load before any real work
└────────┬────────┘
         │ allowed
         ▼
┌─────────────────────┐
│   Retry Wrapper     │ ← OUTERMOST resilience policy
│  exp. backoff +     │   each attempt re-runs LB selection,
│  jitter             │   so retries can land on a NEW backend
└──────────┬──────────┘
           │ attempt N (N = 1, 2, 3, …)
           ▼
┌─────────────────┐
│  Load Balancer  │ ← round-robin via Interlocked
│  (RWLockSlim)   │   filters: backend.Healthy && CB ≠ OPEN
└────────┬────────┘
         │ backend chosen
         ▼
┌─────────────────┐
│ Circuit Breaker │ ← per-backend state machine
│ (per backend)   │   CLOSED / OPEN / HALF_OPEN
└────────┬────────┘   fast-fails when OPEN; records ✓ / ✗
         │ allowed
         ▼
   Backend Server
  (HTTP / gRPC / etc.)
         │
   failure ──► bubbles up to Retry Wrapper for next attempt

         ⬆
┌─────────────────┐
│ Health Checker  │ ← background loop, probes ALL backends
│ (CancellToken)  │   updates volatile _healthy flag
└─────────────────┘   (read by LB on every selection)
```

**Why Retry is the outermost policy.** The most common mistake is to put Retry *inside* the backend call — between the circuit breaker and the HTTP client. That makes every retry hammer the same backend that just failed. The correct composition (Polly's official guidance, Envoy's router→cluster model, resilience4j's standard recipe) is: **Retry wraps the whole selection-and-call sequence**. Each retry attempt re-enters the load balancer, so a failing backend gets bypassed naturally on the next attempt.

**Data flow on failure**: request passes the rate limiter. Retry wrapper begins **attempt #1** → load balancer scans backends and picks B (healthy, CB is `CLOSED`) → HTTP call to B times out → B's circuit breaker records a failure → retry wrapper catches the exception, sleeps 100ms ± jitter → **attempt #2** → load balancer re-scans; if B's failures have now crossed the threshold, B's CB is `OPEN` and the LB excludes it → picks C → C's CB is `CLOSED`, call succeeds → C's CB records success → response returned.

**Concurrency model**: the rate limiter, each circuit breaker, and the backends list all have independent locks. Threads contend per-resource, not on a global lock. The health checker writes `_healthy` flags using `volatile` without blocking request-routing threads. The load balancer reads `_healthy` and CB state on every selection; both are designed to be lock-free or near-lock-free on the read path.

---

## Quick Reference: Which Primitive When?

```
Single bool/int, written rarely, read constantly   → volatile
Counter incremented atomically on every operation  → Interlocked
Short critical section, balanced reads and writes  → lock
Collection read very frequently, modified rarely   → ReaderWriterLockSlim
```

```
No burst accumulation, strict per-window accuracy  → Sliding window
Burst accumulation during idle, O(1) memory        → Token bucket
Smooth output rate, absorb spikes into a queue     → Leaky bucket
Simple quota reset on schedule (acceptable risk)   → Fixed window
```

```
Circuit transitions:
CLOSED   ──[N failures]──→ OPEN
OPEN     ──[timeout]─────→ HALF_OPEN
HALF_OPEN──[probe ✓]─────→ CLOSED
HALF_OPEN──[probe ✗]─────→ OPEN
```

---

<div class="interactive-reference" style="margin: 2rem 0;">

<style>
.ir-container {
  font-family: 'JetBrains Mono', 'Fira Code', 'Courier New', monospace;
  background: #0d1117;
  border-radius: 12px;
  padding: 2rem;
  color: #e6edf3;
  max-width: 900px;
  margin: 0 auto;
  border: 1px solid #21262d;
}

.ir-tabs {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.ir-tab {
  padding: 0.4rem 1rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.8rem;
  font-weight: 600;
  letter-spacing: 0.05em;
  border: 1px solid #30363d;
  background: #161b22;
  color: #8b949e;
  transition: all 0.2s;
}

.ir-tab:hover { background: #21262d; color: #c9d1d9; }
.ir-tab.active { background: #1f6feb; border-color: #1f6feb; color: #fff; }

.ir-panel { display: none; }
.ir-panel.active { display: block; }

.ir-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
}

.ir-card {
  background: #161b22;
  border: 1px solid #21262d;
  border-radius: 8px;
  padding: 1rem;
  transition: border-color 0.2s;
}

.ir-card:hover { border-color: #388bfd; }

.ir-card-title {
  font-size: 0.75rem;
  color: #7ee787;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin-bottom: 0.5rem;
}

.ir-card-body { font-size: 0.85rem; color: #c9d1d9; line-height: 1.6; }
.ir-card-note { font-size: 0.75rem; color: #8b949e; margin-top: 0.4rem; }

.ir-state-machine {
  display: flex;
  align-items: center;
  justify-content: space-around;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin: 1rem 0;
}

.ir-state {
  text-align: center;
  padding: 0.8rem 1.2rem;
  border-radius: 8px;
  font-size: 0.85rem;
  font-weight: 700;
  min-width: 90px;
  cursor: pointer;
  transition: transform 0.15s;
}

.ir-state:hover { transform: scale(1.05); }

.state-closed { background: #1a4731; border: 2px solid #2ea043; color: #7ee787; }
.state-open   { background: #4d1a1a; border: 2px solid #da3633; color: #ff7b72; }
.state-half   { background: #3d2a00; border: 2px solid #d29922; color: #e3b341; }

.ir-arrow {
  color: #8b949e;
  font-size: 0.75rem;
  text-align: center;
  min-width: 70px;
}

.ir-arrow-label { color: #58a6ff; font-size: 0.7rem; }

.ir-timeline {
  position: relative;
  height: 60px;
  background: #161b22;
  border-radius: 6px;
  margin: 0.5rem 0;
  overflow: hidden;
  border: 1px solid #21262d;
}

.ir-window {
  position: absolute;
  top: 0; bottom: 0;
  background: rgba(31, 111, 235, 0.15);
  border-left: 2px solid #1f6feb;
  border-right: 2px solid #1f6feb;
  transition: all 0.5s ease;
}

.ir-req-dot {
  position: absolute;
  top: 50%;
  transform: translate(-50%, -50%);
  width: 12px; height: 12px;
  border-radius: 50%;
  transition: opacity 0.3s;
}

.dot-allowed { background: #7ee787; }
.dot-rejected { background: #ff7b72; }

.ir-btn {
  padding: 0.4rem 1rem;
  border-radius: 6px;
  border: 1px solid #30363d;
  background: #21262d;
  color: #c9d1d9;
  cursor: pointer;
  font-size: 0.8rem;
  font-family: inherit;
  transition: all 0.15s;
  margin-right: 0.5rem;
}

.ir-btn:hover { background: #30363d; color: #fff; }
.ir-btn.primary { background: #1f6feb; border-color: #1f6feb; color: #fff; }
.ir-btn.primary:hover { background: #388bfd; }

.ir-label { font-size: 0.75rem; color: #8b949e; margin: 0.3rem 0; }
.ir-value { font-size: 1.1rem; font-weight: 700; color: #e6edf3; }
.ir-metric { background: #161b22; border-radius: 6px; padding: 0.5rem 0.8rem; display: inline-block; border: 1px solid #21262d; margin: 0.2rem; }

.ir-code {
  background: #0d1117;
  border: 1px solid #21262d;
  border-radius: 6px;
  padding: 1rem;
  font-size: 0.8rem;
  color: #e6edf3;
  line-height: 1.6;
  overflow-x: auto;
  white-space: pre;
}

.kw { color: #ff7b72; }
.cm { color: #8b949e; font-style: italic; }
.st { color: #a5d6ff; }
.nm { color: #79c0ff; }
.fn { color: #d2a8ff; }
</style>

<div class="ir-container">
  <div class="ir-tabs">
    <button class="ir-tab active" onclick="showPanel('primitives')">Primitives</button>
    <button class="ir-tab" onclick="showPanel('ratelimit')">Rate Limiting</button>
    <button class="ir-tab" onclick="showPanel('circuit')">Circuit Breaker</button>
    <button class="ir-tab" onclick="showPanel('retry')">Retry Backoff</button>
  </div>

  <!-- PRIMITIVES PANEL -->
  <div class="ir-panel active" id="panel-primitives">
    <div style="margin-bottom:1rem; font-size:0.8rem; color:#8b949e;">
      Click any card to see when and why to use each primitive.
    </div>
    <div class="ir-grid">
      <div class="ir-card" onclick="showDetail('lock')">
        <div class="ir-card-title">lock</div>
        <div class="ir-card-body">Mutual exclusion<br>One thread at a time</div>
        <div class="ir-card-note">Use: balanced reads &amp; writes</div>
      </div>
      <div class="ir-card" onclick="showDetail('rwlock')">
        <div class="ir-card-title">ReaderWriterLockSlim</div>
        <div class="ir-card-body">Many readers, one writer</div>
        <div class="ir-card-note">Use: read-heavy collections</div>
      </div>
      <div class="ir-card" onclick="showDetail('interlocked')">
        <div class="ir-card-title">Interlocked</div>
        <div class="ir-card-body">Atomic single-field ops<br>No lock overhead</div>
        <div class="ir-card-note">Use: counters on hot path</div>
      </div>
      <div class="ir-card" onclick="showDetail('volatile')">
        <div class="ir-card-title">volatile</div>
        <div class="ir-card-body">Prevent JIT register cache<br>Near-zero cost</div>
        <div class="ir-card-note">Use: flags read by many threads</div>
      </div>
    </div>
    <div id="primitive-detail" style="margin-top:1.5rem; display:none;">
      <div id="detail-lock" class="ir-code" style="display:none"><span class="cm">// lock: one thread executes at a time
// Thread B waits outside until Thread A exits
// Cost: context switch overhead (~1-5µs)</span>
<span class="kw">private readonly</span> <span class="nm">object</span> _lock = <span class="kw">new</span>();

<span class="kw">lock</span> (_lock)
{
    <span class="cm">// atomic from other threads' perspective</span>
    _tokens--;
    _lastRefill = <span class="nm">DateTime</span>.UtcNow;
}

<span class="cm">// WHY: sliding window evict→check→enqueue must
// be one atomic sequence or two threads both pass
// the count check before either enqueues</span></div>
      <div id="detail-rwlock" class="ir-code" style="display:none"><span class="cm">// ReaderWriterLockSlim: read-heavy state
// Multiple threads hold read lock simultaneously
// Write lock is exclusive, blocks all readers</span>
_rwLock.<span class="fn">EnterReadLock</span>();
<span class="kw">try</span> { <span class="cm">/* 1000 concurrent reads — all proceed */</span> }
<span class="kw">finally</span> { _rwLock.<span class="fn">ExitReadLock</span>(); }

<span class="cm">// CRITICAL: always release in finally
// ExitReadLock() after a return statement =
// lock never released = deadlock on next write</span>

_rwLock.<span class="fn">EnterWriteLock</span>();  <span class="cm">// rare — blocks readers</span>
<span class="kw">try</span> { _backends.<span class="fn">Add</span>(newServer); }
<span class="kw">finally</span> { _rwLock.<span class="fn">ExitWriteLock</span>(); }</div>
      <div id="detail-interlocked" class="ir-code" style="display:none"><span class="cm">// _counter++ is NOT atomic. Three CPU steps:
// READ value → ADD 1 → WRITE result
// Two threads can interleave: both read 5,
// both write 6. Lost increment. Wrong index.</span>

<span class="cm">// Interlocked.Increment = one CPU instruction
// (LOCK XADD on x86) — physically uninterruptible</span>
<span class="kw">var</span> idx = <span class="nm">Math</span>.<span class="fn">Abs</span>(
    <span class="nm">Interlocked</span>.<span class="fn">Increment</span>(<span class="kw">ref</span> _counter)
) % n;

<span class="cm">// Math.Abs() because int overflows to negative.
// In C#: -1 % 3 = -1 → IndexOutOfRangeException</span></div>
      <div id="detail-volatile" class="ir-code" style="display:none"><span class="cm">// JIT can cache _healthy in a CPU register:
// while (_healthy) { ... }
// Even if another thread sets _healthy=false,
// this loop runs forever from stale register.</span>

<span class="kw">private volatile bool</span> _healthy = <span class="kw">true</span>;

<span class="cm">// volatile forces re-read from memory each time.
// Cost: L1/L2 cache read (~4-12 cycles) vs 0.
// Hardware (MESI protocol) keeps cache coherent —
// volatile only prevents JIT register caching
// and instruction reordering.</span></div>
    </div>
  </div>

  <!-- RATE LIMITING PANEL -->
  <div class="ir-panel" id="panel-ratelimit">
    <div style="display:flex; gap:1rem; align-items:center; flex-wrap:wrap; margin-bottom:1rem;">
      <button class="ir-btn primary" onclick="fireRequest()">Fire Request</button>
      <button class="ir-btn" onclick="switchLimiter()">Switch: <span id="limiter-name">Sliding Window</span></button>
      <button class="ir-btn" onclick="resetLimiter()">Reset</button>
      <div class="ir-metric"><div class="ir-label">Algorithm</div><div class="ir-value" id="algo-display" style="font-size:0.85rem">Sliding Window</div></div>
      <div class="ir-metric"><div class="ir-label">Count</div><div class="ir-value" id="count-display">0 / 5</div></div>
      <div class="ir-metric"><div class="ir-label">Last result</div><div class="ir-value" id="result-display" style="color:#7ee787">—</div></div>
    </div>
    <div class="ir-label">Request timeline (last 10 seconds):</div>
    <div class="ir-timeline" id="timeline">
      <div class="ir-window" id="window-bar" style="left:0%; right:0%;"></div>
    </div>
    <div id="algo-explanation" style="margin-top:1rem; font-size:0.8rem; color:#8b949e; line-height:1.6;">
      <strong style="color:#c9d1d9">Sliding Window:</strong> 
      Window always ends at NOW. Old requests fall off the left edge as time passes. 
      No boundary burst possible. Memory: O(n) — stores each timestamp.
    </div>
    <div style="margin-top:1.5rem;">
      <div style="font-size:0.75rem; color:#7ee787; margin-bottom:0.5rem;">COMPARISON CHEAT SHEET</div>
      <div class="ir-grid">
        <div class="ir-card">
          <div class="ir-card-title" style="color:#58a6ff">Sliding Window</div>
          <div class="ir-card-body">Strict accuracy<br>No burst accumulation<br>O(n) memory per client</div>
          <div class="ir-card-note">→ API rate limiting</div>
        </div>
        <div class="ir-card">
          <div class="ir-card-title" style="color:#d2a8ff">Token Bucket</div>
          <div class="ir-card-body">Burst during idle<br>Decouple burst/rate<br>O(1) memory</div>
          <div class="ir-card-note">→ Traffic shaping</div>
        </div>
        <div class="ir-card">
          <div class="ir-card-title" style="color:#e3b341">Fixed Window</div>
          <div class="ir-card-body">Simple, cheap<br>Boundary burst flaw<br>O(1) memory</div>
          <div class="ir-card-note">→ Rough quota only</div>
        </div>
        <div class="ir-card">
          <div class="ir-card-title" style="color:#ff7b72">Leaky Bucket</div>
          <div class="ir-card-body">Smoothest output<br>Queue-based<br>Absorbs spikes</div>
          <div class="ir-card-note">→ Network shaping</div>
        </div>
      </div>
    </div>
  </div>

  <!-- CIRCUIT BREAKER PANEL -->
  <div class="ir-panel" id="panel-circuit">
    <div class="ir-state-machine">
      <div class="ir-state state-closed" id="cb-closed" onclick="cbClick('closed')">
        CLOSED<br><span style="font-size:0.7rem;font-weight:400">Normal operation</span>
      </div>
      <div class="ir-arrow">
        <div>──→</div>
        <div class="ir-arrow-label">N failures</div>
        <div>←──</div>
        <div class="ir-arrow-label" style="color:#ff7b72">probe fails</div>
      </div>
      <div class="ir-state state-open" id="cb-open" onclick="cbClick('open')">
        OPEN<br><span style="font-size:0.7rem;font-weight:400">Fast fail</span>
      </div>
      <div class="ir-arrow">
        <div>←──</div>
        <div class="ir-arrow-label" style="color:#7ee787">probe ✓</div>
        <div>──→</div>
        <div class="ir-arrow-label">timeout</div>
      </div>
      <div class="ir-state state-half" id="cb-half" onclick="cbClick('half')">
        HALF_OPEN<br><span style="font-size:0.7rem;font-weight:400">One probe</span>
      </div>
    </div>
    <div style="display:flex; gap:0.5rem; flex-wrap:wrap; margin:1rem 0; align-items:center;">
      <button class="ir-btn primary" onclick="cbSuccess()">Record Success</button>
      <button class="ir-btn" onclick="cbFail()">Record Failure</button>
      <div class="ir-metric"><div class="ir-label">State</div><div class="ir-value" id="cb-state-display" style="color:#7ee787">CLOSED</div></div>
      <div class="ir-metric"><div class="ir-label">Failures</div><div class="ir-value" id="cb-fail-display">0 / 3</div></div>
    </div>
    <div id="cb-detail" style="font-size:0.8rem; color:#8b949e; line-height:1.6; padding:0.8rem; background:#161b22; border-radius:6px;">
      Click a state or use the buttons above. Threshold = 3 failures. OPEN→HALF_OPEN after timeout.
    </div>
    <div style="margin-top:1.5rem;">
      <div style="font-size:0.75rem; color:#7ee787; margin-bottom:0.5rem;">KEY INSIGHT</div>
      <div class="ir-code"><span class="cm">// Allow() and the actual HTTP call are two
// separate operations — cannot be made atomic.
// A thread can slip through between Allow()→true
// and the network call.
//
// Circuit breaker = probabilistic protection.
// Goal: prevent avalanche. Not: mathematical gate.
//
// Always: DateTime.UtcNow, not DateTime.Now.
// DST can move Now backward 1 hour. Timeout
// check fails for entire hour. Circuit stays open.</span></div>
    </div>
  </div>

  <!-- RETRY BACKOFF PANEL -->
  <div class="ir-panel" id="panel-retry">
    <div style="display:flex; gap:0.5rem; flex-wrap:wrap; align-items:center; margin-bottom:1rem;">
      <button class="ir-btn primary" onclick="simRetry()">Simulate Retry</button>
      <button class="ir-btn" onclick="toggleJitter()">Jitter: <span id="jitter-toggle">ON</span></button>
      <button class="ir-btn" onclick="resetRetry()">Reset</button>
    </div>
    <div id="retry-bars" style="display:flex; align-items:flex-end; gap:4px; height:80px; margin:1rem 0; padding:0 0.5rem;">
    </div>
    <div id="retry-summary" style="font-size:0.8rem; color:#8b949e; margin-bottom:1rem;"></div>
    <div style="font-size:0.75rem; color:#7ee787; margin-bottom:0.5rem;">WHY JITTER MATTERS</div>
    <div class="ir-grid">
      <div class="ir-card">
        <div class="ir-card-title" style="color:#ff7b72">Without Jitter</div>
        <div class="ir-card-body">100 clients fail together. All retry at t+100ms. Thundering herd. Recovering service gets re-broken.</div>
      </div>
      <div class="ir-card">
        <div class="ir-card-title" style="color:#7ee787">With Jitter ±25%</div>
        <div class="ir-card-body">100 clients spread across a 500ms window. Service sees gradual ramp. Recovery succeeds.</div>
      </div>
    </div>
    <div style="margin-top:1rem;">
      <div class="ir-code"><span class="cm">// Backoff formula:
// attempt 1: 100ms ± 25ms
// attempt 2: 200ms ± 50ms
// attempt 3: 400ms ± 100ms
// ...capped at 30,000ms
//
// Math.Min(delay * 2, 30_000) prevents:
// 100 × 2^15 = 54 minutes between retries
//
// Random.Shared is thread-safe (.NET 6+)
// Before .NET 6: use [ThreadStatic] Random</span></div>
    </div>
  </div>
</div>

<script>
// Tab switching
function showPanel(name) {
  document.querySelectorAll('.ir-panel').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('.ir-tab').forEach(t => t.classList.remove('active'));
  document.getElementById('panel-' + name).classList.add('active');
  event.target.classList.add('active');
}

// Primitive detail
function showDetail(name) {
  var d = document.getElementById('primitive-detail');
  d.style.display = 'block';
  ['lock','rwlock','interlocked','volatile'].forEach(function(n) {
    document.getElementById('detail-' + n).style.display = n === name ? 'block' : 'none';
  });
}

// Rate limiting simulation
var requests = [];          // sliding-window: timestamps of ALLOWED requests in current window
var requestLog = [];        // {time, allowed} for visualization in BOTH modes
var limit = 5;
var windowMs = 10000;
var useTokenBucket = false;
var maxTokens = 5;
var tokens = maxTokens;
var refillRatePerSec = limit / (windowMs / 1000); // 0.5 tok/sec → matches sliding-window sustained rate
var lastRefill = Date.now();
var timelineLoopStarted = false;

function refillTokens() {
  var now = Date.now();
  var elapsed = (now - lastRefill) / 1000;
  tokens = Math.min(maxTokens, tokens + elapsed * refillRatePerSec);
  lastRefill = now;
}

function fireRequest() {
  var now = Date.now();
  var allowed;

  if (!useTokenBucket) {
    var cutoff = now - windowMs;
    requests = requests.filter(function(t) { return t > cutoff; });
    allowed = requests.length < limit;
    if (allowed) requests.push(now);
  } else {
    refillTokens();
    allowed = tokens >= 1;
    if (allowed) tokens -= 1;
  }

  requestLog.push({time: now, allowed: allowed});

  var resultEl = document.getElementById('result-display');
  resultEl.textContent = allowed ? 'ALLOWED ✓' : 'REJECTED ✗';
  resultEl.style.color = allowed ? '#7ee787' : '#ff7b72';

  renderTimeline();
  if (!timelineLoopStarted) { timelineLoopStarted = true; tickTimeline(); }
}

function renderTimeline() {
  var now = Date.now();
  var timeline = document.getElementById('timeline');
  timeline.querySelectorAll('.ir-req-dot').forEach(function(d) { d.remove(); });

  var cutoff = now - windowMs;
  requestLog = requestLog.filter(function(r) { return r.time > cutoff; });

  requestLog.forEach(function(r) {
    var dot = document.createElement('div');
    dot.className = 'ir-req-dot ' + (r.allowed ? 'dot-allowed' : 'dot-rejected');
    dot.style.left = ((r.time - (now - windowMs)) / windowMs * 100) + '%';
    timeline.appendChild(dot);
  });

  if (!useTokenBucket) {
    requests = requests.filter(function(t) { return t > cutoff; });
    document.getElementById('count-display').textContent = requests.length + ' / ' + limit;
  } else {
    refillTokens();
    document.getElementById('count-display').textContent = tokens.toFixed(1) + ' / ' + maxTokens + ' tok';
  }
}

function tickTimeline() {
  renderTimeline();
  setTimeout(tickTimeline, 500);
}

function switchLimiter() {
  useTokenBucket = !useTokenBucket;
  var name = useTokenBucket ? 'Token Bucket' : 'Sliding Window';
  document.getElementById('limiter-name').textContent = name;
  document.getElementById('algo-display').textContent = name;
  document.getElementById('algo-explanation').innerHTML = useTokenBucket
    ? '<strong style="color:#c9d1d9">Token Bucket:</strong> Maintains virtual token count. Tokens refill at ' + refillRatePerSec + '/sec. Can accumulate burst capacity during idle periods. Memory: O(1) — just two numbers.'
    : '<strong style="color:#c9d1d9">Sliding Window:</strong> Window always ends at NOW. Old requests fall off the left edge as time passes. No boundary burst possible. Memory: O(n) — stores each timestamp.';
  requests = [];
  requestLog = [];
  tokens = maxTokens;
  lastRefill = Date.now();
  renderTimeline();
}

function resetLimiter() {
  requests = [];
  requestLog = [];
  tokens = maxTokens;
  lastRefill = Date.now();
  document.getElementById('result-display').textContent = '—';
  document.getElementById('result-display').style.color = '#7ee787';
  document.getElementById('timeline').querySelectorAll('.ir-req-dot').forEach(function(d) { d.remove(); });
  renderTimeline();
}

// Circuit breaker simulation
var cbState = 'closed';
var cbFailures = 0;
var cbThreshold = 3;

function cbClick(state) {
  var msgs = {
    closed: 'CLOSED: All requests pass through. Failures are counted. After ' + cbThreshold + ' failures, transitions to OPEN.',
    open: 'OPEN: All requests are rejected immediately without calling the dependency. After a timeout elapses, transitions to HALF_OPEN.',
    half: 'HALF_OPEN: One probe request is allowed. If it succeeds → CLOSED. If it fails → back to OPEN.'
  };
  document.getElementById('cb-detail').textContent = msgs[state];
}

function updateCBDisplay() {
  var displays = { closed: '#7ee787', open: '#ff7b72', half: '#e3b341' };
  var names = { closed: 'CLOSED', open: 'OPEN', half: 'HALF_OPEN' };
  var el = document.getElementById('cb-state-display');
  el.textContent = names[cbState];
  el.style.color = displays[cbState];
  document.getElementById('cb-fail-display').textContent = cbFailures + ' / ' + cbThreshold;
  
  ['closed','open','half'].forEach(function(s) {
    document.getElementById('cb-' + s).style.opacity = s === cbState ? '1' : '0.4';
  });
}

function cbSuccess() {
  cbFailures = 0;
  if (cbState === 'half') {
    cbState = 'closed';
    document.getElementById('cb-detail').textContent = 'Probe succeeded. Transitioning HALF_OPEN → CLOSED. Full traffic restored.';
  } else {
    document.getElementById('cb-detail').textContent = 'Success recorded. Failure count reset to 0.';
  }
  updateCBDisplay();
}

function cbFail() {
  cbFailures++;
  if (cbState === 'half') {
    cbState = 'open';
    document.getElementById('cb-detail').textContent = 'Probe failed. Dependency still unhealthy. Back to OPEN. Timeout restarts.';
  } else if (cbFailures >= cbThreshold) {
    cbState = 'open';
    document.getElementById('cb-detail').textContent = 'Failure threshold reached (' + cbThreshold + '/' + cbThreshold + '). CLOSED → OPEN. Requests now fast-fail.';
  } else {
    document.getElementById('cb-detail').textContent = 'Failure recorded (' + cbFailures + '/' + cbThreshold + '). Still CLOSED.';
  }
  updateCBDisplay();
}

// Retry simulation
var jitterOn = true;
var retryAttempts = [];

function simRetry() {
  retryAttempts = [];
  var delay = 100;
  var maxAttempts = 6;
  
  for (var i = 0; i < maxAttempts; i++) {
    var jitter = jitterOn ? (Math.random() * 0.5 - 0.25) * delay : 0;
    retryAttempts.push(Math.round(delay + jitter));
    delay = Math.min(delay * 2, 30000);
  }
  
  renderRetryBars();
}

function renderRetryBars() {
  var container = document.getElementById('retry-bars');
  container.innerHTML = '';
  var max = Math.max.apply(null, retryAttempts);
  
  retryAttempts.forEach(function(ms, i) {
    var bar = document.createElement('div');
    var h = Math.max(4, (ms / max) * 70);
    bar.style.cssText = 'flex:1;height:' + h + 'px;background:#1f6feb;border-radius:3px 3px 0 0;position:relative;transition:height 0.3s;';
    
    var label = document.createElement('div');
    label.textContent = ms >= 1000 ? (ms/1000).toFixed(1)+'s' : ms+'ms';
    label.style.cssText = 'position:absolute;bottom:calc(100% + 2px);left:50%;transform:translateX(-50%);font-size:0.65rem;color:#8b949e;white-space:nowrap;';
    
    var attempt = document.createElement('div');
    attempt.textContent = '#' + (i+1);
    attempt.style.cssText = 'position:absolute;top:calc(100% + 2px);left:50%;transform:translateX(-50%);font-size:0.65rem;color:#8b949e;';
    
    bar.appendChild(label);
    bar.appendChild(attempt);
    container.appendChild(bar);
  });
  
  document.getElementById('retry-summary').textContent = 
    'Attempts: ' + retryAttempts.length + ' | Jitter: ' + (jitterOn ? 'ON (±25%)' : 'OFF') + 
    ' | Cap: 30,000ms | Total wait: ~' + (retryAttempts.reduce(function(a,b){return a+b;},0)/1000).toFixed(1) + 's';
}

function toggleJitter() {
  jitterOn = !jitterOn;
  document.getElementById('jitter-toggle').textContent = jitterOn ? 'ON' : 'OFF';
  if (retryAttempts.length > 0) simRetry();
}

function resetRetry() {
  retryAttempts = [];
  document.getElementById('retry-bars').innerHTML = '';
  document.getElementById('retry-summary').textContent = '';
}

updateCBDisplay();
</script>

</div>

</div>

---

*All code examples are in C# targeting .NET 6+. The patterns themselves are language-agnostic — the same reasoning applies in Go, Java, or Python, only the primitive names change.*