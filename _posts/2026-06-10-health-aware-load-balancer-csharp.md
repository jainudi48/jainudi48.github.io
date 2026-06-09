---
layout: post
title: "Building a Health-Aware Load Balancer in C#: A Machine Coding Walkthrough"
date: 2026-06-10
categories: [backend, system-design, csharp]
tags: [load-balancer, circuit-breaker, rate-limiting, concurrency, machine-coding, interviews]
description: "Round-robin load balancing, circuit breaking, token bucket rate limiting, retries with jitter, and a background health checker — built from scratch in C#, with every concurrency decision justified and every bug documented."
---

If you have ever sat in a machine coding interview, you know the feeling. The problem sounds simple — *"build a load balancer that routes requests across a few backends"* — and then the follow-ups start. What happens when a backend goes down? What if it comes back? What if fifty clients hammer it at once? How do you stop retries from making an outage worse?

This post walks through building exactly that system, end to end, in C#: a `BackendPool` with round-robin routing, per-backend health tracking, circuit breaking, a global token bucket rate limiter, retries with exponential backoff and jitter, and a background health checker that shuts down cleanly. But the code is honestly the least interesting part. What an interviewer actually evaluates is the *reasoning* — why `volatile` here but a `lock` there, why the rate limiter throws instead of returning `false`, why unhealthy backends are never removed from the pool. So this post documents the reasoning, the bugs caught along the way, and what correct behaviour looks like in the logs.

At the end you will find interactive visuals — a circuit breaker you can break, a token bucket you can drain, and a three-way rate limiter comparison — plus the full architecture diagram, so you can revise the whole thing in two minutes before an interview or a design discussion.

The complete source code for this walkthrough — every class below, runnable end to end — lives at [github.com/jainudi48/backendpool](https://github.com/jainudi48/backendpool).

## Why plan before you type

The single highest-leverage thing in a machine coding round is the first five minutes, and most candidates spend them typing. Resist that. The plan below took about four minutes to derive and it prevented every structural rewrite later.

You might ask: isn't planning out loud risky when the clock is running? It is the opposite. Interviewers consistently reward candidates who restate the problem, enumerate the moving parts, and only then open the editor — because that is what distinguishes someone who has built systems from someone who has memorised patterns. The plan is also your safety net: when you are forty minutes in and slightly panicked, you fall back to it instead of improvising.

Here is the class inventory that came out of those four minutes:

```
CLASSES:
  Backend         — address, IsHealthy (volatile), CircuitBreaker
  LoadBalancer    — List<Backend> (RWLock), counter (Interlocked), TokenBucket
  CircuitBreaker  — threshold, timeout, failedRequests, lastFailureTime, state, lock
  TokenBucket     — refillRate, maxTokens, tokens (double), lastRefill, lock
  HealthChecker   — List<Backend>, interval, async loop with CancellationToken
  RetryHelper     — static WithRetry<T>

ENUMS:
  CircuitState    — Closed, Open, HalfOpen
```

Six classes, one enum. Notice what is *not* here: no interfaces, no dependency injection container, no abstract factories. In a timed round, abstraction you do not need is time you do not have. If the interviewer wants extensibility, they will ask, and adding an `ILoadBalancingStrategy` interface later is a five-minute refactor. Starting with one is a twenty-minute tax.

## The shared mutable state audit

Before writing a single class, every piece of shared mutable state gets identified and its protection mechanism justified. This table *is* the concurrency design — everything after it is implementation:

| Variable | Lives in | Written by | Read by | Protection |
|---|---|---|---|---|
| `_backends` list | LoadBalancer | `AddBackend` (rare) | `Next()` (every request) | `ReaderWriterLockSlim` |
| `_counter` | LoadBalancer | every request | every request | `Interlocked.Increment` |
| `_healthy` | Backend | HealthChecker thread | LoadBalancer thread | `volatile` |
| CircuitBreaker state | CircuitBreaker | `RecordSuccess/Failure` | `Allow()` | `lock` |
| `_tokens` | TokenBucket | `Allow()` | `Allow()` | `lock` |

Four different protection mechanisms for five variables. That is not inconsistency — it is the whole point. Each mechanism is matched to the access pattern, and being able to articulate *why* each one fits is worth more in an interview than the code itself. So let's articulate it.

**Why `ReaderWriterLockSlim` for the backends list?** Reads happen on every single request — this is the hot path. Writes happen only when a backend is added, which is rare. Multiple threads reading a list concurrently is perfectly safe; the danger is a write happening mid-read. A plain `lock` would serialise every routing decision behind every other routing decision, throwing away parallelism for no benefit. `ReaderWriterLockSlim` lets all readers proceed in parallel and only blocks when a writer needs exclusive access. It is exactly the read-heavy, write-rare structure this primitive was designed for.

**Why `Interlocked` for the round-robin counter?** Because `_counter++` is a lie. It looks like one operation but compiles to three: read the value, add one, write it back. Two threads can interleave those steps — both read 5, both write 6, and an increment is lost. `Interlocked.Increment` maps to a single atomic CPU instruction (`LOCK XADD` on x86). No lock object, no context switch, no kernel involvement — one uninterruptible hardware operation. And a detail that trips people up: you do **not** also need `volatile` on the counter. `Interlocked` already issues a full memory barrier.

**Why `volatile` for `_healthy`?** It is a single `bool`, written occasionally by the health checker thread and read constantly by the load balancer. Wrapping a one-bool read in a full `lock` is disproportionate. But skipping protection entirely is wrong too — without `volatile`, the JIT is free to cache the value in a register, and the load balancer's loop might never see the health checker's update. `volatile` forbids that caching, forcing every read to go to the memory system, where the hardware cache coherence protocol (MESI) guarantees freshness.

A fair question here: *if MESI already keeps caches coherent, why do we need `volatile` at all?* Because MESI operates below the compiler. The hardware guarantees that if a read reaches the cache, it sees the latest value — but the JIT can optimise the read away entirely, hoisting it out of a loop into a register. `volatile` is a compiler-level contract ("always actually read this"), and MESI is a hardware-level one ("a real read is never stale"). You need both layers, and `volatile` is the one you control.

**Why a plain `lock` for the circuit breaker?** Because its updates are *compound*. `RecordFailure` touches three fields together: the failure count, the last-failure timestamp, and possibly the state. "Check the state, then transition" must be one atomic step — if another thread sneaks in between the check and the act, you get two probe requests in half-open state, or a circuit that opens twice. `Interlocked` works on single variables; `volatile` provides no atomicity at all. When multiple fields must change as a unit, `lock` is the right tool, not the lazy one.

**Why a `lock` for the token bucket?** Same reasoning in miniature: refilling and consuming tokens is a read-modify-write sequence on `_tokens` plus `_lastRefill`. Both must update consistently or the bucket's accounting drifts.

## The happy path, traced before coding

The last planning artefact is a trace of one request through the entire system. If you cannot narrate this path before coding, you will discover the gaps while coding, which is the expensive place to discover them:

```
Request arrives
  → WithRetry wraps the operation
    → LoadBalancer.Next()
      → TokenBucket.Allow()? No → throw RateLimitException → retry with backoff
      → Iterate backends round-robin
        → IsHealthy && CircuitBreaker.Allow()? No → try next backend
        → Yes → return this backend
      → No healthy backend found → return null → throw → retry
    → SimulateRequest(backend)
      → success → RecordSuccess() → return true
      → failure → RecordFailure() → throw → retry
  → All attempts exhausted → re-throw to caller
```

Notice the ordering decision hiding in plain sight: the token bucket is checked *before* iterating backends. Why? Because the rate limit is global — it protects the whole system, not any one backend. If a request is over budget, there is no point evaluating backend health at all. Checking it first also means a rate-limited request consumes zero per-backend state, which keeps the circuit breakers honest: they should only ever see requests that were actually allowed through.

## A detour worth taking: the leaky bucket

Before implementing, it is worth completing the rate limiting picture, because "which rate limiter would you use and why" is one of the most common follow-up questions in this space — and most candidates only know two of the three answers.

A **sliding window** answers the question *"how many requests happened in the last W seconds?"* It keeps a timestamp per request, evicts old ones, and rejects anything over the count. Strict, simple, O(n) memory, and it has no memory of idle time — being quiet for an hour earns you nothing.

A **token bucket** answers *"do I have a token right now?"* Tokens refill at a fixed rate up to a cap, and each request spends one. O(1) memory. Its unique knob is the decoupling of burst capacity (`maxTokens`) from sustained rate (`refillRate`) — set the cap higher than the per-second rate and idle time accumulates into a burst allowance. The sliding window has no equivalent setting.

A **leaky bucket** answers a different question entirely: *"when should this request be processed?"* Requests pour into a queue at any rate; a background processor drains the queue at one fixed, constant rate. Excess requests are not rejected — they are *delayed*. Rejection only happens when the queue itself is full.

```
Requests pour in (any rate)
        ↓ ↓ ↓ ↓ ↓
    [  queue  ]  ← max capacity
        |
        | ← fixed drain rate (e.g. 1 req/100ms)
        ↓
    processed at constant rate
```

So which one do you actually pick? The honest framing is that the rate limit itself is not the differentiator — all three can enforce "100 requests per second." What differs is *what happens to the excess* and *whether idle time is rewarded*. Run the same scenario through all three:

```
Scenario: limit = 3/sec, burst of 6 arrives at once

Sliding window  → allows 3, rejects 3 immediately
Token bucket    → allows 3, rejects 3 immediately (with maxTokens = 3)
Leaky bucket    → queues all 6, processes one every 333ms — no rejection, no burst
```

The interview-ready answer: *if excess requests should fail fast, use a sliding window or token bucket. If excess should wait rather than fail, and output smoothness is critical — say, calling a third-party API with a hard rate limit that must never see a burst — use a leaky bucket, and accept the tradeoffs: added latency for queued requests and O(n) memory proportional to queue depth.* The leaky bucket is less a rate limiter and more a traffic shaper.

For this system, fail-fast is the right behaviour — a client waiting in a hidden queue is worse than a client that gets a fast rejection and retries with backoff — so the token bucket wins.

## The implementation

With the plan locked, the code falls out quickly. Each class below comes with the design notes that an interviewer expects you to volunteer unprompted.

### Backend

```csharp
namespace backendpool;

public class Backend
{
    private volatile bool _healthy = true;
    public CircuitBreaker CircuitBreaker { get; } = new CircuitBreaker(3, TimeSpan.FromSeconds(3));
    public bool IsHealthy => _healthy;
    public void SetHealthy(bool val) => _healthy = val;
    public string Address { get; }
    public Backend(string address) => Address = address;
}
```

Each backend owns its **own** circuit breaker. Why not one global breaker on the load balancer? Because then one flaky backend's failures would open the circuit for everyone — the two healthy backends would be punished for the sick one's behaviour. The circuit breaker's whole job is to isolate failure *per dependency*; sharing it defeats the purpose. `Address` is a get-only property — set once in the constructor, immutable after. And `_healthy` is `volatile` for the reasons established in the audit above.

### CircuitBreaker

```csharp
namespace backendpool;

public enum CircuitState { Open, Closed, HalfOpen }

public class CircuitBreaker
{
    private readonly int _threshold;
    private readonly TimeSpan _timeout;
    private DateTime _lastFailureTime;
    private int _failedRequests;
    private CircuitState _circuitBreakerState = CircuitState.Closed;
    private readonly object _lock = new();

    public CircuitBreaker(int threshold, TimeSpan timeout)
    {
        _threshold = threshold;
        _timeout = timeout;
    }

    public bool Allow()
    {
        lock (_lock)
        {
            return _circuitBreakerState switch
            {
                CircuitState.Closed => true,
                CircuitState.Open when _timeout < DateTime.UtcNow - _lastFailureTime
                    => TransitionState(CircuitState.HalfOpen),
                CircuitState.Open => false,
                _ => false   // HalfOpen — reject while probe is in flight
            };
        }
    }

    public void RecordSuccess()
    {
        lock (_lock)
        {
            _failedRequests = 0;
            _circuitBreakerState = CircuitState.Closed;
        }
    }

    public void RecordFailure()
    {
        lock (_lock)
        {
            _lastFailureTime = DateTime.UtcNow;
            _failedRequests++;
            if (_failedRequests >= _threshold)
                _circuitBreakerState = CircuitState.Open;
        }
    }

    private bool TransitionState(CircuitState circuitState)
    {
        // called from inside Allow() which already holds _lock — no nested lock needed
        _circuitBreakerState = circuitState;
        return true;
    }
}
```

The state machine, in words:

```
CLOSED   --[failures >= threshold]--→ OPEN
OPEN     --[timeout elapsed]--------→ HALF_OPEN  (returns true — one probe request)
HALF_OPEN --[probe succeeds]--------→ CLOSED     (via RecordSuccess)
HALF_OPEN --[probe fails]-----------→ OPEN       (via RecordFailure)
```

The subtlest part is the **half-open probe**. When `Allow()` finds the circuit open *and* the timeout elapsed, it transitions to half-open and returns `true` — that one request *is* the probe. Every other call arriving while the probe is in flight falls through to `_ => false` and is rejected. The probe's outcome then decides everything: `RecordSuccess` closes the circuit, `RecordFailure` reopens it. One request risks itself so the rest don't have to.

Why is `TransitionState` private and lock-free? Because it is only ever called from inside `Allow()`, which already holds `_lock`. C# locks are reentrant, so an inner `lock (_lock)` would not deadlock — but it would be redundant work and, worse, misleading to a reader who assumes the method is safe to call from anywhere. The privacy is the documentation.

Why `DateTime.UtcNow` and never `DateTime.Now`? `Now` is local time, and local time is not monotonic — it jumps backwards an hour when Daylight Saving Time ends. A circuit breaker timing out based on `Now` can decide that a failure from two minutes ago happened *in the future*. `UtcNow` never lies about elapsed time. (Production code would go one step further to `Stopwatch`/monotonic clocks, since even UTC can shift under NTP correction — worth mentioning aloud if asked.)

One limitation to call out proactively, because it earns real credit: this breaker counts **consecutive** failures, and a single success resets the count to zero. A backend that alternates fail-success-fail-success will never trip it, even though half its requests are failing. Production-grade libraries like Polly track failure *rate within a sliding time window* instead — which, satisfyingly, is the same `Queue<DateTime>` data structure as the sliding window rate limiter, just repurposed per backend. The interview line: *"I'm using consecutive failures for simplicity; production would track failure rate within a window to catch intermittent failures."*

And the question everyone forgets to answer: **who calls `RecordSuccess` and `RecordFailure`?** The call site — the code that actually made the request and saw the outcome. Not the load balancer (it only picks backends; it never learns what happened). And emphatically not the health checker — its probes are synthetic, and a synthetic probe succeeding tells you nothing about whether real traffic would succeed. Letting probes drive circuit state would let a trivial health endpoint mask a failing business endpoint. Real request outcomes, and only real request outcomes, drive circuit state.

### TokenBucket

```csharp
namespace backendpool;

public class TokenBucket
{
    private readonly object _lock = new();
    private readonly double _refillRate;
    private readonly int _maxTokens;
    private double _tokens;
    private DateTime _lastRefill = DateTime.UtcNow;

    public TokenBucket(double refillRate, int maxTokens)
    {
        _refillRate = refillRate;
        _maxTokens = maxTokens;
        _tokens = maxTokens;
    }

    public bool Allow()
    {
        lock (_lock)
        {
            var now = DateTime.UtcNow;
            var elapsed = (now - _lastRefill).TotalSeconds;
            _lastRefill = now;
            _tokens += elapsed * _refillRate;
            _tokens = Math.Min(_maxTokens, _tokens);

            if (_tokens < 1) return false;
            _tokens--;
            return true;
        }
    }
}
```

Three decisions hide in these twenty lines, and each one is a classic bug when missed.

First: **`_tokens` is a `double`, not an `int`.** At sub-second call intervals, `elapsed * _refillRate` produces fractional tokens — 50 milliseconds at 10 tokens/sec is 0.5 tokens. Cast that to `int` and it truncates to zero, every time, forever. The bucket never refills at sub-second granularity and the rate limiter silently becomes a denial-of-service machine against your own traffic. `double` preserves the fractions and they accumulate correctly.

Second: **one `now` variable.** Calling `DateTime.UtcNow` twice — once for the elapsed calculation, once to update `_lastRefill` — captures two slightly different moments. The gap between them is time that gets double-counted or lost on every call, and it accumulates. Capture `now` once, use it for both.

Third: **there is no background refill thread.** Tokens are refilled *lazily*, computed from elapsed time at the moment of each `Allow()` call. This is O(1), allocation-free, and needs no timer, no thread, no shutdown logic. Lazy computation from timestamps beats eager background work whenever you can get away with it.

And the placement question: the bucket lives on the **LoadBalancer**, not on each backend. The limit being enforced is global — *the system* accepts N requests per second. If each backend had its own bucket, a request rejected by backend A's bucket could simply be routed to backend B, and the global limit would be unenforceable by construction.

### LoadBalancer

```csharp
namespace backendpool;

public class LoadBalancer
{
    private readonly ReaderWriterLockSlim _rwLock = new();
    private readonly List<Backend> _backends;
    private readonly TokenBucket _tokenBucket;
    private int _counter;

    public LoadBalancer(List<Backend> backends, TokenBucket tokenBucket)
    {
        _backends = backends;
        _tokenBucket = tokenBucket;
    }

    public Backend? Next()
    {
        _rwLock.EnterReadLock();
        try
        {
            if (!_tokenBucket.Allow())
            {
                Console.WriteLine("LoadBalancer.Next(): Token bucket empty");
                return null;
            }

            int totalBackends = _backends.Count;
            for (int i = 0; i < totalBackends; i++)
            {
                var idx = Math.Abs(Interlocked.Increment(ref _counter)) % totalBackends;
                var b = _backends[idx];
                if (b.IsHealthy && b.CircuitBreaker.Allow())
                    return b;
            }
            return null;
        }
        finally { _rwLock.ExitReadLock(); }
    }

    public void AddBackend(Backend b)
    {
        _rwLock.EnterWriteLock();
        try { _backends.Add(b); }
        finally { _rwLock.ExitWriteLock(); }
    }
}
```

**`readonly` means less than you think — and exactly enough.** `readonly` on a reference type freezes the *reference*, not the contents. The list can still be mutated; what cannot happen is an accidental `_backends = new List<Backend>()` somewhere deep in a method, which would silently orphan every registered backend. Cheap insurance, zero cost.

**`Math.Abs()` is not paranoia.** `_counter` is an `int` incremented on every request. It *will* overflow — at 10,000 requests/sec, in about 2.5 days — and wrap negative. In C#, a negative number modulo n yields a *negative* result, so `_backends[idx]` throws `IndexOutOfRangeException`. In production. At 3 a.m. On day three. `Math.Abs()` makes the index non-negative regardless of overflow.

**`finally` is the difference between a bug and an outage.** Put `ExitReadLock()` after the `return` statement and it never runs — `return` leaves the method immediately. Now the read lock is held forever, and the *next* `AddBackend()` call blocks eternally waiting for its write lock. The system does not crash; it just quietly stops accepting new backends. `finally` guarantees release on every exit path: normal return, early return, exception.

**Why are unhealthy backends never removed from the list?** This question separates candidates more reliably than any other in this problem. Removal causes three distinct failures. One: there is no recovery path — a removed backend cannot be routed to when it comes back, so a thirty-second blip becomes a permanent eviction. Two: removal shifts every subsequent index, corrupting the round-robin counter's relationship to the list. Three: every health flap becomes a *write* lock acquisition, stalling all concurrent reads — the hot path pays for the health checker's churn. The correct pattern: mark unhealthy, skip in `Next()`, mark healthy again when the probe recovers. Membership changes rarely; health changes constantly; keep them on separate mechanisms.

**One nested-lock subtlety worth saying out loud:** `Next()` calls `_tokenBucket.Allow()` while holding the read lock, and `Allow()` takes the bucket's own lock — two locks held at once. Safe here, because the acquisition order is globally consistent (always RWLock first, bucket lock second). If any other code path ever acquired them in reverse order, that is a textbook deadlock. Consistent lock ordering is the discipline that makes nested locks survivable.

### HealthChecker

```csharp
namespace backendpool;

public class HealthChecker
{
    private readonly List<Backend> _backends;
    private readonly int _delayIntervalInMs;

    public HealthChecker(List<Backend> backends, int delayIntervalInMs)
    {
        _backends = backends;
        _delayIntervalInMs = delayIntervalInMs;
    }

    public async Task RunHealthCheckLoop(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            await Task.Delay(_delayIntervalInMs, cancellationToken);
            foreach (var backend in _backends)
            {
                var isHealthy = SimulateHealthCheck(backend);
                backend.SetHealthy(isHealthy);
            }
        }
    }

    private bool SimulateHealthCheck(Backend backend)
    {
        var isAlive = Random.Shared.Next(0, 2);
        return isAlive == 1;
    }
}
```

**Probe every backend, especially the unhealthy ones.** It sounds obvious written down, but "skip unhealthy backends to save probe traffic" is a tempting optimisation that quietly removes the only path by which a downed backend ever returns to rotation. If you only probe the healthy, down means down forever.

**Pass the `CancellationToken` into `Task.Delay`.** Without it, a shutdown signal has to wait out the full sleep interval before the loop even notices — a one-minute interval means a one-minute hang on every deploy. With it, `cts.Cancel()` wakes the delay immediately via `OperationCanceledException` and the loop exits cleanly. Cancellable sleep is the difference between graceful shutdown and `kill -9`.

**`Random.Next(0, 2)`, not `(0, 1)`.** The upper bound is *exclusive*. `Next(0, 1)` returns 0 every single time — which here means every backend is permanently unhealthy, and the symptom (no backend ever available) points everywhere except the actual bug.

### RetryHelper

```csharp
namespace backendpool;

public static class RetryHelper
{
    public static async Task<T> WithRetry<T>(
        Func<Task<T>> operation,
        int maxAttempts,
        CancellationToken cancellationToken)
    {
        var backOffTime = TimeSpan.FromMilliseconds(100);

        for (int i = 0; i < maxAttempts; i++)
        {
            try { return await operation(); }
            catch when (maxAttempts - 1 == i) { throw; }
            catch
            {
                var delay = backOffTime +
                    TimeSpan.FromMilliseconds(
                        Random.Shared.Next(
                            -(int)(backOffTime.TotalMilliseconds * 0.25),
                             (int)(backOffTime.TotalMilliseconds * 0.25)));

                await Task.Delay(delay, cancellationToken);
                backOffTime = TimeSpan.FromMilliseconds(
                    Math.Min(backOffTime.TotalMilliseconds * 2, 30_000));
            }
        }
        throw new InvalidOperationException("Unreachable Code");
    }
}
```

**Jitter is not optional.** Picture an outage: a backend dies, fifty clients fail simultaneously, all fifty back off exactly 100ms, and all fifty retry *at the same instant* — a synchronised wave that hits the recovering backend like the original burst, knocking it straight back down. This is the thundering herd, and exponential backoff alone does not fix it; it just synchronises the waves at longer intervals. The ±25% random jitter desynchronises the clients so the recovering backend sees a staggered trickle instead of a wall.

**`Random.Shared` exists for a reason.** It is the thread-safe singleton added in .NET 6. Before it, sharing one `Random` instance across threads produced corrupted internal state — correlated values, or sequences of zeros. Multiple threads retrying concurrently is exactly the scenario that breaks pre-.NET-6 `Random`.

**Cap the backoff.** Doubling without a ceiling is an exponential, and exponentials do what exponentials do: 100ms doubled fifteen times is roughly 54 *minutes*. A retry an hour later is not a retry, it is archaeology. The 30-second cap keeps late attempts useful.

**The unreachable throw is for the compiler, not the runtime.** Every loop iteration either returns or throws — the `catch when (maxAttempts - 1 == i)` filter guarantees the last attempt rethrows. But the compiler cannot prove that, so it demands all code paths return a value. The final `throw new InvalidOperationException("Unreachable Code")` satisfies the type checker and can never execute.

And the design decision that ties the whole system together: **rate limiting must throw, not return false.** Suppose `Next()` returns `null` because the bucket is empty, and the caller translates that into `return false`. From `WithRetry`'s perspective, the operation *completed* — no exception means success, so it stops retrying. The request was never actually made, no circuit breaker recorded anything, and the caller received a quiet, misleading `false`. A rate-limited request is a *failed attempt* and the retry machinery only understands failure as exceptions — so throw, let the backoff absorb the burst, and let the retry land when tokens have refilled. This is the contract between the layers, and breaking it makes the failure invisible, which is the worst kind of failure.

## The bugs that actually happened

Every one of these was caught during review of this exact implementation. They are worth memorising not as trivia but as *categories* — each represents a class of mistake that recurs across systems.

**The silent enum typo.** `HealhOpen` instead of `HalfOpen`. The compiler catches it if you reference the right name elsewhere — but during fast typing, a consistent typo compiles fine and produces a state machine with a misspelt state. Category: names the compiler cannot save you from.

**Lock-free `RecordSuccess`/`RecordFailure`.** The first draft incremented `_failedRequests` without a lock. Two threads failing simultaneously lose increments, and the circuit opens late or never. Category: "it's just an increment" — there is no such thing as *just* an increment under concurrency.

**Off-by-one in the threshold.** `>` instead of `>=` means a threshold of 3 opens the circuit on the *fourth* failure. Category: boundary conditions on counters.

**The redundant nested lock.** `TransitionState` originally took `_lock` even though every caller already held it. Reentrancy means no deadlock in C# — but it is wasted work and a misleading signal about the method's contract. Category: know your lock's reentrancy semantics, then design so you don't depend on them.

**`int` tokens.** Covered above; the bucket that never refills. Category: integer truncation in rate/time arithmetic.

**Two `UtcNow` calls.** Covered above; the accumulating gap. Category: capture time once per logical operation.

**The missing `Math.Abs()`.** Covered above; the negative-modulo crash that takes days of uptime to manifest. Category: overflow is not hypothetical, it is scheduled.

**The string concatenation address bug.** `"192.168.0" + i + 1` produces `"192.168.001"`, `"192.168.011"` — because `+` left-associates and concatenates `i` before ever considering the `1`. `$"192.168.0.{i + 1}"` produces what was meant. Category: operator precedence in string building; interpolation removes the ambiguity.

**The oversized token bucket.** `new TokenBucket(1000, 10000)` against 500 test requests means the bucket never empties and the rate limiting code path is never exercised — it could be completely broken and the demo would look perfect. Sizing it to `(100, 100)` made the limiter visibly trigger under retry load. Category: test parameters that cannot reach the code you wrote.

**Returning `false` on rate limit.** Covered above; the retry machinery silently declaring victory. Category: error signalling contracts between layers.

## Reading the logs: what correct looks like

After the fixes, the running system produced logs with five recognisable signatures. Learning to *read* these patterns is how you verify resilience behaviour without a debugger.

The circuit opening at exactly the threshold:

```
RecordFailure(): _failedRequests: 1, _circuitBreakerState: Closed
RecordFailure(): _failedRequests: 2, _circuitBreakerState: Closed
RecordFailure(): _failedRequests: 3, _circuitBreakerState: Open   ← opens at exactly 3
```

A success resetting it:

```
RecordSuccess(): _failedRequests: 0, _circuitBreakerState: Closed
```

The token bucket saturating in blocks — rejections cluster while a retry burst outpaces the refill, then routing resumes as tokens trickle back:

```
Token bucket empty
Token bucket empty
...
LoadBalancer.Next(): IP: 192.168.0.3   ← resumes when tokens refill
```

Retries exhausting:

```
Attempt: 5, System.InvalidOperationException: Backend Request Failed!
```

And traffic correctly skewing: when the health checker marks two of three backends unhealthy, every request routes to the survivor — the round-robin loop tries each index, skips the unhealthy, and lands on the one that passes.

One log pattern looks like a bug but is not: occasionally a request routes to a backend whose circuit *just* opened. That is the inherent race between `Allow()` returning true on one thread and `RecordFailure()` flipping the state on another, microseconds later. Could you eliminate it? Only by holding the breaker's lock across the entire downstream request — serialising all traffic through every backend, which is a cure categorically worse than the disease. The circuit breaker is **probabilistic protection, not a hard gate**: it stops the flood, and a few drops through the closing door are the accepted cost. Saying that sentence, unprompted, is the moment an interviewer relaxes.

## The full picture

Everything above, on one diagram. This is the drawing to reproduce on the whiteboard — and the concurrency annotations in the corners are the things to say *while* drawing it.

<style>
.bp-wrap { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; margin: 2rem 0; }
.bp-arch { width: 100%; height: auto; display: block; background: #fbfaf7; border: 1px solid #e2ddd3; border-radius: 12px; }
.bp-card { background: #fbfaf7; border: 1px solid #e2ddd3; border-radius: 12px; padding: 1.25rem 1.5rem; margin: 1.5rem 0; }
.bp-card h4 { margin: 0 0 .25rem 0; font-size: 1.05rem; color: #1f2933; }
.bp-card .bp-sub { font-size: .85rem; color: #6b7280; margin-bottom: 1rem; }
.bp-row { display: flex; gap: .5rem; flex-wrap: wrap; align-items: center; }
.bp-btn { border: 1.5px solid #1f2933; background: #fff; color: #1f2933; font: inherit; font-size: .85rem; font-weight: 600; padding: .45rem .9rem; border-radius: 8px; cursor: pointer; transition: transform .08s ease, background .15s ease; }
.bp-btn:hover { background: #1f2933; color: #fff; }
.bp-btn:active { transform: scale(.96); }
.bp-btn:disabled { opacity: .35; cursor: not-allowed; }
.bp-btn.bp-ok { border-color: #15803d; color: #15803d; } .bp-btn.bp-ok:hover { background: #15803d; color:#fff; }
.bp-btn.bp-bad { border-color: #b91c1c; color: #b91c1c; } .bp-btn.bp-bad:hover { background: #b91c1c; color:#fff; }
.bp-statebox { display: inline-flex; align-items: center; gap: .5rem; padding: .5rem 1rem; border-radius: 999px; font-weight: 700; font-size: .9rem; letter-spacing: .03em; border: 2px solid; transition: all .25s ease; }
.bp-closed { color: #15803d; border-color: #15803d; background: #f0fdf4; }
.bp-open { color: #b91c1c; border-color: #b91c1c; background: #fef2f2; }
.bp-half { color: #b45309; border-color: #b45309; background: #fffbeb; }
.bp-dot { width: 10px; height: 10px; border-radius: 50%; background: currentColor; }
.bp-log { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: .78rem; background: #1f2933; color: #d1fae5; border-radius: 8px; padding: .75rem 1rem; margin-top: .9rem; height: 130px; overflow-y: auto; line-height: 1.55; }
.bp-log .bp-l-fail { color: #fca5a5; } .bp-log .bp-l-warn { color: #fcd34d; } .bp-log .bp-l-info { color: #93c5fd; }
.bp-meter { height: 26px; background: #eceae4; border-radius: 8px; overflow: hidden; position: relative; margin: .6rem 0; border: 1px solid #d6d1c4; }
.bp-meter-fill { height: 100%; background: linear-gradient(90deg, #0e7490, #0891b2); transition: width .2s linear; border-radius: 8px 0 0 8px; }
.bp-meter-label { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-size: .78rem; font-weight: 700; color: #1f2933; }
.bp-cols { display: grid; grid-template-columns: repeat(3, 1fr); gap: .8rem; margin-top: 1rem; }
@media (max-width: 640px) { .bp-cols { grid-template-columns: 1fr; } }
.bp-col { background: #fff; border: 1px solid #e2ddd3; border-radius: 10px; padding: .8rem; }
.bp-col h5 { margin: 0 0 .15rem 0; font-size: .85rem; color: #1f2933; }
.bp-col .bp-tag { font-size: .7rem; color: #6b7280; display: block; margin-bottom: .6rem; }
.bp-slots { display: flex; gap: 5px; flex-wrap: wrap; min-height: 26px; }
.bp-slot { width: 22px; height: 22px; border-radius: 6px; background: #eceae4; border: 1px solid #d6d1c4; transition: all .25s ease; }
.bp-slot.bp-pass { background: #22c55e; border-color: #15803d; }
.bp-slot.bp-rej { background: #ef4444; border-color: #b91c1c; }
.bp-slot.bp-wait { background: #fde68a; border-color: #d97706; }
.bp-verdict { font-size: .74rem; margin-top: .55rem; color: #374151; min-height: 2em; }
.bp-grid2 { display: grid; grid-template-columns: repeat(auto-fit, minmax(230px, 1fr)); gap: .8rem; }
.bp-flip { background: #fff; border: 1px solid #e2ddd3; border-radius: 10px; padding: .9rem 1rem; cursor: pointer; transition: border-color .15s ease, box-shadow .15s ease; }
.bp-flip:hover { border-color: #0891b2; box-shadow: 0 2px 8px rgba(8,145,178,.12); }
.bp-flip .bp-q { font-weight: 700; font-size: .88rem; color: #1f2933; }
.bp-flip .bp-mech { font-family: ui-monospace, Menlo, monospace; font-size: .76rem; color: #0e7490; margin-top: .2rem; }
.bp-flip .bp-why { font-size: .78rem; color: #374151; margin-top: .55rem; line-height: 1.5; display: none; }
.bp-flip.bp-openf .bp-why { display: block; }
.bp-flip .bp-hint { font-size: .68rem; color: #9ca3af; margin-top: .45rem; }
</style>

<div class="bp-wrap">
<svg class="bp-arch" viewBox="0 0 860 640" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="BackendPool architecture diagram">
  <defs>
    <marker id="bpArr" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0,0 L9,4.5 L0,9 z" fill="#1f2933"/></marker>
    <marker id="bpArrG" markerWidth="9" markerHeight="9" refX="7" refY="4.5" orient="auto"><path d="M0,0 L9,4.5 L0,9 z" fill="#15803d"/></marker>
  </defs>
  <style>
    .bx { fill:#ffffff; stroke:#1f2933; stroke-width:1.6; rx:10; }
    .bxs { fill:#f0f9ff; stroke:#0e7490; stroke-width:1.4; }
    .t { font:700 15px ui-monospace,Menlo,monospace; fill:#1f2933; }
    .ts { font:600 11.5px ui-monospace,Menlo,monospace; fill:#0e7490; }
    .tn { font:11.5px -apple-system,'Segoe UI',sans-serif; fill:#6b7280; }
    .ln { stroke:#1f2933; stroke-width:1.6; fill:none; marker-end:url(#bpArr); }
    .lng { stroke:#15803d; stroke-width:1.6; stroke-dasharray:6 5; fill:none; marker-end:url(#bpArrG); }
  </style>

  <rect class="bx" x="280" y="18" width="300" height="46"/>
  <text class="t" x="430" y="46" text-anchor="middle">50 concurrent clients</text>

  <line class="ln" x1="430" y1="64" x2="430" y2="96"/>

  <rect class="bx" x="295" y="98" width="270" height="58"/>
  <text class="t" x="430" y="122" text-anchor="middle">WithRetry</text>
  <text class="ts" x="430" y="142" text-anchor="middle">5 attempts · exp backoff · ±25% jitter</text>

  <line class="ln" x1="430" y1="156" x2="430" y2="188"/>

  <rect class="bx" x="255" y="190" width="350" height="76"/>
  <text class="t" x="430" y="216" text-anchor="middle">LoadBalancer</text>
  <text class="ts" x="430" y="236" text-anchor="middle">RWLock (list) · Interlocked (counter)</text>
  <text class="ts" x="430" y="254" text-anchor="middle">round-robin</text>

  <line class="ln" x1="320" y1="266" x2="170" y2="316"/>
  <text class="tn" x="200" y="284">over budget?</text>
  <rect class="bxs" x="48" y="318" width="244" height="72" rx="10"/>
  <text class="t" x="170" y="344" text-anchor="middle">TokenBucket</text>
  <text class="ts" x="170" y="362" text-anchor="middle">global · lock · lazy refill</text>
  <text class="ts" x="170" y="379" text-anchor="middle">empty → THROW → retry</text>

  <line class="ln" x1="500" y1="266" x2="600" y2="316"/>
  <text class="tn" x="560" y="284">healthy &amp;&amp; CB.Allow()?</text>

  <rect class="bx" x="380" y="318" width="440" height="200"/>
  <text class="t" x="600" y="344" text-anchor="middle">Backend ×3 (each owns its state)</text>

  <rect class="bxs" x="404" y="362" width="186" height="64" rx="10"/>
  <text class="t" x="497" y="386" text-anchor="middle" style="font-size:13.5px">CircuitBreaker</text>
  <text class="ts" x="497" y="404" text-anchor="middle">lock · Closed/Open/Half</text>

  <rect class="bxs" x="610" y="362" width="186" height="64" rx="10"/>
  <text class="t" x="703" y="386" text-anchor="middle" style="font-size:13.5px">IsHealthy</text>
  <text class="ts" x="703" y="404" text-anchor="middle">volatile bool</text>

  <rect class="bx" x="430" y="448" width="340" height="54" style="stroke:#0e7490"/>
  <text class="t" x="600" y="470" text-anchor="middle" style="font-size:13.5px">SimulateRequest</text>
  <text class="ts" x="600" y="488" text-anchor="middle">ok → RecordSuccess · fail → RecordFailure → throw</text>

  <line class="ln" x1="497" y1="426" x2="540" y2="446"/>
  <line class="ln" x1="703" y1="426" x2="660" y2="446"/>

  <rect x="48" y="548" width="772" height="74" rx="10" fill="#f0fdf4" stroke="#15803d" stroke-width="1.5" stroke-dasharray="7 5"/>
  <text class="t" x="434" y="574" text-anchor="middle" style="fill:#15803d">HealthChecker · background · every 1s · CancellationToken</text>
  <text class="ts" x="434" y="594" text-anchor="middle" style="fill:#166534">probes ALL backends → SetHealthy(bool)</text>
  <text class="ts" x="434" y="611" text-anchor="middle" style="fill:#166534">never calls RecordSuccess / RecordFailure — synthetic probes must not drive circuit state</text>
  <line class="lng" x1="703" y1="546" x2="703" y2="430"/>
</svg>
</div>

The narration that goes with it, condensed to five lines — one per shared variable:

- `IsHealthy` → **volatile**: lock-free read on the hot path, written rarely by one background thread.
- Round-robin counter → **Interlocked**: atomic hardware increment, no lock, memory barrier included.
- Backends list → **ReaderWriterLockSlim**: parallel reads every request, exclusive write on rare membership change.
- CircuitBreaker → **lock**: compound multi-field state transitions must be atomic.
- TokenBucket → **lock**: read-modify-write on token accounting must be atomic.

## Interactive quick reference

Reading about these patterns builds familiarity; *operating* them builds intuition. The widgets below are the live versions of everything above — two minutes here before an interview or design review is the fastest revision you can do.

### Break the circuit yourself

Threshold is 3 consecutive failures; open timeout is 5 seconds. Fail it three times, watch it open, wait out the timeout, then watch one probe decide its fate.

<div class="bp-wrap bp-card" id="bpCB">
  <h4>Circuit breaker simulator</h4>
  <div class="bp-sub">threshold = 3 · timeout = 5s · half-open allows exactly one probe</div>
  <div class="bp-row">
    <span class="bp-statebox bp-closed" id="bpCBState"><span class="bp-dot"></span><span id="bpCBStateText">CLOSED</span></span>
    <span style="font-size:.82rem;color:#6b7280;">failures: <b id="bpCBFails">0</b>/3 <span id="bpCBTimer"></span></span>
  </div>
  <div class="bp-row" style="margin-top:.8rem;">
    <button class="bp-btn bp-ok" id="bpCBOk">Request succeeds</button>
    <button class="bp-btn bp-bad" id="bpCBBad">Request fails</button>
    <button class="bp-btn" id="bpCBReset">Reset</button>
  </div>
  <div class="bp-log" id="bpCBLog"><span class="bp-l-info">// send a request to begin</span></div>
</div>

### Drain the token bucket

Capacity 5, refilling at 1 token per second — continuously, in fractions, with no background thread (the page recomputes from elapsed time, exactly like the lazy refill in `Allow()`). Spam the button and watch fail-fast behaviour, then idle and watch the burst allowance rebuild.

<div class="bp-wrap bp-card" id="bpTB">
  <h4>Token bucket simulator</h4>
  <div class="bp-sub">maxTokens = 5 · refill = 1/sec · tokens are a <b>double</b> — watch the fractions</div>
  <div class="bp-meter"><div class="bp-meter-fill" id="bpTBFill" style="width:100%"></div><div class="bp-meter-label" id="bpTBLabel">5.00 / 5 tokens</div></div>
  <div class="bp-row">
    <button class="bp-btn" id="bpTBSend">Send request</button>
    <button class="bp-btn" id="bpTBBurst">Send burst of 8</button>
    <span id="bpTBResult" style="font-size:.85rem;font-weight:700;"></span>
  </div>
  <div class="bp-row" style="margin-top:.6rem;font-size:.78rem;color:#6b7280;">
    allowed: <b id="bpTBOk" style="color:#15803d;">&nbsp;0</b>&nbsp;·&nbsp;rejected: <b id="bpTBRej" style="color:#b91c1c;">&nbsp;0</b>
  </div>
</div>

### One burst, three rate limiters

The same burst of 6 against a limit of 3/sec. Watch *where the excess goes* — that is the entire difference between the algorithms.

<div class="bp-wrap bp-card" id="bpRL">
  <h4>Rate limiter showdown</h4>
  <div class="bp-sub">limit = 3 requests/sec · burst of 6 arrives at t=0</div>
  <div class="bp-row"><button class="bp-btn" id="bpRLGo">Send burst of 6</button></div>
  <div class="bp-cols">
    <div class="bp-col"><h5>Sliding window</h5><span class="bp-tag">counts last W seconds · O(n)</span><div class="bp-slots" id="bpRLSW"></div><div class="bp-verdict" id="bpRLSWv"></div></div>
    <div class="bp-col"><h5>Token bucket</h5><span class="bp-tag">spend a token now · O(1)</span><div class="bp-slots" id="bpRLTB"></div><div class="bp-verdict" id="bpRLTBv"></div></div>
    <div class="bp-col"><h5>Leaky bucket</h5><span class="bp-tag">queue + fixed drain · shaper</span><div class="bp-slots" id="bpRLLB"></div><div class="bp-verdict" id="bpRLLBv"></div></div>
  </div>
</div>

### Which primitive protects what?

Tap each card. If you can answer before flipping, you are ready for the follow-up questions.

<div class="bp-wrap bp-card">
  <h4>Concurrency primitive picker</h4>
  <div class="bp-sub">five shared variables, four mechanisms — and the why behind each</div>
  <div class="bp-grid2" id="bpPick">
    <div class="bp-flip"><div class="bp-q">Single bool, one writer, many readers</div><div class="bp-mech">volatile</div><div class="bp-why">A lock is disproportionate for one bool. <b>volatile</b> stops the JIT register-caching the value across loop iterations; the hardware (MESI) keeps the actual read fresh. Compiler contract + hardware contract, both needed.</div><div class="bp-hint">tap for why ▾</div></div>
    <div class="bp-flip"><div class="bp-q">Hot-path counter, incremented every request</div><div class="bp-mech">Interlocked.Increment</div><div class="bp-why"><code>_counter++</code> is read+add+write — three interleavable steps. Interlocked is one atomic instruction (LOCK XADD), no lock, no context switch, memory barrier included. And remember <code>Math.Abs()</code> — the int will overflow negative.</div><div class="bp-hint">tap for why ▾</div></div>
    <div class="bp-flip"><div class="bp-q">List read every request, written rarely</div><div class="bp-mech">ReaderWriterLockSlim</div><div class="bp-why">Plain lock serialises all readers behind each other for no reason. RWLock lets reads run in parallel and only blocks for the rare write. Read-heavy, write-rare is its exact use case. Release in <b>finally</b>, always.</div><div class="bp-hint">tap for why ▾</div></div>
    <div class="bp-flip"><div class="bp-q">Multiple fields that change together</div><div class="bp-mech">lock</div><div class="bp-why">Circuit breaker state + failure count + timestamp must transition as one unit. Interlocked covers one variable; volatile covers zero atomicity. Compound invariants need mutual exclusion — lock is the correct tool, not the lazy one.</div><div class="bp-hint">tap for why ▾</div></div>
    <div class="bp-flip"><div class="bp-q">Read-modify-write on token accounting</div><div class="bp-mech">lock</div><div class="bp-why">Refill (from elapsed time) and spend must be consistent with <code>_lastRefill</code>. Two fields, one logical operation → lock. Inside: <b>double</b> tokens (int truncates fractions to zero) and a <b>single</b> <code>now</code> variable.</div><div class="bp-hint">tap for why ▾</div></div>
    <div class="bp-flip"><div class="bp-q">Background loop that must shut down cleanly</div><div class="bp-mech">CancellationToken</div><div class="bp-why">Pass the token <b>into</b> <code>Task.Delay</code>. Without it, shutdown waits out the full sleep interval. With it, Cancel() wakes the delay instantly via OperationCanceledException and the loop exits — graceful shutdown instead of kill -9.</div><div class="bp-hint">tap for why ▾</div></div>
  </div>
</div>

<script>
(function () {
  /* ---------- circuit breaker ---------- */
  var cbState = 'CLOSED', cbFails = 0, cbOpenedAt = 0, cbTimerIv = null;
  var THRESH = 3, TIMEOUT = 5000;
  var $ = function (id) { return document.getElementById(id); };
  function cbLog(msg, cls) {
    var el = $('bpCBLog'), line = document.createElement('div');
    if (cls) line.className = cls;
    line.textContent = msg;
    el.appendChild(line); el.scrollTop = el.scrollHeight;
  }
  function cbRender() {
    var box = $('bpCBState'), txt = $('bpCBStateText');
    box.className = 'bp-statebox ' + (cbState === 'CLOSED' ? 'bp-closed' : cbState === 'OPEN' ? 'bp-open' : 'bp-half');
    txt.textContent = cbState;
    $('bpCBFails').textContent = cbFails;
  }
  function cbTick() {
    if (cbState !== 'OPEN') { $('bpCBTimer').textContent = ''; return; }
    var left = TIMEOUT - (Date.now() - cbOpenedAt);
    $('bpCBTimer').textContent = left > 0 ? '· retry window opens in ' + (left / 1000).toFixed(1) + 's' : '· timeout elapsed — next request is the probe';
  }
  function cbRequest(success) {
    if (cbState === 'OPEN') {
      if (Date.now() - cbOpenedAt >= TIMEOUT) {
        cbState = 'HALF_OPEN';
        cbLog('Allow(): timeout elapsed → HALF_OPEN — this request is the probe', 'bp-l-warn');
      } else {
        cbLog('Allow(): false — circuit OPEN, request rejected without touching backend', 'bp-l-fail');
        cbRender(); return;
      }
    } else if (cbState === 'HALF_OPEN') {
      cbLog('Allow(): false — probe already in flight, rejected', 'bp-l-fail');
      cbRender(); return;
    }
    if (success) {
      cbFails = 0;
      var was = cbState; cbState = 'CLOSED';
      cbLog('RecordSuccess(): failures reset → CLOSED' + (was === 'HALF_OPEN' ? ' (probe succeeded)' : ''));
    } else {
      cbFails++;
      cbLog('RecordFailure(): _failedRequests = ' + cbFails, 'bp-l-fail');
      if (cbFails >= THRESH || cbState === 'HALF_OPEN') {
        cbState = 'OPEN'; cbOpenedAt = Date.now();
        cbLog('threshold reached → OPEN for ' + (TIMEOUT / 1000) + 's', 'bp-l-warn');
      }
    }
    cbRender();
  }
  if ($('bpCBOk')) {
    $('bpCBOk').addEventListener('click', function () { cbRequest(true); });
    $('bpCBBad').addEventListener('click', function () { cbRequest(false); });
    $('bpCBReset').addEventListener('click', function () {
      cbState = 'CLOSED'; cbFails = 0; $('bpCBLog').innerHTML = '<span class="bp-l-info">// send a request to begin</span>'; cbRender();
    });
    cbTimerIv = setInterval(cbTick, 200);
    cbRender();
  }

  /* ---------- token bucket ---------- */
  var TB_MAX = 5, TB_RATE = 1, tbTokens = TB_MAX, tbLast = Date.now(), tbOk = 0, tbRej = 0;
  function tbRefill() {
    var now = Date.now();                      // single `now` — same rule as the C# code
    tbTokens = Math.min(TB_MAX, tbTokens + ((now - tbLast) / 1000) * TB_RATE);
    tbLast = now;
  }
  function tbRender() {
    $('bpTBFill').style.width = (tbTokens / TB_MAX * 100) + '%';
    $('bpTBLabel').textContent = tbTokens.toFixed(2) + ' / ' + TB_MAX + ' tokens';
    $('bpTBOk').textContent = ' ' + tbOk; $('bpTBRej').textContent = ' ' + tbRej;
  }
  function tbAllow() {
    tbRefill();
    if (tbTokens < 1) { tbRej++; return false; }
    tbTokens--; tbOk++; return true;
  }
  function tbFlash(ok) {
    var r = $('bpTBResult');
    r.textContent = ok ? '✓ allowed' : '✗ rejected — fail fast, let retry+backoff absorb it';
    r.style.color = ok ? '#15803d' : '#b91c1c';
  }
  if ($('bpTBSend')) {
    $('bpTBSend').addEventListener('click', function () { tbFlash(tbAllow()); tbRender(); });
    $('bpTBBurst').addEventListener('click', function () {
      var i = 0, iv = setInterval(function () {
        tbFlash(tbAllow()); tbRender();
        if (++i >= 8) clearInterval(iv);
      }, 90);
    });
    setInterval(function () { tbRefill(); tbRender(); }, 120);
    tbRender();
  }

  /* ---------- rate limiter showdown ---------- */
  function slots(id) {
    var c = $(id); c.innerHTML = '';
    var arr = [];
    for (var i = 0; i < 6; i++) { var s = document.createElement('div'); s.className = 'bp-slot'; c.appendChild(s); arr.push(s); }
    return arr;
  }
  if ($('bpRLGo')) {
    $('bpRLGo').addEventListener('click', function () {
      var btn = this; btn.disabled = true;
      var sw = slots('bpRLSW'), tb = slots('bpRLTB'), lb = slots('bpRLLB');
      $('bpRLSWv').textContent = ''; $('bpRLTBv').textContent = ''; $('bpRLLBv').textContent = '';
      // sliding window & token bucket: instant verdicts
      setTimeout(function () {
        for (var i = 0; i < 6; i++) {
          sw[i].className = 'bp-slot ' + (i < 3 ? 'bp-pass' : 'bp-rej');
          tb[i].className = 'bp-slot ' + (i < 3 ? 'bp-pass' : 'bp-rej');
          lb[i].className = 'bp-slot bp-wait';
        }
        $('bpRLSWv').textContent = '3 allowed, 3 rejected instantly. Strict; idle time earns nothing.';
        $('bpRLTBv').textContent = '3 allowed, 3 rejected instantly. But raise maxTokens above the rate and idle time buys burst.';
        $('bpRLLBv').textContent = 'All 6 queued — draining one every 333ms…';
      }, 250);
      // leaky bucket: drain one per 333ms
      var d = 0, iv = setInterval(function () {
        lb[d].className = 'bp-slot bp-pass'; d++;
        if (d >= 6) {
          clearInterval(iv);
          $('bpRLLBv').textContent = 'All 6 processed, zero rejected — but the last one waited ~2s. Smoothness traded for latency + queue memory.';
          btn.disabled = false;
        }
      }, 333);
    });
  }

  /* ---------- primitive picker ---------- */
  var picks = document.querySelectorAll('#bpPick .bp-flip');
  for (var p = 0; p < picks.length; p++) {
    picks[p].addEventListener('click', function () {
      this.classList.toggle('bp-openf');
      this.querySelector('.bp-hint').textContent = this.classList.contains('bp-openf') ? 'tap to hide ▴' : 'tap for why ▾';
    });
  }
})();
</script>

## The pre-interview checklist

Sixty seconds before walking in, this is the scan:

**Before writing code** — restate the problem; list all shared mutable state; justify each protection mechanism; walk the happy path end to end; define what counts as failure for the circuit breaker.

**While coding** — `readonly` on every constructor-set field; `finally` on every lock release; `Math.Abs()` on any `% n` fed by an incrementing counter; `double` not `int` for token counts; `DateTime.UtcNow` never `Now`; one `now` variable per method; rate limit *throws*, never returns false; `Random.Next(0, 2)` not `(0, 1)` for a coin flip.

**Say proactively** — why `volatile` over lock for the health flag; why `Interlocked` over lock for the counter; why `ReaderWriterLockSlim` over plain lock for the list; why `TransitionState` takes no inner lock; why the token bucket is global, not per-backend; why the health checker never touches circuit state; and why the circuit breaker is probabilistic protection, not a hard gate.

If this walkthrough helped, the same series covers Python internals, API protocols at scale, and idempotency — and API versioning is next.