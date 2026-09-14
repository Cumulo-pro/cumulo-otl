# Chain Profile: XRPL EVM Testnet

**Chain:** XRPL EVM Sidechain  
**Network:** Testnet (`xrplevm_1449000-1`)  
**OTL Status:** Live (deployed 2026-09-10)  
**Dashboard:** [cumulo.pro/services/xrplevm/otl](https://cumulo.pro/services/xrplevm/otl) · [Documentation](https://cumulo.pro/services/xrplevm/otl-docs)  
**API:** [otl-api.cumulo.com.es/otl/v2/xrplevm/testnet/posture](https://otl-api.cumulo.com.es/otl/v2/xrplevm/testnet/posture)

---

## About XRPL EVM

XRPL EVM is an EVM-compatible Cosmos SDK sidechain bridged to the XRP Ledger, giving XRPL smart-contract programmability (Solidity, MetaMask-compatible tooling) while settling back to XRPL. It uses CometBFT consensus with a Cosmos EVM execution layer (`exrpd` binary) — technically distinct from the classic XRP Ledger (`rippled`, RPCA consensus), even though it shares the XRPL brand and bridge. Testnet is a fully independent network from Mainnet (separate chain ID, separate validator set).

- **Docs:** [docs.xrplevm.org](https://docs.xrplevm.org/)
- **Governance:** [governance.xrplevm.org](https://governance.xrplevm.org/)
- **GitHub:** [github.com/xrplevm](https://github.com/xrplevm)

Cumulo has validated on XRPL EVM Testnet since the network approved Cumulo as a validator (Governance Proposal #29, "Add Cumulo validator", 2025-04-30) — confirmed independently both by direct on-chain query (`exrpd query gov proposals`) and by the validator's own `commission.update_time`, which lines up with the proposal's `voting_end_time` to the second.

---

## Cumulo Infrastructure

Same split as Mainnet and Story Aeneid: a dedicated validator setup signs blocks, and a separate node serves the public endpoints below. Unlike Mainnet (single signer with manual backup), Testnet runs its signing through Horcrux across two signer nodes.

### Validator
| Parameter | Value |
|---|---|
| Chain software | `exrpd v11.1.0` |
| Validator moniker | Cumulo |
| Consensus address | `AC3E00CD0B9D97D933850853E00BCB88D2BCD89A` |

### RPC Node — Public Endpoints
| Endpoint | Value |
|---|---|
| RPC | `rpc.xrpl.cumulo.com.es` |
| RPC WebSocket (CometBFT) | `wss://rpc.xrpl.cumulo.com.es/websocket` |
| API | `api.xrpl.cumulo.com.es` |
| gRPC | `grpc.xrpl.cumulo.com.es` |
| JSON-RPC (EVM) | `json-rpc.xrpl.cumulo.com.es` |
| JSON WebSocket (EVM) | `wss://ws.xrpl.cumulo.com.es` |

### Archive / Snapshot RPC
| Endpoint | Value |
|---|---|
| RPC | `rpc-xrplevm-testnet.archive.cumulo.com.es` |
| API | `api-xrplevm-testnet.archive.cumulo.com.es` |
| gRPC | `grpc-xrplevm-testnet.archive.cumulo.com.es` |
| JSON-RPC | `jsonrpc-xrplevm-testnet.archive.cumulo.com.es` |

---

## Chain Parameters

| Parameter | Value |
|---|---|
| Chain ID | `xrplevm_1449000-1` |
| Consensus engine | CometBFT |
| Execution layer | Cosmos EVM |
| Consensus algorithm | BFT (Tendermint-derived) |
| Minimum signed-per-window | 0.5 (verified against the live slashing module params) |

---

## Prometheus Configuration

| Job | Target | Metrics |
|---|---|---|
| `xrplevm_testnet_validator` | Validator signer node | Consensus, validator performance, P2P |
| `xrplevm_testnet_rpc` | RPC node | Block context, network health |

**Chain selector (PromQL):** `chain_id="xrplevm_1449000-1"`.

---

## On-Chain Record Source

Testnet has its own block explorer collector (distinct domain from Mainnet's), which the OTL reads as a soft, Prometheus-like signal to cross-check `signing_uptime_pct`, `missed_blocks`, `jail_margin`, `jailed`/`tombstoned`/`bonded`/`active` status, and voting share directly against the chain's staking/slashing modules. As with all `on_chain` sources in the OTL, this degrades gracefully (`capabilities.on_chain: false`) rather than breaking the rest of the posture grade if the source is ever unavailable.

---

## Known Metric Behaviours

Testnet shares the same step-duration histogram aggregation fix and vote-anomaly-as-context-only treatment documented in the [XRPL EVM Mainnet profile](xrplevm-mainnet.md#known-metric-behaviours) — both are chain-agnostic fixes in the shared scoring code, not chain-specific configuration.

---

## Snapshots

Cumulo provides a snapshot service for XRPL EVM Testnet to help new operators bootstrap quickly.

| Snapshot | URL |
|---|---|
| Index | [xrpl.cumulo.com.es/snapshots/](https://xrpl.cumulo.com.es/snapshots/) |

`snapshot_checker` cross-checking (size/freshness/speed from an independent collector) is not yet wired up for Testnet the way it is for Mainnet — tracked in the [Roadmap](../README.md#roadmap).

---

## Independent Endpoint Verification

XRPL EVM Testnet is covered by Cumulo's `check_d` endpoint checker — a second, independent measurement of the same public endpoints (latency, reliability history, TLS), separate from what the OTL itself reports.

See [check_d resources](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/check_End-Points/check_d%20resources.md).

---

## Activity History

All upgrades, governance votes, and incidents for XRPL EVM Testnet are tracked in Cumulo's public activity log:

[github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json)

Filter by `"chain": "XRPL EVM"`, `"network": "testnet"`.

---

## OTL Posture API Example

```bash
# Current posture
curl https://otl-api.cumulo.com.es/otl/v2/xrplevm/testnet/posture | jq '{
  status: .posture_status,
  height: .chain_context.head_height,
  freshness: .reliability.chain_freshness_sec,
  grade: .slo,
  signing: .validator_performance.last_signed_height
}'

# SLO score
curl https://otl-api.cumulo.com.es/otl/v2/xrplevm/testnet/slo | jq '.slo'
```
