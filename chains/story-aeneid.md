# Chain Profile: Story Aeneid

**Chain:** Story Protocol  
**Network:** Aeneid Testnet (devnet-1)  
**OTL Status:** Live  
**Dashboard:** [cumulo.pro/services/story_aeneid](https://cumulo.pro/services/story_aeneid)  
**API:** [otl-api.cumulo.com.es/otl/v2/story/aeneid/posture](https://otl-api.cumulo.com.es/otl/v2/story/aeneid/posture)

---

## About Story

Story is the World's IP Blockchain — a purpose-built Layer 1 for programmable intellectual property. It uses a hybrid architecture combining a CometBFT consensus layer (EVM-compatible execution via `story-geth`) with native IP management primitives.

- **Story Protocol:** [story.foundation](https://www.story.foundation/)
- **Documentation:** [docs.story.foundation](https://docs.story.foundation/)
- **GitHub:** [github.com/piplabs/story](https://github.com/piplabs/story)

---

## Cumulo Infrastructure

### Validator Node
| Parameter | Value |
|---|---|
| Server | Velia 12 |
| Chain software | storyd v1.5.2 |
| Execution client | story-geth v1.2.1 |
| Consensus port | 26646 |
| Validator address | `7FBE0E10EF7E41CEECC598AB43195F84B6DC517F` |

### RPC Node
| Parameter | Value |
|---|---|
| Server | Velia 2 |
| RPC endpoint | `rpc.story-aeneid.cumulo.me` |
| API endpoint | `api.story-aeneid.cumulo.me` |
| JSON-RPC endpoint | `json-rpc.story-aeneid.cumulo.me` |
| WebSocket / EVM | `wss://ws-evm.story-aeneid.cumulo.me` |

---

## Chain Parameters

| Parameter | Value |
|---|---|
| Chain ID | `devnet-1` |
| Consensus engine | CometBFT |
| Target block time | ~2s |
| Validator set size | ~60 active validators |
| Consensus algorithm | BFT (Tendermint-derived) |

---

## Prometheus Configuration

| Job | Target | Metrics |
|---|---|---|
| `storyAened` | Validator node metrics endpoint | Consensus, validator performance, P2P |
| `storyAenedRPC` | RPC node metrics endpoint | Block context, network health |

**Chain selector (PromQL):** `chain_id="devnet-1"`

---

## Known Metric Behaviours

### Vote anomaly counters
`cometbft_consensus_duplicate_vote` and `cometbft_consensus_late_votes` are monotonically increasing counters that grow continuously as normal network behaviour on Story Aeneid testnet. High absolute values are expected and do not indicate validator misbehaviour. See [methodology.md](../docs/methodology.md#5-vote-anomaly-counters-informational).

### Block processing metric units
`cometbft_state_block_processing_time` in Story's build exports values in **milliseconds** (not seconds as in standard CometBFT). The OTL applies no unit conversion for this metric.

### Availability accumulation
`node_availability_7d_pct` requires 7 days of Prometheus scrape history to be meaningful. During the first 7 days after a Prometheus target is added or reconfigured, this field shows `0` or partial data.

---

## Snapshots

Cumulo provides daily pruned snapshots for Story Aeneid to help new operators bootstrap quickly.

| Snapshot | URL |
|---|---|
| Index | [snap.testnet.story.cumulo.com.es/story-snapshots/](https://snap.testnet.story.cumulo.com.es/story-snapshots/) |
| Consensus | `{index}/{date}_story_consensus_snapshot.tar.zst` |
| Geth | `{index}/{date}_story_geth_snapshot.tar.zst` |

Snapshot frequency: daily  
Typical size: ~68 GiB (consensus) + ~17 GiB (geth)

---

## Activity History

All upgrades, governance votes, and incidents for Story Aeneid are tracked in Cumulo's public activity log:

[github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json](https://github.com/Cumulo-pro/Cumulo-Front-Chain/blob/main/incidents.json)

Filter by `"chain": "Story"`.

---

## OTL Posture API Example

```bash
# Current posture
curl https://otl-api.cumulo.com.es/otl/v2/story/aeneid/posture | jq '{
  status: .posture_status,
  height: .chain_context.head_height,
  freshness: .reliability.chain_freshness_sec,
  grade: .slo,
  signing: .validator_performance.last_signed_height
}'

# SLO score
curl https://otl-api.cumulo.com.es/otl/v2/story/aeneid/slo | jq '.slo'
```
