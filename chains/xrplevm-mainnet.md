# Chain Profile: XRPL EVM Mainnet

**Chain:** XRPL EVM Sidechain  
**Network:** Mainnet (`xrplevm_1440000-1`)  
**OTL Status:** Live (deployed 2026-09-04)  
**Dashboard:** [cumulo.pro/services/xrplevm_mainnet/otl](https://cumulo.pro/services/xrplevm_mainnet/otl) · [Documentation](https://cumulo.pro/services/xrplevm_mainnet/otl-docs) · [Audit report (PDF)](https://cumulo.pro/services/xrplevm_mainnet/otl-report)  
**API:** [otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture](https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture)

---

## About XRPL EVM

XRPL EVM is an EVM-compatible Cosmos SDK sidechain bridged to the XRP Ledger, giving XRPL smart-contract programmability (Solidity, MetaMask-compatible tooling) while settling back to XRPL. It uses CometBFT consensus with a Cosmos EVM execution layer (`exrpd` binary), technically distinct from the classic XRP Ledger (`rippled`, RPCA consensus), even though it shares the XRPL brand and bridge.

- **Docs:** [docs.xrplevm.org](https://docs.xrplevm.org/)
- **Governance:** [governance.xrplevm.org](https://governance.xrplevm.org/)
- **GitHub:** [github.com/xrplevm](https://github.com/xrplevm)

XRPL EVM is a **Proof of Governance** chain, not a permissionless Proof of Stake one: the active validator set is admitted by on-chain governance, voting weight is assigned by governance rather than acquired with stake, validators run at 0% commission and there is no delegation market. The OTL therefore measures what stays in the operator's hands (signing, availability of its infrastructure, upgrade discipline, governance participation) and does not grade commission, delegation or stake concentration, which dominate on permissionless chains.

Cumulo has run a mainnet validator since the network approved Cumulo as an official validator (Governance Proposal #18, 2026-01-01), and has executed every subsequent mandatory upgrade and security-driven precautionary action since.

---

## Cumulo Infrastructure

A dedicated validator node signs blocks (its RPC is bound to localhost only, with no public traffic), and a separate node serves the public endpoints below. Mainnet runs a single active signer with a manually managed backup node; Horcrux threshold signing is used on Testnet and is planned for Mainnet. Exact IP addresses and hosting providers are deliberately not published.

### Validator Node
| Parameter | Value |
|---|---|
| Chain software | `exrpd` (current version shown live on the dashboard) |
| Validator moniker | Cumulo |
| Validator consensus address | `45A2E210C9F6EFECBC2F46DEA1DECDDD1007AEDF` |

### RPC Node: Public Endpoints
| Endpoint | Value |
|---|---|
| RPC | `rpc.xrpl.cumulo.org.es` |
| RPC WebSocket (CometBFT) | `wss://rpc.xrpl.cumulo.org.es/websocket` |
| API | `api.xrpl.cumulo.org.es` |
| gRPC | `grpc.xrpl.cumulo.org.es` |
| JSON-RPC (EVM) | `json-rpc.xrpl.cumulo.org.es` |
| JSON WebSocket (EVM) | `wss://ws.xrpl.cumulo.org.es` |

### Backup node and IBC relayer
The dashboard also reports a standby backup node (manual failover, disclosed as text, not backed by a live health signal) and Cumulo's IBC relayer, listing the channels it serves (Celestia, Cosmos Hub, Injective) with their state read live from the chain's own IBC module. Neither is part of the grade.

---

## Chain Parameters

| Parameter | Value |
|---|---|
| Chain ID | `xrplevm_1440000-1` |
| Consensus engine | CometBFT |
| Execution layer | Cosmos EVM |
| Consensus algorithm | BFT (Tendermint-derived) |
| Validator set model | Proof of Governance (admission by on-chain vote) |

---

## Prometheus Configuration

| Job | Target | Metrics |
|---|---|---|
| `xrplevm_mainnet_validator` | Validator node | Consensus, validator performance, P2P |
| `xrplevm_mainnet_rpc` | RPC node | Block context, network health |

**Chain selector (PromQL):** `chain_id="xrplevm_1440000-1"`.

---

## Governance

Governance vote participation is a scored signal on this chain (target 90%, see [slo-scoring.md](../docs/slo-scoring.md)). Eligible proposals are those that reached a vote after Cumulo's admission. While a proposal is open, the vote is read live from the chain; once it closes, the chain no longer keeps the individual vote record (standard Cosmos SDK behaviour), so closed proposals are read from the public activity log. Disclosure (a published transaction hash per vote) is shown separately and never moves the grade.

---

## Known Metric Behaviours

### `step_duration_ms_p95_5m`
`cometbft_consensus_step_duration_seconds` is actually seven separate per-`step` histograms (`Commit`, `NewHeight`, `NewRound`, `Precommit`, `Prevote`, `PrevoteWait`, `Propose`), each with wildly different real-world scales. The `*Wait` steps intentionally include protocol wait time (waiting for 2/3+ voting power), not processing time. The OTL aggregates across the signing steps only (`Propose|Prevote|Precommit|Commit`), excluding the `*Wait`/`NewRound`/`NewHeight` transition steps, so the published p95 reflects actual signing latency rather than one arbitrary step's own distribution. This aggregation is chain-agnostic and applies to any chain onboarded with a `step`-labeled histogram.

### Vote timing counters
`late_votes_5m` and `duplicate_vote_5m` are published as informational context only, never scored (shown on the dashboard as "Vote Timing Counters"). See [methodology.md](../docs/methodology.md). Cumulo runs inside a small, curated validator set on this sidechain, so baselines here may differ from a large public network.

---

## Snapshots

Cumulo provides a pruned archive snapshot for XRPL EVM Mainnet to help new operators bootstrap quickly.

| Snapshot | URL |
|---|---|
| Archive (pruned) | [cumulo.pro/services/xrplevm_mainnet/snapshot](https://cumulo.pro/services/xrplevm_mainnet/snapshot) |

XRPL EVM Mainnet ships a single combined archive snapshot.

---

## External Endpoint Measurement

XRPL EVM Mainnet is covered by Cumulo's `check_d` endpoint checker, an out-of-band measurement of the same public endpoints (latency, reliability history, TLS), separate from what the OTL itself reports. It is Cumulo-built, open source and documented, and it measures from outside the node, so it is independent of the node's self-report, not of Cumulo as operator. The OTL dashboard renders it live: reliability, smoothed/raw latency, per-region latency, TLS days-to-expiry, sync/CORS/pruning, node version, and a block-height cross-check against the OTL's own `/status` read.

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
