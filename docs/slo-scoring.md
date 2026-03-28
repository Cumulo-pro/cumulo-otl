# OTL SLO Scoring Model

This document defines the complete SLO scoring formula used by the Cumulo OTL to compute `consensus_health_score` and `posture_grade`.

---

## Philosophy

The scoring model is based on the **error budget** concept from Google SRE:

> *"SLOs set the target level of reliability for a service's customers. Error budgets provide a clear, objective metric that determines how unreliable a service is allowed to be within a single quarter."*
> — Beyer et al., *SRE Book*, ch. 3

Applied to blockchain validator operations, the score answers:
**"How much of its reliability budget has Cumulo consumed right now?"**

A score of 100 means no active reliability concerns. Each detected risk condition reduces the budget.

---

## Score Formula

```
score = 100

# ── Network security (highest penalty) ──────────────────
if byzantine_validators > 0:
    score -= 40
    flags += ["byzantine_detected"]

# ── Liveness (chain-wide) ────────────────────────────────
if missing_validators > 1:
    score -= 20
    flags += ["multiple_missing_validators"]
elif missing_validators == 1:
    score -= 10
    flags += ["one_validator_missing"]

# ── Consensus latency ────────────────────────────────────
if block_processing_p50 > 500:   # ms
    score -= 15
    flags += ["high_block_processing_latency"]
elif block_processing_p50 > 200: # ms
    score -= 5
    flags += ["moderate_block_processing_latency"]

if round_duration_p95 > 2000:    # ms
    score -= 10
    flags += ["slow_consensus_rounds"]
elif round_duration_p95 > 1000:  # ms
    score -= 4
    flags += ["moderate_consensus_rounds"]

# ── Floor ────────────────────────────────────────────────
score = max(0, score)
```

---

## Grade Assignment

| Score | Grade | Meaning |
|---|---|---|
| 85–100 | **A** | Healthy — no significant concerns |
| 60–84 | **B** | Degraded — one or more moderate issues |
| 40–59 | **C** | Stressed — multiple issues present |
| 0–39 | **D** | Critical — severe reliability concerns |

---

## Risk Flags

Risk flags are attached to the SLO response as an array. They provide the "why" behind any score reduction.

| Flag | Trigger | Score Impact |
|---|---|---|
| `byzantine_detected` | byzantine_validators > 0 | −40 |
| `multiple_missing_validators` | missing_validators > 1 | −20 |
| `one_validator_missing` | missing_validators == 1 | −10 |
| `high_block_processing_latency` | block_processing_p50 > 500ms | −15 |
| `moderate_block_processing_latency` | block_processing_p50 > 200ms | −5 |
| `slow_consensus_rounds` | round_duration_p95 > 2000ms | −10 |
| `moderate_consensus_rounds` | round_duration_p95 > 1000ms | −4 |
| `duplicate_votes_present` | duplicate_vote counter > 0 | 0 (informational) |
| `late_votes_present` | late_votes counter > 0 | 0 (informational) |

---

## What the Score Does NOT Penalise

The following conditions are explicitly excluded from scoring:

| Condition | Reason for exclusion |
|---|---|
| `duplicate_vote` / `late_votes` counters | Monotonically increasing counters — high values are normal testnet noise, not actionable signals |
| `node_availability_7d_pct` when < 7 days of Prometheus history | Insufficient data window produces artificially low values |
| `rpc_p95_latency_ms` when null | Not instrumented by Story's CometBFT build |
| Snapshot lag | Service metric, not consensus reliability |

**Design principle:** The model errs on the side of false positives (unfairly *high* scores) rather than false negatives (unfairly *low* scores). An incorrectly low score damages institutional trust; an incorrectly high score would be caught by other signals (posture_status, last_signed_height).

---

## Posture Status vs Score

These are two independent signals:

| Signal | Source | What it measures |
|---|---|---|
| `posture_status` | RPC (real-time) | Is the node currently serving the chain? |
| `consensus_health_score` | Prometheus (5m window) | How healthy is consensus participation? |

A node can be `ready` (RPC responding, not catching up) but have a score below 100 if chain-wide consensus conditions are suboptimal. Conversely, a node can have score 100 but status `syncing` if it recently restarted.

Both signals together give a complete picture.

---

## Roadmap

Future versions of the scoring model will incorporate:

- **Rolling 30-day availability** (`availability_7d_pct` extended to 30d with persistent storage)
- **Incident count** (from `incidents.json` — unplanned downtime events reduce score)
- **Validator commission changes** (institutional transparency signal)
- **Governance participation rate** (votes cast vs proposals active)

---

## References

- Beyer, B. et al. — *The Site Reliability Workbook*, ch. 2: Implementing SLOs. O'Reilly, 2018. [https://sre.google/workbook/implementing-slos/](https://sre.google/workbook/implementing-slos/)
- Buchman, E. — *Tendermint: Byzantine Fault Tolerance*. [https://arxiv.org/abs/1807.04938](https://arxiv.org/abs/1807.04938)
