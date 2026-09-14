# OTL SLO Scoring Model

This document defines the complete SLO scoring formula used by the Cumulo OTL to compute `posture_score` (formerly `consensus_health_score`, kept as a deprecated alias) and `posture_grade`.

---

## Philosophy

The scoring model is based on the **error budget** concept from Google SRE:

> *"SLOs set the target level of reliability for a service's customers. Error budgets provide a clear, objective metric that determines how unreliable a service is allowed to be within a single quarter."*
> — Beyer et al., *SRE Book*, ch. 3

A score of 100 means no active reliability concerns. Each detected risk condition reduces the budget.

**Design principle — infrastructure, not the chain.** Earlier versions of this model scored chain-wide consensus health observed from Cumulo's node (byzantine / missing validator counts, round duration). That conflated the *network's* health with *Cumulo's*, and could mark the score down for conditions entirely outside Cumulo's control. The model was rewritten around one question instead: **is Cumulo fulfilling its validator duty, and are its own core node and public RPC endpoint healthy right now?** Conditions belonging to the network as a whole (byzantine/missing validators, round duration) are still published, but only as context — they never move `posture_score`. See [What the Score Does NOT Penalise](#what-the-score-does-not-penalise).

The validator-core thresholds are not invented percentages — they are derived from the chain's own slashing parameters, so the score stays objective and auditable against on-chain state that anyone can re-derive.

---

## Score Formula

Field sources: `on_chain` = chain state (staking + slashing modules, `CHAIN`); `reliability` / `chain_context` / `validator_performance` = Prometheus + RPC (`PROM`/`RPC`, soft — degrade to `null`, never fabricated); `governance` = chain governance module for open proposals, `incidents.json` for closed ones (see [methodology.md](methodology.md)).

```
score = 100
flags = []

# ── 1. Validator core (on-chain, load-bearing) ───────────────────────
# jail_margin comes straight from the chain's own slashing params:
jail_margin = signed_blocks_window * (1 - min_signed_per_window)   # cosmos/slashing/v1beta1/params

if on_chain_core_available:      # jail_margin resolvable and missed_blocks numeric
    score -= min(60, round(60 * missed_blocks / jail_margin))
else:
    flags += ["on_chain_record_unavailable"]

# Grade-band ceilings used throughout (B = 70-84, C = 50-69, D < 50):
GRADE_B_CEIL, GRADE_C_CEIL, GRADE_D_CEIL, GRADE_TOMBSTONED_CEIL = 84, 69, 40, 20

# On-chain confirmation is the one source treated as load-bearing, not soft:
# Prometheus/RPC degrade gracefully without capping the grade, but "not
# jailed / bonded / slashed" can only be asserted from chain state, so its
# absence caps the grade instead of defaulting to a clean bill of health.
if on_chain.available != true:
    score = min(score, GRADE_B_CEIL)
    flags += ["on_chain_unconfirmed_grade_capped"]

# Live signing gap: head_height vs last_signed_height, right now.
gap = head_height - last_signed_height
if gap > 100:
    score -= 25 ; score = min(score, GRADE_D_CEIL)
    flags += ["signing_gap_outage"]                 # ongoing outage
elif gap > 20:
    score -= 10 ; score = min(score, GRADE_C_CEIL)
    flags += ["signing_gap"]                         # active signing gap

# ── 2. Operator-run infrastructure (RPC + own node), applied on top ──
target = chain_config.rpc_availability_target_pct or 99.0   # per-chain override, else default
if rpc_availability_7d_pct < target - 1:
    score -= 24 ; score = min(score, GRADE_C_CEIL)
    flags += ["rpc_availability_far_below_target"]
elif rpc_availability_7d_pct < target:
    score -= 12 ; score = min(score, GRADE_B_CEIL)
    flags += ["rpc_availability_below_target"]

if catching_up or (chain_freshness_sec > max(60, 10 * avg_block_time_sec)
                    and network_missing_validators <= 1):
    score -= 12 ; score = min(score, GRADE_B_CEIL)
    flags += ["own_node_stalled"]

# ── 3. State-machine ceilings (override the continuous score) ────────
if missed_blocks >= 0.5 * jail_margin:
    score = min(score, GRADE_C_CEIL)
    flags += ["signing_at_risk"]

if missed_blocks >= jail_margin or jailed or not bonded:
    score = min(score, GRADE_D_CEIL)
    flags += [ "jailed" if jailed else "not_bonded" if not bonded else "signing_critical" ]

if ever_slashed:                    # permanent — does not decay with time
    score = min(score, GRADE_C_CEIL)
    flags += ["historical_slashing_event"]

if tombstoned:
    score = min(score, GRADE_TOMBSTONED_CEIL)
    flags += ["tombstoned"]

# ── 4. Track record: upgrade discipline + governance participation ───
if ever_delayed_upgrade:            # a logged upgrade applied outside its window; permanent
    score = min(score, GRADE_B_CEIL)
    flags += ["delayed_upgrade_on_record"]

gov_target = chain_config.gov_participation_target_pct or 90.0
if governance_participation_pct < gov_target - 10:
    score -= 24 ; score = min(score, GRADE_C_CEIL)
    flags += ["governance_participation_far_below_target"]
elif governance_participation_pct < gov_target:
    score -= 12 ; score = min(score, GRADE_B_CEIL)
    flags += ["governance_participation_below_target"]
# only scored when governance.available and votes_eligible > 0

if governance_disclosure_gaps > 0:  # a cast vote not logged in incidents.json
    flags += ["governance_disclosure_gap"]   # informational — never scored

# ── Floor + grade ──────────────────────────────────────────────────
score = max(0, score)
grade = "A" if score >= 85 else "B" if score >= 70 else "C" if score >= 50 else "D"
```

---

## Grade Assignment

| Score | Grade | Meaning |
|---|---|---|
| 85–100 | **A** | Healthy — validator core sound, own infrastructure within target |
| 70–84 | **B** | One moderate issue — e.g. RPC/own node below target, an unconfirmed on-chain record, a delayed upgrade, or governance participation moderately below target |
| 50–69 | **C** | At-risk validator core, an active signing gap, a historical slashing event, or participation/RPC availability far below target |
| 0–49 | **D** | Critical — jailed, not bonded, at or past the jail margin, or an ongoing signing outage |

Several conditions **cap** the grade directly (state-machine ceilings) rather than only subtracting points — a single historical slashing event, for example, permanently caps the grade at C regardless of every other signal being clean, because the point deduction alone could be outweighed by an otherwise-perfect score.

---

## Risk Flags

Risk flags are attached to the SLO response as an array — the "why" behind a score reduction or ceiling.

| Flag | Trigger | Effect |
|---|---|---|
| `on_chain_record_unavailable` | Chain state fetch failed — `jail_margin`/`missed_blocks` not resolvable | No continuous core term applied (not the same as a penalty) |
| `on_chain_unconfirmed_grade_capped` | `on_chain.available !== true` | Cap at B (84) |
| `signing_gap_outage` | `head_height − last_signed_height` > 100 | −25, cap at D (40) |
| `signing_gap` | Gap 21–100 blocks | −10, cap at C (69) |
| `rpc_availability_far_below_target` | RPC 7d availability < target − 1pp | −24, cap at C (69) |
| `rpc_availability_below_target` | RPC 7d availability < target | −12, cap at B (84) |
| `own_node_stalled` | `catching_up`, or stale freshness with ≤1 missing validator network-wide | −12, cap at B (84) |
| `signing_at_risk` | Missed blocks ≥ 0.5 × jail margin | Cap at C (69), no separate deduction |
| `jailed` / `not_bonded` / `signing_critical` | Jailed, not bonded, or missed blocks ≥ jail margin | Cap at D (40) |
| `historical_slashing_event` | Ever slashed/double-signed (any point in Cumulo's record for this chain) | Cap at C (69), **permanent** |
| `tombstoned` | Validator currently tombstoned | Cap at 20 |
| `delayed_upgrade_on_record` | A logged mandatory upgrade was applied outside its announced window | Cap at B (84), **permanent** |
| `governance_participation_far_below_target` | Participation < target − 10pp | −24, cap at C (69) |
| `governance_participation_below_target` | Participation < target | −12, cap at B (84) |
| `governance_disclosure_gap` | A cast vote has no matching public disclosure in `incidents.json` | Informational — **0, never scored** |

`historical_slashing_event` and `delayed_upgrade_on_record` are deliberately permanent — a statute of limitations on either would mean the grade stops reflecting a real event in the validator's history. This is a policy decision, not an oversight.

---

## What the Score Does NOT Penalise

| Condition | Reason for exclusion |
|---|---|
| Byzantine / missing validators, round duration, block-processing latency | Chain-wide consensus conditions, not Cumulo's infrastructure — the superseded model scored these; published today only as network context |
| `duplicate_vote` / `late_votes` counters | Per-node, network-position-dependent signals — informational, not actionable |
| Snapshot lag / availability | A community service metric, not a consensus-reliability signal |
| Vote disclosure gaps | A transparency measure (is the vote publicly logged), not a reliability measure — surfaced as `governance_disclosure_gap` but never scored |
| Availability with < 1 day of Prometheus history | Insufficient data window — reported as `null`, never extrapolated or estimated |

**Design principle:** the model never fabricates a number to fill a data gap. A missing signal degrades to `null`/unscored rather than being coerced into a value that could produce a false reading in either direction.

---

## Posture Status vs Score

These are two independent signals:

| Signal | Source | What it measures |
|---|---|---|
| `posture_status` | RPC (real-time) | Is the node currently serving the chain? (`ready` / `syncing` / `stale` / `bootstrapping` / `maintenance` / `down`) |
| `posture_score` / `posture_grade` | On-chain state + Prometheus + governance | Is Cumulo fulfilling its validator duty, and is its own infrastructure within target? |

A node can be `ready` (RPC responding, not catching up) with a grade below A if, say, a historical slashing event caps it, or governance participation is below target. Conversely a node can score 100 but show `syncing` if it recently restarted. Both signals together give the complete picture — neither substitutes for the other.

---

## Implemented Since v2 Launch

The formula above superseded the original v2 model (chain-wide byzantine/missing-validator and latency penalties) over the course of September 2026, alongside the following additions:

- **On-chain validator core** (jail margin, signing gap, bonded/jailed/tombstoned state) — replaces chain-wide consensus penalties with signals specific to Cumulo's own validator.
- **Historical slashing/double-sign integrity cap** — permanent, sourced from `incidents.json`.
- **Upgrade discipline** — a logged upgrade applied outside its announced window permanently caps the grade at B.
- **Governance participation** — scored continuously against a per-chain target (default 90%); vote disclosure tracked alongside it but never scored.
- **On-chain confirmation as load-bearing** — an unresolvable chain-state fetch caps the grade at B rather than defaulting to a clean bill of health.
- **Rolling 30d/90d/lifetime availability with persistent storage** — daily rollup, `reliability.long_term` in the posture payload; degrades to `null` per window until enough history exists, never extrapolated.

No further changes to the scoring model are open in the backlog as of this writing. New chains inherit this same formula unchanged — only `jail_margin` inputs, RPC/governance targets, and `min_signed_per_window` are chain-specific configuration.

---

## References

- Beyer, B. et al. — *Site Reliability Engineering*. O'Reilly, 2016. Ch. 3 (Error Budgets), ch. 4 (SLOs). [https://sre.google/sre-book/](https://sre.google/sre-book/)
- Beyer, B. et al. — *The Site Reliability Workbook*, ch. 2: Implementing SLOs. O'Reilly, 2018. [https://sre.google/workbook/implementing-slos/](https://sre.google/workbook/implementing-slos/)
- Buchman, E. — *Tendermint: Byzantine Fault Tolerance in the Age of Blockchains*. [https://arxiv.org/abs/1807.04938](https://arxiv.org/abs/1807.04938)
- Cosmos SDK — `x/slashing` module documentation (source of `signed_blocks_window` / `min_signed_per_window`).
