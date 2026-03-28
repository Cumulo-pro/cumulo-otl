# OTL Methodology — Metric Catalogue & Justification

This document describes every metric published by the Cumulo OTL, its data source, what it measures, and the institutional justification for its inclusion.

---

## Data Source Labels

| Label | Source | Description |
|---|---|---|
| `RPC` | CometBFT `/status` | Direct from the node's standard status endpoint |
| `PROM` | Prometheus exporter | From `cometbft_*` metrics scraped at 15s intervals |
| `VALIDATOR` | Prometheus (validator-specific) | Filtered by Cumulo's validator address |
| `DERIVED` | Computed | Calculated from one or more raw sources |

---

## 1. Service Posture

### `posture_status` · `DERIVED`

**States:** `ready` · `syncing` · `stale` · `bootstrapping` · `maintenance` · `down`

Derived from two RPC fields:
- `catching_up` (boolean) — from `/status → sync_info.catching_up`
- `chain_freshness_sec` (now − latest_block_time) — computed from the RPC block timestamp

**Logic:**
```
if maintenance.enabled       → "maintenance"
if rpc unreachable           → "down"
if freshness > 120s AND catching_up   → "bootstrapping"
if freshness > 120s                   → "stale"
if catching_up                        → "syncing"
else                                  → "ready"
```

**Justification:** SRE teams require a single binary signal for escalation decisions. The traffic-light pattern is standard in on-call runbooks. *(Beyer et al., SRE Book, ch. 6 — Monitoring Distributed Systems)*

---

### `consensus_health_score` / `posture_grade` · `DERIVED`

A composite 0–100 score starting at 100, reduced by detected risk conditions. Expressed as a letter grade: **A** (≥85) · **B** (≥60) · **C** (≥40) · **D** (<40).

See [slo-scoring.md](slo-scoring.md) for the complete formula.

**Justification:** Composite scoring is the standard method for communicating complex system health to non-technical stakeholders. Used in Datadog SLO dashboards, Google SRE error budgets, and institutional SLA reports.

---

## 2. Chain at a Glance

### `head_height` · `RPC`

Latest block height seen by the RPC node. From `/status → sync_info.latest_block_height`.

**Justification:** Block height is the fundamental liveness indicator for any blockchain node. A stagnant height is the first observable sign of a node outage.

---

### `chain_freshness_sec` · `DERIVED`

Age of the most recent block seen by the RPC node: `now − latest_block_time` (seconds).

- ≤10s: real-time sync (green)
- ≤120s: acceptable
- >120s: triggers `stale` posture

**Justification:** Freshness is more reliable than block height alone because it accounts for clock drift and network partitions. It is the standard "freshness probe" in blockchain health checks (Chainstack, Infura, Alchemy status pages all use this signal).

---

### `avg_block_time_sec` · `PROM`

Average block production interval over 5 minutes. Derived from `cometbft_consensus_block_interval_seconds` histogram (sum/count ratio).

**Justification:** Block time variability is a leading indicator of consensus stress. Sustained elevation above the target (Story Aeneid: ~2s) precedes missed blocks and potential slashing. It provides earlier warning than availability metrics alone.

---

### `validators_count` / `missing_validators` / `byzantine_validators` · `PROM`

Chain-wide validator set health metrics:
- `cometbft_consensus_validators` — total active validators
- `cometbft_consensus_missing_validators` — validators not voting in current round
- `cometbft_consensus_byzantine_validators` — validators with equivocation evidence

**Justification:** Network-level validator health directly affects service quality. If >1/3 of stake weight is offline, the chain halts (liveness failure). Byzantine validators represent a security event. These are the primary consensus health signals in BFT consensus theory. *(Buchman et al., 2018 — Tendermint: Byzantine Fault Tolerance in the Age of Blockchains)*

---

## 3. Validator Performance

### `voting_power` / `voting_power_pct` · `VALIDATOR`

Cumulo's stake weight in the current validator set. From `cometbft_consensus_validator_power` filtered by Cumulo's validator address, expressed both in absolute units and as a percentage of `cometbft_consensus_validators_power` (total network stake).

**Justification:** Voting power determines Cumulo's influence in consensus and is the primary metric delegators use to evaluate concentration risk. Standard disclosure in all institutional validator performance reports.

---

### `last_signed_height` · `VALIDATOR`

The most recent block height at which Cumulo's validator successfully signed. From `cometbft_consensus_validator_last_signed_height`.

Compared against `head_height` to compute lag. Normal lag is 5–20 blocks (one Prometheus scrape interval at 15s). Lag above 50 blocks indicates potential signing failure.

**Justification:** This is the most direct and honest indicator of active consensus participation. Unlike "uptime" percentage, which only measures whether a process is running, `last_signed_height` verifies that the validator is actually signing blocks — which is what delegators depend on for reward accrual. A node can be "online" but not signing; this metric detects that scenario.

---

### `node_availability_7d_pct` / `rpc_availability_7d_pct` · `PROM`

7-day probe-based uptime percentage. Computed from:

```promql
avg_over_time(up{job="storyAened"}[7d]) * 100
avg_over_time(up{job="storyAenedRPC"}[7d]) * 100
```

Where `up` is Prometheus' standard scrape success indicator (1 = reachable, 0 = unreachable), evaluated every 15 seconds.

**Justification:** The 7-day availability window is the standard SLA reporting period in infrastructure services. It smooths transient outages while capturing meaningful reliability signals. Directly comparable to cloud provider SLAs (AWS, GCP, Azure all publish 99.9% monthly availability targets). *(Google SRE Workbook, ch. 2 — Implementing SLOs)*

---

### `validator_address_prom` · `VALIDATOR`

Cumulo's on-chain validator identity (CometBFT consensus address), as identified by Prometheus metric labels.

**Justification:** Publishing the validator address is a basic transparency requirement. It allows any third party to independently verify Cumulo's signing activity using any block explorer, without relying on Cumulo's own reporting. This is the foundation of the OTL's verifiability claim.

---

## 4. Consensus Performance

### `block_processing_ms` (p50 / p95) · `PROM`

Time to process each block. From `cometbft_state_block_processing_time` histogram.

> **Unit note:** In Story's CometBFT build, this metric is in **milliseconds** (not seconds). Normal range for Story Aeneid: p50 ~15–90ms, p95 ~90ms.

**Justification:** Block processing time is the primary latency indicator for consensus participation. If processing exceeds the prevote timeout window, the validator misses the voting round. *(CometBFT docs: Metrics — consensus)*

---

### `round_duration_ms_p95` · `PROM`

p95 consensus round duration. From `cometbft_consensus_round_duration_seconds` × 1000.

Measures total time from round start to completion. Normal for Story Aeneid: ~95–200ms.

**Justification:** Round duration directly determines block time. Sustained elevation indicates consensus difficulty — missing validators, network latency, or Byzantine behaviour — before these conditions manifest as missed blocks.

---

### `quorum_prevote_delay_ms` / `quorum_precommit_delay_ms` · `PROM`

Time for 2/3+ of voting power to submit prevotes and precommits respectively. From `cometbft_consensus_quorum_prevote_delay` and `cometbft_consensus_quorum_precommit_delay` (gauge, ms).

**Justification:** Quorum delays are early warning signals for network partition or validator degradation. They directly precede round timeouts, which cascade into missed blocks and liveness failures in BFT consensus.

---

### `peers_avg` · `PROM`

Average number of active P2P connections. From `cometbft_p2p_peers`.

Normal range for Story Aeneid: 7–12 peers.

**Justification:** Peer count is a connectivity health indicator. Fewer than 3 peers significantly increases partition risk. This is a standard network health metric in P2P system monitoring.

---

### `p2p_recv_bytes_per_sec` / `p2p_send_bytes_per_sec` · `PROM`

P2P network bandwidth, 5-minute rate. From `cometbft_p2p_peer_receive_bytes_total` and `cometbft_p2p_peer_send_bytes_total`.

**Justification:** P2P traffic is a liveness proxy. A node with near-zero send/receive has effectively disconnected from the network. Unusual spikes may indicate transaction spam or gossip amplification attacks.

---

## 5. Vote Anomaly Counters (Informational)

### `duplicate_vote_5m` / `late_votes_5m` · `PROM`

Chain-wide vote anomaly counters. From `cometbft_consensus_duplicate_vote` and `cometbft_consensus_late_votes`.

> **Important:** These are **monotonically increasing counters** that accumulate from genesis. In long-running testnets like Story Aeneid (devnet-1), high absolute values are expected and do not indicate validator misbehaviour.

These metrics are published for transparency but **do not affect the SLO score**. They are displayed with an explicit context note on the dashboard.

**Justification for inclusion:** Duplicate votes are a Byzantine fault signal in BFT consensus. Late votes contribute to round timeouts. While chain-wide testnet noise makes absolute values unreliable, the metrics are published as part of the OTL's commitment to full data disclosure.

---

## 6. Snapshots

The OTL monitors Cumulo's daily snapshot service for Story Aeneid:

- **Snapshot lag:** head_height − snapshot_block_height. Acceptable: <5,000 blocks (~2.5h at 2s/block).
- **Availability:** The OTL fetches the snapshot index on each request and verifies both consensus and execution client snapshots are present.

**Justification:** Snapshot availability is a community service metric. New validators depend on snapshots to bootstrap in hours rather than days. Publishing snapshot health is part of Cumulo's commitment to ecosystem infrastructure.

---

## References

1. Beyer, B. et al. — *Site Reliability Engineering: How Google Runs Production Systems*. O'Reilly, 2016. [https://sre.google/sre-book/](https://sre.google/sre-book/)
2. Beyer, B. et al. — *The Site Reliability Workbook*. O'Reilly, 2018. [https://sre.google/workbook/](https://sre.google/workbook/)
3. Buchman, E., Kwon, J., Milosevic, Z. — *Tendermint: Byzantine Fault Tolerance in the Age of Blockchains*. 2018. [https://arxiv.org/abs/1807.04938](https://arxiv.org/abs/1807.04938)
4. CometBFT — *Metrics Reference*. [https://docs.cometbft.com/v1.0/references/metrics](https://docs.cometbft.com/v1.0/references/metrics)
5. Prometheus Authors — *Data Model & PromQL*. [https://prometheus.io/docs/concepts/data_model/](https://prometheus.io/docs/concepts/data_model/)
6. Story Protocol — *Node Documentation*. [https://docs.story.foundation/](https://docs.story.foundation/)
