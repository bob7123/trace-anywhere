# Architecture Overview

- **Agent** performs probes with ICMP, TCP, UDP Paris fallback.
- **API (FastAPI)** schedules probes, ingests results.
- **Web UI** streams updates, visualizes hops, diffs, badges.
- **Storage**: SQLite default, Timescale plugin optional.

### Protocol Intelligence
1. ICMP (if CAP_NET_RAW available)
2. TCP SYN 443
3. UDP Paris with stable flow-ID `hash(run_id, hop) % 16384 + 49152`
Lockout period prevents flapping.

### Data Model
- TraceRun with hops, route changes, method transitions.
- HopResult with method, ip_version, latency, confidence intervals.

### Observability
- OTel spans: trace.run, trace.hop_batch, ingest.persist.
- Attributes: method, ip_version, route_change, n_samples.

### UI Performance
- Downsample >5k points (LTTB) but preserve events.
- Virtualize hop table beyond 50 rows.
