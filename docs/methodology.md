# OTL Methodology: Metric Catalogue & Justification

This document describes every metric published by the Cumulo OTL, its data source, what it measures, and the institutional justification for its inclusion. It is chain-agnostic: the OTL applies the same `cosmos-cometbft` profile to every CometBFT chain it supports, with only endpoints, chain filters, and a handful of thresholds configured per chain. For a specific chain's concrete values (job labels, thresholds, validator address), see its profile under [`chains/`](../chains/).

---

## Data Source Labels

| Label | Source | Description |
|---|---|---|
| `RPC` | CometBFT `/status` | Direct from the node's standard status endpoint |
| `PROM` | Prometheus exporter | From `cometbft_*` metrics scraped at 15s intervals, self-reported by the node binary |
| `VALIDATOR` | Prometheus (validator-specific) | Filtered by Cumulo's validator address |
| `CHAIN` | Chain state (staking + slashing modules) | Read from the chain itself via Cumulo's own block explorer, re-derivable by anyone from any LCD, independent of the node's own telemetry |
| `CHECK_D` | check_d (independent collector) | External probe of the public RPC endpoint, run by Cumulo but observing from outside the node, same as any third party would |
| `SNAPCHECK` | snapshot_checker (independent collector) | External download and inspection of the published snapshot / state-sync endpoints |
| `DERIVED` | Computed | Calculated from one or more of the above |

The two independent collectors (`CHECK_D`, `SNAPCHECK`) are Cumulo-built and Cumulo-run, open source. What makes their output useful as a cross-check is that they measure from outside the node and don't edit results after collection, not that a third party operates them. See [Independent Collectors](../README.md#independent-collectors) in the README.

---

## 1. Service Posture

### `posture_status` · `DERIVED`

**States:** `ready` · `syncing` · `stale` · `bootstrapping` · `maintenance` · `down`

Derived from RPC fields:
- `catching_up` (boolean), from `/status → sync_info.catching_up`
- `chain_freshness_sec` (now − latest_block_time), computed from the RPC block timestamp

**Logic:**
```
if maintenance.enabled                → "maintenance"
if rpc unreachable                    → "down"
if freshness > 120s AND catching_up   → "bootstrapping"
if freshness > 120s                   → "stale"
if catching_up                        → "syncing"
else                                  → "ready"
```

**Justification:** SRE teams require a single binary signal for escalation decisions. The traffic-light pattern is standard in on-call runbooks. (Beyer et al., SRE Book, ch. 6, Monitoring Distributed Systems)

---

### `posture_score` / `posture_grade` · `DERIVED`

A composite 0-100 score starting at 100, reduced by detected risk conditions specific to **Cumulo's own infrastructure**, not chain-wide consensus health. Expressed as a letter grade: **A** (≥85) · **B** (70-84) · **C** (50-69) · **D** (<50).

See [slo-scoring.md](slo-scoring.md) for the complete formula, risk-flag catalogue, and the design rationale for scoring only Cumulo's infrastructure rather than the network as a whole.

**Justification:** Composite scoring is the standard method for communicating complex system health to non-technical stakeholders. Anchoring thresholds to the chain's own slashing parameters (`jail_margin`), rather than arbitrary percentages, keeps the grade objective and auditable.

---

## 2. Chain at a Glance

### `head_height` · `RPC`

Latest block height seen by the RPC node. From `/status → sync_info.latest_block_height`.

**Justification:** Block height is the fundamental liveness indicator for any blockchain node. A stagnant height is the first observable sign of a node outage.

---

### `chain_freshness_sec` · `DERIVED`

Age of the most recent block seen by the RPC node: `now − latest_block_time` (seconds). Triggers `stale` posture above 120s.

**Justification:** Freshness is more reliable than block height alone because it accounts for clock drift and network partitions. It is the standard "freshness probe" used in blockchain health checks.

---

### `avg_block_time_sec` · `PROM`

Average block production interval over 5 minutes. Derived from `cometbft_consensus_block_interval_seconds` (sum/count ratio over the window).

**Justification:** Block time variability is a leading indicator of consensus stress. Sustained elevation above a chain's target precedes missed blocks and potential slashing, giving earlier warning than availability metrics alone.

---

### `validators_count` / `missing_validators` / `byzantine_validators` · `PROM`

Chain-wide validator set health metrics:
- `cometbft_consensus_validators`, total active validators
- `cometbft_consensus_missing_validators`, validators not voting in the current round
- `cometbft_consensus_byzantine_validators`, validators with equivocation evidence

**Not scored.** These are network-wide conditions, not Cumulo's responsibility, and do **not** move `posture_score`. See [slo-scoring.md § What the Score Does NOT Penalise](slo-scoring.md#what-the-score-does-not-penalise). The payload also derives a summary `network_context.state` (`nominal` / `degraded`) from these same counters plus round duration, purely for dashboard display.

**Justification for publishing them anyway:** network-level validator health still affects the service Cumulo can provide. If more than 1/3 of stake weight is offline, the chain halts, so it is disclosed as context even though it is not graded. These are the primary consensus-health signals in BFT theory. (Buchman et al., 2018, Tendermint: Byzantine Fault Tolerance in the Age of Blockchains)

---

## 3. Validator Performance

### `voting_power` / `voting_power_pct` · `VALIDATOR`

Cumulo's consensus voting power in the current validator set. From `cometbft_consensus_validator_power` filtered by Cumulo's validator address, expressed both in absolute units and as a percentage of `cometbft_consensus_validators_power` (total network power).

This is Cumulo's share of the network's total consensus voting power, however that power is allocated on a given chain: proportional to bonded stake on a standard Proof-of-Stake chain, or a weight assigned directly by governance on a Proof-of-Governance chain such as XRPL EVM (see the chain's profile for which model applies, and for XRPL EVM specifically, an equal weight across the active set rather than stake-proportional). This is a **separate, independently sourced** figure from the `CHAIN`-labeled voting weight in § 4 below (Prometheus/consensus vs. staking-module/governance). On a standard Proof-of-Stake chain the two track each other closely by construction, since consensus power is derived from bonded stake. On XRPL EVM they coincide only because that chain's governance-assigned allocation happens to be even across the set, a property of that chain's token allocation, not a computation this system performs.

**Justification:** Voting power determines Cumulo's influence in consensus and is standard disclosure in institutional validator performance reports.

---

### `last_signed_height` · `VALIDATOR`

The most recent block height at which Cumulo's validator signed. From `cometbft_consensus_validator_last_signed_height`, compared against `head_height` to compute the live signing gap that feeds `posture_score` (see slo-scoring.md).

**Justification:** This is the most direct and honest indicator of active consensus participation, more honest than a raw "uptime" percentage, which only measures whether a process is running. A node can be online but not signing; this metric, cross-checked against the chain's own missed-blocks counter (§ 4), detects that scenario.

---

### `node_availability_7d_pct` / `rpc_availability_7d_pct` · `PROM`

Probe-based uptime percentage. Computed from:

```promql
avg_over_time(up{job="<node job label>"}[Nd]) * 100
avg_over_time(up{job="<rpc job label>"}[Nd]) * 100
```

Where `up` is Prometheus' standard scrape success indicator (1 = reachable, 0 = unreachable), evaluated every 15 seconds. `N` is 7 when that much scrape history exists; when a chain was onboarded more recently, the window is the actual coverage instead (never extrapolated to a full 7 days). The payload states the real window in `data_window.availability` and numerically in `reliability.availability_coverage_days`.

`rpc_availability_7d_pct` (against the configured per-chain target, default 99.0%) is the figure that feeds `posture_score`.

**Justification:** A probe-based availability window is the standard SLA reporting period in infrastructure services, directly comparable to cloud-provider SLAs. (Google SRE Workbook, ch. 2, Implementing SLOs)

---

### `reliability.long_term` (30d / 90d / lifetime) · `DERIVED`

A daily rollup of `node_availability_pct` / `rpc_availability_pct`, persisted independently of Prometheus' own retention window, read back and averaged over 30 days, 90 days, and since `validator_since` (or since the rollup began, if that date isn't configured for the chain). Each window is `null`, never a guess, until at least one full day of rollup history exists for it, and only fully-closed days are counted (never an in-progress day averaged in as if complete).

**Justification:** Prometheus' own retention is finite and operational (tuned for debugging, not institutional reporting). A persistent daily rollup gives delegators and partners a long-horizon reliability figure that outlives Prometheus' TSDB retention window, with the same "never fabricate, never extrapolate" discipline as the 7-day figure above.

---

### `validator_address` / `validator_address_prom` · `RPC` + `VALIDATOR`

Cumulo's canonical CometBFT consensus address: `SHA256(consensus_pubkey)` truncated to 20 bytes, the same value a block explorer shows, and the same string used as the `validator_address` Prometheus label. Both fields carry the same canonical address, resolved with a fixed precedence (chain config, then Prometheus-detected, then RPC) rather than whichever node a given RPC target happens to front. A public RPC endpoint is commonly a non-signing sentry node, and reading its own `/status` address would silently report the wrong validator.

**Justification:** Publishing the validator address is a basic transparency requirement. It lets any third party independently verify Cumulo's signing activity via a block explorer, without depending on Cumulo's own reporting.

---

## 4. On-Chain Validator Record · `CHAIN`

Sourced entirely from chain state (staking + slashing modules), independent of the node's own telemetry, and re-derivable by anyone from any LCD or full node via `cosmos/staking/v1beta1/validators/<valoper>` and `cosmos/slashing/v1beta1/signing_infos/<valcons>`. If this fetch fails, `posture_grade` is capped at B (`on_chain_unconfirmed_grade_capped`, see slo-scoring.md) rather than defaulting to a clean bill of health. This is the one source in the system treated as load-bearing instead of soft.

These are the standard `x/staking` and `x/slashing` fields present on any Cosmos SDK / CometBFT chain, regardless of its consensus-economics model. What a given field represents in practice, delegated stake vs. a governance-assigned weight, a real commission market vs. one fixed at zero, depends on the chain, noted per field below and detailed fully in that chain's profile under [`chains/`](../chains/).

### `signing_uptime_pct` / `missed_blocks` / `signed_blocks_window` / `jail_margin`

`missed_blocks_counter` over `signed_blocks_window`, straight from the chain's `x/slashing` module: the canonical, third-party-verifiable signing record the chain itself uses to decide jailing, and the only signing metric that can't be gamed by keeping a node running without signing. `jail_margin = signed_blocks_window × (1 − min_signed_per_window)` is the number of blocks Cumulo can still miss in the window before the chain itself would jail the validator; it anchors the validator-core term of `posture_score`.

**Justification:** Unlike a self-reported uptime percentage, this is the exact figure the chain uses to decide jailing; nothing about it can be improved by anything other than actually signing blocks.

### `bonded` / `jailed` / `tombstoned` / `active`

Position in the active set. `status` (`BOND_STATUS_BONDED`) and the `jailed` / `tombstoned` flags from the staking module.

**Justification:** An unbonded or jailed validator isn't earning or participating; this is the first thing a delegator or integrator checks.

### `tokens` / `voting_share_pct`

Bonded weight (tokens) and its share of the active set, from the staking module. On a standard Proof-of-Stake chain this is the validator's total bonded stake (self-bond plus delegations), accumulated through staking and adjustable by delegators moving their stake. On a Proof-of-Governance chain such as XRPL EVM, the same field instead carries a weight assigned directly by governance vote, equal across the active set rather than accumulated through delegation. The chain's profile states which model applies and, for a Proof-of-Stake chain, the actual delegation figures. See § 3 above for how this compares to the separately sourced consensus voting-power figure.

**Justification:** The on-chain counterpart to the Prometheus-derived `voting_power_pct` in § 3, from an independent source, and on a Proof-of-Stake chain the primary concentration-risk figure delegators use to evaluate a validator.

### `commission_rate` / `commission_max_rate` / `commission_max_change_rate` / `commission_updated_at`

From the staking module's `commission` object: the delegator-facing fee a validator charges on staking rewards, and the ceiling and rate-of-change limit on how far it can move. On a standard Proof-of-Stake chain this is an operator-set figure with real weight, since it directly determines delegator returns. On a Proof-of-Governance chain with no delegation market or delegator rewards, such as XRPL EVM, these fields are commonly fixed at 0% by protocol design rather than by operator choice. The OTL states the figure explicitly either way, with the chain's profile explaining which case applies, rather than omitting fields that don't carry their usual meaning on a given chain.

**Justification:** Commission and its change history is the most frequently asked question among delegators on Proof-of-Stake chains; on a Proof-of-Governance chain the answer is structurally fixed, and showing it explicitly, rather than omitting a field that doesn't apply, keeps the disclosure honest either way.

---

## 5. Track Record & Governance

Computed from the public [incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json) activity log and, for open proposals, the live chain governance module, independent of node telemetry.

### `integrity.ever_slashed` / `events`

Whether any slashing/jailing/double-sign incident is on record for this chain and network, matched by structured `incident_type` (never a free-text search). A single event caps `posture_grade` at C, permanently. See slo-scoring.md.

### `integrity.ever_delayed_upgrade` / `delayed_upgrades` / `total_upgrades_tracked`

Whether any logged mandatory upgrade (`incident_type` in the `upgrade` family) was applied outside its announced window. A single occurrence caps `posture_grade` at B, permanently.

**Justification (both):** A validator's integrity and process-discipline record shouldn't decay with time just because nothing has gone wrong recently. These are permanent marks on record by design, not metrics that age out.

### `governance.votes_eligible` / `votes_cast` / `participation_pct`

Every proposal Cumulo was eligible to vote on since becoming a validator on this chain (excludes proposals still in deposit period, and anything whose voting window closed before `validator_since`), cross-referenced against Cumulo's actual vote. `participation_pct` feeds `posture_score` against a per-chain target (default 90%).

**Source, by proposal state, which is not a design choice but a constraint of the chain itself:** while a proposal is still open, its vote is verified live against the chain's governance module. Once a proposal closes, most Cosmos SDK chains (inherited by CometBFT sidechains generally, XRPL EVM included) prune the individual vote record after tallying; only the aggregate result survives on-chain. For closed proposals, which is the large majority of any validator's history, participation is therefore read from `incidents.json`, the only place the fact still exists. A consequence, stated explicitly rather than left implicit: for closed proposals, `disclosure_pct` below coincides with `participation_pct` by construction, since both draw on the same tracker entry. The distinction between the two is only meaningful for proposals still open, where an on-chain vote can be confirmed before Cumulo has logged it.

### `governance.votes_disclosed` / `disclosure_pct` / `disclosure_gaps`

Whether a cast vote also has a public disclosure entry (with its transaction hash) logged in `incidents.json`. **Never scored**: `governance_disclosure_gap` is informational only, a transparency measure rather than a reliability one.

**Justification:** Governance participation, verified against the most complete source available for each proposal's state, is a standard accountability signal for institutional validators. Separating it from voluntary public disclosure keeps the score honest about what it actually measures.

---

## 6. Consensus Performance · `PROM`

All histogram quantiles are aggregated across scraped instances with `sum(rate(bucket[5m])) by (le)`.

### `block_processing_ms` (p50 / p95)

Time to process each block. From the `cometbft_state_block_processing_time` histogram; check the chain's exporter build for its unit (seconds vs. milliseconds). The payload always reports milliseconds.

**Justification:** Block processing time is the primary latency indicator for consensus participation. If processing exceeds the prevote timeout window, the validator misses the voting round. (CometBFT docs: Metrics, consensus)

### `round_duration_ms_p95`

p95 consensus round duration, from `cometbft_consensus_round_duration_seconds` × 1000: total time from round start to completion.

**Justification:** Round duration directly determines block time. Sustained elevation indicates consensus difficulty (missing validators, network latency, byzantine behaviour) before it manifests as missed blocks. Not scored (see § 2); published as network context.

### `step_duration_ms_p95`

p95 duration of each consensus step (Propose / Prevote / Precommit / Commit), excluding protocol wait steps.

**Justification:** Breaks round duration down by phase, useful for diagnosing where consensus time is actually spent, rather than only the aggregate.

### `quorum_prevote_delay_ms` / `quorum_precommit_delay_ms`

Time for ≥2/3 of voting power to submit prevotes and precommits respectively. Gauge metrics, in milliseconds.

**Justification:** Quorum delays are early-warning signals for network partition or validator degradation, directly preceding round timeouts that cascade into missed blocks.

### `peers_avg`

Average number of active P2P connections, from `cometbft_p2p_peers`.

**Justification:** Fewer than roughly 3 peers significantly increases partition risk, a standard connectivity health indicator in P2P system monitoring.

### `p2p_recv_bytes_per_sec` / `p2p_send_bytes_per_sec`

P2P bandwidth, 5-minute rate, from `cometbft_p2p_peer_receive_bytes_total` / `..._send_bytes_total`.

**Justification:** Near-zero traffic indicates effective disconnection from the network; unusual spikes may indicate spam or gossip amplification.

---

## 7. Vote Anomaly Counters (Informational) · `PROM`

### `duplicate_vote_5m` / `late_votes_5m` / `duplicate_block_part_5m`

Per-node counters, scoped to Cumulo's own validator instance and split by vote type. **Not** a chain-wide count and **not** attributable to any particular validator's behaviour, including Cumulo's.

> **Note on magnitude:** these track the underlying CometBFT counter's own units, which aren't formally defined, and the figure is dependent on network position, not a global chain quantity. Never scored.

**Justification for inclusion:** published for full data disclosure even though the raw magnitude alone isn't independently actionable; only an extreme spike per interval would warrant investigation.

---

## 8. Independent Verification (check_d) · `CHECK_D`

A second, external measurement of the public RPC endpoint, run by Cumulo but observing from outside the node on its own schedule, independent of both the OTL node's self-reporting and of Prometheus.

- **Reliability / average and smoothed latency:** probe history and response time, smoothed exponentially and as the latest raw sample.
- **Per-region latency:** measured from multiple probe locations, something neither the node itself nor local Prometheus scraping could ever produce, since it requires observing from outside the network position of the node.
- **TLS validity / days to expiry:** whether the HTTPS certificate on the public endpoint is currently valid, and how soon it expires.
- **Block height cross-check:** check_d's own last-probed height compared against the OTL's live `head_height`, flagged only when the difference exceeds what check_d's own probe age can explain.

**Justification:** a system's report about itself can be wrong in ways an external prober would catch, the same double-verification principle applied to signing activity in § 4, here applied to endpoint responsiveness instead.

---

## 9. Snapshots · `SNAPCHECK` + `DERIVED`

- **Snapshot lag:** `head_height − snapshot_block_height`, computed live on every dashboard load from whatever height the published snapshot page reports.
- **Availability:** the OTL fetches the snapshot page on each request and verifies the archive is present and reachable. The archive layout is per-chain, a single combined archive covering full node state, or separate consensus/execution-client files, depending on what the chain publishes.
- **State-sync parameters**, where the chain provides them (seed/RPC servers, trust height/hash).
- **Independent cross-check:** `snapshot_checker` downloads and inspects the same published snapshot from outside this page's own fetch (size, last-modified time, measured download speed), rather than trusting the page's own retrieval of the index. Shows "not yet cross-checked" for a chain with no entry in `chains_snapshot.json`.

**Justification:** the snapshot is a community service, not a validator obligation. An external download check verifies it is actually current and fetchable, not merely present.

---

## References

1. Beyer, B. et al., *Site Reliability Engineering: How Google Runs Production Systems*. O'Reilly, 2016. [https://sre.google/sre-book/](https://sre.google/sre-book/)
2. Beyer, B. et al., *The Site Reliability Workbook*. O'Reilly, 2018. [https://sre.google/workbook/](https://sre.google/workbook/)
3. Buchman, E., Kwon, J., Milosevic, Z., *Tendermint: Byzantine Fault Tolerance in the Age of Blockchains*. 2018. [https://arxiv.org/abs/1807.04938](https://arxiv.org/abs/1807.04938)
4. CometBFT, *Metrics Reference*. [https://docs.cometbft.com/v1.0/references/metrics](https://docs.cometbft.com/v1.0/references/metrics)
5. Prometheus Authors, *Data Model & PromQL*. [https://prometheus.io/docs/concepts/data_model/](https://prometheus.io/docs/concepts/data_model/)
6. Lido Finance, *ValOS: Validator Operator Standard*. Formal reference for the operational-risk disclosure categories this document and the OTL dashboard's infrastructure-disclosure section follow. [https://github.com/lidofinance/valos](https://github.com/lidofinance/valos)
7. Rated Network, independent validator rating/analytics; cited as commercial precedent for validator-transparency products. [https://rated.network/](https://rated.network/)
