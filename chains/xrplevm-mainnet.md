# Chain Profile: XRPL EVM Mainnet

**Chain:** XRPL EVM Sidechain  
**Network:** Mainnet (`xrplevm_1440000-1`)  
**OTL Status:** Live (deployed 2026-09-04)  
**Dashboard:** [cumulo.pro/services/xrplevm_mainnet/otl](https://cumulo.pro/services/xrplevm_mainnet/otl) · [Documentation](https://cumulo.pro/services/xrplevm_mainnet/otl-docs)  
**API:** [otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture](https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture)

---

## About XRPL EVM

XRPL EVM is an EVM-compatible Cosmos SDK sidechain bridged to the XRP Ledger, giving XRPL smart-contract programmability (Solidity, MetaMask-compatible tooling) while settling back to XRPL. It uses CometBFT consensus with a Cosmos EVM execution layer (`exrpd` binary) — technically distinct from the classic XRP Ledger (`rippled`, RPCA consensus), even though it shares the XRPL brand and bridge.

- **Docs:** [docs.xrplevm.org](https://docs.xrplevm.org/)
- **Governance:** [governance.xrplevm.org](https://governance.xrplevm.org/)
- **GitHub:** [github.com/xrplevm](https://github.com/xrplevm)

Cumulo has run a mainnet validator since the network approved Cumulo as an official validator (Governance Proposal #18, 2026-01-01), and has executed every subsequent mandatory upgrade and security-driven precautionary action since.

---

## Cumulo Infrastructure

Same split as Story Aeneid: a dedicated validator node signs blocks (RPC bound to localhost only — no public traffic), and a separate node serves the public endpoints below.

### Validator Node
| Parameter | Value |
|---|---|
| Chain software | `exrpd v10.2.0` |
| Validator moniker | Cumulo |
| Validator consensus address | `45A2E210C9F6EFECBC2F46DEA1DECDDD1007AEDF` |

### RPC Node — Public Endpoints
| Endpoint | Value |
|---|---|
| RPC | `rpc.xrpl.cumulo.org.es` |
| RPC WebSocket (CometBFT) | `wss://rpc.xrpl.cumulo.org.es/websocket` |
| API | `api.xrpl.cumulo.org.es` |
| gRPC | `grpc.xrpl.cumulo.org.es` |
| JSON-RPC (EVM) | `json-rpc.xrpl.cumulo.org.es` |
| JSON WebSocket (EVM) | `wss://ws.xrpl.cumulo.org.es` |

---

## Chain Parameters

| Parameter | Value |
|---|---|
| Chain ID | `xrplevm_1440000-1` |
| Consensus engine | CometBFT |
| Execution layer | Cosmos EVM |
| Consensus algorithm | BFT (Tendermint-derived) |

---

## Prometheus Configuration

| Job | Target | Metrics |
|---|---|---|
| `xrplevm_mainnet_validator` | Validator node | Consensus, validator performance, P2P |
| `xrplevm_mainnet_rpc` | RPC node | Block context, network health |

**Chain selector (PromQL):** `chain_id="xrplevm_1440000-1"`.

---

## Known Metric Behaviours

### `step_duration_ms_p95_5m`
`cometbft_consensus_step_duration_seconds` is actually seven separate per-`step` histograms (`Commit`, `NewHeight`, `NewRound`, `Precommit`, `Prevote`, `PrevoteWait`, `Propose`), each with wildly different real-world scales — the `*Wait` steps intentionally include protocol wait time (waiting for 2/3+ voting power), not processing time. The OTL aggregates across the signing steps only (`Propose|Prevote|Precommit|Commit`), excluding the `*Wait`/`NewRound`/`NewHeight` transition steps, so the published p95 reflects actual signing latency rather than one arbitrary step's own distribution. This aggregation is chain-agnostic and applies to any chain onboarded with a `step`-labeled histogram.

### Vote anomaly counters
`late_votes_5m` and `duplicate_vote_5m` are published as informational context only, never scored — see [methodology.md](../docs/methodology.md). Cumulo runs a small, curated validator set (17 validators observed) on this sidechain, so baselines here may differ from a large public testnet.

---

## Snapshots

Cumulo provides a pruned archive snapshot for XRPL EVM Mainnet to help new operators bootstrap quickly.

| Snapshot | URL |
|---|---|
| Archive (pruned) | [cumulo.pro/services/xrplevm_mainnet/snapshot](https://cumulo.pro/services/xrplevm_mainnet/snapshot) |

Unlike Story Aeneid (separate consensus + execution-client snapshot files), XRPL EVM Mainnet ships a single combined archive snapshot.

---

## Independent Endpoint Verification

XRPL EVM Mainnet is covered by Cumulo's `check_d` endpoint checker — a second, independent measurement of the same public endpoints (latency, reliability history, TLS), separate from what the OTL itself reports. The OTL dashboard renders this live as its own "Independent Verification" section: reliability, smoothed/raw latency, per-region latency, TLS days-to-expiry, sync/CORS/pruning, node version, and a block-height cross-check against the OTL's own `/status` read.

| Checker | Dashboard |
|---|---|
| RPC | [cumulo.pro/services/xrplevm_mainnet/rpcscan.php](https://cumulo.pro/services/xrplevm_mainnet/rpcscan.php) |
| REST API | [cumulo.pro/services/xrplevm_mainnet/apiscan.php](https://cumulo.pro/services/xrplevm_mainnet/apiscan.php) |

See [check_d resources](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/check_End-Points/check_d%20resources.md).

---

## Activity History

All upgrades, governance votes, and incidents for XRPL EVM Mainnet are tracked in Cumulo's public activity log:

[github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json)

Filter by `"chain": "XRPL EVM"`, `"network": "mainnet"`.

---

## OTL Posture API Example

```bash
# Current posture
curl https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture | jq '{
  status: .posture_status,
  height: .chain_context.head_height,
  freshness: .reliability.chain_freshness_sec,
  grade: .slo,
  signing: .validator_performance.last_signed_height
}'

# SLO score
curl https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/slo | jq '.slo'
```
