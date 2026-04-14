# PiMonitor — Tech Stack

## Runtime

| Layer | Technology | Version | Notes |
|-------|-----------|---------|-------|
| OS | Raspberry Pi OS (Debian Bookworm) | 64-bit | Also works on any Linux |
| Python | CPython | 3.11+ | Ships with Pi OS; 3.13 used in dev |
| Web framework | Flask | ≥ 3.0.0 | Node agent and hub both use Flask |
| HTTP client | Requests | ≥ 2.31 | Hub only — proxies to node APIs |

## Frontend

| Technology | Role |
|-----------|------|
| Vanilla JS (ES2020) | Dashboard polling, DOM updates, tab management |
| Bootstrap 5 | Layout, modals, utility classes |
| Google Fonts (Exo 2, Plus Jakarta Sans, IBM Plex Mono) | Typography |
| CSS custom properties | Brand theming (navy/silver/blue/orange) |

No build step — the dashboard is a single self-contained `index.html`.

## System Interfaces

| Interface | Used By | Data |
|-----------|---------|------|
| `/proc/stat` | `get_cpu_usage()` | Per-core CPU jiffies (delta method) |
| `/proc/meminfo` | `get_memory()` | Total, free, available, cached RAM |
| `/proc/uptime` | `get_uptime()` | Seconds since boot |
| `/proc/net/dev` | `get_network()` | Interface RX/TX bytes |
| `/proc/<pid>/status` | `get_top_processes()` | Process name, PID, state, memory |
| `/sys/class/thermal/thermal_zone0/temp` | `get_cpu_temperature()` | CPU temp in millidegrees |
| `/sys/block/*/stat` | `get_storage()` | Disk I/O read/write sectors |
| `df -h` (subprocess) | `get_storage()` | Mount points, used/free space |
| `systemctl` (subprocess) | `get_services()`, `control_service()` | Service status, start/stop/restart/enable/disable |
| `sudo reboot / shutdown` (subprocess) | `system_power()` | Power actions |

## Process Management (Node)

| Tool | Purpose |
|------|---------|
| `os.kill(pid, sig)` | Send signal to process (default SIGTERM=15) |
| `threading.Lock` | Guards CPU stat differential and event log |

## Concurrency (Hub)

| Tool | Purpose |
|------|---------|
| `threading.Thread` | Background poller loop |
| `concurrent.futures.ThreadPoolExecutor` | Parallel node polling (max 10 workers) |
| `threading.Lock` | Guards node registry dict |

## Persistence

| File | Location | Contents |
|------|----------|---------|
| `services.json` | `PIMONITOR_SERVICES_FILE` (default: project root) | Ordered list of monitored service names |
| `hub_nodes.json` | `PIHUB_NODES_FILE` (default: `hub/`) | Node registry (host, port, label, token) |

## Process Supervisor

| Tool | Config File |
|------|------------|
| systemd | `pi-monitor.service` / `hub/pi-monitor-hub.service` |
