# TraceAnywhere

TraceAnywhere is a Docker-native network diagnostic and **teaching** tool. It runs probes (ICMP, TCP:443, UDP Paris) with platform-aware fallbacks, streams results via WebSockets, and renders a UI designed to **explain networking concepts**.

## Highlights
- Works across Linux, Docker Desktop (Windows/macOS), and servers with reduced capability mode.
- Protocol intelligence with ICMP→TCP:443→UDP Paris fallback.
- Storage with rollups: hot (raw), warm (1-min), cold (5-min).
- Budgets, allow-lists, non-root probes with `CAP_NET_RAW` only when required.
- Observability: OpenTelemetry spans emitted by the tool itself.
- Education: docs, tutorials, concepts, route diffs, badges.

## Quick Start
```bash
docker compose up --build
```

Visit `http://localhost:8080` for UI.
