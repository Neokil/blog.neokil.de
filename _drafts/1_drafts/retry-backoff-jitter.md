---
layout: post
title: "Retries, Backoff, and Jitter"
subtitle: "How to stay resilient without DDoSing your dependencies"
description: "A practical guide to retries: how backoff and jitter shape traffic during spikes, with lessons from a real notification-system incident."
date:   2026-02-19 17:00:00 +0100
categories: general distributed-systems reliability
---

Retries look easy at first:

```text
if request failed: retry
```

In production, that tiny line can make your system either resilient or unstable.

We learned this the hard way in our notification system.

We had traffic spikes where a lot of notifications were triggered in a short window. Our downstream provider started timing out and returning transient errors.

At that point, we were still on linear backoff with no jitter. It looked simple, but under spike conditions it kept retries too synchronized.

But in reality, retries lined up too closely. We got repeated retry waves: a big spike, then another spike, then another smaller one. It eventually recovered, but only after several rounds of avoidable pressure.

This is where **backoff** and **jitter** matter.

# Why naive retries fail
Imagine a burst of notification jobs hitting one downstream API. The API returns `503` for a short period.

If every client retries immediately, then retries again immediately, you get synchronized spikes:

1. Original traffic spike.
2. First retry spike.
3. Second retry spike.

The dependency has no breathing room to recover.

Even worse, if you have multiple layers (frontend -> service A -> service B -> database), retries can multiply across layers. A single user action can fan out into many repeated calls.

# The core pattern: exponential backoff
Backoff means waiting before retrying. Exponential backoff increases the wait each time.

Example with base delay of `100ms`:

- attempt 1 retry delay: `100ms`
- attempt 2 retry delay: `200ms`
- attempt 3 retry delay: `400ms`
- attempt 4 retry delay: `800ms`

Usually you also cap the delay:

```text
delay = min(maxDelay, base * 2^attempt)
```

This reduces pressure on the dependency and avoids burning your own CPU/network on tight retry loops.

# Why jitter is non-negotiable
Without jitter, all clients using the same backoff formula still retry at nearly the same timestamps.

Jitter randomizes each delay so retries spread out over time. That smooths traffic and gives recovering systems a chance.

Common strategies:

1. Full jitter:
`sleep = random(0, backoffDelay)`
2. Equal jitter:
`sleep = backoffDelay/2 + random(0, backoffDelay/2)`
3. Decorrelated jitter:
Next delay is random between a base and ~3x previous delay (bounded by max).

Full jitter is often a strong default, and AWS guidance plus simulations show why it works well in many systems.

# Why we went with `+-50%` jitter
Our journey looked like this:
1. Linear backoff, no jitter (original setup).
2. Exponential backoff, no jitter (first improvement).
3. Exponential backoff, `+-10%` jitter (second improvement).
4. Exponential backoff, `+-50%` jitter (final choice).

Each step improved things, but only the last one spread retries enough for our notification spikes while still keeping a minimum retry delay.

Example with base backoff `30ms`:
- no jitter: all retries at `30ms`
- `+-10%` jitter: retries still cluster in a narrow `27-33ms` window
- `+-50%` jitter: retries spread much wider in a `15-45ms` window
- full jitter: retries spread across `0-30ms`

When many retries are involved, `+-10%` still creates dense mini-spikes. They are smaller than no jitter, but still synchronized enough to keep overwhelming the same downstream bottleneck.

In theory, full jitter spreads load even more aggressively. But for our notification flow, we intentionally wanted to keep a minimum delay between attempts and avoid retries happening too close to zero delay.

So we moved to `+-50%` jitter. That gave us much better spreading than `+-10%`, while preserving a floor on delay.

Another way to think about it:
- `+-10%` jitter mostly shifts a spike.
- `+-50%` jitter actually broadens and flattens it for our latency budget.
- full jitter can retry too early for our use case, when the downstream service still has not recovered.

# What should be retried?
Not every error is retryable.

Usually retryable:
- network timeouts
- connection resets
- HTTP `429` (rate limited)
- HTTP `502`, `503`, `504`

Usually not retryable:
- HTTP `400`, `401`, `403`, `404`
- validation errors
- business rule violations
- authentication/authorization failures (unless token refresh is part of logic)

Also: only retry operations that are **idempotent** or protected by an idempotency key. Stripe has an excellent practical write-up on why this matters in real APIs.

# Anti-patterns to avoid
- Retrying forever.
- Retrying every error blindly.
- No timeout or deadline.
- Using retries as a substitute for fixing latency/availability issues.
- Stacking retries at every layer without coordination.

# Retries are only one part of resilience
Retries help with short-lived failures. They do not solve systemic overload by themselves.

Pair retries with:
- circuit breakers: prevent calls to unhealthy dependencies and allow fast failure while they recover.
- load shedding: drop or defer non-critical work when the system is overloaded.
- concurrency limits: cap in-flight requests so downstream systems are not saturated.
- queues/buffers where appropriate: absorb short spikes and smooth bursty traffic over time.
- proper observability (retry count, attempt latency, final failure reason): make retry behavior visible so you can tune policies and detect retry storms early.

If your dashboards do not show retry volume and success-after-retry rates, you are mostly flying blind.

# Conclusion
Retries are a powerful tool, but only when controlled.

The formula is simple:

1. Retry only transient failures.
2. Back off exponentially.
3. Add jitter.
4. Bound everything with attempt and time limits.

Done well, retries make your system graceful under turbulence. Done poorly, they become the turbulence.

In our case, the biggest change was moving from linear/no-jitter behavior to exponential backoff with `+-50%` jitter, after evaluating exponential backoff without jitter and with `+-10%` jitter on the way.

Our timings are also intentionally short:
- initial backoff: `30ms`
- max retry timeout cap: around `1.5s`

That combination fit our notification workload better than a full-jitter strategy with near-zero delays.

# References
- AWS Architecture Blog: [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- Amazon Builders’ Library: [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- Stripe Blog: [Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
