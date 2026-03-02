# TraceAnywhere — Project Specification

**Version:** 2.0 (clean start)
**Date:** 2026-03-02
**Status:** Pre-implementation

---

## 1. What Is TraceAnywhere?

TraceAnywhere is a Docker-native network diagnostic tool. It runs continuous ICMP traceroutes from the host where it's deployed and presents the results in a real-time web UI — similar in spirit to Ping Plotter Pro, but open-source, self-hosted, and server-side.

**The core question it answers:** "Where between point A and point B is the network broken or degraded?"

**Target audience:** Network admins, homelabbers, sysadmins, developers debugging connectivity. Anyone who would run `mtr` but wants persistent history and a visual timeline.

---

## 2. How It Works (User Perspective)

1. User runs `docker compose up` on any machine with Docker.
2. Opens `http://localhost:8080` in a browser.
3. Types a hostname or IP into the target bar. Hits Trace.
4. A hop table appears showing every router between the Docker host and the target, with latency and packet loss per hop.
5. Each hop row has a sparkline showing latency over time. Spikes and loss are immediately visible.
6. Clicking a hop row shows a detailed timeline chart for that hop.
7. The trace runs continuously (default every 2 seconds). The user can leave it running for hours or days to catch intermittent issues.
8. Multiple traces can run simultaneously in separate tabs.
9. Traces are saved automatically and can be exported as JSON for sharing or later analysis.

---

## 3. Architecture

### 3.1 Overview

Single Docker Compose stack with two services:

```
┌─────────────────────────────────────────────┐
│  Docker Compose                             │
│                                             │
│  ┌──────────────┐    ┌───────────────────┐  │
│  │   Backend    │◄──►│    Frontend       │  │
│  │  (FastAPI)   │    │  (Static SPA)     │  │
│  │              │    │  served by nginx   │  │
│  │  - REST API  │    │  or backend       │  │
│  │  - WebSocket │    │                   │  │
│  │  - mtr proc  │    │                   │  │
│  │  - SQLite    │    │                   │  │
│  └──────────────┘    └───────────────────┘  │
│        │                                    │
│        ▼                                    │
│  ┌──────────────┐                           │
│  │  mtr binary  │  (ICMP probes)            │
│  │  CAP_NET_RAW │                           │
│  └──────────────┘                           │
└─────────────────────────────────────────────┘
```

For v1, the backend can also serve the frontend static files directly (no need for a separate nginx container). One container is fine to start.

### 3.2 Backend (Python / FastAPI)

- **REST API** for managing traces (create, stop, list, get history, export).
- **WebSocket endpoint** for real-time streaming of probe results to the UI.
- **Probe manager** that spawns and manages `mtr` subprocesses.
- **SQLite database** for persisting trace history.
- Runs as non-root user inside the container, with `CAP_NET_RAW` granted to `mtr`.

### 3.3 Frontend (JavaScript SPA)

- Single-page application.
- Technology: vanilla JS with a lightweight charting library, or React if complexity warrants it. Decision deferred to implementation.
- Connects to backend via WebSocket for real-time updates.
- Responsive layout but primarily designed for desktop/tablet use (this is a diagnostic tool, not a phone app).

### 3.4 Probe Engine

- Wraps `mtr --json` as a subprocess.
- Clean internal interface (`ProbeEngine` abstraction) so the engine can be swapped later without touching the rest of the app.
- Parses `mtr` JSON output into a normalized internal data model.
- Default probe interval: 2 seconds (configurable per trace).

### 3.5 Storage

- **SQLite** — single file, zero configuration, ships in the container.
- Database file lives on a Docker volume so data survives container restarts.
- Schema designed for efficient time-range queries (traces over a given period).

### 3.6 Data Model

```
Trace
  id              UUID
  target          string (hostname or IP)
  display_name    string (optional user label)
  created_at      timestamp
  status          enum: running | paused | stopped
  probe_interval  int (seconds, default 2)
  config          JSON (future: probe type, etc.)

TraceResult (one per probe cycle)
  id              UUID
  trace_id        FK → Trace
  timestamp       timestamp
  hop_count       int

HopResult (one per hop per probe cycle)
  id              UUID
  result_id       FK → TraceResult
  hop_number      int
  ip_address      string (nullable — for * * * hops)
  hostname        string (nullable — reverse DNS)
  latency_ms      float (nullable — for lost packets)
  loss_percent    float
  sent            int
  received        int
```

### 3.7 Export Format

JSON. One file per trace. Structure:

```json
{
  "traceanywhere_version": "1.0",
  "export_version": "1",
  "trace": {
    "id": "...",
    "target": "google.com",
    "created_at": "2026-03-02T10:00:00Z",
    "probe_interval_s": 2,
    "probe_type": "icmp"
  },
  "results": [
    {
      "timestamp": "2026-03-02T10:00:02Z",
      "hops": [
        {
          "hop": 1,
          "ip": "96.43.131.201",
          "hostname": "gateway.dc.example.com",
          "latency_ms": 0.4,
          "loss_percent": 0.0
        }
      ]
    }
  ]
}
```

No existing standard exists for traceroute history. This is our format. Keep it simple, human-readable, and easy to parse.

---

## 4. User Interface

### 4.1 Layout

```
┌──────────────────────────────────────────────────────────┐
│  TraceAnywhere                          [+ New Trace]    │
├──────────────────────────────────────────────────────────┤
│  [ Tab: google.com ▶ ] [ Tab: 1.1.1.1 ▶ ] [ + ]        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Target: [ google.com          ] [Trace] [⏸ Pause][⏹]   │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │         Timeline Chart (selected hop)              │  │
│  │                                                    │  │
│  │   ms                                               │  │
│  │  40│          ╱╲                                    │  │
│  │  20│   ──────╱  ╲──────────────────                │  │
│  │   0│                                               │  │
│  │    └──────────────────────────────── time           │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Hop │ Host              │ IP          │ Avg  │ ... │  │
│  │  1  │ gateway           │ 96.43.131.1 │ 0.4  │ ▁▁▂ │  │
│  │  2  │ core-rtr.dc       │ 10.0.0.1    │ 2.1  │ ▁▁▁ │  │
│  │  3  │ * * *             │ —           │ —    │     │  │
│  │  4  │ chi.backbone.net  │ 12.122.2.5  │ 18.3 │ ▁▃▁ │  │
│  │  5  │ target.google.com │ 142.250.x.x │ 22.1 │ ▁▁▁ │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Status: Running | Probes: 1,247 | Duration: 41m        │
│  [Export JSON] [Clear History]                           │
└──────────────────────────────────────────────────────────┘
```

### 4.2 Hop Table Columns

| Column | Description |
|--------|-------------|
| Hop | Hop number (1, 2, 3...) |
| Host | Reverse DNS hostname (or "* * *" for non-responding hops) |
| IP | IP address |
| Avg | Average latency (ms) over the trace lifetime or selected window |
| Min | Minimum latency |
| Max | Maximum latency |
| Loss | Packet loss percentage |
| Sparkline | Mini latency chart showing trend over time |

### 4.3 Timeline Chart

- X axis: time
- Y axis: latency (ms)
- Shows data for the currently selected hop (click a row to select)
- Packet loss events marked as red dots or gaps
- Zoomable/pannable for long-running traces
- Auto-scrolls to follow live data, stops auto-scroll if user pans back

### 4.4 Interactions

- Click hop row → select it → timeline chart updates
- Tabs for multiple simultaneous traces
- Pause/Resume a trace without losing history
- Stop a trace → it becomes read-only history
- Export button downloads JSON file

---

## 5. Docker Packaging

### 5.1 docker-compose.yml

```yaml
services:
  traceanywhere:
    build: .
    ports:
      - "8080:8080"
    cap_add:
      - NET_RAW
    volumes:
      - trace-data:/app/data
    restart: unless-stopped

volumes:
  trace-data:
```

### 5.2 Dockerfile

- Base: `python:3.12-slim`
- Install `mtr` via apt
- Install Python dependencies
- Copy app code
- Non-root user with `CAP_NET_RAW` on the `mtr` binary via `setcap`
- Expose port 8080

### 5.3 One-command startup

```bash
git clone https://github.com/bob7123/trace-anywhere.git
cd trace-anywhere
docker compose up --build
# Open http://localhost:8080
```

---

## 6. API Endpoints

### 6.1 REST

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/traces` | Start a new trace (body: `{target, probe_interval}`) |
| GET | `/api/traces` | List all traces (active + history) |
| GET | `/api/traces/{id}` | Get trace details + recent results |
| PATCH | `/api/traces/{id}` | Pause/resume/stop a trace |
| DELETE | `/api/traces/{id}` | Delete a trace and its history |
| GET | `/api/traces/{id}/export` | Download trace as JSON file |
| GET | `/api/traces/{id}/results?from=&to=` | Query results in time range |

### 6.2 WebSocket

`ws://localhost:8080/ws/traces/{id}`

Server pushes each new probe result as JSON as it arrives. Client connects when a trace tab is active, disconnects when switching away.

---

## 7. Implementation Phases

### Phase 1 — MVP (Get it working)

- [ ] Project scaffolding: Dockerfile, docker-compose.yml, FastAPI app skeleton
- [ ] Probe engine: wrap `mtr --json`, parse output, normalize to data model
- [ ] SQLite schema and basic CRUD for traces/results
- [ ] REST API: start trace, list traces, stop trace, get results
- [ ] WebSocket: stream live results to client
- [ ] Frontend: target bar, hop table (no sparklines yet), basic timeline chart
- [ ] Export as JSON
- [ ] Single container, `docker compose up` works end to end

**Exit criteria:** You can start a trace, see hops updating live, stop it, and export the results.

### Phase 2 — Make it good

- [ ] Multiple simultaneous traces with tabs
- [ ] Sparklines in the hop table
- [ ] Pause/resume traces
- [ ] Timeline chart zoom/pan
- [ ] Saved trace history (browse and reload past traces)
- [ ] Responsive polish, loading states, error handling
- [ ] Configurable probe interval per trace

**Exit criteria:** A non-technical person could use this without help.

### Phase 3 — Additional probe types

Future probe types, each slotting into the `ProbeEngine` interface:

| Probe Type | What It Answers |
|------------|-----------------|
| **TCP SYN (443/80)** | "Is the ICMP loss real, or is a firewall just dropping pings?" |
| **HTTP/HTTPS check** | "Is the target server responding, and how fast?" (latency + status code) |
| **DNS resolution timing** | "Is slow DNS the hidden culprit?" |
| **UDP traceroute** | "Is ECMP load balancing routing UDP differently than ICMP?" |

Each adds a probe type selector in the UI. Results share the same storage and export format.

### Phase 4 — Portfolio polish

- [ ] Public demo instance
- [ ] Proper README with screenshots/GIF
- [ ] Logo and branding
- [ ] GitHub Actions CI (build, test, push to Docker Hub)
- [ ] Docker Hub listing: `docker pull traceanywhere/traceanywhere`
- [ ] Landing page / project site
- [ ] Comparison table vs Ping Plotter, WinMTR, etc.
- [ ] Blog post / write-up

---

## 8. Technology Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Backend language | Python 3.12 | Fast to develop, great async support, FastAPI is mature |
| Backend framework | FastAPI | Async, WebSocket support, auto-generated API docs |
| Probe tool | mtr | Battle-tested, JSON output, does exactly what we need |
| Database | SQLite | Zero config, single file, Docker-volume friendly |
| Frontend | TBD (vanilla JS or React) | Decide during Phase 1 based on complexity |
| Charting | TBD (Chart.js, uPlot, or Lightweight Charts) | Need real-time append performance |
| Container base | python:3.12-slim | Small image, official, well-maintained |

---

## 9. Testing & Code Quality

### 9.1 Design for Testability

Every component must be testable in isolation. This means:

- **Dependency injection everywhere.** The probe engine, database, and WebSocket manager are passed in, not imported as globals. Tests swap in fakes/mocks without monkeypatching.
- **ProbeEngine is an interface.** A `FakeProbeEngine` ships with the test suite that returns canned `mtr` output — no real network calls needed for 90% of tests.
- **Database layer is abstract.** Tests use an in-memory SQLite instance, created and destroyed per test. No shared state between tests.
- **WebSocket testing via HTTPX/TestClient.** FastAPI's test client supports WebSocket — use it. No real server needed.
- **No hidden state.** If a function's behavior depends on something, that something is a parameter, not a module-level variable.

### 9.2 Graceful Failure

AI-generated code tends to test the happy path. This project must explicitly handle failure modes.

**Principle: Every error response must correctly attribute fault to the right layer — caller, system, or dependency — so that the entity receiving it takes the right corrective action.**

There are three error categories, and they must never be confused:

1. **Caller error** → tell them what to fix. ("Hostname could not be resolved — check the target.")
2. **Internal system problem** → say so. ("Unable to store results — disk may be full.")
3. **External dependency failure** → name it. ("Probe engine error — mtr process crashed.")

Never report a category 2 or 3 error as category 1. The caller — whether human, API client, or automated agent — must be able to determine whether the failure is something they can fix, something they should retry, or something that requires intervention elsewhere.

Specific failure modes to handle:

- **`mtr` process crashes or hangs:** Probe manager must detect subprocess death (non-zero exit, no output within timeout) and report the failure to the UI — not silently stop updating. Implement a per-probe timeout.
- **`mtr` not found in container:** Fail at startup with a clear error message, not a buried traceback.
- **Invalid target (unresolvable hostname):** Return a meaningful error to the UI, not a 500.
- **Database full / disk full:** Catch write failures, surface them in the API response, don't crash the server.
- **WebSocket client disconnects mid-stream:** Clean up without exceptions. Don't leak probe subprocesses.
- **Concurrent trace limit:** If someone starts 50 traces, don't fork-bomb the container. Enforce a sane max (e.g., 10) and return an error for the 11th.
- **Malformed mtr output:** Parse defensively. If `mtr --json` returns unexpected structure, log it and skip that cycle — don't crash the trace.
- **Container restart with running traces:** On startup, mark any previously-running traces as `stopped`. Don't try to resume them with stale state.

**Rule:** Every error the user can encounter must produce a message they can understand and act on. No blank screens, no spinner-forever, no silent failures.

### 9.3 Test Plan

**Unit tests** (run without Docker, no network, no filesystem):

| Area | What's tested |
|------|---------------|
| mtr output parser | Valid JSON, malformed JSON, empty output, timeout, unexpected fields |
| Data model | Trace/TraceResult/HopResult creation, validation, edge cases (null IPs, zero latency, 100% loss) |
| Probe manager | Start/stop lifecycle, max concurrent limit, subprocess failure handling, timeout behavior |
| Export builder | Correct JSON structure, empty trace export, large trace export |
| API request validation | Missing fields, invalid targets, bad UUIDs, out-of-range intervals |

**Integration tests** (run in Docker, but no real network probes):

| Area | What's tested |
|------|---------------|
| REST API end-to-end | Create trace → get results → export → delete, using FakeProbeEngine |
| WebSocket streaming | Connect → receive updates → disconnect cleanly, verify no leaked resources |
| Database persistence | Create trace, restart app (simulate container restart), verify data survived |
| Startup checks | Missing `mtr` binary, unwritable data directory, port already in use |

**Smoke tests** (run in Docker with real network):

| Area | What's tested |
|------|---------------|
| Real trace | `docker compose up`, trace `8.8.8.8`, verify hops appear in UI within 10 seconds |
| Export round-trip | Run a trace, export JSON, verify the file is valid and contains expected fields |

### 9.4 Test Execution

```bash
# Unit tests (fast, no Docker needed)
pytest tests/unit/ -v

# Integration tests (needs Docker)
pytest tests/integration/ -v

# Smoke tests (needs Docker + network)
pytest tests/smoke/ -v

# All tests
pytest -v
```

Tests run in CI (Phase 4) and must pass before merge. During Phase 1-3, run locally before committing.

### 9.5 What "Done" Means for Each Feature

No feature is complete unless:

1. The happy path works
2. At least one failure mode is tested (what happens when it breaks?)
3. The UI shows a meaningful message for every error state
4. The test can run without network access (except smoke tests)

---

## 10. Non-Goals (Explicitly Out of Scope)

- User accounts or authentication (it's a tool, not a SaaS)
- Mobile-first design
- Windows/Mac native client
- Packet capture or deep packet inspection
- Network topology mapping
- Cloud-hosted multi-tenant service
- OpenTelemetry integration
- SBOM validation in CI

These may be revisited later but are not part of any current phase.

---

## 11. Reference: v1 Documentation

The original GPT-generated documentation (architecture, security model, contributing guide, tutorials, etc.) is preserved in `/v1/GPT Doc Suggestions/` for reference. Those documents cast a wide net and contain ideas that may be drawn from in future phases, but they do not govern the current implementation.
