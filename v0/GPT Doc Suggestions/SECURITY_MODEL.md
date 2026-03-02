# Security Model

## Friendly Overview
- Probes run non-root. ICMP uses CAP_NET_RAW only when needed.
- No payload capture, only metadata (latency, IP, ASN).
- Allow-lists required. Budgets enforced (30 traces/min, 200 pps default).
- Reduced capability mode for Docker Desktop auto-badges TCP-only.
- CI fails build if dependency has non-permissive license.

## Threat Model (STRIDE-lite)
- Spoofing: Agents authenticate via rotating API keys (mTLS optional).
- Tampering: Results schema-validated. Audit logs of policy changes.
- Info disclosure: No packet payload storage. Target hashes used for analytics.
- DoS: Rate limits, jittered schedules, bounded agent queue.
- Elevation: Containers run non-root, CAP_NET_RAW only for probes.

## Checklist
- [ ] Run non-root containers
- [ ] Apply allow-lists, budgets
- [ ] Enable OTel export
- [ ] Review SBOM in CI
