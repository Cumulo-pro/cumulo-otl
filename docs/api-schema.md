# OTL API Schema

The Cumulo OTL exposes a public, stable, versioned JSON API. All endpoints are read-only and require no authentication.

**Base URL:** `https://otl-api.cumulo.com.es`

---

## Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Service health check |
| `GET` | `/otl/chains` | List all configured chains |
| `GET` | `/otl/v2/:chain/:network/posture` | Full posture snapshot |
| `GET` | `/otl/v2/:chain/:network/slo` | SLO score and grade |
| `GET` | `/otl/v2/:chain/:network/diagnostics` | Debug — data source connectivity |

---

## GET /health

```json
{
  "ok": true,
  "service": "cumulo-otl-api",
  "version": "v2",
  "time": "2026-03-28T15:00:00.000Z"
}
```

---

## GET /otl/chains

```json
{
  "ok": true,
  "count": 1,
  "chains": [
    {
      "chain": "story",
      "network": "aeneid",
      "display_name": "Story Aeneid",
      "posture_url": "/otl/v2/story/aeneid/posture",
      "slo_url": "/otl/v2/story/aeneid/slo"
    }
  ],
  "time": "2026-03-28T15:00:00.000Z"
}
```

---

## GET /otl/v2/:chain/:network/posture

Full posture snapshot. Cached for 60 seconds server-side.

### Response Schema

```jsonc
{
  // ── Identity ──────────────────────────────────────────
  "id":             "story-aeneid-posture",
  "schema_version": "v2",
  "chain":          "Story",
  "network":        "aeneid",
  "chain_id":       "devnet-1",
  "display_name":   "Story Aeneid",
  "version":        "storyd v1.5.2 · geth v1.2.1",

  // ── Status ────────────────────────────────────────────
  "posture_status": "ready",        // ready | syncing | stale | bootstrapping | maintenance | down
  "ok":             true,           // false if RPC unreachable
  "generated_at":   "2026-03-28T15:00:00.000Z",
  "last_updated":   "2026-03-28T15:00:00.000Z",

  "data_window": {
    "availability": "7d",
    "latency":      "5m",
    "freshness":    "now"
  },

  // ── Reliability ───────────────────────────────────────
  "reliability": {
    "chain_freshness_sec":       3,      // now − latest_block_time
    "node_availability_7d_pct":  99.97,  // null if < 7d Prometheus history
    "rpc_availability_7d_pct":   92.63,
    "rpc_p95_latency_ms":        null    // null if not instrumented
  },

  // ── Chain context (RPC fields always present) ─────────
  "chain_context": {
    "head_height":        16162363,
    "latest_block_time":  "2026-03-28T15:00:00.000Z",
    "catching_up":        false,

    // Prometheus fields (null if unavailable)
    "peers_avg":                          9,
    "avg_block_time_sec":                 1.92,
    "txs_per_sec_5m":                     0.0,
    "block_size_bytes_avg_5m":            7466,
    "block_processing_ms_avg_5m":         17,     // p50 median, ms
    "block_processing_ms_p95_5m":         91,     // ms
    "round_duration_ms_p95_5m":           98,     // ms
    "step_duration_ms_p95_5m":            189,    // ms
    "quorum_prevote_delay_ms_p95_5m":     126,    // ms
    "quorum_precommit_delay_ms_p95_5m":   245     // ms
  },

  // ── Network health (chain-wide, Prometheus) ───────────
  "network_health": {
    "validators_count":                   60,
    "missing_validators":                 0,
    "byzantine_validators":               0,
    "late_votes_5m":                      63448,  // cumulative counter
    "duplicate_vote_5m":                  64599,  // cumulative counter
    "duplicate_block_part_5m":            1138,   // cumulative counter
    "p2p_recv_bytes_per_sec_5m":          108372, // bytes/s
    "p2p_send_bytes_per_sec_5m":          67249,  // bytes/s
    "p2p_peer_pending_send_bytes_avg_5m": 14,
    "blocksync_syncing":                  0,
    "blocksync_latest_block_height":      null    // null when not syncing
  },

  // ── Validator performance ─────────────────────────────
  "validator_performance": {
    "state":                   "active",   // active | syncing | unknown
    "validator_address":       "E2E81B5B0302E36F9EA67B7600A392DA82C41AF2",
    "validator_address_prom":  "7FBE0E10EF7E41CEECC598AB43195F84B6DC517F",
    "voting_power":            26024000,
    "voting_power_pct":        0.015896,
    "last_signed_height":      16162361
  },

  // ── Capabilities ─────────────────────────────────────
  // Tells consumers which data sources are currently available
  "capabilities": {
    "rpc":            true,
    "prometheus":     true,
    "availability":   true,
    "chain_context":  true,
    "network_health": true,
    "validator":      true,
    "latency":        false   // not instrumented in Story's build
  },

  // ── Sources (public-safe, no IPs) ─────────────────────
  "sources": {
    "rpc_status":     "(internal)",
    "prometheus":     "(internal)",
    "prom_labels": {
      "node": "job=\"storyAened\"",
      "rpc":  "job=\"storyAenedRPC\""
    },
    "selector_comet": "chain_id=\"devnet-1\""
  },

  // ── Public endpoints ──────────────────────────────────
  "public_endpoints": [
    { "label": "RPC",      "value": "rpc.story-aeneid.cumulo.me" },
    { "label": "API",      "value": "api.story-aeneid.cumulo.me" },
    { "label": "JSON-RPC", "value": "json-rpc.story-aeneid.cumulo.me" },
    { "label": "WS / EVM", "value": "wss://ws-evm.story-aeneid.cumulo.me" }
  ],

  "notes": []  // operational notes array, empty when all systems normal
}
```

### Null fields

Fields are `null` when:
- The data source is unavailable (Prometheus unreachable)
- Insufficient history (availability metrics need 7d of scrape data)
- The metric is not instrumented by the chain's node binary

Consumers must handle `null` for all Prometheus-sourced fields.

---

## GET /otl/v2/:chain/:network/slo

```jsonc
{
  "ok":             true,
  "schema_version": "v2",
  "chain":          "story",
  "network":        "aeneid",
  "display_name":   "Story Aeneid",
  "generated_at":   "2026-03-28T15:00:00.000Z",
  "posture_status": "ready",

  "slo": {
    "consensus_health_score": 100,    // 0–100
    "posture_grade":          "A",    // A | B | C | D
    "availability_7d_pct":    99.97,  // null if insufficient history
    "rpc_availability_7d_pct": 92.63,
    "risk_flags": [
      "duplicate_votes_present",      // informational, no score impact
      "late_votes_present"            // informational, no score impact
    ]
  },

  "notes": [
    "SLO is computed from current posture snapshot.",
    "Availability uses 7d Prometheus window when available."
  ],

  "time": "2026-03-28T15:00:00.000Z"
}
```

See [slo-scoring.md](slo-scoring.md) for the complete scoring formula.

---

## GET /otl/v2/:chain/:network/diagnostics

Debug endpoint. Returns connectivity status for all data sources.

```jsonc
{
  "ok":      true,
  "chain":   "story",
  "network": "aeneid",

  "rpc": {
    "url":        "(internal)",
    "ok":         true,
    "latency_ms": 162,
    "error":      null
  },

  "prometheus": {
    "url":            "(internal)",
    "label_node":     "job=\"storyAened\"",
    "label_rpc":      "job=\"storyAenedRPC\"",
    "selector_comet": "chain_id=\"devnet-1\"",
    "height_sample":  16162363,
    "available":      true
  },

  "config": {
    "validator_address_prom":    "7FBE0E10EF7E41CEECC598AB43195F84B6DC517F",
    "freshness_threshold_sec":   120
  },

  "time": "2026-03-28T15:00:00.000Z"
}
```

---

## Error Responses

All endpoints return standard error objects:

```jsonc
{
  "ok":     false,
  "chain":  "story",
  "network": "aeneid",
  "error":  "RPC unavailable: fetch failed",
  "code":   "RPC_ERROR",     // UNKNOWN_CHAIN | NO_TARGET | RPC_ERROR
  "time":   "2026-03-28T15:00:00.000Z"
}
```

HTTP status codes: `200` (ok), `404` (unknown chain), `502` (upstream error).

---

## Versioning

The API is versioned via the URL path (`/v2/`). Breaking schema changes will increment the version. The `schema_version` field in responses reflects the current schema version.

---

## Rate Limits

The API is served behind Nginx with standard rate limiting. It is intended for dashboard consumption (one request per 60s per chain). Please do not poll more frequently than the cache TTL (60s).
