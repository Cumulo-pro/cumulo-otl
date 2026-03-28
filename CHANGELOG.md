# Changelog

All notable changes to the Cumulo OTL are documented here.

---

## [v2] — 2026-03-28

### Added
- Multi-chain architecture — adding a chain requires only editing `chains.json`
- `last_signed_height` metric replacing misleading `missed_blocks_7d` counter
- Snapshot monitoring — lag vs head height, direct download links
- Activity & Incidents summary from public `incidents.json`
- Node version extraction from `incidents.json` (Story + Story-Geth versions)
- `capabilities` object — tells consumers which data sources are available
- `/otl/chains` endpoint listing all configured chains
- `/otl/v2/:chain/:network/diagnostics` endpoint for data source debugging
- Public documentation page with metric justification and SLO formula
- GitHub repository `cumulo-otl` with full methodology documentation

### Changed
- SLO scoring: `duplicate_vote` and `late_votes` counters are now informational only (no score impact) — they reflect normal Story testnet behaviour
- `block_processing_ms` uses histogram p50 (not sum/count ratio which produced inflated values)
- `block_processing_ms` threshold adjusted for Story's ms-unit exporter (not seconds)
- `blocksync_latest_block_height` hidden when `blocksync_syncing = 0` (value is stale when idle)
- `voting_power_rpc` removed — RPC nodes never have validator voting power

### Fixed
- Prometheus target `storyAened` was pointing to wrong IP — now correctly targets `85.195.116.219:26666`
- Round duration and step duration queries now use correct `_seconds` histogram suffix
- Quorum delay queries changed from histogram to gauge (correct metric type for Story)
- P2P bytes queries use `peer_receive/send_bytes_total` (not `recv/send_bytes_total`)

---

## [v1] — 2026-02-01

### Added
- Initial OTL deployment for Story Aeneid
- CometBFT RPC integration (head height, freshness, catching_up)
- Prometheus integration (availability, peers, validators, consensus timings)
- SLO scoring v1 (posture grade A–F)
- Public dashboard at cumulo.pro/services/story_aeneid
- Public API at otl-api.cumulo.com.es

---

## Roadmap

- [ ] Rolling 30-day availability with persistent storage
- [ ] Incident count integrated into SLO score
- [ ] Monthly PDF SLO reports
- [ ] Celestia chain profile
- [ ] XRPL EVM chain profile
- [ ] Dymension chain profile
