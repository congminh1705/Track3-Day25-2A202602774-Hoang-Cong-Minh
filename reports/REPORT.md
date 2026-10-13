# Lab 25B — Game Day Report

**Họ tên / nhóm:** Hoang Cong Minh / 2A202602774  **Commit:** pending  **Điểm tự động (`make grade`):** 84 / 88

Tables below show seed 25. Final validation passed seeds 25, 26, and 27 for all six Game Day tapes and C1; unit, workflow, and public SLO checks also passed. The hidden SLO dataset is instructor-only and is not included in the 84/88 automatic score.

## 1. Game Day results

| Tape | Client | Availability | p95 (s) | Amplification | Retry ratio | Cost/turn | Recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4 | No |
| G1 | resilient | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0 | Yes |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155 | No |
| G2 | resilient | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0 | Yes |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0 | No |
| G3 | resilient | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0 | Yes |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0 | No |
| G4 | resilient | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0 | Yes |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | No |
| G5 | resilient | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | — | Yes |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0 | No |
| G6 | resilient | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0 | Yes |

G5 passes its 78% availability floor, but still loses 413 malformed turns (18.2% of turns); correctness of the request generator remains an upstream dependency. G2 resilient p95 is 10.92s, above the normal-turn target one might choose, while still meeting the tape's stated availability/amplification/recovery checks.

## 2. Client configuration and decisions

| Setting | Value | Reason |
|---|---:|---|
| attempt_timeout_s | 8 | Leaves time for fallback inside the 30s user deadline; G6 stayed at 100% availability with 5.961s p95. |
| max_attempts_per_provider | 2 | One same-provider retry limits load; across tapes amplification stayed 1.003–1.062. |
| backoff_base_s / cap_s | 0.25 / 4 | Full jitter spreads transient retries; G3 retry ratio was 0.034. |
| retry_budget_ratio | 0.1 | Bounds retries during sustained incidents; G2 amplification was 1.041. |
| breaker window_size / min_calls | 20 / 10 | Requires a meaningful sample while reacting within a short failure run. |
| breaker failure_rate_threshold | 0.5 | Opens at half unhealthy outcomes in the window. |
| breaker slow_call_s | 5 | Detects G4's slow A-region responses before timeout-only detection. |
| breaker open_duration_s | 10 | Allows recovery while limiting repeated probes. |
| breaker half_open_max_probes | 2 | Caps concurrent probes against a recovering deployment. |

Error decisions: `400/401/403/404/422 → release permit → GiveUp (no retry/failover)`; quota exhaustion → release, long provider cooldown, immediate fallback; rate limit → release, honor retry-after via shared cooldown, immediate fallback; overload/server error/timeout → record provider health, retry same provider only within attempt limit, budget and deadline, otherwise advance the ladder. HALF_OPEN admits capped probes; slow/failing probes reopen the breaker. Tool calls are only routed to tool-capable deployments.

## 3. Experiments and interpretation

The first experiment disabled effective breaker trips (`min_calls=1e9`) and allowed five attempts per provider. With the retry budget, G2 amplification/retry ratio/recovery were 1.380/0.058/0s; with the budget disabled they were 1.675/0.495/25s, and availability fell from 99.87% to 96.66%. On G3, disabling the budget increased retry ratio from 0.068 to 0.159. A functioning breaker reduces retries during known outages, but the budget remains a last bound when breaker thresholds are misconfigured or an incident slips through the window.

For the 30s-timeout experiment, G6 availability dropped from 100% to 94.88%, with 122 deadline give-ups; p95 among successful calls fell from 5.961s to 5.315s. That lower p95 is survivorship bias: more slow/hung turns were excluded as failures. G4 remained at 100% availability and p95 moved only from 5.562s to 5.608s because its provider responds slowly rather than hanging. In G4, the 8s timeout preserves room for fallback under the 30s user deadline.

Removing tool-capability filtering on G6 routed 258 tool attempts to gamma-mini, produced 258 `incompatible_model_output` give-ups, and reduced availability to 89.18% (from 100%). In product terms, the request asks the model to invoke a tool that model cannot invoke; its answer cannot complete the requested action.

## 4. Refund workflow

| Executor | Completion | Crashes | Duplicate refunds | Duplicate emails | Lost refunds | Overpaid | Wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 100% | 81 | 128 | 81 | 0 | 1,496,700 cents | 1.123 |
| yours | 100% | 66 | 0 | 0 | 0 | 0 | 0.303 |

Idempotency keys deduplicate a side effect when its response is lost, but do not let a restarted worker know which earlier pure steps already completed. Durable checkpoints avoid repeating those LLM calls. Checkpointing before payment is unsafe: a crash after writing “done” but before payment would cause resume to skip a refund that never happened. The implementation checkpoints only after a successful result and retries uncertain payment responses using the same stable key.

## 5. Alerting and error budget

| Incident | Expected | Naive | Burn-rate | Detection delay |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | false page | none | — |
| slow_leak | ticket | missed | ticket | 1,337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | false page | none | — |
| small_bump | none | false page | none | — |

30-day SLI: 25,901,755 requests, 55,618 bad, 99.7853% availability. The 99.5% budget allows 129,508.8 bad requests; 42.95% is consumed and 57.05% remains, so the measured budget does not call for a release freeze. `small_bump` affects 12% of requests for only 20 minutes: its short-window burn does not sustain the paired long/short thresholds, so paging would interrupt responders without indicating a continuing budget-threatening incident.

## 6. Postmortem — G2 naive client

**Summary.** At t=120 alpha lost 75% GPU capacity for three minutes. The naive client multiplied traffic through three retry layers without jitter, a shared retry budget, or a circuit breaker, keeping the provider queue overloaded after capacity returned. Overall availability was 45.60%, p95 latency 15.034s, and amplification 9.904.

![G2 attempts per second and rolling success ratio](gameday/G2_overload_storm.png)
Source data: [naive G2 timeseries](gameday/naive_G2_overload_storm_timeseries.csv) and [resilient G2 timeseries](gameday/resilient_G2_overload_storm_timeseries.csv).

**Impact.** Of 2,362 turns, 1,285 failed (54.4% of this tape's turns). The tape recovery metric was 155s after provider restoration at t=300. Assuming the same baseline traffic of 4 turns/s throughout a 30-day month and no other incidents, the month would contain 10,368,000 turns, with a 99.5% SLO budget of 51,840 failed turns. This incident would consume 1,285 / 51,840 = 2.48% of that monthly budget; the tape's failure percentage is not the monthly failure percentage.

| t (virtual s) | Event | Evidence from naive CSV |
|---:|---|---|
| 0–119 | Baseline | 7.54 attempts/s on average; all 895 completed calls succeeded. |
| 120 | Capacity loss starts | 5 attempts, 7/7 calls succeed at the boundary. |
| 130 | Retries begin multiplying | 10 attempts/s; 3/3 successes. |
| 150 | Queue failure | 163 attempts/s; 0/3 success. |
| 180–299 | Retry storm sustains failure | 138–151 attempts/s; sampled successful calls remain 0. |
| 300 | Alpha fully restored | 115 attempts in that second; 0/5 calls succeed. |
| 330–399 | System remains unstable | Average 67.46 attempts/s; only 0 successes in the sampled interval. |
| 400–499 | Partial recovery | 26.75 attempts/s average; 70.3% of completed calls succeeded. |
| 500–719 | Recovered | 3.49 attempts/s average; all 780 completed calls succeeded. |

**Trigger vs. root cause.** The trigger was the temporary GPU-capacity reduction on alpha. The root cause was retry amplification across application, gateway, and SDK layers with fixed delays and no budget or breaker; retries filled queues and consumed capacity needed by new calls. At t=300 the provider recovered, but abandoned timed-out requests continued consuming server slots, while the retry loop continued adding work faster than the queue could drain. This metastable feedback loop ended only after queued work cleared around t=500.

**What helped / hurt.** Restored capacity helped, but delayed recovery because in-flight abandoned work still occupied slots. The lack of jitter synchronized retries; retries after client timeout paid for work that could no longer help the user.

| # | Action | Type | Owner | Verification |
|---|---|---|---|---|
| 1 | Enforce one retry-owning layer and exponential full jitter. | Prevent | SDK platform | G2 CI amplification ≤ 1.2. |
| 2 | Add shared token-bucket retry budgets and provider breakers. | Mitigate | Agent runtime | G2 recovery ≤ 30s across three seeds. |
| 3 | Alert on attempt amplification, queue wait, and success ratio, not just provider health. | Detect | On-call/SRE | Alert within 5s when attempts exceed 2× logical calls or rolling success falls below 95%. |

## 7. Remaining weakness

G2 resilient p95 is 10.92s even though availability is 100%. A fallback can keep the service available while adding queue and cross-region latency. I would tune attempt deadlines from measured normal p99 and user deadline slack, then validate p95 and availability across hidden seeds before tightening timeout or opening breakers sooner.

## 8. Semantic cache

| Client | Availability | FAQ availability | Hit ratio | Stale served | Wrong answers | Outdated | Cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient, no cache | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | No |
| cached | 0.9120 | 0.9645 | 0.637 | 150 | 0 | 3 | 0.926 | Yes |

| Cache setting | Value | Evidence |
|---|---:|---|
| similarity_threshold | 0.96 | At 0.80 there were 175 wrong `other_question` answers (86.05% FAQ availability); 0.96 had zero wrong answers with 63.7% hits. At 0.97 there were also zero wrong answers but hit ratio fell to 57.9%. |
| ttl_s | 60 | 30s TTL yielded 2 outdated answers and 56.2% hits; 300s TTL yielded 95 outdated answers. Webhook invalidation handles announced refund-policy changes; TTL bounds unannounced shipping-fee changes. |
| max_stale_s | 600 | Serves known FAQ answers during the outage; 150 stale answers supported 96.45% FAQ availability. |

Personal questions are never cached because an order, refund, or points response is account data; partitioning by user still risks stale or incorrect state and expands the privacy boundary. During stale serving, page on provider health and stale-serving rate/age, and show cache contribution separately from live-provider success so a green aggregate availability number does not hide a provider outage. With `max_stale_s=0`, FAQ availability fell to 87.73%, versus 96.45% at 600s.
