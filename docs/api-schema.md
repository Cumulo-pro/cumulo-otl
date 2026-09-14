# OTL API Schema

The Cumulo OTL exposes a public, stable, versioned JSON API. All endpoints are read-only and require no authentication.

**Base URL:** `https://otl-api.cumulo.com.es`

Example values below are illustrative (rounded, representative numbers), not a live capture. For real, current figures, query the API directly, e.g. `curl https://otl-api.cumulo.com.es/otl/v2/xrplevm/mainnet/posture`.

---

## Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Service health check |
| `GET` | `/otl/chains` | List all configured chains |
| `GET` | `/otl/v2/:chain/:network/posture` | Full posture snapshot (cached, 60s TTL) |
| `GET` | `/otl/v2/:chain/:network/slo` | SLO score, grade, and risk flags |
| `POST` | `/otl/v2/:chain/:network/refresh` | Force a fresh fetch, bypassing the cache |
| `GET` | `/otl/v2/:chain/:network/diagnostics` | Debug: raw connectivity check against RPC and Prometheus |

Two further endpoints exist (`GET /otl/cache`, `GET /otl/cache/:id`, cache inspection, and a legacy `GET /otl/v1/story-aeneid/posture` compatibility route kept for the original v1 consumer). Neither is part of the stable public contract: the cache endpoints are operational tooling and the v1 route is a compatibility shim for one legacy consumer. Don't build against either.

---

## GET /health

```json
{
  "ok": true,
  "service": "cumulo-otl-api",
  "version": "v2",
  "time": "2026-09-14T12:00:00.000Z"
}
```

---

## GET /otl/chains

```json
{
  "ok": true,
  "count": 2,
  "chains": [
    {
      "chain": "xrplevm",
      "network": "mainnet",
      "display_name": "XRPL EVM Mainnet",
      "posture_url": "/otl/v2/xrplevm/mainnet/posture",
      "slo_url": "/otl/v2/xrplevm/mainnet/slo"
    },
    {
      "chain": "xrplevm",
      "network": "testnet",
      "display_name": "XRPL EVM Testnet",
      "posture_url": "/otl/v2/xrplevm/testnet/posture",
      "slo_url": "/otl/v2/xrplevm/testnet/slo"
    }
  ],
  "time": "2026-09-14T12:00:00.000Z"
}
```

---

## GET /otl/v2/:chain/:network/posture

Full posture snapshot. Cached for 60 seconds server side.

### Response Schema

```jsonc
{
  // ── Identity ──────────────────────────────────────────
  "id":             "xrplevm-mainnet-posture",
  "schema_version": "v2",
  "chain":          "XRPL EVM",
  "network":        "mainnet",
  "chain_id":       "xrplevm_1440000-1",
  "display_name":   "XRPL EVM Mainnet",
  "version":        "exrpd v10.2.0",

  // ── Status ────────────────────────────────────────────
  "posture_status": "ready",        // ready | syncing | stale | bootstrapping | maintenance | down
  "ok":             true,           // false if RPC unreachable
  "generated_at":   "2026-09-14T12:00:00.000Z",
  "last_updated":   "2026-09-14T12:00:00.000Z",

  // Truthful label: reflects how much scrape/rollup history actually backs
  // the figures below, never a fixed "7d" regardless of onboarding age.
  "data_window": {
    "availability": "7d",   // or "<N>d" when less history exists, or "unknown"
    "latency":      "5m",
    "freshness":    "now"
  },

  // ── SLO configuration (operator-chosen; also echoed in /slo) ─────────
  "slo_config": {
    "rpc_availability_target_pct":  99.0,
    "gov_participation_target_pct": 90.0
  },

  // ── Reliability ───────────────────────────────────────
  "reliability": {
    "chain_freshness_sec":       3,      // now − latest_block_time
    "node_availability_7d_pct":  99.98,  // null if < 1d Prometheus history
    "rpc_availability_7d_pct":   99.6,
    "rpc_p95_latency_ms":        null,   // null if not instrumented (optional LATENCY capability)
    "availability_coverage_days": 7,     // real coverage backing the *_7d_pct figures above

    // Persisted daily rollup, independent of Prometheus retention. Each
    // window is null (never a guess) until at least one full day of rollup
    // history exists for it; only fully-closed days are averaged in.
    "long_term": {
      "30d": {
        "node_availability_pct": 99.95,
        "rpc_availability_pct":  99.4,
        "coverage_days": 30,
        "full_window": true,
        "since":   "2026-08-15",
        "through": "2026-09-13"
      },
      "90d": null,          // fewer than 90 days of rollup history exist yet
      "lifetime": {
        "node_availability_pct": 99.96,
        "rpc_availability_pct":  99.5,
        "coverage_days": 60,
        "full_window": true,   // "full" for lifetime means every day the rollup has, not a fixed target
        "since":   "2026-07-15",
        "through": "2026-09-13"
      }
    }
  },

  // ── Chain context (RPC fields always present; Prometheus fields may be null) ──
  "chain_context": {
    "head_height":        16162363,
    "latest_block_time":  "2026-09-14T12:00:00.000Z",
    "catching_up":        false,

    "peers_avg":                          9,
    "avg_block_time_sec":                 1.9,
    "txs_per_sec_5m":                     0.02,
    "block_size_bytes_avg_5m":            7466,
    "block_processing_ms_avg_5m":         17,     // p50 median, ms
    "block_processing_ms_p95_5m":         91,     // ms
    "round_duration_ms_p95_5m":           98,     // ms
    "step_duration_ms_p95_5m":            189,    // ms
    "quorum_prevote_delay_ms_p95_5m":     126,    // ms
    "quorum_precommit_delay_ms_p95_5m":   245     // ms
  },

  // ── Network health (chain-wide, Prometheus; NOT scored) ───────────────
  "network_health": {
    "validators_count":                   17,
    "missing_validators":                 0,
    "byzantine_validators":               0,
    "late_votes_5m":                      12,     // per-node counter, network-position dependent
    "duplicate_vote_5m":                  0,
    "duplicate_block_part_5m":            3,
    "p2p_recv_bytes_per_sec_5m":          108372, // bytes/s
    "p2p_send_bytes_per_sec_5m":          67249,  // bytes/s
    "p2p_peer_pending_send_bytes_avg_5m": 14,
    "blocksync_syncing":                  0,
    "blocksync_latest_block_height":      null    // null when not syncing
  },

  // ── Network context: same chain-wide conditions, summarized for the
  // dashboard. Never scored, see slo-scoring.md § What the Score Does NOT
  // Penalise. ──────────────────────────────────────────────────────────
  "network_context": {
    "byzantine_validators":     0,
    "missing_validators":       0,
    "round_duration_ms_p95_5m": 98,
    "state":                    "nominal"   // "degraded" if byzantine>0, missing>1, or round_duration>3000ms
  },

  // ── On-chain validator record (staking + slashing, CHAIN-labeled) ─────
  // `available` is always present explicitly. buildSlo()'s
  // on_chain_unconfirmed_grade_capped check reads this field directly.
  "on_chain": {
    "available": true,
    "signing_uptime_pct":   99.99,
    "missed_blocks":        6,
    "signed_blocks_window": 88888,
    "min_signed_per_window": 0.5,
    "jail_margin":          44444,
    "jailed":     false,
    "tombstoned": false,
    "bonded":     true,
    "active":     true,
    "tokens":     "1000000",
    "voting_share_pct":     5.88,
    "commission_rate":            0,
    "commission_max_rate":        0,
    "commission_max_change_rate": 0,
    "commission_updated_at":      "2026-01-01T15:11:36.000Z",

    "integrity": {
      "ever_slashed":  false,
      "events":        0,
      "ever_delayed_upgrade":  false,
      "delayed_upgrades":      0,
      "total_upgrades_tracked": 6,
      "source_checked": true
    }
  },

  // ── Governance participation ───────────────────────────────────────────
  // participation_pct feeds posture_score. disclosure_pct/disclosure_gaps
  // are informational only, never scored. See methodology.md § 5 for the
  // hybrid live-chain / incidents.json sourcing by proposal state.
  "governance": {
    "available":        true,
    "votes_eligible":   19,
    "votes_cast":       19,
    "participation_pct": 100,
    "votes_disclosed":  19,
    "disclosure_pct":   100,
    "disclosure_gaps":  0
  },

  // ── Validator performance ─────────────────────────────
  "validator_performance": {
    "state":                   "active",   // active | syncing | unknown
    "validator_address":       "45A2E210C9F6EFECBC2F46DEA1DECDDD1007AEDF",
    "validator_address_prom":  "45A2E210C9F6EFECBC2F46DEA1DECDDD1007AEDF",
    "voting_power":            1000000,
    "voting_power_pct":        5.8823529,
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
    "latency":        false,   // optional; true only where a latency probe is configured
    "on_chain":       true
  },

  // ── Sources (public-safe, no IPs) ─────────────────────
  "sources": {
    "rpc_status":     "(internal)",
    "prometheus":     "(internal)",
    "prom_labels": {
      "node": "job=\"xrplevm_mainnet_validator\"",
      "rpc":  "job=\"xrplevm_mainnet_rpc\""
    },
    "selector_comet": "chain_id=\"xrplevm_1440000-1\""
  },

  // ── Public endpoints ──────────────────────────────────
  "public_endpoints": [
    { "label": "RPC",      "value": "rpc.xrpl.cumulo.com.es" },
    { "label": "API",      "value": "api.xrpl.cumulo.com.es" },
    { "label": "gRPC",     "value": "grpc.xrpl.cumulo.com.es" },
    { "label": "JSON-RPC", "value": "json-rpc.xrpl.cumulo.com.es" }
  ],

  "notes": []  // operational notes array, e.g. "Prometheus metrics unavailable...", empty when all systems normal
}
```

### Null and soft-unavailable fields

Every field sourced from Prometheus, RPC latency probes, or the on-chain fetch is `null` (or its parent block carries `available: false`) when:
- The data source is unreachable
- Insufficient history exists (availability figures need at least one day of scrape or rollup data)
- The metric isn't instrumented by the chain's node binary, or no probe is configured for it

None of these fields are ever estimated or extrapolated to fill the gap. `on_chain` is the one exception treated as load bearing rather than soft: when it can't be fetched, `posture_grade` is explicitly capped rather than silently assumed healthy (see slo-scoring.md).

Consumers must handle `null` for every Prometheus-, latency-, and on-chain-sourced field.

---

## GET /otl/v2/:chain/:network/slo

```jsonc
{
  "ok":             true,
  "schema_version": "v2",
  "chain":          "xrplevm",
  "network":        "mainnet",
  "display_name":   "XRPL EVM Mainnet",
  "generated_at":   "2026-09-14T12:00:00.000Z",
  "posture_status": "ready",

  "slo": {
    "posture_score":            100,     // 0-100
    "consensus_health_score":   100,     // deprecated alias of posture_score, kept for existing dashboards
    "posture_grade":            "A",     // A | B | C | D
    "rpc_availability_target_pct": 99.0,
    "availability_7d_pct":      99.98,   // null if insufficient history
    "rpc_availability_7d_pct":  99.6,
    "signing_uptime_pct":       99.99,
    "missed_blocks":            6,
    "jail_margin":              44444,
    "network_context_state":    "nominal",
    "governance_participation_target_pct": 90.0,
    "governance_participation_pct":        100,
    "ever_delayed_upgrade":     false,
    "risk_flags": []            // e.g. ["governance_participation_below_target", "signing_gap"]
  },

  "notes": [
    "SLO is computed from current posture snapshot.",
    "Availability uses a rolling 7d Prometheus window (see data_window.availability).",
    "Long-term availability (30d/90d/lifetime), when enough daily rollup history exists, is in posture.reliability.long_term."
  ],

  "time": "2026-09-14T12:00:00.000Z"
}
```

See [slo-scoring.md](slo-scoring.md) for the complete formula and the full risk-flag catalogue.

---

## POST /otl/v2/:chain/:network/refresh

Forces a fresh fetch of all sources, bypassing the 60-second posture cache, and writes the result back into it.

```jsonc
{
  "ok": true,
  "chain": "xrplevm",
  "network": "mainnet",
  "posture_status": "ready",
  "time": "2026-09-14T12:00:00.000Z"
}
```

---

## GET /otl/v2/:chain/:network/diagnostics

Debug endpoint. Tests RPC and Prometheus connectivity directly, independent of the cached posture.

```jsonc
{
  "ok":      true,
  "chain":   "xrplevm",
  "network": "mainnet",

  "rpc": {
    "url":        "(internal)",
    "ok":         true,
    "latency_ms": 162,
    "error":      null
  },

  "prometheus": {
    "url":            "(internal)",
    "label_node":     "job=\"xrplevm_mainnet_validator\"",
    "label_rpc":      "job=\"xrplevm_mainnet_rpc\"",
    "selector_comet": "chain_id=\"xrplevm_1440000-1\"",
    "height_sample":  16162363,
    "available":      true
  },

  "config": {
    "validator_address_prom":   "45A2E210C9F6EFECBC2F46DEA1DECDDD1007AEDF",
    "freshness_threshold_sec":  120
  },

  "time": "2026-09-14T12:00:00.000Z"
}
```

---

## Error Responses

All endpoints return standard error objects:

```jsonc
{
  "ok":      false,
  "chain":   "xrplevm",
  "network": "mainnet",
  "error":   "RPC unavailable: fetch failed",
  "code":    "RPC_ERROR",     // UNKNOWN_CHAIN | NO_TARGET | RPC_ERROR | null
  "time":    "2026-09-14T12:00:00.000Z"
}
```

HTTP status codes: `200` (ok, including a degraded-but-valid posture), `404` (unknown chain), `502` (upstream fetch error).

A degraded source (Prometheus down, on-chain fetch failed) does **not** produce an error response. It still returns `200` with the affected fields `null`/`available: false` and a note in the `notes` array; only a failed RPC fetch or an unrecognised `chain`/`network` produces an error.

---

## Versioning

The API is versioned via the URL path (`/v2/`). Breaking schema changes will increment the version. The `schema_version` field in responses reflects the current schema version. A legacy `/otl/v1/story-aeneid/posture` route remains for one existing consumer; new integrations should use `/v2/`.

---

## Rate Limits

The API is served behind Nginx with standard rate limiting. It is intended for dashboard consumption (one request per 60s per chain, matching the cache TTL). Please do not poll more frequently than that; use `POST /refresh` only when a fresh read is actually needed, not on a regular schedule.
