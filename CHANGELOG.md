# Changelog

All notable changes to the Cumulo OTL are documented here. The API's own `schema_version` field has stayed `v2` since the March 2026 rewrite and remains `v2` through the September 2026 batch below — despite its size, it does not break the `v2` response contract. Entries are grouped by when the work shipped; a section only gets its own `[vN]` heading when the schema itself is actually bumped.

---

## September 2026 (schema `v2`, unchanged): XRPL EVM Mainnet and Testnet, SLO model rewrite, governance and track record

The OTL's pilot chain (Story Aeneid) was retired from active development in favor of XRPL EVM as the primary deployment. This batch of changes also rewrote the scoring model from the ground up: see [docs/slo-scoring.md](docs/slo-scoring.md) for the full rationale.

### Added
- XRPL EVM Mainnet: full production deployment (`chains.json` entry, dashboard, API), live since early September 2026.
- XRPL EVM Testnet: full production deployment, live since 2026-09-10.
- **On-chain validator record** (`on_chain`, `CHAIN`-labeled): signing uptime, `jail_margin`, bonded/jailed/tombstoned status, voting weight, and commission, read directly from the chain's staking and slashing modules via Cumulo's block explorer. Re-derivable by anyone from any LCD.
- **SLO model rewrite**: `posture_score`/`posture_grade` is now built entirely from signals specific to Cumulo's own infrastructure (on-chain jail margin, live signing gap, RPC/own-node availability against a published target) instead of chain-wide consensus health. Byzantine/missing validators and round duration are still published, but only as context, never scored. `consensus_health_score` is kept as a deprecated alias of the new `posture_score` field.
- **Integrity cap**: a historical slashing/double-sign event on record, from `incidents.json`, permanently caps the grade at C.
- **Upgrade discipline**: a logged mandatory upgrade applied outside its announced window permanently caps the grade at B. Recognizes the same `upgrade`/`update`/`security_upgrade`/`infrastructure_update` family the public dashboard has grouped for a while, not only the literal string `upgrade`.
- **Governance participation and disclosure**: `governance.votes_eligible`/`votes_cast`/`participation_pct` (scored against a per-chain target, default 90%) and `votes_disclosed`/`disclosure_pct`/`disclosure_gaps` (informational only, never scored). See methodology.md § 5 for the hybrid live-chain/tracker sourcing this required.
- **Rolling 30d/90d/lifetime availability** (`reliability.long_term`): a daily rollup persisted independently of Prometheus' own retention, read back and averaged per window; each window is `null` until enough closed-day history exists for it, never extrapolated.
- `on_chain_unconfirmed_grade_capped`, `signing_gap_outage`, `signing_gap`, `rpc_availability_far_below_target`, `rpc_availability_below_target`, `own_node_stalled`, `signing_at_risk`, `jailed`/`not_bonded`/`signing_critical`, `historical_slashing_event`, `tombstoned`, `delayed_upgrade_on_record`, `governance_participation_far_below_target`, `governance_participation_below_target`, and `governance_disclosure_gap` risk flags.
- `GRADE_TOMBSTONED_CEIL` and the other grade-band ceiling constants named explicitly in the scoring code.
- 44 unit tests over the pure scoring/governance functions, plus an end-to-end test harness (a real server instance against a mocked RPC/explorer/incidents.json/governance-LCD) that exercises the full posture/slo contract with real HTTP traffic.

### Changed
- `data_window.availability` in `/slo` now reflects the real Prometheus coverage (`reliability.availability_coverage_days`) instead of a fixed `"7d"` label regardless of how much scrape history actually exists.
- `validator_address`/`validator_address_prom` now resolve with a fixed precedence (chain config, then Prometheus-detected, then RPC), rather than the RPC target's own `/status` address; a public RPC endpoint is commonly a non-signing sentry node, and the old behavior could silently report that node's address instead of the actual signing validator's.
- `chains.json`: `validator_since` now carries a precise timestamp, not just a date, so a proposal that closed hours before a validator's admission proposal was approved on the same calendar day is no longer counted as an eligible vote it was structurally impossible to cast.

### Fixed
- **2026-09-13**: `computeDisclosedProposalIds()` was systematically undercounting governance disclosures, especially on XRPL EVM Mainnet. Root cause: `incidents.json` had changed `incident_type` convention mid-flight (`governance_vote` before proposal #27, `governance_proposal` from #27 on, only the newer value was recognized) and the `id` slug format had drifted across at least four shapes. Fixed by recognizing both `incident_type` values and extracting the proposal number from the (far more consistent) `url` field first, falling back to the `id` only when the URL doesn't fit. `votes_eligible`/`votes_cast` on XRPL EVM Mainnet went from 9/19 (47.4%) to 17/19 (89.5%) on deploy; a missing tracker entry for Proposal #24 was found and added in the same pass.
- **2026-09-13**: `fetchGovernanceParticipation()` reported 0% participation for a validator with a real 100% historical participation record. Root cause: Cosmos SDK's `x/gov` module (inherited by XRPL EVM) prunes individual vote records once a proposal has been tallied, only the aggregate result survives on-chain, so a live per-voter LCD query against any closed proposal always came back empty, for every validator, not just Cumulo. Redesigned to a hybrid source: live LCD verification for proposals still open, `incidents.json` for closed ones (the only place the fact still exists). As a stated consequence, `disclosure_pct` now coincides with `participation_pct` by construction for closed proposals; the two remain independently meaningful only for proposals still open.
- **2026-09-13**: two pre-existing null-coercion bugs in `buildSlo()`, found during end-to-end testing of the governance changes above (not introduced by them): `Number(null)` evaluates to `0`, not `NaN`, so a `last_signed_height` or `rpc_availability_7d_pct` that was `null` because Prometheus hadn't scraped that metric yet (a normal, soft-unavailable state) was silently read as a real zero, always firing `signing_gap_outage` (cap at D) or `rpc_availability_far_below_target` (cap at C) for any chain missing that one Prometheus metric, regardless of actual health. Fixed by checking `!= null` before the numeric conversion instead of after.
- An earlier gap in the same on-chain confirmation logic: the healthy branch of the on-chain fetch was missing an explicit `available: true`, so `on_chain.available` read as `undefined`, and `on_chain_unconfirmed_grade_capped` fired unconditionally, even when the on-chain record was fetched successfully and the validator was perfectly healthy.
- `blocksync_latest_block_height` hidden when `blocksync_syncing = 0` (value is stale when idle).
- `voting_power_rpc` removed: RPC nodes never have validator voting power.

---

## [v2] - 2026-03-28

### Added
- Multi-chain architecture: adding a chain requires only editing `chains.json`.
- `last_signed_height` metric, replacing the misleading `missed_blocks_7d` counter.
- Snapshot monitoring: lag vs head height, direct download links.
- Activity & Incidents summary from public `incidents.json`.
- Node version extraction from `incidents.json`.
- `capabilities` object: tells consumers which data sources are available.
- `/otl/chains` endpoint listing all configured chains.
- `/otl/v2/:chain/:network/diagnostics` endpoint for data source debugging.
- Public documentation page with metric justification and SLO formula.
- GitHub repository `cumulo-otl` with full methodology documentation.

### Changed
- SLO scoring: `duplicate_vote` and `late_votes` counters became informational only (no score impact).
- `block_processing_ms` uses histogram p50 (not the sum/count ratio, which produced inflated values).
- `blocksync_latest_block_height` hidden when idle.

### Fixed
- A Prometheus scrape target was pointing at the wrong IP.
- Round duration and step duration queries now use the correct `_seconds` histogram suffix.
- Quorum delay queries changed from histogram to gauge (the correct metric type).
- P2P bytes queries use `peer_receive/send_bytes_total` (not `recv/send_bytes_total`).

---

## [v1] - 2026-02-01

### Added
- Initial OTL deployment, on the pilot chain used to validate the architecture before the multi-chain rewrite.
- CometBFT RPC integration (head height, freshness, catching_up).
- Prometheus integration (availability, peers, validators, consensus timings).
- SLO scoring v1 (posture grade A to F).
- Public dashboard and public API at otl-api.cumulo.com.es.

---

## Roadmap

See the [Roadmap section in README.md](README.md#roadmap) for current status: it is kept in one place to avoid this file and the README drifting apart on what is actually shipped.
