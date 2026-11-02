# Chain Profile: Celestia Mainnet

**Chain:** Celestia
**Network:** Mainnet (`celestia`)
**OTL Status:** Live (extended with block proposals and Fibre, November 2026)
**Dashboard:** [cumulo.pro/services/celestia/otl](https://cumulo.pro/services/celestia/otl) · [Documentation](https://cumulo.pro/services/celestia/otl-docs) · [Audit report (PDF)](https://cumulo.pro/services/celestia/otl-report)
**API:** [otl-api.cumulo.com.es/otl/v2/celestia/mainnet/posture](https://otl-api.cumulo.com.es/otl/v2/celestia/mainnet/posture)

---

## About Celestia

Celestia is a modular data availability network built on CometBFT and the Cosmos SDK. It is a permissionless Proof-of-Stake chain: the active set is the top validators by bonded stake, capped by the staking module's `MaxValidators` parameter, and there is no governance approval to become a validator (unlike XRPL EVM, Proposal #18). Cumulo has validated since the genesis of Celestia Mainnet (launched 2023-10-31).

- **Docs:** [docs.celestia.org](https://docs.celestia.org/)
- **GitHub:** [github.com/celestiaorg](https://github.com/celestiaorg)

---

## Cumulo Infrastructure

| Component | Role |
|---|---|
| Validator node | Signs blocks and proposes them. Not exposed to public traffic. |
| Public RPC node | Serves the public endpoints below. Holds no signing key. |
| Fibre server | Serves Fibre on its own host, separate from the validator, reading chain state from a full node that holds no consensus key. |
| Bridge node | Relays chain data from consensus to the data availability network. Reported on the dashboard, not part of the grade. |
| IBC relayer | Hermes relayer for the Celestia mainnet channels. Reported on the dashboard, not part of the grade. |

The validator signs with a single consensus key held in a local key file on the validator host, over a mutually authenticated gRPC channel firewalled to the Fibre host. Cumulo no longer runs threshold signing (Horcrux) on this chain: the signing path follows the requirements of Celestia's own infrastructure. Failover is manual and deliberate, confirming that the previous signer is stopped before the replacement starts, because two signers with the same key is the failure that must never happen. The details are in the [documentation](https://cumulo.pro/services/celestia/otl-docs) and in the activity log. Server addresses are not published.

### Validator

| Parameter | Value |
|---|---|
| Chain software | `celestia-app` (current version shown live on the dashboard) |
| Validator moniker | Cumulo |
| Operator address | `celestiavaloper1cs37tvmahavw8xcnzcgyz342sh0al37ma4zqat` |
| Validator consensus address | `5047F9F1E0CFBE4BCB45BBEA0D1AE70B8243412D` |

### Public Endpoints

| Endpoint | Value |
|---|---|
| RPC | `celestia.cumulo.org.es` |
| RPC WebSocket (CometBFT) | `wss://celestia.cumulo.org.es/websocket` |
| API | `celestia.api.cumulo.org.es` |
| gRPC | `celestia.grpc.cumulo.org.es` |

---

## Chain Parameters

| Parameter | Value |
|---|---|
| Chain ID | `celestia` |
| Consensus engine | CometBFT |
| Staking | Permissionless Proof of Stake |
| Profile | `cosmos-cometbft` |

---

## What is Scored on This Chain

On top of the signals every chain has (on-chain signing record against the jailing threshold, public RPC availability against a 99% target, governance participation, slashing and upgrade record), Celestia adds two duties. Both are opt-in blocks in `chains.json`.

| Block | Source | Summary |
|---|---|---|
| 1b. Block proposals | Cumulo's missed-proposals monitor | Miss rate above 0.1%: 15 points, cap B. Above 0.5%: 30 points, cap C. Scored from 2,000 turns. |
| 1c. Fibre | Cumulo's Fibre Network Monitor | The worst of host ready now, 7-day uptime against 99% and 30-day data custody against 99%, applied once (maximum 30 points). Endorsements and network readiness are context only. |

Full formulas and the flag catalogue: [slo-scoring.md](../docs/slo-scoring.md). Metric definitions: [methodology.md](../docs/methodology.md) §§ 10 and 11.

The `own_node_stalled` flag describes the **public RPC node** (`catching_up`, or a stale head), not the validator. Its id is kept for API stability.

---

## Prometheus Configuration

| Selector | Meaning |
|---|---|
| `job="celestia-validator-mainnet"` | Validator node: consensus, signing, P2P, block sync state |
| `job="celestia-rpc"` | Public RPC node: block context, availability |
| `job="fibre-mainnet"` | Fibre server: up, version, storage (context only) |

**Chain selector (PromQL):** `chain_id="celestia"`.

The daily availability rollup follows these selectors. Until 1 November 2026 the "validator node" series was read from one of Cumulo's sentry nodes; from 2 November 2026 it is read from the validator node itself, so the 30-day, 90-day and lifetime figures combine both sources.

---

## Known Metric Behaviours

### Availability measures reachability
Node and RPC availability are the share of Prometheus scrapes that succeeded. A node that is running but behind the chain head still counts as up; its sync state is shown separately as chain freshness and, for the grade, by the `own_node_stalled` flag.

### Small Fibre samples
Fibre on mainnet carries little traffic so far, so data custody and endorsements rest on small samples. Every figure is published with the number of checks behind it, and the weight of Fibre in the grade will be reviewed as traffic grows.

### Missed proposals are inferred
A missed proposal is inferred from public consensus data (the scheduled proposer for a height compared with the block's actual proposer). It is not an on-chain record. The complete history is published as JSON.

### Vote anomaly counters
`late_votes_5m` and `duplicate_vote_5m` are published as informational context only, never scored. With about 99 validators they read in the hundreds of thousands, which is expected.

---

## Snapshots

| Snapshot | URL |
|---|---|
| Archive (pruned) | [cumulo.pro/services/celestia/snapshot](https://cumulo.pro/services/celestia/snapshot) |

Cross-checked from outside by `snapshot_checker`.

---

## Independent Verification

| Monitor | Page |
|---|---|
| `check_d` (public RPC endpoint) | Rendered on the OTL dashboard as "Independent Verification" |
| Missed proposals | [cumulo.pro/services/celestia/missed-proposals](https://cumulo.pro/services/celestia/missed-proposals) · [methodology](https://github.com/Cumulo-pro/cumulo-block-explorer/blob/main/docs/celestia-missed-proposals.md) |
| Fibre Network Monitor | [cumulo.pro/services/celestia/fibre](https://cumulo.pro/services/celestia/fibre) · [methodology](https://cumulo.pro/services/celestia/fibre-methodology) · [API and archive](https://cumulo.pro/services/celestia/fibre-api) |

Fibre server setup guide: [Fibre_Mainnet_Setup.md](https://github.com/Cumulo-pro/cumulo-celestia-infra/blob/main/docs/fibre-setup/Fibre_Mainnet_Setup.md).

---

## Activity History

All upgrades, governance votes, infrastructure migrations and incidents for Celestia Mainnet are tracked in Cumulo's public activity log:

[github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json)

Filter by `"chain": "Celestia"`, `"network": "mainnet"`.

---

## OTL Posture API Example

```bash
# Current posture
curl https://otl-api.cumulo.com.es/otl/v2/celestia/mainnet/posture | jq '{
  status: .posture_status,
  height: .chain_context.head_height,
  signing: .validator_performance.last_signed_height,
  fibre: .fibre.reason
}'

# SLO score, with the proposals and Fibre fields
curl https://otl-api.cumulo.com.es/otl/v2/celestia/mainnet/slo | jq '.slo | {
  posture_score, posture_grade, risk_flags,
  proposal_miss_rate_pct, fibre_scored, fibre_uptime_7d_pct, fibre_custody_30d_pct
}'
```
