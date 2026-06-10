---
layout: post
title: "The Design Was the Engineering: An LLM Router Specified by Hand, Typed by AI"
date: 2026-06-10
categories: [backend, system-design, ai]
tags: [llm-router, rate-limiting, circuit-breaker, concurrency, ai-assisted-development, machine-coding, csharp]
description: "Priority routing across five LLM providers with a sliding-window rate limiter and per-provider circuit breakers — designed by hand down to the lock boundaries, then handed to AI as a prompt. The planning doc was the work; the code generation just proved it."
---

Every week there is another post that says *"I built X with AI in an afternoon."* This is not that post — or rather, it is that post inverted. I built an LLM router in an afternoon, and AI typed essentially all of the code. But the afternoon was not spent prompting. It was spent on a planning document: which modules exist, how requests flow between them, where the shared state lives, what gets locked and when the lock is released. The code generation at the end was almost an anticlimax — and that anticlimax is the entire point.

The claim I want to defend with a concrete artifact: **the quality of AI-generated code is a function of the quality of the design handed to it.** When the design specifies the flow to the level of "acquire lock → loop providers → check → release lock → then execute," the implementation becomes nearly mechanical, and reviewing it becomes a diff against your own intent rather than an archaeology dig. When the design is "build me a router," you get plausible code that you now have to reverse-engineer to find out what decisions were made on your behalf.

So this post walks through the design as the main act: the routing problem, the concurrency decisions, the ordering decisions that look trivial and are not, and the failure semantics. Then it shows the planning doc verbatim — because that doc *was* the prompt — and treats the generated C# as what it was: a validation run. At the end you will find the full architecture diagram, three flow walkthroughs, a condensed checklist, and interactive simulators where you can operate the router, trip the breaker, and saturate the rate limiter yourself.

The complete runnable project lives alongside this post as `WebSearchRouter` — a C# / .NET console app, pure standard library, no external dependencies.

## The pragmatic default, acknowledged

If you need to route requests across LLM providers in production today, you do not write this. You reach for [LiteLLM](https://github.com/BerriAI/litellm) or a gateway like OpenRouter, which give you provider fallbacks, rate limit handling, and retries as configuration. And if you are in .NET and need resilience primitives, [Polly](https://github.com/App-vNext/Polly) ships circuit breakers and rate limiters that are battle-tested in ways a from-scratch version never will be.

So why build it by hand? Two reasons. First, this is exactly the shape of problem that machine coding interviews love — "route requests across N providers with rate limits and failure isolation" is a one-hour round at half the companies hiring backend engineers right now, and the libraries are not available in that room. Second, and more relevant to this post: I wanted a clean experiment for the design-first workflow. A problem small enough to specify completely, gnarly enough (concurrency, shared state, state machines) that an underspecified prompt would visibly go wrong.

## The problem

The system does web search through multiple LLM providers. Five providers are configured statically, each with a priority, its own rate limit, and its own circuit breaker parameters. A request should go to the highest-priority provider that is currently *willing* (circuit not open) and *able* (rate limit not exhausted). If a provider is tripped or saturated, fall through to the next one. If all five are out, reject the request — fast and explicitly. Thousands of requests arrive in parallel, so every routing decision races with every other one.

That last sentence is where the design effort actually goes. Everything else is bookkeeping.

## Designing the flow before the modules

The first artifact in my planning doc was not a class list — it was a single line tracing one request through the entire system:

```
Request → Router → acquire lock → loop providers by priority
        → rate limiter (sliding window) + circuit breaker checks
        → first provider that passes wins → release the lock
        → enter the executor → spin a parallel task → run the request
        → structured response
```

Writing the flow first forces the two decisions that define this system, both hiding in that line.

**Decision one: selection is serialized, execution is parallel.** The router holds a lock while it picks a provider, and releases it *before* the provider does any work. Why serialize selection at all? Because the decision is compound: "is the circuit closed?" and "is there rate budget?" are two reads against two pieces of shared state, followed by a write (consuming a rate slot). If two threads interleave there, both can observe "one slot left" and both can take it — the classic check-then-act race, except spread across two components, so no single component's internal lock can prevent it. A selection-wide lock makes the combined decision atomic.

The immediate objection: doesn't a global lock on the hot path destroy throughput? It would, if the work inside the lock were slow. But selection is a handful of dictionary lookups and integer comparisons — microseconds. The expensive part, the actual provider call with its tens of milliseconds of latency, happens strictly *after* the lock is released, on its own task. Serializing microsecond decisions while parallelizing millisecond work is the standard shape for this problem; you accept a tiny serial section to buy correctness, and you keep the parallelism where it pays. The 5,000-request test exists precisely to prove this tradeoff holds up.

**Decision two: the order of the two checks matters, and the obvious order is wrong.** My flow line above — taken verbatim from the planning doc — lists the rate limiter before the circuit breaker. The implementation does the opposite, and the flip is deliberate. Here is the asymmetry that forces it: asking the circuit breaker "is this call allowed?" is (in the closed state) a pure read. Asking the sliding-window rate limiter "can I have a slot?" *consumes the slot* — `TryAcquire` enqueues a timestamp on success. If you acquire the slot first and the breaker then says no, you have burned rate budget on a request that never ran. Under load, a tripped provider would keep eating its own rate window while blocked, and the denial counts in your logs would lie to you about why traffic is being refused. Cheap, side-effect-free checks go first; consuming checks go last, when everything else has already said yes. It is the same principle as "validate before you mutate," dressed up in resilience clothing — and it is exactly the kind of one-line subtlety that survives in a design review and dies in a vague prompt.

## The module inventory

With the flow fixed, the modules name themselves. From the planning doc: static provider registry, router, sliding-window rate limiter, circuit breaker, executor (dummy), structured response, a bulk-firing test module, and a logger that writes `log.txt` to the working directory and clears it on every launch.

```
Models/
  LLMProvider.cs         — provider config (priority, RL, CB, sim fields)
  LLMRequest.cs          — request DTO (8-char id, query, timestamp)
  StructuredResponse.cs  — status: Success | Failure | Rejected
Core/
  ProviderRegistry.cs    — 5 static providers, priorities 1–5
  Router.cs              — selection gate + priority loop
  SlidingWindowRateLimiter.cs
  CircuitBreaker.cs      — Closed / Open / HalfOpen per provider
  LLMExecutor.cs         — parallel task, simulated call, reports to CB
  Logger.cs              — thread-safe file logger
Tests/
  TestHarness.cs         — happy path, CB trip, bulk 500/5000, iterated
```

Notice what is absent, again: no interfaces, no DI container, no strategy abstractions. The planning doc asked for "different classes for each module" and module relationships — not extensibility theater. If a second selection policy ever shows up, extracting an interface is a five-minute refactor; carrying one from day zero is a permanent tax on readability.

The five providers and their parameters, exactly as configured:

| Provider | Priority | Rate limit | CB threshold | CB reset | Sim. failure | Sim. latency |
|---|---|---|---|---|---|---|
| OpenAI-GPT | 1 | 100 / 1s | 5 | 2s | 5% | 30ms |
| Anthropic-Claude | 2 | 80 / 1s | 5 | 2s | 8% | 40ms |
| Google-Gemini | 3 | 60 / 1s | 5 | 2s | 10% | 50ms |
| Perplexity | 4 | 50 / 1s | 5 | 2s | 12% | 60ms |
| Mistral | 5 | 40 / 1s | 5 | 2s | 15% | 70ms |

Two derived numbers worth committing to memory before reading the test results: total system capacity is **330 requests per second** (100+80+60+50+40), and failure rates climb as priority drops — which is realistic (your fallbacks are fallbacks for a reason) and conveniently makes lower tiers exercise the circuit breaker harder.

## The shared mutable state audit

Same discipline as always: before any implementation, list every piece of state that more than one thread touches, and justify its protection. This table is the concurrency design; everything after it is typing.

| State | Lives in | Touched by | Protection |
|---|---|---|---|
| Provider selection (the compound RL+CB decision) | Router | every request thread | `SemaphoreSlim(1,1)` |
| `Queue<DateTime>` per provider | RateLimiter | every selection | `lock (_lock)` |
| Breaker state, counters, `OpenedAt`, probe flag | CircuitBreaker | selections + executor callbacks | `lock (_lock)` |
| Log buffer | Logger | every component | `ConcurrentQueue` + `Monitor.TryEnter` |
| Random number generator | Executor | every parallel task | `ThreadLocal<Random>` |

**Why `SemaphoreSlim` and not `lock` for the router?** Because the router's method is `async`. You cannot `await` inside a `lock` block — the compiler refuses, because a lock is thread-affine and an `await` may resume on a different thread, leaving you releasing a lock you no longer hold. `SemaphoreSlim(1,1)` with `WaitAsync` is the async-compatible mutex: same exclusivity, no thread affinity, and waiting requests park without burning a thread. The release lives in a `finally` so a cancelled or faulted selection can never strand the gate.

**Why a plain `lock` inside the rate limiter and breaker, when the router already serializes selection?** Defense in depth, but not redundantly: the executor calls `RecordSuccess`/`RecordFailure` on the circuit breaker from its parallel tasks, *outside* the selection gate. So the breaker's state genuinely is touched by threads the router never sees, and it must protect itself. The rate limiter's lock additionally guards `CurrentCount` queries arriving from outside the gate. Each component is safe alone; the router's gate adds the cross-component atomicity that no per-component lock can provide.

**Why `ThreadLocal<Random>`?** `Random` is not thread-safe; concurrent access can corrupt its internal state to the point of returning all zeros. One instance per thread, seeded uniquely, costs nothing and removes the hazard.

**Why `Monitor.TryEnter` in the logger instead of `lock`?** This one is a favorite. Every component logs on the hot path. If the logger took a blocking lock around file I/O, every routing decision would occasionally stall behind a disk write. Instead, lines go into a `ConcurrentQueue` (lock-free), and each log call makes a *non-blocking attempt* to flush: `Monitor.TryEnter` either gets the file lock immediately or gives up and walks away, leaving the lines for whoever holds the lock to drain. The hot path never waits on the disk. The cost is a final `Logger.Flush()` (a blocking drain) at process exit, or buffered lines could die with the process.

## The circuit breaker, designed before typed

The breaker is the only real state machine in the system, so it got the most design attention. Three states per provider — `Closed`, `Open`, `HalfOpen` — with the transitions specified in the doc before any code existed:

```
CLOSED    --[5 consecutive failures]--→ OPEN
OPEN      --[2s reset timeout elapsed, next call arrives]--→ HALF_OPEN
HALF_OPEN --[probe succeeds]--→ CLOSED
HALF_OPEN --[probe fails]-----→ OPEN  (timer restarts)
```

Two design subtleties here separate a working breaker from a decorative one.

First, **`Open → HalfOpen` is a lazy transition, not a timer.** There is no background thread watching the clock. The state flips when the *next call* arrives after the timeout has elapsed — `IsCallAllowed` checks `DateTime.UtcNow - OpenedAt >= 2s` and transitions inline. This costs you nothing (a breaker nobody is calling does not need to recover promptly, because nobody would notice) and saves you a timer per provider plus the shutdown coordination that timers drag in. If no traffic arrives for an hour, the first request after the hour performs the transition. The design doc should say this explicitly, because "the breaker resets after 2 seconds" reads like a timer to most people, and an AI given that sentence may well build one.

Second, **half-open admits exactly one probe.** When the breaker goes half-open, the temptation is to "let a little traffic through and see." Under 5,000 parallel requests, "a little traffic" is a stampede: hundreds of threads observe the half-open state simultaneously and all conclude they are the probe. The fix is a `HalfOpenProbeInFlight` flag set atomically under the breaker's lock: the first caller through flips it and becomes the probe; everyone else sees the flag and is rejected. One request risks itself on behalf of the rest. The probe's outcome then decides the next state — success closes the breaker (and clears the flag), failure reopens it (and restarts the 2-second clock).

Who calls `RecordSuccess` and `RecordFailure`? The executor — the component that actually made the call and saw the outcome. Not the router: it only selects, and by the time the outcome exists the router is long gone. This was written into the planning doc as a module relationship ("Executor should update the Circuit Breaker of the LLM it used"), because outcome-reporting responsibility is exactly the kind of thing that ends up nowhere if no one assigns it.

One honest limitation, stated proactively: this breaker counts **consecutive** failures, and a single success resets the count. A provider failing every other request never trips it. Production breakers track failure *rate over a window* instead — same `Queue<DateTime>` idea as the rate limiter, repurposed. For a priority-fallback router the consecutive-failure model is defensible (a provider that succeeds half the time is still doing useful work at its priority slot), but say the words "consecutive, and here is what that misses" before your reviewer does.

## Failure is not rejection

The response DTO has three terminal states, and collapsing any two of them is a design bug:

- **Success** — a provider ran the request and returned a result.
- **Failure** — a provider ran the request and it failed. The breaker hears about this.
- **Rejected** — no provider ever ran it: the priority loop visited all five, each was either circuit-blocked or rate-limited, and the router returned `"All providers unavailable (rate limited or circuit open)"`. The breaker hears nothing, because no provider did anything wrong.

The distinction earns its keep twice. Operationally, rejections are a *capacity* signal (add providers, raise limits, shed load) while failures are a *health* signal (a provider is degrading) — alert on them differently or drown. And mechanically, it answers "what does retry exhaustion look like here?": this design deliberately has **no retry loop after execution**. If a provider accepts a request and then fails it, the response goes back as `Failure` — the router does not catch it and re-route to the next provider. The failure feeds the breaker, the breaker trips after 5 in a row, and *subsequent* requests route around the sick provider. Exhaustion lives entirely inside the selection loop: running out of providers *before* execution is the `Rejected` path, and it is the only exhaustion this system has. You could add per-request re-routing on failure — at the cost of double-charging rate budgets for one logical request and muddying what "Failure" means in the stats. For a system whose job is to demonstrate isolation and shedding, the simple semantics are the better ones.

## The planning doc, verbatim — because it was the prompt

Everything above crystallized into one short document. I am reproducing it exactly as written, typos and all, because this document is the experiment: it is the entire prompt I handed to Claude to generate the project. No follow-up clarifications about architecture, no "actually, what I meant was." What you see is what the model got.

```
Goal: To do web search using various providers. These providers can be selected
on the basis of the priority queue which is already defined.
Language & Framework: Use C# and .NET to develop this.
Flow:
Request 1 -> Router -> aquire lock -> Loop LLMs in queue -> Rate Limiter Sliding
window -> circuit breaker with threshold & timeout -> If allowed for being
iterated LLMs, proceed -> release the lock -> enter the executor -> spin a new
parallel task -> run the request on that task -> structured response from LLM

Instructions for system design:
1. Let us have different classes for each of the module that is part of this design.
    a. Static LLM providers
    b. Router for the LLM
    c. Rate Limiter - sliding window
    d. Circuit Breaker - Update the success and failures
    e. LLM Executor (dummy)
    f. Structured reponse
    g. Dummy requests firing in bulk to test the system
    h. Create a log.txt file in pwd for logging the actions throughout the flow.
       This should be deleted on each new launch of app.
2. Module relationships
    a. Request should come to Router layer
    b. Router should choose what LLM to pick on the basis on the allowance from
       Rate Limiter and Circuit Breaker. This part should have locking mechanism
       to ensure multi requests should be handled synchronously
    c. Once LLM is chosen then proceed with the executor (dummy)
    d. Executor should update the Circuit Breaker of the LLM it used to execute.
    e. Multiple executors can be as executed in parallel as the requests are coming.
3. Let us have a testing module to test this system.
    a. Ensure that happy flow works.
    b. Ensure that flows where multiple requests come in parallel should be handled
       well by the rate limiter and circuit breaker
    c. Ensure to test with 5000 requests in parallel and validate if the system is
       working as defined
    d. Run this loop in iteration to fix any of the issue in the design
```

Look at what this document is and is not. It does not name a single class member, type, or library. It *does* pin down: the module boundaries (eight of them), the ownership of every responsibility (who locks, who updates the breaker, who spawns tasks), the exact flow ordering including where the lock is acquired and released, the testing ladder (happy path → parallel → 5,000 → iterate), and even operational details like the log file lifecycle. It is roughly 300 words, and it took most of the design time. That ratio — minutes of typing, hours of thinking — is what design-first means in practice.

Point 3d deserves a highlight: *"Run this loop in iteration to fix any of the issue in the design."* The prompt builds the verification loop into the deliverable. The model is not asked to produce code that looks right; it is asked to produce code, run a 5,000-request load test against it, and iterate until the system's observed behavior matches the specified one. Designing the test harness into the prompt is what turns "code generation" into "validated implementation."

## What came back

The generated code matched the design closely enough that reviewing it felt like reviewing my own work — which is the whole thesis. Three excerpts, verbatim from the project, with what to notice in each.

### The router: lock boundaries exactly where the doc drew them

```csharp
public async Task<StructuredResponse> RouteAsync(LLMRequest request, CancellationToken ct = default)
{
    Logger.Log("ROUTER", $"req={request.RequestId} arrived query='{request.Query}'");

    LLMProvider? selected = null;

    await _selectionGate.WaitAsync(ct);
    try
    {
        Logger.Log("ROUTER", $"req={request.RequestId} acquired selection lock");
        foreach (var provider in _priorityOrdered)
        {
            if (!_circuitBreaker.IsCallAllowed(provider.Name))
            {
                Logger.Log("ROUTER", $"req={request.RequestId} skip {provider.Name} (circuit not allowed)");
                continue;
            }

            if (!_rateLimiter.TryAcquire(provider.Name))
            {
                Logger.Log("ROUTER", $"req={request.RequestId} skip {provider.Name} (rate-limited)");
                continue;
            }

            selected = provider;
            Logger.Log("ROUTER", $"req={request.RequestId} SELECTED {provider.Name} (priority={provider.Priority})");
            break;
        }
    }
    finally
    {
        _selectionGate.Release();
        Logger.Log("ROUTER", $"req={request.RequestId} released selection lock");
    }

    if (selected == null)
    {
        Logger.Log("ROUTER", $"req={request.RequestId} REJECTED — no provider available");
        return new StructuredResponse
        {
            RequestId = request.RequestId,
            Query = request.Query,
            Status = ResponseStatus.Rejected,
            ErrorMessage = "All providers unavailable (rate limited or circuit open)"
        };
    }

    return await _executor.ExecuteAsync(request, selected, ct);
}
```

The things the design demanded are all here, in the right places: `SemaphoreSlim` because the method is async; the release in a `finally`; the breaker checked *before* the rate limiter (the ordering flip from the flow line — the generated code got the consuming-check-last ordering right, which is exactly the kind of judgment call you verify first in review); and `ExecuteAsync` called strictly after the gate is released, so execution never holds up selection. Note also that the executor call sits outside the `try/finally` — if it threw inside, the gate would already be released, but more importantly a request that fails *execution* must not look like a request that failed *selection*.

### The rate limiter: evict, count, consume — under one lock

```csharp
public bool TryAcquire(string providerName)
{
    if (!_providers.TryGetValue(providerName, out var provider)) return false;

    var now = DateTime.UtcNow;
    lock (_lock)
    {
        var queue = _hits[providerName];
        var cutoff = now - provider.RateLimitWindow;
        while (queue.Count > 0 && queue.Peek() < cutoff)
        {
            queue.Dequeue();
        }
        if (queue.Count >= provider.RateLimitPerWindow)
        {
            Logger.Log("RATE_LIMIT", $"{providerName} DENIED — {queue.Count}/{provider.RateLimitPerWindow} in window {provider.RateLimitWindow.TotalMilliseconds}ms");
            return false;
        }
        queue.Enqueue(now);
        Logger.Log("RATE_LIMIT", $"{providerName} ALLOWED — {queue.Count}/{provider.RateLimitPerWindow}");
        return true;
    }
}
```

A true sliding window: one `Queue<DateTime>` per provider, timestamps older than the window evicted on every call, admission only while `count < limit`. Strict and memory-proportional to the limit — at 100 timestamps per provider, who cares. The eviction-then-check-then-enqueue sequence is a compound operation, hence the lock around all three; `UtcNow` is captured once per call and never re-read mid-operation, so eviction and admission reason about a single instant instead of a drifting one.

### The breaker: the probe flag doing its quiet work

```csharp
case BreakerState.Open:
    if (DateTime.UtcNow - s.OpenedAt >= provider.CircuitBreakerResetTimeout)
    {
        s.Status = BreakerState.HalfOpen;
        s.HalfOpenProbeInFlight = false;
        Logger.Log("CIRCUIT", $"{providerName} OPEN -> HALF_OPEN (reset timeout elapsed)");
    }
    else
    {
        Logger.Log("CIRCUIT", $"{providerName} OPEN — blocking call");
        return false;
    }
    goto case BreakerState.HalfOpen;

case BreakerState.HalfOpen:
    if (s.HalfOpenProbeInFlight)
    {
        Logger.Log("CIRCUIT", $"{providerName} HALF_OPEN — probe already in flight, blocking");
        return false;
    }
    s.HalfOpenProbeInFlight = true;
    Logger.Log("CIRCUIT", $"{providerName} HALF_OPEN — admitting probe");
    return true;
```

The lazy transition and the single-probe rule, both as designed. The `goto case` is C#'s explicit fall-through — after the lazy `Open → HalfOpen` flip, the call is immediately evaluated under half-open rules in the same pass, so the transitioning request itself becomes the probe candidate rather than being bounced and forcing a second round-trip. Everything happens under the breaker's lock, so two threads cannot both pass the `HalfOpenProbeInFlight` check.

One more generated detail worth flagging because it is easy to miss in review: the executor's randomness is `ThreadLocal<Random>`, seeded per thread. The naive `new Random()` shared across 5,000 parallel tasks is both thread-unsafe and, when created in a tight loop, time-seeded into near-identical sequences. The model handled it unprompted — but "unprompted" is precisely why review still matters. You verify the gifts too.

## Running it: the numbers the design predicts

The test harness runs a fixed ladder on every launch: happy path, a forced circuit breaker trip, a 500-request parallel burst, a 5,000-request parallel burst, then 3 iterations of 2,000 — with 1.5-second pauses between bulk phases so the rate windows drain and each phase starts clean. Total runtime is about ten seconds.

The design lets you predict the bulk results before running anything. Total capacity is 330 requests per second across the five providers. A burst of 500 fired simultaneously therefore cannot all land: roughly the first window's worth (330, give or take requests that slip into a second window as time passes during the burst) are admitted and spread across providers in priority order — OpenAI fills its 100 first, overflow cascades to Claude's 80, then Gemini's 60, Perplexity's 50, Mistral's 40 — and the remainder come back `Rejected`. At 5,000 the same arithmetic holds, just with more elapsed windows during the burst: admitted counts match capacity × elapsed windows, the rest are rejected, and the rejection message tells you it was capacity, not health. If you see *failures* spike during a bulk test instead of rejections, something is actually wrong — that is the three-state response DTO paying rent.

The forced breaker trip uses its own two-provider rig: a `FlakyPrimary` at 100% failure rate with threshold 3 and an 800ms reset timeout, and a `ReliableFallback` that never fails. Ten sequential requests: the first three fail on the primary and trip it (`CLOSED -> OPEN` at exactly failure #3), requests four through ten route to the fallback without touching the primary. Wait 900ms — past the reset timeout — and send one request: the primary lazily transitions to half-open, admits it as the probe, fails it, and reopens (`HALF_OPEN -> OPEN`). Swap the primary to healthy, wait another 900ms, send again: probe succeeds, breaker closes (`HALF_OPEN -> CLOSED`). The full state machine, exercised in four lines of console output.

Everything is also traceable post-hoc, because the logger tags every line with timestamp, thread id, and category:

```
[10:16:06.781][T001][ROUTER] req=71a8d4ac SELECTED OpenAI-GPT (priority=1)
```

```bash
grep "req=71a8d4ac" log.txt        # one request, end to end
grep CIRCUIT log.txt               # every breaker transition
grep "RATE_LIMIT.*DENIED" log.txt  # who is shedding, and how much
```

## What this says about working with AI

The honest accounting of where the time went: most of it on the flow line, the state audit, and the module relationships; minutes on prompting; and a review pass that read like a code review of a competent colleague who had been handed an unusually precise spec — because that is literally what happened.

The inversion I want to leave you with: *"I used AI to write the code"* is the least interesting sentence in this whole exercise. The interesting one is that **a 300-word design document fully determined a 900-line concurrent system.** Every decision that mattered — lock boundaries, check ordering, probe semantics, failure taxonomy — was made before generation and survived generation. The model filled in syntax, idiom, and diligence (the `ThreadLocal<Random>`, the `finally`, the log lines), and the load test in point 3d closed the loop. Where a design is complete, codegen is mechanical. Where a design is vague, the model makes your decisions for you, silently, and you inherit them unread. The skill that is appreciating in value is not prompting — it is the old one: knowing exactly what you want built, precisely enough that anyone, human or machine, could build it.

## The full picture

Everything above, on one diagram — happy path, rejection path, breaker trip and recovery, where every piece of shared state lives and what protects it. This is the drawing to reproduce on a whiteboard, annotations and all.

<a href="/assets/llm-router-system-design.jpg" target="_blank"><img src="/assets/llm-router-system-design.jpg" alt="WebSearchRouter system design: priority routing across 5 LLM providers with sliding-window rate limiter, per-provider circuit breakers, selection gate, executor, and rejection path" style="width:100%;height:auto;border:1px solid #e2ddd3;border-radius:12px" loading="lazy"/></a>

<em style="font-size:.85rem;color:#6b7280">Click to open full size (2K).</em>

## Three walkthroughs to narrate aloud

**Happy path.** A request arrives at the router and waits its turn at the selection gate (`SemaphoreSlim(1,1)` — async-safe, FIFO-ish). Inside, the loop starts at priority 1: OpenAI-GPT. The breaker is closed — a pure read says go. `TryAcquire` evicts stale timestamps, sees 73/100 in the current 1-second window, enqueues `now`, returns true. Selected. The gate releases — total time inside: microseconds. The executor spins a parallel task, simulates the call (30ms), rolls against the 5% failure rate, succeeds, calls `RecordSuccess` (consecutive-failure counter resets to 0), and a `StructuredResponse { Success, OpenAI-GPT, 30ms }` flows back to the caller.

**Rejection (the only exhaustion).** Same arrival, but mid-burst. Priority 1: breaker closed but `TryAcquire` finds 100/100 — denied, skip. Priority 2: Claude tripped two seconds ago, breaker open, timeout not yet elapsed — blocked, skip (and note: its rate budget was never touched, because the breaker is checked first). Priorities 3–5: rate windows full — skip, skip, skip. Loop exhausted, `selected == null`, gate releases, and the caller gets `Rejected: "All providers unavailable (rate limited or circuit open)"`. No provider executed anything; no breaker state changed. This response in bulk means *add capacity*, not *something is broken*.

**Trip and recovery.** Five consecutive failures on Gemini (threshold 5) and its breaker flips `CLOSED → OPEN`, stamping `OpenedAt`. For the next 2 seconds every selection skips Gemini at the breaker check — costlessly. The first call arriving after 2s triggers the lazy transition: `OPEN → HALF_OPEN`, probe flag clear, and that same call becomes the probe (`goto case` fall-through). Hundreds of concurrent contemporaries see `HalfOpenProbeInFlight == true` and are bounced. The probe executes for real: success → `HALF_OPEN → CLOSED`, traffic resumes; failure → `HALF_OPEN → OPEN`, clock restarts, repeat in 2s. Throughout the recovery dance, traffic kept flowing through the other four providers — isolation is the feature, the probe is just bookkeeping.

## The condensed checklist

Run through this before an interview or design review on routing, rate limiting, or circuit breaking:

- **Flow first.** One line tracing a request end to end, including where locks open and close. If you cannot write the line, you cannot write the code.
- **Shared state audit.** Every mutable thing touched by 2+ threads, each with a named protection: async path → `SemaphoreSlim`; compound updates → `lock`; hot-path logging → lock-free queue + `Monitor.TryEnter`; per-thread hazards (e.g. `Random`) → `ThreadLocal`.
- **Serialize decisions, parallelize work.** Lock the microsecond selection; release before the millisecond execution. Never hold a selection lock across I/O.
- **Order checks by side effect.** Pure reads (circuit state) before consuming checks (`TryAcquire` eats a slot). A blocked call must not burn rate budget.
- **Cross-component atomicity needs its own lock.** Each component being thread-safe does not make a check-A-then-take-B sequence atomic.
- **Breaker = 3 states + 2 rules.** Lazy `Open → HalfOpen` on next call (no timers); exactly one probe via an in-flight flag set under the lock.
- **Outcomes are reported by whoever saw them.** The executor updates the breaker. The router never learns what happened, and synthetic checks must not drive circuit state.
- **Failure ≠ Rejection.** Ran-and-failed feeds the breaker (health); never-ran feeds nobody (capacity). Alert on them separately.
- **Know your exhaustion.** Here: priority loop exhausts → `Rejected`. No post-execution re-routing — say it is a choice and defend it.
- **Consecutive-failure counting misses intermittent failure.** Production: failure rate over a window. Say it before they ask.
- **Predict the load test, then run it.** 100+80+60+50+40 = 330/s capacity ⇒ a 5,000-burst must show ~capacity×windows admitted, rest rejected, failures ≈ failure rates. If you cannot predict the numbers, the test cannot validate anything.
- **Spec the verification into the prompt/design.** "Iterate until the 5,000-request run matches the design" is what makes generated code trustworthy.

## Interactive quick reference

Reading about routing is one thing; operating it is another. Three simulators below run the actual algorithms from this post — same parameters, same state machines. No libraries, no network, just the logic.

<style>
.lr-card { background: #fbfaf7; border: 1px solid #e2ddd3; border-radius: 12px; padding: 1.25rem 1.5rem; margin: 1.5rem 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
.lr-card h4 { margin: 0 0 .25rem 0; font-size: 1.05rem; color: #1f2933; }
.lr-card .lr-sub { font-size: .85rem; color: #6b7280; margin-bottom: 1rem; }
.lr-row { display: flex; gap: .5rem; flex-wrap: wrap; align-items: center; margin-bottom: .6rem; }
.lr-btn { border: 1.5px solid #1f2933; background: #fff; color: #1f2933; font: inherit; font-size: .85rem; font-weight: 600; padding: .45rem .9rem; border-radius: 8px; cursor: pointer; transition: transform .08s ease, background .15s ease; }
.lr-btn:hover { background: #1f2933; color: #fff; }
.lr-btn:active { transform: scale(.96); }
.lr-btn.lr-ok { border-color: #15803d; color: #15803d; } .lr-btn.lr-ok:hover { background: #15803d; color: #fff; }
.lr-btn.lr-bad { border-color: #b91c1c; color: #b91c1c; } .lr-btn.lr-bad:hover { background: #b91c1c; color: #fff; }
.lr-btn.lr-time { border-color: #6d28d9; color: #6d28d9; } .lr-btn.lr-time:hover { background: #6d28d9; color: #fff; }
.lr-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: .6rem; margin: .8rem 0; }
.lr-prov { background: #fff; border: 1px solid #e2ddd3; border-radius: 10px; padding: .6rem .7rem; }
.lr-prov h5 { margin: 0; font-size: .82rem; color: #1f2933; display: flex; justify-content: space-between; align-items: center; }
.lr-prov .lr-pr { font-size: .68rem; color: #6b7280; font-weight: 400; }
.lr-chip { display: inline-block; font-size: .62rem; font-weight: 700; letter-spacing: .04em; padding: .12rem .5rem; border-radius: 999px; border: 1.5px solid; margin: .35rem 0; }
.lr-chip.lr-closed { color: #15803d; border-color: #15803d; background: #f0fdf4; }
.lr-chip.lr-open { color: #b91c1c; border-color: #b91c1c; background: #fef2f2; }
.lr-chip.lr-half { color: #b45309; border-color: #b45309; background: #fffbeb; }
.lr-meter { height: 16px; background: #eceae4; border: 1px solid #d6d1c4; border-radius: 6px; overflow: hidden; position: relative; margin-top: .3rem; }
.lr-meter-fill { height: 100%; background: linear-gradient(90deg, #0e7490, #0891b2); transition: width .15s ease; }
.lr-meter-fill.lr-full { background: linear-gradient(90deg, #b91c1c, #ef4444); }
.lr-meter-label { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-size: .62rem; font-weight: 700; color: #1f2933; }
.lr-stat { font-size: .72rem; color: #6b7280; margin-top: .25rem; }
.lr-totals { display: flex; gap: 1rem; flex-wrap: wrap; font-size: .82rem; font-weight: 600; margin: .5rem 0; }
.lr-totals .lr-t-ok { color: #15803d; } .lr-totals .lr-t-fail { color: #b91c1c; } .lr-totals .lr-t-rej { color: #b45309; }
.lr-log { font-family: ui-monospace, SFMono-Regular, Menlo, monospace; font-size: .74rem; background: #1f2933; color: #d1fae5; border-radius: 8px; padding: .7rem .9rem; margin-top: .8rem; height: 140px; overflow-y: auto; line-height: 1.5; }
.lr-log .lr-l-fail { color: #fca5a5; } .lr-log .lr-l-warn { color: #fcd34d; } .lr-log .lr-l-info { color: #93c5fd; } .lr-log .lr-l-dim { color: #9ca3af; }
.lr-statebox { display: inline-flex; align-items: center; gap: .5rem; padding: .45rem 1rem; border-radius: 999px; font-weight: 700; font-size: .88rem; border: 2px solid; transition: all .25s ease; }
.lr-statebox.lr-closed { color: #15803d; border-color: #15803d; background: #f0fdf4; }
.lr-statebox.lr-open { color: #b91c1c; border-color: #b91c1c; background: #fef2f2; }
.lr-statebox.lr-half { color: #b45309; border-color: #b45309; background: #fffbeb; }
.lr-dot { width: 10px; height: 10px; border-radius: 50%; background: currentColor; }
.lr-kv { font-size: .8rem; color: #374151; margin: .2rem 0; }
.lr-track { position: relative; height: 64px; background: #fff; border: 1px solid #e2ddd3; border-radius: 10px; margin: .7rem 0 .3rem 0; overflow: hidden; }
.lr-track .lr-ts { position: absolute; top: 18px; width: 12px; height: 12px; border-radius: 50%; background: #0891b2; border: 2px solid #0e7490; transform: translateX(-50%); transition: left .2s ease, opacity .3s ease; }
.lr-track .lr-now { position: absolute; top: 0; bottom: 0; width: 2px; background: #1f2933; right: 8px; }
.lr-track .lr-cap { position: absolute; bottom: 4px; right: 12px; font-size: .65rem; color: #6b7280; }
.lr-axis { display: flex; justify-content: space-between; font-size: .65rem; color: #9ca3af; margin-bottom: .4rem; }
@media (max-width: 640px) {
  .lr-card { padding: 1rem; }
  .lr-grid { grid-template-columns: repeat(auto-fit, minmax(125px, 1fr)); }
  .lr-log { font-size: .68rem; }
}
</style>

### Drive the router

The five real providers, real limits, real breaker rules. Fire single requests or bursts and watch the priority loop skip, cascade, and shed. Time is simulated — advance it yourself to drain windows and let opened breakers reach their reset timeout. Trip a breaker manually and notice rejected bursts never touch a tripped provider's rate budget (breaker is checked first).

<div class="lr-card" id="lr-router">
  <h4>Priority router — 330 req/s total capacity</h4>
  <div class="lr-sub">Selection: breaker check → rate check → first pass wins. CB: threshold 5, reset 2s (lazy), single probe.</div>
  <div class="lr-row">
    <button class="lr-btn" id="lr-r-send1">Send 1</button>
    <button class="lr-btn" id="lr-r-send50">Burst 50</button>
    <button class="lr-btn" id="lr-r-send400">Burst 400</button>
    <button class="lr-btn lr-bad" id="lr-r-trip">Trip OpenAI-GPT</button>
    <button class="lr-btn lr-time" id="lr-r-tick">Advance 1s</button>
    <button class="lr-btn" id="lr-r-reset">Reset</button>
  </div>
  <div class="lr-totals">
    <span>t = <span id="lr-r-clock">0.0</span>s</span>
    <span class="lr-t-ok">Success: <span id="lr-r-ok">0</span></span>
    <span class="lr-t-fail">Failure: <span id="lr-r-fail">0</span></span>
    <span class="lr-t-rej">Rejected: <span id="lr-r-rej">0</span></span>
  </div>
  <div class="lr-grid" id="lr-r-grid"></div>
  <div class="lr-log" id="lr-r-log"></div>
</div>

### Operate one circuit breaker

The exact state machine from `CircuitBreaker.cs`: threshold 5 consecutive failures, 2s reset, lazy transition, one probe. Try to get a probe admitted, then decide its fate.

<div class="lr-card" id="lr-cb">
  <h4>Circuit breaker — Closed / Open / HalfOpen</h4>
  <div class="lr-sub">Open → HalfOpen happens on the <em>next call</em> after the timeout, not on a timer. Watch the log when you call into an expired Open breaker.</div>
  <div class="lr-row">
    <span class="lr-statebox lr-closed" id="lr-cb-state"><span class="lr-dot"></span>CLOSED</span>
  </div>
  <div class="lr-kv">consecutive failures: <strong id="lr-cb-cons">0</strong> / 5 &nbsp;·&nbsp; probe in flight: <strong id="lr-cb-probe">no</strong> &nbsp;·&nbsp; t = <strong id="lr-cb-clock">0.0</strong>s</div>
  <div class="lr-row" style="margin-top:.6rem">
    <button class="lr-btn lr-ok" id="lr-cb-okcall">Call → succeeds</button>
    <button class="lr-btn lr-bad" id="lr-cb-failcall">Call → fails</button>
    <button class="lr-btn lr-time" id="lr-cb-wait">Wait 2s</button>
    <button class="lr-btn" id="lr-cb-reset">Reset</button>
  </div>
  <div class="lr-log" id="lr-cb-log"></div>
</div>

### Saturate a sliding window

One provider, limit 10 per rolling 1-second window — a scaled-down OpenAI-GPT. Each dot is a timestamp in the queue. Old dots are evicted as the window slides; admission requires `count < limit` <em>after</em> eviction.

<div class="lr-card" id="lr-sw">
  <h4>Sliding window — 10 per 1s</h4>
  <div class="lr-sub">The queue holds timestamps, not counts: that is what makes the window slide instead of snap.</div>
  <div class="lr-row">
    <button class="lr-btn" id="lr-sw-one">1 request</button>
    <button class="lr-btn" id="lr-sw-burst">Burst of 6</button>
    <button class="lr-btn lr-time" id="lr-sw-tick">Advance 250ms</button>
    <button class="lr-btn" id="lr-sw-reset">Reset</button>
  </div>
  <div class="lr-axis"><span>now − 1s</span><span>now</span></div>
  <div class="lr-track" id="lr-sw-track"><div class="lr-now"></div><div class="lr-cap" id="lr-sw-cap">0 / 10 in window</div></div>
  <div class="lr-totals">
    <span>t = <span id="lr-sw-clock">0.00</span>s</span>
    <span class="lr-t-ok">Allowed: <span id="lr-sw-ok">0</span></span>
    <span class="lr-t-rej">Denied: <span id="lr-sw-den">0</span></span>
  </div>
  <div class="lr-log" id="lr-sw-log"></div>
</div>

<script>
(function () {
  "use strict";

  function logTo(el, msg, cls) {
    var line = document.createElement("div");
    if (cls) line.className = cls;
    line.textContent = msg;
    el.appendChild(line);
    el.scrollTop = el.scrollHeight;
    while (el.children.length > 200) el.removeChild(el.firstChild);
  }

  /* ---------------- Widget 1: priority router ---------------- */
  (function () {
    var PROVIDERS = [
      { name: "OpenAI-GPT", prio: 1, limit: 100, failRate: 0.05 },
      { name: "Anthropic-Claude", prio: 2, limit: 80, failRate: 0.08 },
      { name: "Google-Gemini", prio: 3, limit: 60, failRate: 0.10 },
      { name: "Perplexity", prio: 4, limit: 50, failRate: 0.12 },
      { name: "Mistral", prio: 5, limit: 40, failRate: 0.15 }
    ];
    var WINDOW = 1000, THRESHOLD = 5, RESET = 2000;
    var grid = document.getElementById("lr-r-grid");
    var logEl = document.getElementById("lr-r-log");
    var st, totals, clock;

    function freshState() {
      clock = 0;
      totals = { ok: 0, fail: 0, rej: 0 };
      st = PROVIDERS.map(function (p) {
        return { p: p, hits: [], cb: "closed", cons: 0, openedAt: 0, probe: false };
      });
    }

    function evict(s) {
      var cutoff = clock - WINDOW;
      while (s.hits.length && s.hits[0] <= cutoff) s.hits.shift();
    }

    function cbAllowed(s) {
      if (s.cb === "closed") return true;
      if (s.cb === "open") {
        if (clock - s.openedAt >= RESET) {
          s.cb = "half";
          s.probe = false;
          logTo(logEl, "[CIRCUIT] " + s.p.name + " OPEN -> HALF_OPEN (reset timeout elapsed)", "lr-l-warn");
        } else {
          return false;
        }
      }
      if (s.probe) return false;
      s.probe = true;
      logTo(logEl, "[CIRCUIT] " + s.p.name + " HALF_OPEN — admitting probe", "lr-l-warn");
      return true;
    }

    function recordOutcome(s, failed) {
      if (failed) {
        s.cons += 1;
        if (s.cb === "half") {
          s.cb = "open"; s.openedAt = clock; s.probe = false;
          logTo(logEl, "[CIRCUIT] " + s.p.name + " HALF_OPEN -> OPEN (probe failed)", "lr-l-fail");
        } else if (s.cb === "closed" && s.cons >= THRESHOLD) {
          s.cb = "open"; s.openedAt = clock;
          logTo(logEl, "[CIRCUIT] " + s.p.name + " CLOSED -> OPEN (failures=" + s.cons + " >= " + THRESHOLD + ")", "lr-l-fail");
        }
      } else {
        s.cons = 0;
        if (s.cb === "half") {
          s.cb = "closed"; s.probe = false;
          logTo(logEl, "[CIRCUIT] " + s.p.name + " HALF_OPEN -> CLOSED (probe succeeded)", "lr-l-info");
        }
      }
    }

    function route(quiet) {
      var selected = null;
      for (var i = 0; i < st.length; i++) {
        var s = st[i];
        if (!cbAllowed(s)) {
          if (!quiet) logTo(logEl, "[ROUTER] skip " + s.p.name + " (circuit not allowed — rate budget untouched)", "lr-l-dim");
          continue;
        }
        evict(s);
        if (s.hits.length >= s.p.limit) {
          if (!quiet) logTo(logEl, "[ROUTER] skip " + s.p.name + " (rate-limited " + s.hits.length + "/" + s.p.limit + ")", "lr-l-dim");
          continue;
        }
        s.hits.push(clock);
        selected = s;
        break;
      }
      if (!selected) {
        totals.rej += 1;
        if (!quiet) logTo(logEl, "[ROUTER] REJECTED — all providers unavailable (rate limited or circuit open)", "lr-l-warn");
        return;
      }
      var failed = Math.random() < selected.p.failRate;
      recordOutcome(selected, failed);
      if (failed) {
        totals.fail += 1;
        if (!quiet) logTo(logEl, "[EXECUTOR] FAILED on " + selected.p.name, "lr-l-fail");
      } else {
        totals.ok += 1;
        if (!quiet) logTo(logEl, "[EXECUTOR] SUCCESS on " + selected.p.name + " (priority " + selected.p.prio + ")");
      }
    }

    function burst(n) {
      logTo(logEl, "--- burst of " + n + " at t=" + (clock / 1000).toFixed(1) + "s ---", "lr-l-info");
      var before = { ok: totals.ok, fail: totals.fail, rej: totals.rej };
      for (var i = 0; i < n; i++) route(true);
      logTo(logEl, "burst result: +" + (totals.ok - before.ok) + " success, +" + (totals.fail - before.fail) + " failure, +" + (totals.rej - before.rej) + " rejected", "lr-l-info");
    }

    function render() {
      document.getElementById("lr-r-clock").textContent = (clock / 1000).toFixed(1);
      document.getElementById("lr-r-ok").textContent = String(totals.ok);
      document.getElementById("lr-r-fail").textContent = String(totals.fail);
      document.getElementById("lr-r-rej").textContent = String(totals.rej);
      grid.innerHTML = "";
      st.forEach(function (s) {
        evict(s);
        var pct = Math.min(100, Math.round(100 * s.hits.length / s.p.limit));
        var chip = s.cb === "closed" ? "lr-closed" : (s.cb === "open" ? "lr-open" : "lr-half");
        var label = s.cb === "closed" ? "CLOSED" : (s.cb === "open" ? "OPEN" : "HALF-OPEN");
        var card = document.createElement("div");
        card.className = "lr-prov";
        card.innerHTML =
          '<h5>' + s.p.name + ' <span class="lr-pr">P' + s.p.prio + "</span></h5>" +
          '<span class="lr-chip ' + chip + '">' + label + "</span>" +
          '<div class="lr-meter"><div class="lr-meter-fill' + (pct >= 100 ? " lr-full" : "") + '" style="width:' + pct + '%"></div>' +
          '<div class="lr-meter-label">' + s.hits.length + "/" + s.p.limit + "</div></div>" +
          '<div class="lr-stat">consec fails: ' + s.cons + "/" + THRESHOLD + "</div>";
        grid.appendChild(card);
      });
    }

    function wrap(fn) { return function () { fn(); render(); }; }

    document.getElementById("lr-r-send1").addEventListener("click", wrap(function () { route(false); }));
    document.getElementById("lr-r-send50").addEventListener("click", wrap(function () { burst(50); }));
    document.getElementById("lr-r-send400").addEventListener("click", wrap(function () { burst(400); }));
    document.getElementById("lr-r-trip").addEventListener("click", wrap(function () {
      var s = st[0];
      s.cb = "open"; s.openedAt = clock; s.cons = THRESHOLD; s.probe = false;
      logTo(logEl, "[CIRCUIT] OpenAI-GPT forced CLOSED -> OPEN (manual trip)", "lr-l-fail");
    }));
    document.getElementById("lr-r-tick").addEventListener("click", wrap(function () {
      clock += 1000;
      logTo(logEl, "--- t advanced to " + (clock / 1000).toFixed(1) + "s (windows drain; open breakers age) ---", "lr-l-info");
    }));
    document.getElementById("lr-r-reset").addEventListener("click", wrap(function () {
      freshState();
      logEl.innerHTML = "";
      logTo(logEl, "[INIT] fresh state — all breakers closed, all windows empty");
    }));

    freshState();
    logTo(logEl, "[INIT] fresh state — all breakers closed, all windows empty");
    render();
  })();

  /* ---------------- Widget 2: one circuit breaker ---------------- */
  (function () {
    var THRESHOLD = 5, RESET = 2000;
    var logEl = document.getElementById("lr-cb-log");
    var s, clock;

    function fresh() { s = { state: "closed", cons: 0, openedAt: 0, probe: false }; clock = 0; }

    function render() {
      var box = document.getElementById("lr-cb-state");
      var cls = s.state === "closed" ? "lr-closed" : (s.state === "open" ? "lr-open" : "lr-half");
      var label = s.state === "closed" ? "CLOSED" : (s.state === "open" ? "OPEN" : "HALF-OPEN");
      box.className = "lr-statebox " + cls;
      box.innerHTML = '<span class="lr-dot"></span>' + label;
      document.getElementById("lr-cb-cons").textContent = String(s.cons);
      document.getElementById("lr-cb-probe").textContent = s.probe ? "YES" : "no";
      document.getElementById("lr-cb-clock").textContent = (clock / 1000).toFixed(1);
    }

    function allowed() {
      if (s.state === "closed") return true;
      if (s.state === "open") {
        var aged = clock - s.openedAt;
        if (aged >= RESET) {
          s.state = "half"; s.probe = false;
          logTo(logEl, "[CIRCUIT] OPEN -> HALF_OPEN (this call triggered the lazy transition)", "lr-l-warn");
        } else {
          logTo(logEl, "[CIRCUIT] OPEN — blocked (" + ((RESET - aged) / 1000).toFixed(1) + "s until reset)", "lr-l-fail");
          return false;
        }
      }
      if (s.probe) {
        logTo(logEl, "[CIRCUIT] HALF_OPEN — probe already in flight, blocked", "lr-l-fail");
        return false;
      }
      s.probe = true;
      logTo(logEl, "[CIRCUIT] HALF_OPEN — this call is the probe", "lr-l-warn");
      return true;
    }

    function call(fails) {
      clock += 100;
      if (!allowed()) { render(); return; }
      if (fails) {
        s.cons += 1;
        if (s.state === "half") {
          s.state = "open"; s.openedAt = clock; s.probe = false;
          logTo(logEl, "[CIRCUIT] probe FAILED → HALF_OPEN -> OPEN (clock restarts)", "lr-l-fail");
        } else if (s.cons >= THRESHOLD) {
          s.state = "open"; s.openedAt = clock;
          logTo(logEl, "[CIRCUIT] failure #" + s.cons + " → CLOSED -> OPEN", "lr-l-fail");
        } else {
          logTo(logEl, "[EXECUTOR] call failed (consecutive: " + s.cons + "/" + THRESHOLD + ")", "lr-l-fail");
        }
      } else {
        if (s.state === "half") {
          s.state = "closed"; s.probe = false; s.cons = 0;
          logTo(logEl, "[CIRCUIT] probe SUCCEEDED → HALF_OPEN -> CLOSED", "lr-l-info");
        } else {
          s.cons = 0;
          logTo(logEl, "[EXECUTOR] call succeeded (consecutive failures reset to 0)");
        }
      }
      render();
    }

    document.getElementById("lr-cb-okcall").addEventListener("click", function () { call(false); });
    document.getElementById("lr-cb-failcall").addEventListener("click", function () { call(true); });
    document.getElementById("lr-cb-wait").addEventListener("click", function () {
      clock += RESET;
      logTo(logEl, "--- waited 2.0s (no transition yet — the breaker has no timer; call it) ---", "lr-l-info");
      render();
    });
    document.getElementById("lr-cb-reset").addEventListener("click", function () {
      fresh(); logEl.innerHTML = "";
      logTo(logEl, "[INIT] breaker closed, counters zeroed");
      render();
    });

    fresh();
    logTo(logEl, "[INIT] breaker closed, counters zeroed");
    render();
  })();

  /* ---------------- Widget 3: sliding window ---------------- */
  (function () {
    var LIMIT = 10, WINDOW = 1000;
    var logEl = document.getElementById("lr-sw-log");
    var track = document.getElementById("lr-sw-track");
    var hits, clock, ok, den;

    function fresh() { hits = []; clock = 0; ok = 0; den = 0; }

    function evict() {
      var cutoff = clock - WINDOW;
      var evicted = 0;
      while (hits.length && hits[0] <= cutoff) { hits.shift(); evicted += 1; }
      return evicted;
    }

    function tryAcquire() {
      var ev = evict();
      if (ev > 0) logTo(logEl, "[RATE_LIMIT] evicted " + ev + " timestamp(s) older than 1s", "lr-l-dim");
      if (hits.length >= LIMIT) {
        den += 1;
        logTo(logEl, "[RATE_LIMIT] DENIED — " + hits.length + "/" + LIMIT + " in window", "lr-l-fail");
        return;
      }
      hits.push(clock);
      ok += 1;
      logTo(logEl, "[RATE_LIMIT] ALLOWED — " + hits.length + "/" + LIMIT);
    }

    function render() {
      evict();
      var old = track.querySelectorAll(".lr-ts");
      for (var i = 0; i < old.length; i++) track.removeChild(old[i]);
      var w = track.clientWidth - 20;
      hits.forEach(function (t) {
        var age = clock - t;
        var x = 10 + w * (1 - age / WINDOW);
        var d = document.createElement("div");
        d.className = "lr-ts";
        d.style.left = x + "px";
        track.appendChild(d);
      });
      document.getElementById("lr-sw-cap").textContent = hits.length + " / " + LIMIT + " in window";
      document.getElementById("lr-sw-clock").textContent = (clock / 1000).toFixed(2);
      document.getElementById("lr-sw-ok").textContent = String(ok);
      document.getElementById("lr-sw-den").textContent = String(den);
    }

    document.getElementById("lr-sw-one").addEventListener("click", function () { clock += 20; tryAcquire(); render(); });
    document.getElementById("lr-sw-burst").addEventListener("click", function () {
      logTo(logEl, "--- burst of 6 ---", "lr-l-info");
      for (var i = 0; i < 6; i++) { clock += 5; tryAcquire(); }
      render();
    });
    document.getElementById("lr-sw-tick").addEventListener("click", function () {
      clock += 250;
      logTo(logEl, "--- advanced 250ms — window slides right ---", "lr-l-info");
      render();
    });
    document.getElementById("lr-sw-reset").addEventListener("click", function () {
      fresh(); logEl.innerHTML = "";
      logTo(logEl, "[INIT] queue empty");
      render();
    });

    fresh();
    logTo(logEl, "[INIT] queue empty");
    render();
  })();
})();
</script>

---

*The full project — every class in this post, runnable with `dotnet run`, no dependencies — accompanies this article as `WebSearchRouter`. The planning doc above is the complete prompt that generated it.*
