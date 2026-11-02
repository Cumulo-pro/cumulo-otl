# Cumulo OTL: Operational Transparency Layer

**Stake with us. [cumulo.pro](https://cumulo.pro)**

---

The **Operational Transparency Layer (OTL)** is Cumulo's institutional-grade public accountability system for its validator and node infrastructure across supported blockchains.

> *Transparency is not a feature. It is the foundation of institutional trust.*

## What is the OTL?

The OTL is a structured, versioned, publicly accessible API and dashboard that exposes the real operational posture of Cumulo's infrastructure, in real time, on every supported chain.

It answers a question that Grafana dashboards, uptime monitors, and block explorers cannot answer on their own:

**"Is Cumulo a reliable infrastructure provider right now, and how do we prove it?"**

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
| XRPL EVM | Mainnet | 🟢 Live | [cumulo.pro/services/xrplevm_mainnet/otl](https://cumulo.pro/services/xrplevm_mainnet/otl) · [Documentation](https://cumulo.pro/services/xrplevm_mainnet/otl-docs) | [otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture](https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture) |
| XRPL EVM | Testnet | 🟢 Live | [cumulo.pro/services/xrplevm/otl](https://cumulo.pro/services/xrplevm/otl) · [Documentation](https://cumulo.pro/services/xrplevm/otl-docs) | [otl-api.cumulo.com.es/otl/v2/xrplevm/testnet/posture](https://otl-api.cumulo.com.es/otl/v2/xrplevm/testnet/posture) |
| Celestia | Mainnet | 🟢 Live | [cumulo.pro/services/celestia/otl](https://cumulo.pro/services/celestia/otl) · [Documentation](https://cumulo.pro/services/celestia/otl-docs) · [Audit report (PDF)](https://cumulo.pro/services/celestia/otl-report) | [otl-api.cumulo.com.es/otl/v2/celestia/mainnet/posture](https://otl-api.cumulo.com.es/otl/v2/celestia/mainnet/posture) |
| Celestia | Mocha (testnet) | 🟢 Live | [cumulo.pro/services/celestia_mocha/otl](https://cumulo.pro/services/celestia_mocha/otl) · [Documentation](https://cumulo.pro/services/celestia_mocha/otl-docs) · [Audit report (PDF)](https://cumulo.pro/services/celestia_mocha/otl-report) | [otl-api.cumulo.com.es/otl/v2/celestia/mocha/posture](https://otl-api.cumulo.com.es/otl/v2/celestia/mocha/posture) |
| Gno.land | Pearl Testnet | 🟢 Live | [cumulo.pro/services/gnoland_testnet/otl](https://cumulo.pro/services/gnoland_testnet/otl) · [Documentation](https://cumulo.pro/services/gnoland_testnet/otl-docs) | [otl-api.cumulo.com.es/otl/v2/gnoland/testnet/posture](https://otl-api.cumulo.com.es/otl/v2/gnoland/testnet/posture) |

Celestia Mainnet is the first standard permissionless Proof-of-Stake chain on the OTL, as opposed to XRPL EVM's Proof-of-Governance model. It runs on the same `cosmos-cometbft` profile and scoring model, with two chain-specific duties added to the grade: block proposals and Fibre (see below). Celestia Mocha, the public testnet, runs the same model on its own `mocha-5` data; its token has no economic value, so there the grade measures operational discipline, not financial exposure. Gno.land is not a Cosmos SDK chain (no staking, slashing or `x/gov` module), so its dashboard adapts the layout and marks the signals that do not apply as unavailable. Further CometBFT chains are onboarded by adding one entry to `chains.json`; see [Roadmap](#roadmap).

---

## What the OTL Publishes

For each supported chain, the OTL exposes:

### Service Posture
A single-signal summary of infrastructure status: `ready` · `syncing` · `stale` · `bootstrapping` · `maintenance` · `down`

### SLO Score & Grade
A composite 0-100 score, expressed as a letter grade (A to D), built entirely from signals specific to **Cumulo's own infrastructure**: on-chain signing record against the chain's own jailing threshold, public RPC availability against a published target, and the track-record signals below. On chains where they exist, delivering the blocks the validator is scheduled to propose and keeping its Fibre server ready are scored too (see the chain's profile). It deliberately does **not** penalise chain-wide conditions outside Cumulo's control (byzantine/missing validators, consensus round duration) or availability computed over an incomplete history. See [docs/slo-scoring.md](docs/slo-scoring.md) for the full formula, thresholds, and risk-flag catalogue.

### Track Record & Governance
Computed from the public [incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json) activity log and, for proposals still open, the live chain governance module, independent of live node metrics:
- Validator since / governance proposal of approval
- Slashing / jailing / double-signing events (target: zero; caps the grade permanently if any is on record)
- Mandatory upgrades applied on schedule, and the testnet-first upgrade policy (a late upgrade caps the grade)
- Governance vote participation against a per-chain target, and public disclosure of each vote (disclosure is informational only, never scored)

### Chain Context
- Head height and block freshness (from CometBFT RPC `/status`)
- Average block time and TPS
- Active validators, missing validators, byzantine validators

### Validator Performance
- Cumulo's voting power and share of the active set
- Last signed block height (active consensus participation)
- Probe-based availability over a rolling window (up to 7 days; uses whatever Prometheus history exists if shorter, and the payload declares this in `data_window`)
- Consensus address (`SHA256(consensus_pubkey)` truncated to 20 bytes, the value block explorers show)

### Block Proposals (Celestia)
On a chain where a validator's reliability is measured above all by delivering the blocks it is scheduled to propose, signing uptime alone does not show it. The OTL publishes proposals made and missed, the miss rate, the validator's share of all proposals against its voting share, and how many active validators have zero misses. Counts come from Cumulo's own [missed proposals monitor](https://cumulo.pro/services/celestia/missed-proposals), inferred from public consensus data, with the complete history published as JSON. The miss rate is part of the grade once a minimum sample exists. See [docs/methodology.md](docs/methodology.md) § 10.

### Fibre (Celestia)
Every Celestia validator operates a Fibre server, and Fibre payments only settle when validators holding two thirds of the stake have a ready one. The OTL shows whether Cumulo's host is ready (as seen by the majority of four probe regions), Fibre uptime over 24 hours, 7 and 30 days, data custody (the share of the blobs Cumulo signed that its host actually served, checked against the blob commitment), endorsements, latency by region, identity certificate expiry and the network's readiness against the two-thirds line. Measured by Cumulo's own [Fibre Network Monitor](https://cumulo.pro/services/celestia/fibre), whose raw data is public. Only the host's country is shown, never an address. The worst of host readiness, uptime and custody is part of the grade, applied once. See [docs/methodology.md](docs/methodology.md) § 11.

### On-Chain Validator Record
Sourced entirely from chain state (staking + slashing modules), independent of the node's own telemetry and re-derivable by anyone from any LCD. These are the same standard fields on every Cosmos SDK / CometBFT chain; what they represent depends on that chain's economic model (see each chain's profile):
- **Signing uptime**: `missed_blocks_counter` over `signed_blocks_window`; the canonical, third-party-verifiable signing record the chain itself uses for jailing
- **Bonded status / jailed flag**
- **Voting weight** (tokens) and share of the set: delegated stake on a standard Proof-of-Stake chain, governance-assigned on a Proof-of-Governance chain
- **Commission** (rate / max rate / max change rate); on a Proof-of-Governance chain with no delegation market this is commonly structurally `0%` and is stated explicitly rather than omitted

### Consensus Timings
All histogram quantiles are aggregated across scraped instances with `sum(rate(bucket[5m])) by (le)`:
- Block processing time (p50 / p95)
- Round duration p95
- Step duration p95 (Propose / Prevote / Precommit / Commit phases, excluding protocol wait steps)
- Quorum prevote and precommit delays

### Network Health
- P2P bandwidth (recv / send)
- Peer count
- Blocksync status
- Per-node vote-anomaly counters (`late_votes` / `duplicate_vote` / `duplicate_block_part`): informational, network-position dependent, excluded from the SLO score

### Snapshots
- Snapshot availability and lag vs head height, monitored on every dashboard load
- The archive layout is per-chain: a single combined archive covering full node state (e.g. XRPL EVM Mainnet), or separate consensus / execution-client files where the chain publishes them
- **State-sync** parameters where the chain provides them (`nodeid@ip:port` seed / RPC servers, trust height / hash)
- Cross-checked independently by `snapshot_checker`. See [Independent Collectors](#independent-collectors).

### Independent Verification (check_d)
A second, external measurement of the same public endpoints, not derived from the node's own reporting. See [Independent Collectors](#independent-collectors).

### Activity & Incidents
Summary of upgrades, governance votes, and incidents from the public [incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json).

---

## Data Sources

OTL data derives from open, verifiable sources with no proprietary intermediary. Two are self-reported by the node itself; the rest are separate systems that observe the same infrastructure from the outside or read it straight from chain state.

| Source | Label | Kind | Description |
|---|---|---|---|
| CometBFT RPC `/status` | `RPC` | self-reported | Standard endpoint on every CometBFT node. Head height, freshness, `catching_up`, validator address. |
| Prometheus (CometBFT exporter) | `PROM` | self-reported | Standard metrics exported by the node binary. Consensus timings, peer counts, validator counters, network health, probe-based availability. |
| check_d (aggregate-rpcs collector) | `CHECK_D` | independent | External endpoint scan: reliability, per-region latency, TLS validity, CORS, pruning, node version, block height. |
| snapshot_checker (aggregate-snapshots collector) | `SNAPCHECK` | independent | External snapshot check: size, last-modified time, measured download speed, and state-sync availability. |
| Missed proposals monitor | `PROPOSALS` | independent | Cumulo's own monitor of block proposals made and missed, inferred from public consensus data (Celestia). |
| Fibre Network Monitor | `FIBRE` | independent | Cumulo's own monitor of every Fibre server: four probes (US, EU, CA, AS) check each host and verify the data it holds against the blob commitment (Celestia). |
| Chain state (staking + slashing modules) | `CHAIN` | on-chain | Signing uptime, bonded / jailed status, voting weight (delegated stake or governance-assigned, per chain), commission. Consensus state, re-derivable from any LCD via `cosmos/staking/v1beta1/validators/<valoper>` and `cosmos/slashing/v1beta1/signing_infos`. |

No proprietary data sources. No black boxes. Every metric is independently verifiable.

### Independent Collectors

Standalone Cumulo collectors run separately from the OTL API and cross-check what the node reports about itself. Two cover the public endpoints and snapshots on every chain; the other two (missed proposals and Fibre) are specific to Celestia and feed the grade through the OTL API:

**check_d: endpoint scan**
- Live output: <https://aggregate-rpcs.cumulo.com.es/aggregate-rpcs>
- Documentation: <https://github.com/Cumulo-pro/Cumulo-Front-Chain/tree/main/check_End-Points>
- Probes the public RPC endpoint from multiple regions and reports reliability history, latency (raw + smoothed, per region), TLS certificate validity and days to expiry, CORS, sync status, pruning mode and node version. The OTL filters this output to Cumulo's entry and renders it as its "Independent Verification" section; it also currently fills the gap left by the OTL's optional `LATENCY` capability.

**snapshot_checker: snapshot & state-sync check**
- Live output: <https://aggregate-snapshots.cumulo.com.es/aggregate-snapshots>
- Chains under test: [`snapshot_checker/chains_snapshot.json`](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/snapshot_checker/chains_snapshot.json)
- Independently downloads and inspects each published snapshot (size, freshness, throughput) and the state-sync endpoints, rather than trusting the OTL page's own fetch of the snapshot index. The OTL renders this as a cross-check in its Snapshot section; if a chain has no entry in `chains_snapshot.json` the section shows "not yet cross-checked".

---

## API

The OTL exposes a public, stable, versioned JSON API:

```
GET https://otl-api.cumulo.com.es/otl/v2/{chain}/{network}/posture
GET https://otl-api.cumulo.com.es/otl/v2/{chain}/{network}/slo
GET https://otl-api.cumulo.com.es/otl/v2/{chain}/{network}/diagnostics
GET https://otl-api.cumulo.com.es/otl/chains
GET https://otl-api.cumulo.com.es/health
```

See [docs/api-schema.md](docs/api-schema.md) for the full response schema.

**Example:**
```bash
curl https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture \
  | jq '{status: .posture_status, height: .chain_context.head_height, grade: .slo}'

curl https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/slo | jq '.slo'
```

---

## Documentation

- [docs/methodology.md](docs/methodology.md): what each metric measures and why it is included, chain-agnostic
- [docs/slo-scoring.md](docs/slo-scoring.md): the SLO scoring formula, thresholds, and grade assignment
- [docs/api-schema.md](docs/api-schema.md): full API response schema
- [chains/xrplevm-mainnet.md](chains/xrplevm-mainnet.md): XRPL EVM Mainnet chain profile
- [chains/xrplevm-testnet.md](chains/xrplevm-testnet.md): XRPL EVM Testnet chain profile
- [chains/celestia-mainnet.md](chains/celestia-mainnet.md): Celestia Mainnet chain profile
- [chains/celestia-mocha.md](chains/celestia-mocha.md): Celestia Mocha (testnet) chain profile
- [CHANGELOG.md](CHANGELOG.md): what changed and when
- Live per-chain methodology: [cumulo.pro/services/xrplevm_mainnet/otl-docs](https://cumulo.pro/services/xrplevm_mainnet/otl-docs) (Mainnet) · [cumulo.pro/services/xrplevm/otl-docs](https://cumulo.pro/services/xrplevm/otl-docs) (Testnet) · [cumulo.pro/services/celestia/otl-docs](https://cumulo.pro/services/celestia/otl-docs) (Celestia Mainnet) · [cumulo.pro/services/celestia_mocha/otl-docs](https://cumulo.pro/services/celestia_mocha/otl-docs) (Celestia Mocha) · [cumulo.pro/services/gnoland_testnet/otl-docs](https://cumulo.pro/services/gnoland_testnet/otl-docs) (Gno.land Pearl Testnet)

---

## Architecture

```
SELF-REPORTED (into OTL API)
  CometBFT RPC /status            ->  head height, freshness, catching_up, validator address
  Prometheus (CometBFT exporter)  ->  consensus timings, peers, validator + network counters, availability

ON-CHAIN (into OTL API)
  staking + slashing modules (via LCD)  ->  signing uptime, bonded/jailed, voting weight, commission
  governance module (via LCD)           ->  live vote check, open proposals only
  incidents.json (GitHub)               ->  slashing/upgrade history, governance participation for closed
                                             proposals; feeds the score, not only the dashboard display

        |
        v
  OTL API (Node.js + Express)  ->  posture / slo / diagnostics as JSON, 60s TTL cache
        |
        v
  OTL Dashboard (PHP)  ->  cached fetch + stale-serve, human-readable rendering
        +  incidents.json (GitHub) consumed again, directly, for the Track Record & Governance display

INDEPENDENT (consumed directly by the Dashboard, not by the OTL API)
  check_d (aggregate-rpcs)               ->  external endpoint scan
  snapshot_checker (aggregate-snapshots) ->  external snapshot + state-sync check

INDEPENDENT, ALSO FEEDING THE GRADE (read by the OTL API, Celestia)
  missed proposals monitor               ->  proposals made / missed, miss rate
  Fibre Network Monitor                  ->  host readiness, Fibre uptime, data custody
  Fibre server Prometheus job            ->  server up, version, storage (context only)
```

**Stack:** Node.js · Express · Prometheus · CometBFT · PHP · Nginx · Tailwind CSS

**Multi-chain design:** Adding a new chain requires only one entry in `chains.json`. The dashboard page for each chain is a single PHP file with a handful of configurable variables, using the same shared `cosmos-cometbft` profile and rendering code, differing only in API endpoints, chain filter and version-extraction pattern.

---

## Roadmap

- [x] First production deployment live
- [x] XRPL EVM Mainnet live
- [x] XRPL EVM Testnet live
- [x] Independent collectors integrated (check_d, snapshot_checker)
- [x] On-chain validator record (signing uptime, bonded status, voting weight, commission) from staking + slashing
- [x] Rolling 30d / 90d / lifetime availability with persistent daily rollup
- [x] Incident history (slashing, delayed upgrades) integrated into the SLO score
- [x] Governance vote participation and public-disclosure tracking integrated into the SLO score
- [x] Celestia Mainnet live: the first standard Proof-of-Stake chain, validating the chain-agnostic claim end to end
- [x] Block proposals scored (Celestia)
- [x] Fibre duty scored (Celestia)
- [x] Institutional PDF audit report (Celestia Mainnet and Mocha, generated from live data on every request)
- [x] Celestia Mocha (testnet) live, with the same scoring model, block proposals and Fibre as Mainnet
- [x] Gno.land Pearl Testnet dashboard (non-Cosmos-SDK chain)
- [ ] Additional chains: Dymension and others
- [ ] Review the weight of Fibre in the grade once mainnet Fibre carries more traffic
- [ ] Availability that also requires the node to be at the chain head, not only answering its metrics endpoint
- [ ] Scheduled monthly PDF report

---

## References

- Beyer et al., *Site Reliability Engineering* (Google, 2016): SLO / error-budget model
- Beyer et al., *The Site Reliability Workbook* (O'Reilly, 2018): implementing SLOs
- Buchman, Kwon, Milosevic, *Tendermint: Byzantine Fault Tolerance in the Age of Blockchains* (2018)
- [CometBFT Metrics Documentation](https://docs.cometbft.com/v1.0/references/metrics)
- [Prometheus Data Model](https://prometheus.io/docs/concepts/data_model/)
- [XRPL EVM Node & Validator Documentation](https://docs.xrplevm.org/)

---

## About Cumulo

Cumulo is an institutional-grade validator and node infrastructure provider operating across multiple Cosmos SDK / CometBFT blockchains, on both proof-of-stake and proof-of-governance networks. We believe transparency is a prerequisite for trust, not an optional feature.

**[cumulo.pro](https://cumulo.pro) · [@Cumulo_pro](https://twitter.com/Cumulo_pro)**

---

*OTL v2 · Schema version: v2 · Last updated: November 2026*
