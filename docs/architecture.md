# RPiMonitor — Architecture

## High-Level Design

```
Browser
  │
  │  HTTP / polling (every 2s)
  ▼
┌─────────────────────────────────────────────┐
│  Node Agent  (rpi_monitor.py : 8585)         │
│                                             │
│  Flask API ──► /proc, /sys, systemctl       │
│  In-memory ring buffer (200 events)         │
│  services.json  (persisted service list)    │
└─────────────────────────────────────────────┘

       ─ ─ ─ OR, with Hub ─ ─ ─

Browser
  │
  ▼
┌─────────────────────────────────────────────┐
│  Fleet Hub  (rpi_monitor_hub.py : 8686)      │
│                                             │
│  Node registry  (hub_nodes.json)            │
│  Background poller  (ThreadPoolExecutor)    │
│    └─► polls each node every 5s             │
│  Proxy routes  ──► Node Agent APIs          │
└──────────────┬──────────────────────────────┘
               │  HTTP (per-node)
      ┌────────┴────────┐
      ▼                 ▼
 Node :8585       Node :8585
 (Pi A)           (Pi B)
```

## Node Agent — Data Flow

1. Browser loads `index.html` (served by Flask).
2. Dashboard JS polls `/api/status` every `PIMONITOR_REFRESH` seconds (default 2s).
3. `/api/status` calls the metric functions:
   - `get_cpu_usage()` — reads `/proc/stat` and computes usage from the delta since the previous poll call (stored in `_cpu_prev`).
   - `get_cpu_temperature()` — reads `/sys/class/thermal/thermal_zone0/temp`.
   - `get_memory()` — parses `/proc/meminfo`.
   - `get_uptime()` — reads `/proc/uptime`.
4. On-demand endpoints (`/api/storage`, `/api/network`, `/api/processes`, `/api/services-with-ports`) are fetched only when the dashboard tab is active.
5. `/api/services-with-ports` merges:
   - `get_services()` — systemd service status via `systemctl is-active/is-enabled`
   - `get_open_ports()` — listening ports via `ss -tuln`
   - `_SERVICE_PORTS` mapping — common service→port associations
6. `/api/logs?system=true` merges:
   - In-memory event log (ring buffer)
   - `get_system_errors()` — journalctl error entries
7. Mutating actions (service control, process kill, power) require a POST and pass through `require_auth` middleware if `PIMONITOR_TOKEN` is set.
8. All mutations are written to the in-memory event log (ring buffer, 200 entries, accessible via `/api/logs`).

## Hub — Data Flow

1. At startup, `_load_nodes()` restores the node registry from `hub_nodes.json`.
2. `_start_poller()` launches a background thread that calls `_poll_node()` for every registered node on `PIHUB_POLL_INTERVAL` (default 5s) using a `ThreadPoolExecutor(max_workers=min(node_count, 8))`.
3. Poll results are cached in the in-memory `_nodes` dict under a `_lock`.
4. Browser requests `/api/fleet` — returns the cached snapshot for all nodes instantly.
5. Proxy routes (e.g. `/api/nodes/<nid>/services`) forward to the corresponding node's API via `_fetch_node()` with per-request timeouts.
6. Network discovery (`/api/discover`) scans the local subnet in parallel using `ThreadPoolExecutor` — it probes `<ip>:PIHUB_DISCOVERY_PORT/api/ping` for every host.

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `/proc`/`/sys` reads over `psutil` | No extra dependency; works on stock Pi OS |
| In-memory ring buffer for logs | Avoids disk I/O; survives service restarts (ephemeral by design) |
| Per-device `services.json` | Survives restarts; decoupled from env vars after first run |
| Hub caches polls, serves stale data | Dashboard stays responsive even when a node is slow or unreachable |
| `require_auth` decorator | Opt-in; safe to run token-free on a trusted LAN |
