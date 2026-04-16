# Migration Guide: pi-monitor → rpi-monitor

This guide covers upgrading from pi-monitor v2.0.0 to rpi-monitor v2.0.0.

## What Changed

| Category | Old Value | New Value |
|----------|-----------|-----------|
| File name | `pi_monitor.py` | `rpi_monitor.py` |
| Service name | `pi-monitor` | `rpi-monitor` |
| Hub service | `pi-monitor-hub` | `rpi-monitor-hub` |
| Install directory | `/opt/pi-monitor` | `/opt/rpi-monitor` |
| GitHub URL | `coreconduit/pi-monitor` | `coreconduit/rpi-monitor` |
| GitHub URL (bitsandbots) | `bitsandbots/pi-monitor` | `bitsandbots/rpi-monitor` |

## What Stayed the Same

- **Environment variables:** `PIMONITOR_*` and `PIHUB_*` prefixes unchanged
- **Ports:** Node agent (8585), Hub (8686)
- **API endpoints:** All `/api/*` paths unchanged
- **Web UI:** Same interface, same functionality
- **Configuration format:** Same structure

---

## Migration Steps

### Option 1: Fresh Install (Recommended)

```bash
# Backup existing configuration (optional)
sudo cp /opt/pi-monitor/services.json /opt/pi-monitor/services.json.bak 2>/dev/null || true

# Install new version
git clone https://github.com/bitsandbots/rpi-monitor
cd rpi-monitor
sudo ./install.sh
```

This will install to `/opt/rpi-monitor` with the new service name.

### Option 2: In-Place Upgrade (Existing Installation)

If you want to keep your existing configuration:

```bash
# 1. Stop the current service
sudo systemctl stop pi-monitor pi-monitor-hub

# 2. Clone the new repo
git clone https://github.com/bitsandbots/rpi-monitor /opt/rpi-monitor

# 3. Copy your existing configuration
sudo cp /opt/pi-monitor/services.json /opt/rpi-monitor/
sudo cp /opt/pi-monitor/.env /opt/rpi-monitor/ 2>/dev/null || true

# 4. Install as new service
cd /opt/rpi-monitor
sudo ./install.sh
```

### Option 3: Manual Upgrade (Full Control)

```bash
# Stop services
sudo systemctl stop pi-monitor pi-monitor-hub

# Download and extract release
curl -LO https://github.com/bitsandbots/rpi-monitor/releases/download/v2.0.0/rpi-monitor-2.0.0.tar.gz
tar -xzf rpi-monitor-2.0.0.tar.gz

# Install new service (this removes old service)
cd rpi-monitor-2.0.0
sudo ./install.sh

# If you had custom services.json, restore it:
sudo cp /tmp/rpi-monitor-2.0.0.tar.gz-backup/services.json /opt/rpi-monitor/
```

---

## Post-Migration Checklist

After upgrading, verify:

- [ ] Node agent accessible at `http://<ip>:8585`
- [ ] Hub accessible at `http://<ip>:8686` (if installed)
- [ ] Service running: `systemctl status rpi-monitor`
- [ ] Logs showing correct version: `journalctl -u rpi-monitor -f`
- [ ] Services list persisted: Check Services tab in UI

---

## Uninstalling Old Installation

If you did a fresh install and want to clean up:

```bash
# Remove old directory
sudo rm -rf /opt/pi-monitor 2>/dev/null || true

# Uninstall hub if installed separately
cd /opt/pi-monitor/hub  # if it still exists
sudo ./install.sh --uninstall
```

---

## Troubleshooting

### "Permission denied" on /opt/rpi-monitor

The new service expects `/opt/rpi-monitor`. Update your sudoers rules if running as non-root:

```bash
# /etc/sudoers.d/rpi-monitor (was: pi-monitor)
rpi-monitor ALL=(ALL) NOPASSWD: /usr/bin/systemctl start ssh nginx docker
rpi-monitor ALL=(ALL) NOPASSWD: /usr/bin/systemctl stop ssh nginx docker
rpi-monitor ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart ssh nginx docker
rpi-monitor ALL=(ALL) NOPASSWD: /usr/bin/systemctl enable ssh nginx docker
rpi-monitor ALL=(ALL) NOPASSWD: /usr/bin/systemctl disable ssh nginx docker
rpi-monitor ALL=(ALL) NOPASSWD: /sbin/reboot
rpi-monitor ALL=(ALL) NOPASSWD: /sbin/shutdown
```

### Old service still running after upgrade

```bash
# Check for old service
systemctl list-units | grep pi-monitor

# Stop and remove if found
sudo systemctl stop pi-monitor pi-monitor-hub
sudo systemctl disable pi-monitor pi-monitor-hub
sudo rm -f /etc/systemd/system/pi-monitor*.service
sudo systemctl daemon-reload
```

### Configuration not persisting

Environment variables in systemd unit files may reference the old paths. Check:

```bash
sudo systemctl cat rpi-monitor | grep -i environment
```

If needed, update the service file:

```bash
sudo systemctl edit rpi-monitor
# Add: Environment=PIMONITOR_SERVICES_FILE=/opt/rpi-monitor/services.json
sudo systemctl daemon-reload
sudo systemctl restart rpi-monitor
```

---

## Support

For issues, check:
- Wiki: https://github.com/bitsandbots/rpi-monitor/wiki
- Documentation: `docs/` folder in the repository
