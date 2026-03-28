# Cumulo OTL — Operational Transparency Layer

**Stake with us. [cumulo.pro](https://cumulo.pro)**

---

The **Operational Transparency Layer (OTL)** is Cumulo's institutional-grade public accountability system for its validator and node infrastructure across supported blockchains.

> *Transparency is not a feature. It is the foundation of institutional trust.*

## What is the OTL?

The OTL is a structured, versioned, publicly accessible API and dashboard that exposes the real operational posture of Cumulo's infrastructure — in real time, on every supported chain.

It answers a question that Grafana dashboards, uptime monitors, and block explorers cannot answer on their own:

**"Is Cumulo a reliable infrastructure provider right now — and how do we prove it?"**

### OTL vs Internal Monitoring

| System | Audience | Purpose | Public |
|---|---|---|---|
| Grafana dashboards | Cumulo operators | Real-time anomaly detection | No |
| Alertmanager | On-call team | Incident notification | No |
| Activity Tracker | Community | Audit trail of upgrades & governance | Yes |
| **OTL Dashboard** | **Delegators, partners, evaluators** | **Institutional accountability** | **Yes** |

The OTL is not a replacement for internal monitoring. It is a different product for a different audience.

---

## Live Deployments

| Chain | Network | Status | Dashboard | API |
|---|---|---|---|---|
| Story | Aeneid (testnet) | 🟢 Live | [cumulo.pro/services/story_aeneid](https://cumulo.pro/services/story_aeneid) · [Documentation](https://cumulo.pro/services/story_aeneid/docs.php) | [otl-api.cumulo.com.es/otl/v2/story/aeneid/posture](https://otl-api.cumulo.com.es/otl/v2/story/aeneid/posture) |

---

## What the OTL Publishes

For each supported chain, the OTL exposes:

### Service Posture
A single-signal summary of infrastructure status: `ready` · `syncing` · `stale` · `maintenance` · `down`

### SLO Score & Grade
A composite 0–100 score derived from consensus health signals, expressed as a letter grade (A–D). See [docs/slo-scoring.md](docs/slo-scoring.md) for the full formula.

### Chain Context
- Head height and block freshness (from CometBFT RPC)
- Average block time and TPS
- Active validators, missing validators, byzantine validators

### Validator Performance
- Cumulo's voting power and share of total stake
- Last signed block height (active consensus participation)
- 7-day probe-based availability

### Consensus Timings
- Block processing time (p50 / p95)
- Round duration p95
- Quorum prevote and precommit delays

### Network Health
- P2P bandwidth (recv/send)
- Peer count
- Blocksync status

### Snapshots
- Daily snapshot availability and lag vs head height
- Direct download links (consensus + execution client)

### Activity & Incidents
- Summary of upgrades, governance votes, and incidents from the public [incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json)

---

## Data Sources

All OTL data derives from two open, verifiable, standard sources:

| Source | Label | Description |
|---|---|---|
| CometBFT RPC `/status` | `RPC` | Standard endpoint on every CometBFT node. Head height, freshness, catching_up, validator address. |
| Prometheus (CometBFT exporter) | `PROM` | Standard metrics exported by the node binary. Consensus timings, peer counts, validator counters. |

No proprietary data sources. No black boxes. Every metric is independently verifiable.

---

## API

The OTL exposes a public, stable, versioned JSON API:

```
GET https://otl-api.cumulo.com.es/otl/v2/{chain}/{network}/posture
GET https://otl-api.cumulo.com.es/otl/v2/{chain}/{network}/slo
GET https://otl-api.cumulo.com.es/otl/chains
GET https://otl-api.cumulo.com.es/health
```

See [docs/api-schema.md](docs/api-schema.md) for the full response schema.

**Example:**
```bash
curl https://otl-api.cumulo.com.es/otl/v2/story/aeneid/posture | jq '.posture_status, .chain_context.head_height'
```

---

## Documentation

- [docs/methodology.md](docs/methodology.md) — What each metric measures and why it is included
- [docs/slo-scoring.md](docs/slo-scoring.md) — The SLO scoring formula, thresholds, and grade assignment
- [docs/api-schema.md](docs/api-schema.md) — Full API response schema
- [chains/story-aeneid.md](chains/story-aeneid.md) — Story Aeneid chain profile

---

## Architecture

```
CometBFT RPC ──────────────────────────┐
                                       ▼
Prometheus ────────────────────► OTL API (Node.js)
(CometBFT exporter)                    │
                                       ▼ JSON (60s TTL)
incidents.json (GitHub) ──────► OTL Dashboard (PHP)
                                       │
Snapshot index (HTTP) ─────────────────┘
```

**Stack:** Node.js · Express · Prometheus · CometBFT · PHP · Nginx · Tailwind CSS

**Multi-chain design:** Adding a new chain requires only one entry in `chains.json`. Zero code changes.

---

## Roadmap

- [x] v1 — Story Aeneid (testnet) live
- [ ] v2 — Rolling 30-day availability with persistent storage
- [ ] v3 — Incident counter integrated into SLO score
- [ ] Multi-chain — Celestia, XRPL EVM, Dymension
- [ ] PDF report generation (monthly SLO report)

---

## References

- Beyer et al. — *Site Reliability Engineering* (Google, 2016) — SLO/error budget model
- Buchman, Kwon, Milosevic — *Tendermint: Byzantine Fault Tolerance in the Age of Blockchains* (2018)
- [CometBFT Metrics Documentation](https://docs.cometbft.com/v1.0/references/metrics)
- [Prometheus Data Model](https://prometheus.io/docs/concepts/data_model/)
- [Story Protocol Documentation](https://docs.story.foundation/)

---

## About Cumulo

Cumulo is an institutional-grade validator and node infrastructure provider operating across multiple proof-of-stake blockchains. We believe transparency is a prerequisite for trust, not an optional feature.

**[cumulo.pro](https://cumulo.pro) · [@Cumulo_pro](https://twitter.com/Cumulo_pro)**

---

*OTL v2 · Schema version: v2 · Last updated: March 2026*
