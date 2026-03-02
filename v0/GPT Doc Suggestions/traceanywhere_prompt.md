# TraceAnywhere Claude Code Prompt

**Always output**: PLAN, DIFFS, TESTS, DOCS, OPS NOTES.

### Requirements
- Probing with ICMP→TCP:443→UDP Paris fallback.
- Paris stability: src_port = hash(run_id, hop) % 16384 + 49152.
- Prefer IPv6 when available; show ip_version badges.
- Reduced capability mode on Docker Desktop/Windows/macOS with badge.
- Storage rollups: hot 72h, warm 1m 2w, cold 5m 3m.
- Budgets + allow-lists (OSS: org == local scope).
- OTel spans: trace.run, trace.hop_batch, ingest.persist.
- CI must fail on non-permissive deps.

### Test Profiles
- lite: mocks, default
- kernel: netem, raw sockets
- full: privileged/nightly

### UI
- Downsampling >5k points with event preservation.
- Virtualized hop table beyond 50 rows.
