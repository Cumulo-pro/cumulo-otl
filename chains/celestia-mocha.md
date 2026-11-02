# Chain Profile: Celestia Mocha (testnet)

**Chain:** Celestia
**Network:** Mocha testnet (`mocha-5`)
**OTL Status:** Live (November 2026), same scoring model as Celestia Mainnet
**Dashboard:** [cumulo.pro/services/celestia_mocha/otl](https://cumulo.pro/services/celestia_mocha/otl) · [Documentation](https://cumulo.pro/services/celestia_mocha/otl-docs) · [Audit report (PDF)](https://cumulo.pro/services/celestia_mocha/otl-report)
**API:** [otl-api.cumulo.com.es/otl/v2/celestia/mocha/posture](https://otl-api.cumulo.com.es/otl/v2/celestia/mocha/posture)

---

## About Celestia Mocha

Mocha is Celestia's public testnet. Like Mainnet it is a permissionless Proof-of-Stake chain on CometBFT and the Cosmos SDK: the active set is the top validators by bonded stake, capped by the staking module's `MaxValidators` parameter, with no governance approval to become a validator. Its token has no economic value, so what the OTL measures here is operational discipline, not financial exposure.

Cumulo's Mocha validator predates the Celestia Mainnet genesis (2023-10-31). Mocha networks are replaced over time (`mocha-5` followed `mocha-4`), so there is no single genesis file in which the validator creation can be cited, and the OTL does not claim a start date it cannot verify. The dashboard states this as "Pre-mainnet".

Mocha is also where Cumulo's upgrade policy is exercised: upgrades are tried here before they reach Mainnet.

- **Docs:** [docs.celestia.org](https://docs.celestia.org/)
- **GitHub:** [github.com/celestiaorg](https://github.com/celestiaorg)

---

## Cumulo Infrastructure

| Component | Role |
|---|---|
| Validator node | Signs blocks and proposes them. |
| Fibre server | Runs on the same host as the validator and signs through the validator's local privval link. |
| Public RPC node | Serves the public endpoints below, on its own host. |
| Bridge node | Relays chain data from consensus to the data availability network. Reported on the dashboard, not part of the grade. |

The validator signs with a single consensus key held in a local key file on the validator host. Cumulo does not run threshold signing (Horcrux) on this chain. The Fibre server runs on the same host and asks the validator to co-sign payment promises through the validator's privval gRPC endpoint, which is bound to the loopback interface of that host and is not reachable from the network. Server addresses are not published. The details are in the [documentation](https://cumulo.pro/services/celestia_mocha/otl-docs) and in the activity log.

There is no IBC relayer section: IBC relaying on Mocha is not part of this OTL.

### Validator

| Parameter | Value |
|---|---|
| Chain software | `celestia-app` (current version shown live on the dashboard) |
| Validator moniker | Cumulo |
| Operator address | `celestiavaloper16yv2p64nzps0lqcealnumrsqc2rcszkd0uzlp8` |
| Validator consensus address | `AC2A961260B80FC88DDB9E52F1A957E2F5D60D64` |

### Public Endpoints

| Endpoint | Value |
|---|---|
| RPC | `mocha5.celestia.rpc.cumulo.me` |
| RPC WebSocket (CometBFT) | `wss://mocha5.celestia.rpc.cumulo.me/websocket` |
| API | `mocha5.celestia.api.cumulo.me` |
| gRPC | `mocha5.celestia.grpc.cumulo.me` |

---

## Chain Parameters

| Parameter | Value |
|---|---|
| Chain ID | `mocha-5` |
| Consensus engine | CometBFT |
| Staking | Permissionless Proof of Stake |
| Signed blocks window | 10,000 |
| Minimum signed per window | 0.001 (jail margin 9,990 blocks) |
| Downtime slash fraction | 0 |
| Downtime jail duration | 60 seconds |
| Profile | `cosmos-cometbft` |

Mocha's punishment parameters are lenient. The score does not relax for that: it still treats the jail margin as the limit and reads the same signals, so a testnet grade means the same operational discipline as a Mainnet one.

---

## What is Scored on This Chain

The same blocks as Celestia Mainnet: the on-chain signing record against the jailing threshold, public RPC availability against a 99% target, governance participation, slashing and upgrade record, plus the two Celestia duties.

| Block | Source | Summary |
|---|---|---|
| 1b. Block proposals | Cumulo's missed-proposals monitor | Miss rate above 0.1%: 15 points, cap B. Above 0.5%: 30 points, cap C. Scored from 2,000 turns. |
| 1c. Fibre | Cumulo's Fibre Network Monitor (`mocha-5`) | The worst of host ready now, 7-day uptime against 99% and 30-day data custody against 99%, applied once (maximum 30 points). Endorsements and network readiness are context only. |

Full formulas and the flag catalogue: [slo-scoring.md](../docs/slo-scoring.md). Metric definitions: [methodology.md](../docs/methodology.md) §§ 10 and 11.

**Governance.** `mocha-5` has had no governance proposals so far, so there are no eligible votes. Governance participation is shown as "no proposals yet" and is not scored: no penalty and no bonus. Once a proposal opens, the vote is verified live while voting is open and counted from the activity log afterwards.

The `own_node_stalled` flag describes the **public RPC node** (`catching_up`, or a stale head), not the validator. Its id is kept for API stability.

---

## Prometheus Configuration

| Selector | Meaning |
|---|---|
| `job="celestia-validator-mocha"` | Validator node: consensus, signing, P2P, block sync state |
| `job="celestia-rpc-mocha"` | Public RPC node: block context, availability |
| Fibre server job | Fibre server: up, version, storage (context only). Named after its host, so the name is not published. |

**Chain selector (PromQL):** `chain_id="mocha-5"`.

Prometheus has scraped these jobs since 2 November 2026, so the daily availability rollup starts on that date. The 30-day, 90-day and lifetime figures read "Not enough history yet" until the rollup has recorded enough full days, and a window that is not yet full is marked as partial.

---

## Known Metric Behaviours

### Availability measures reachability
Node and RPC availability are the share of Prometheus scrapes that succeeded. A node that is running but behind the chain head still counts as up; its sync state is shown separately as chain freshness and, for the grade, by the `own_node_stalled` flag.

### Small Fibre samples
Fibre on Mocha carries little traffic so far, so data custody and endorsements rest on small samples. Every figure is published with the number of checks behind it, and the weight of Fibre in the grade will be reviewed as traffic grows.

### Missed proposals are inferred
A missed proposal is inferred from public consensus data (the scheduled proposer for a height compared with the block's actual proposer). It is not an on-chain record.

### Vote anomaly counters
`late_votes_5m` and `duplicate_vote_5m` are published as informational context only, never scored. With about 90 active validators they read in the hundreds of thousands, which is expected.

### Network names in the activity log
The activity log spells this network several ways (`testnet (mocha)`, `testnet (mocha-5)` and `mocha-5`). The OTL API compares the first word of the network label and matches all of them through the optional `incident_networks` list in the chain's `chains.json` entry (`["testnet", "mocha-5"]`). Without it, the chain's upgrades and slashing events would not be found.

---

## Snapshots

| Snapshot | URL |
|---|---|
| Archive (pruned) | [cumulo.pro/services/celestia_mocha/snapshot](https://cumulo.pro/services/celestia_mocha/snapshot) |

Cross-checked from outside by `snapshot_checker` (network `mocha-5`).

---

## Independent Verification

| Monitor | Page |
|---|---|
| `check_d` (public RPC endpoint) | Rendered on the OTL dashboard as "Independent Verification" |
| Missed proposals | [cumulo.pro/services/celestia_mocha/missed-proposals](https://cumulo.pro/services/celestia_mocha/missed-proposals) · [methodology](https://github.com/Cumulo-pro/cumulo-block-explorer/blob/main/docs/celestia-missed-proposals.md) |
| Fibre Network Monitor | [cumulo.pro/services/celestia_mocha/fibre](https://cumulo.pro/services/celestia_mocha/fibre) · [methodology](https://cumulo.pro/services/celestia_mocha/fibre-methodology) · [API and archive](https://cumulo.pro/services/celestia_mocha/fibre-api) |
| Bridge nodes | [cumulo.pro/services/celestia_mocha/bridges](https://cumulo.pro/services/celestia_mocha/bridges) (network-wide dashboard, with Cumulo's own bridge node) |

Fibre server setup guide: [Fibre_Server_Setup.md](https://github.com/Cumulo-pro/cumulo-celestia-infra/blob/main/docs/fibre-setup/Fibre_Server_Setup.md).

---

## Activity History

All upgrades, governance votes, infrastructure migrations and incidents for Celestia Mocha are tracked in Cumulo's public activity log:

[github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json)

Filter by `"chain": "Celestia"` and a `"network"` that starts with `testnet` or `mocha-5` (for example `testnet (mocha)` or `testnet (mocha-5)`).

---

## OTL Posture API Example

```bash
# Current posture
curl https://otl-api.cumulo.com.es/otl/v2/celestia/mocha/posture | jq '{
  status: .posture_status,
  height: .chain_context.head_height,
  signing: .validator_performance.last_signed_height,
  fibre: .fibre.reason
}'

# SLO score, with the proposals and Fibre fields
curl https://otl-api.cumulo.com.es/otl/v2/celestia/mocha/slo | jq '.slo | {
  posture_score, posture_grade, risk_flags,
  proposal_miss_rate_pct, fibre_scored, fibre_uptime_7d_pct, fibre_custody_30d_pct
}'
```
