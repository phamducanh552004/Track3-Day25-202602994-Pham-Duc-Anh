# Lab 25B - Reliability Game Day Report

**Ho ten / nhom:** Pham Duc Anh - 2A202602994
**Commit:** `2a4d7fd` (core implementation; this report and generated artifacts are committed immediately after it)
**Automatic score:** 84 / 88 (the remaining 4 automatic points use the instructor's hidden SLO dataset).

All numbers below come from the deterministic public simulator, seed 25. The source artifacts are committed under `reports/`.

## 1. Game Day result: naive versus resilient client

| Tape | Client | Availability | P95 s | Amplification | Retry ratio | Cost/good turn | Recovery s | Pass |
|---|---|---:|---:|---:|---:|---:|---:|---|
| G1 rate limit | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | No |
| G1 rate limit | resilient | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | Yes |
| G2 overload | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | No |
| G2 overload | resilient | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | Yes |
| G3 brownout | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | No |
| G3 brownout | resilient | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | Yes |
| G4 slow region | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | No |
| G4 slow region | resilient | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | Yes |
| G5 bad deploy/quota | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | n/a | No |
| G5 bad deploy/quota | resilient | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | n/a | Yes |
| G6 outage | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | No |
| G6 outage | resilient | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | Yes |

The resilient client passes all six public tapes. In particular it has zero retry-after violations, zero client-error retries, zero incompatible tool routes, and zero unneeded failovers in these runs.

## 2. Client configuration and decisions

| Parameter | Value | Reason and observed evidence |
|---|---:|---|
| attempt timeout | 8 s | Leaves room inside the 30 s user deadline for failover; G4 P95 is 5.562 s versus naive 13.764 s. |
| attempts/provider | 2 | One bounded retry prevents a transient failure from immediately moving traffic while avoiding a retry storm. |
| full-jitter backoff | 0.25 s / 4 s cap | G1 has zero retry-after violations and amplification 1.062. |
| retry budget | ratio 0.10, min 0.5/s, cap 20 | G3 retry ratio is only 0.034 despite a 30% brownout. |
| breaker window/min calls | 20 / 10 | Avoids opening on a few isolated errors while responding to sustained degradation. |
| breaker failure rate | 0.50 | Opens only after meaningful evidence of provider trouble. |
| slow call threshold | 5 s | G4 moves traffic away from the slow primary before timeouts accumulate. |
| breaker open time/probes | 10 s / 2 probes | Prevents a recovering provider from receiving a thundering herd. |

Decision path:

```text
client error -> release permit -> GiveUp (no retry, no failover)
quota error  -> release -> 300 s provider cooldown -> compatible fallback
rate limit   -> release -> respect retry-after cooldown -> compatible fallback
500/503/529/timeout -> record in breaker -> one budgeted full-jitter retry
                    -> compatible fallback when retry is unavailable
all routes unavailable -> wait only for a short cooldown that fits deadline, else GiveUp
```

## 3. Reliability experiments

The public baseline illustrates why retry limiting is required: in G2, three nested fixed retries produce 9.904 physical attempts per logical call and only 45.60% availability; the resilient design uses 1.041 attempts and reaches 100%. The retry budget remains a last line of defence even when the breaker is healthy: breakers react to provider-health samples, while the budget globally limits retries during a bad configuration or an unexpected failure mode.

An 8-second per-attempt timeout is intentionally below the 30-second user deadline. G6 can fail over before the user deadline, while G4 still has 100% availability and a 5.562-second P95. Measuring only successful calls can hide a long-timeout outage, so availability and give-up reasons must be evaluated alongside latency.

`ladder("tool")` filters out `gamma-mini`; tool steps therefore never route to a model that cannot execute the requested tool. In G6 `incompatible_routes=0`: in product terms, users do not receive a low-cost but incapable model for an operation that needs a tool call.

## 4. Exactly-once refund workflow

| Executor | Completion | Crashes | Duplicate refunds | Duplicate emails | Lost refunds | Overpaid cents | Wasted LLM/crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,496,700 | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

The executor derives a stable idempotency key from `(session_id, step)`, so a timeout after the payment side effect returns the already committed refund rather than issuing a second one. Checkpoints are still required: idempotency protects one side effect but cannot avoid repeated LLM/read work after a worker crash. Conversely, a checkpoint alone cannot protect a payment if a crash happens after the payment provider commits but before the checkpoint is written. Writing a checkpoint before payment would be worse: recovery would skip a refund that was never actually issued.

## 5. Burn-rate alerting and error budget

| Incident | Expected | Naive result | Burn-rate result |
|---|---|---|---|
| checkout_bug | page | page, delay 1 min | page, delay 27 min |
| provider_outage | page | page, delay 0 min | page, delay 4 min |
| deploy_blip | none | false page | no page |
| slow_leak | ticket | missed | ticket, delay 1337 min |
| degraded_evening | page | page, delay 214 min | page, delay 291 min |
| night_spike | none | false page | no page |
| small_bump | none | false page | no page |

The burn-rate rules generate three pages, one ticket, and zero false pages. Over the public 30-day data: 25,901,755 requests, 55,618 bad requests, and a 99.7853% availability. At a 99.5% SLO the allowed budget is 129,508.8 bad requests; 42.95% is consumed and 57.05% remains, so `freeze_releases=false`. A 20-minute 12% `small_bump` should not page because it does not burn both required windows fast enough to threaten the monthly budget.

## 6. Blameless postmortem: G2 overload storm with naive client

![G2 overload time series](gameday/G2_overload_storm.png)

**Summary.** At t=120 the primary lost 75% of its capacity. The naive client repeatedly retried at three layers, creating 34,475 physical attempts for 3,481 logical calls (amplification 9.904). Even though the provider restores at t=300, recovery takes 155 seconds because its FIFO queues still contain retry-generated work.

**Impact.** 1,285 of 2,362 user turns fail (availability 45.60%); successful-turn P95 reaches 15.034 seconds. The incident lasts 180 virtual seconds, then an additional 155 seconds of metastable recovery. Under a simple 4 turns/s monthly-throughput assumption (10,368,000 turns/month), a 0.5% budget is 51,840 failed turns; the observed 1,285 failures alone consume about 2.48% of that budget.

| Time | Event | Evidence |
|---:|---|---|
| 120 s | alpha capacity falls | scenario trigger: 75% GPU loss |
| 120-300 s | nested retries amplify queueing | 3,194 overloads, 2,737 timeouts, 26,298 rate limits |
| 300 s | provider capacity restores | scenario restore point |
| 300-455 s | queue remains congested | recovery_s=155, not 0 |
| 455 s | 95% success window returns | simulator recovery metric |

The trigger was a capacity loss; the root cause of the extended outage was unbounded, synchronized, nested retry behavior. A timeout does not cancel provider work, so client retries create still more queued work. Helpful controls are a single retry layer, full jitter, retry budget, breaker, and capability-aware fallback. The naive fixed delay and 36-attempt retry stack make the incident worse.

| # | Action | Type | Owner | Verification |
|---:|---|---|---|---|
| 1 | Remove nested retry layers; keep one budgeted retry policy | Prevent | Agent platform | G2 amplification <= 1.6 |
| 2 | Alert on amplification, queue delay, and breaker-open rate | Detect | SRE on-call | Synthetic G2 alert fires before deadline loss |
| 3 | Add breaker and compatible fallback ladder | Mitigate | Runtime owner | G2 recovery <= 30 s and G6 tool routes = 0 |

## 7. Remaining weakness

The design trades cost for availability during a broad outage: in G6, `beta` receives 2,118 attempts and cost per good turn rises to 2.189. If all compatible providers become slow rather than failing, the breaker can only reduce harm, not create capacity. A next improvement is provider-specific concurrency/admission control with cost-aware routing, then validation against the hidden tapes.

## 8. Semantic cache and stale-if-error

| Client | Availability | FAQ availability | Hit ratio | Stale served | Wrong answers | Outdated answers | Cost/good turn | Pass |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| resilient, no cache | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | No |
| cached | 0.9106 | 0.9616 | 0.561 | 160 | 0 | 7 | 0.977 | Yes |

| Cache parameter | Value | Reason |
|---|---:|---|
| similarity threshold | 0.99 | A loose 0.80 run gave 181 `other_question` wrong answers; strict matching gives zero. |
| TTL | 75 s | Bounds silent shipping-policy drift; final run has only 7 outdated answers (limit 15). |
| max stale | 180 s | Covers the 120-second all-provider outage and serves 160 stale responses safely. |

Only non-personal FAQ requests are eligible, so an order/refund/points answer cannot leak to another user even if the embeddings are similar. A cache dashboard should alert on `stale_served`, cache hit rate, invalidation rate, and provider give-ups: high stale serving may keep user availability green while providers are unhealthy.

The content-update webhook invalidates answers grounded in a changed source. It cannot cover the intentionally silent shipping update; TTL is the remaining bound on those outdated answers.
