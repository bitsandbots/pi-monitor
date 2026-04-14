# PiMonitor — API Reference

All responses are JSON. Authentication (if enabled) requires `Authorization: Bearer <token>` on every request.

---

## Node Agent API (`pi_monitor.py`, default port 8585)

### Health

#### `GET /api/ping`
Returns immediately. Use to check if the agent is up.

```json
{"ok": true, "ts": 1713100000.0}
```

---

### System

#### `GET /api/boot`
Hardware identity, detected once at startup.

```json
{
  "model": "Raspberry Pi 5 Model B Rev 1.0",
  "soc": "BCM2712",
  "architecture": "aarch64",
  "revision": "d04170",
  "serial": "10000000abcdef12",
  "hardware": "BCM2835",
  "hostname": "pi5",
  "kernel": "6.6.31+rpt-rpi-2712",
  "is_raspberry_pi": true
}
```

#### `GET /api/status`
Live snapshot — CPU, memory, temperature, uptime. Polled by the dashboard on every refresh cycle.

```json
{
  "cpu": {
    "percent": 12.4,
    "per_core": [10.1, 14.7, 11.2, 13.6],
    "frequency_mhz": 2400
  },
  "memory": {
    "total_mb": 8192,
    "used_mb": 2048,
    "available_mb": 5900,
    "cached_mb": 512,
    "percent": 25.0
  },
  "temperature": 52.3,
  "uptime": {
    "seconds": 345600,
    "human": "4 days, 0:00:00",
    "boot_time": "2026-04-10 07:00:00"
  },
  "hostname": "pi5",
  "timestamp": "14:22:01"
}
```

---

### Storage

#### `GET /api/storage`
Disk mounts and I/O statistics.

```json
[
  {
    "mount": "/",
    "device": "/dev/mmcblk0p2",
    "total": "59G",
    "used": "12G",
    "free": "44G",
    "percent": "21%",
    "reads": 104532,
    "writes": 88201
  }
]
```

---

### Network

#### `GET /api/network`
Per-interface RX/TX byte counters.

```json
{
  "eth0": {"rx_bytes": 1048576000, "tx_bytes": 524288000},
  "wlan0": {"rx_bytes": 2097152, "tx_bytes": 1048576}
}
```

---

### Processes

#### `GET /api/processes`
Top 12 processes by memory usage.

Query params: none.

```json
[
  {
    "pid": 1234,
    "name": "python3",
    "state": "S",
    "memory_kb": 45200,
    "user": "root"
  }
]
```

#### `DELETE /api/processes/<pid>`
Send a signal to a process.

Request body (optional):
```json
{"signal": 9}
```
Default signal: `15` (SIGTERM).

Response:
```json
{"success": true, "pid": 1234, "signal": 15}
```

Error (process not found):
```json
{"success": false, "error": "No such process"}
```

---

### Services

#### `GET /api/services`
Status of all monitored services.

```json
[
  {
    "name": "ssh",
    "active": "active",
    "sub": "running",
    "load": "loaded",
    "enabled": true
  }
]
```

#### `POST /api/services/<name>/<action>`
Control a service. Valid actions: `start`, `stop`, `restart`, `enable`, `disable`.

```bash
curl -X POST http://pi:8585/api/services/nginx/restart
```

Response:
```json
{"success": true, "service": "nginx", "action": "restart", "output": ""}
```

#### `GET /api/services/config`
Return the current persisted service list.

```json
{"services": ["ssh", "nginx", "docker"]}
```

#### `POST /api/services/config`
Add a service to the monitored list.

```json
{"name": "myapp"}
```

#### `DELETE /api/services/config/<name>`
Remove a service from the list.

#### `PUT /api/services/config/<name>`
Rename a service entry.

```json
{"name": "myapp-v2"}
```

---

### Power

#### `POST /api/power/<action>`
Valid actions: `reboot`, `shutdown`.

```bash
curl -X POST http://pi:8585/api/power/reboot
```

Response:
```json
{"success": true, "action": "reboot"}
```

---

### Logs

#### `GET /api/logs`
Retrieve the in-memory event log (up to 200 entries, ring buffer).

Query params: `?limit=50` (default 100).

```json
[
  {"ts": "14:22:01", "msg": "PiMonitor ready.", "level": "success"},
  {"ts": "14:23:10", "msg": "Service restarted: nginx", "level": "info"}
]
```

Log levels: `info`, `success`, `warning`, `error`.

---

## Fleet Hub API (`hub/pi_monitor_hub.py`, default port 8686)

### Health

#### `GET /api/ping`
```json
{"ok": true, "hub": true, "ts": 1713100000.0}
```

---

### Fleet

#### `GET /api/fleet`
Cached snapshot of all registered nodes (updated every poll interval).

```json
{
  "nodes": [
    {
      "id": "192-168-1-42-8585",
      "host": "192.168.1.42",
      "port": 8585,
      "label": "Pi-Living-Room",
      "online": true,
      "last_seen": "14:22:05",
      "status": { /* same shape as /api/status */ }
    }
  ],
  "total": 1,
  "online": 1
}
```

---

### Node Registry

#### `POST /api/nodes`
Register a new node.

```json
{"host": "192.168.1.42", "port": 8585, "label": "Pi-A", "token": ""}
```

#### `DELETE /api/nodes/<nid>`
Remove a node from the registry.

#### `PUT /api/nodes/<nid>`
Update a node's label or token.

---

### Discovery

#### `POST /api/discover`
Scan the local subnet for PiMonitor agents. Probes `<ip>:PIHUB_DISCOVERY_PORT/api/ping` in parallel.

```json
{"found": 3, "nodes": ["192.168.1.42", "192.168.1.55", "192.168.1.71"]}
```

---

### Node Proxy Routes

All proxy routes forward to the corresponding node's API. Response shape matches the node API docs above.

| Method | Hub Route | Forwards To |
|--------|-----------|------------|
| `GET` | `/api/nodes/<nid>/status` | `/api/status` |
| `GET` | `/api/nodes/<nid>/boot` | `/api/boot` |
| `GET` | `/api/nodes/<nid>/services` | `/api/services` |
| `POST` | `/api/nodes/<nid>/services/<svc>/<action>` | `/api/services/<svc>/<action>` |
| `GET` | `/api/nodes/<nid>/storage` | `/api/storage` |
| `GET` | `/api/nodes/<nid>/processes` | `/api/processes` |
| `GET` | `/api/nodes/<nid>/network` | `/api/network` |
| `GET` | `/api/nodes/<nid>/logs` | `/api/logs` |
| `POST` | `/api/nodes/<nid>/power/<action>` | `/api/power/<action>` |

If a node is unreachable, proxy routes return `502` with `{"error": "unreachable"}`.

---

## Error Responses

| Status | Meaning |
|--------|---------|
| `400` | Bad request — invalid parameters |
| `401` | Unauthorized — missing or invalid token |
| `404` | Not found — service or node doesn't exist |
| `409` | Conflict — duplicate service name |
| `502` | Bad gateway — hub cannot reach the node |
