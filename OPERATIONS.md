# OpenVAS / Greenbone Community Edition — Operations Guide

> Step-by-step deployment and day-2 operations for a containerized Greenbone Community Edition vulnerability scanner on Proxmox. Covers VM creation, Docker setup, scan configuration, authenticated scanning, feed management, and recovery procedures.

---

## At a Glance

| Field | Value |
|-------|-------|
| VM/CT Name | VMVULN01 |
| Host Node | Proxmox VE cluster |
| IP Address | (internal) |
| OS | Ubuntu Server 24.04 (minimal) |
| vCPU / RAM / Disk | 2 (4+ rec) / 4 GB min (8 GB rec) / 40–60 GB |
| Key Ports | 9392 (GSA Web UI), 22 (SSH) |
| Credentials | Default: admin / admin — change immediately |
| Depends On | Proxmox VE, flat LAN (10.0.0.0/24) |

> **Warning:** GSA is HTTP-only on port 9392. Keep this VM on your internal LAN or behind a reverse proxy / VPN. Do not expose it directly to the Internet.

> **Note:** This VM is scanner-only. Prometheus, Grafana, and other monitoring components live elsewhere.

---

## Prerequisites

- Working Proxmox VE cluster with available resources
- Flat homelab network (e.g. `10.0.0.0/24`)
- Static IP reserved or DHCP reservation configured
- Ubuntu Server 24.04 (minimal) ISO uploaded to Proxmox storage

---

## 0 — Create the VM in Proxmox

Create a new VM on your Proxmox cluster with these settings:

- Name: `VMVULN01`
- Guest OS: Linux → Ubuntu → 24.04 (or "Other 5.x+ Linux")
- CPU: 2 vCPUs (4+ recommended)
- RAM: 4 GB minimum (8 GB recommended for larger scans)
- Disk: 40–60 GB on `local-lvm` (or your thin pool)
- BIOS / Machine: `OVMF (UEFI)` or `SeaBIOS` (your cluster standard), `q35` if you prefer modern chipset
- Network: VirtIO on `vmbr0` (flat LAN)
- QEMU Guest Agent: Enabled (tick "Qemu Agent" checkbox)

Install Ubuntu Server 24.04 (minimal) inside this VM.

---

## 1 — Base OS Prep & Hostname

Log in as your normal sudo user on the VM.

### 1.1 Update packages & basic tools

```bash
sudo apt update
sudo apt -y full-upgrade

# Helpful base packages
sudo apt install -y qemu-guest-agent vim curl ca-certificates gnupg lsb-release

# Enable guest agent for Proxmox
sudo systemctl enable --now qemu-guest-agent
```

Reboot if the kernel was updated:

```bash
sudo reboot
```

### 1.2 Ensure hostname and /etc/hosts are correct

After reboot:

```bash
hostnamectl set-hostname vmvuln01

sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1\tvmvuln01/' /etc/hosts
grep '^127\.0\.1\.1' /etc/hosts
hostnamectl
```

---

## 2 — Install Docker CE (Official Repository)

All container work uses Docker Engine + docker compose plugin from the official Docker repo.

### 2.1 Remove conflicting Docker packages (if any)

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y "$pkg" 2>/dev/null || true
done
```

### 2.2 Add the Docker APT repository

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

### 2.3 Install Docker Engine + compose plugin

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Enable and verify:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
```

Quick test:

```bash
sudo docker run --rm hello-world
```

### 2.4 Add your user to the docker group

```bash
sudo usermod -aG docker "$USER"
```

Log out and log back in, then confirm:

```bash
id | sed 's/ /\n/g' | grep docker || echo "docker group NOT present in this session"
```

---

## 3 — Greenbone Community Containers

Following the official Greenbone container docs, pinning everything under `/opt/greenbone-community-container` and exposing GSA on all interfaces.

### 3.1 Create working directory

```bash
sudo mkdir -p /opt/greenbone-community-container
sudo chown "$USER":"$USER" /opt/greenbone-community-container

export DOWNLOAD_DIR=/opt/greenbone-community-container
cd "$DOWNLOAD_DIR"

# Add to .bashrc for persistence
echo 'export DOWNLOAD_DIR=/opt/greenbone-community-container' >> ~/.bashrc
```

### 3.2 Download the docker-compose.yml

```bash
curl -f -O -L https://greenbone.github.io/docs/latest/_static/docker-compose.yml \
  --output-dir "$DOWNLOAD_DIR"

ls -l "$DOWNLOAD_DIR"/docker-compose.yml
```

### 3.3 Expose GSA on all interfaces (0.0.0.0)

By default, the `gsa` service only binds to `127.0.0.1:9392`. For LAN access:

```bash
cd "$DOWNLOAD_DIR"
sed -i 's/127\.0\.0\.1:9392:80/0.0.0.0:9392:80/' docker-compose.yml
grep -n '9392:80' docker-compose.yml
```

> **Warning:** This exposes GSA on all interfaces. Keep the VM on a trusted LAN.

### 3.4 Pull images and start the stack

```bash
docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition pull

docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition up -d
```

Watch logs (Ctrl+C to stop):

```bash
docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition logs -f
```

> **Note:** First startup can take a while. Several "feed" containers will exit after populating data volumes (that's expected). The long-lived services are: `gvmd`, `gsa`, `ospd-openvas`, `redis-server`, `pg-gvm`, `openvasd`.

### 3.5 Confirm container and port status

```bash
docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition ps

sudo ss -ltnp | grep 9392 || echo "No listener on 9392!"
```

---

## 4 — Web Login & Admin Account

### 4.1 Access GSA

Open: `http://<vmvuln01-ip>:9392/`

### 4.2 Default credentials & password change

- Username: `admin`
- Password: `admin`

Change immediately via CLI:

```bash
cd "$DOWNLOAD_DIR"
docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" \
  exec -u gvmd gvmd gvmd --user=admin --new-password='<new-strong-password>'
```

> **Note:** Quote the password in single quotes if it contains `$` or other special characters.

---

## 5 — Validation

### Host & Docker sanity

```bash
hostnamectl
ip -4 addr show
sudo systemctl status docker --no-pager

cd /opt/greenbone-community-container
docker compose -f docker-compose.yml -p greenbone-community-edition ps

sudo ss -ltnp | grep 9392 || echo "No GSA listener on 9392!"
```

### Web UI

- [ ] GSA login page loads at `http://<vmvuln01-ip>:9392/`
- [ ] Login succeeds with admin credentials
- [ ] Administration → Feed Status: NVT, SCAP, CERT, GVMD_DATA all show Current
- [ ] Configuration → Scanners: `OpenVAS Default` scanner present, no error badges
- [ ] Sample scan Task completes (Queued → Running → Done) with results

---

## 6 — Post-Install

- [ ] Change default `admin` password immediately
- [ ] Configure log forwarding to Graylog/Cribl
- [ ] Add scan targets for other lab hosts
- [ ] Store credentials in password manager
- [ ] Add VM to Proxmox backup schedule

---

# Part 2 — Day-2 Operations: Scanning, Feeds & Recovery

> Everything below covers ongoing operations after the initial deployment is complete.

---

## 7 — Create the Scan Service Account

Authenticated (credentialed) scanning finds dramatically more vulnerabilities than network-only scanning. Create a dedicated service account on every Linux target.

### 7.1 On each target VM/CT

```bash
sudo useradd \
  --create-home \
  --shell /bin/bash \
  --comment "OpenVAS / Greenbone scan service account" \
  vas_scan

sudo passwd vas_scan
# Use a strong password — store it in your password manager

sudo usermod -aG sudo vas_scan
```

> **Note:** The scanner needs `sudo` for local checks (installed packages, kernel version, config files). Without sudo, credentialed scan coverage drops ~60%.

### 7.2 Register the credential in GVM

In the GSA web UI:

1. Navigate to Configuration → Credentials
2. Click New Credential
3. Fill in:
   - Name: `Linux`
   - Type: Username + Password
   - Login: `vas_scan`
   - Password: (the password you set above)
4. Save

---

## 8 — Create Scan Targets

### 8.1 Recommended target groups

| Target Group | Hosts | Credential | Notes |
|-------------|-------|------------|-------|
| Proxmox Hosts | hypervisor IPs | Linux (vas_scan) | High-value, low count |
| VMs (Linux) | VM IPs, comma-separated | Linux (vas_scan) | All Linux VMs and containers |
| IoT and Workstations | mixed device IPs | Linux (vas_scan) | Everything else worth scanning |

### 8.2 Create a target in GSA

1. Navigate to Configuration → Targets
2. Click New Target
3. Fill in:
   - Name: (e.g., `VMs (Linux)`)
   - Hosts — Manual: comma-separated IPs
   - Port List: `All IANA assigned TCP`
   - Alive Test: `Scan Config Default`
   - SSH Credential: `Linux` (created above)
   - SSH Port: `22`
4. Save

---

## 9 — Create Scan Tasks & Schedules

### 9.1 Create schedules in GSA

1. Navigate to Configuration → Schedules
2. Create two schedules:
   - **Weekly (Wave 1):** Sunday 01:00 AM, FREQ=WEEKLY
   - **Weekly (Wave 2):** Sunday 05:00 AM, FREQ=WEEKLY

Staggering scans in waves prevents saturating your network link.

### 9.2 Create tasks in GSA

1. Navigate to Scans → Tasks
2. Click New Task
3. Fill in:
   - Name: (e.g., `VMs (Linux)`)
   - Scan Config: `Full and fast`
   - Target: (the target from Step 8)
   - Scanner: `OpenVAS Default`
   - Schedule: (Wave 1 or Wave 2)
4. Save

### 9.3 Scanner concurrency tuning

Configure per-task based on scanner VM resources:

| Task | max_hosts | max_checks | Notes |
|------|-----------|------------|-------|
| Standard weekly scans | 20 | 4 | 4 vCPU / 8 GB RAM budget |
| Full subnet discovery | 30 | 6 | On-demand only, more aggressive |

---

## 10 — Automated Feed Updates

### 10.1 Why this matters

OpenVAS is only as good as its vulnerability feed (~95K NVTs). If feeds go stale, scans produce incomplete results. After a VM restore from backup, feeds can be months old.

### 10.2 Create the update script

```bash
sudo tee /opt/greenbone-community-container/feed-update.sh << 'EOF'
#!/bin/bash
# Greenbone Community Edition - Weekly Feed Update
# Schedule: Saturday midnight (before Sunday scans)

LOG="/var/log/greenbone-feed-update.log"
COMPOSE_DIR="/opt/greenbone-community-container"

echo "$(date): Starting feed update..." >> "$LOG"
cd "$COMPOSE_DIR" || exit 1

# Pull latest feed images
docker compose -f docker-compose.yml -p greenbone-community-edition \
  pull vulnerability-tests notus-data scap-data cert-bund-data \
  dfn-cert-data data-objects report-formats >> "$LOG" 2>&1

# Run feed containers to populate volumes
docker compose -f docker-compose.yml -p greenbone-community-edition \
  up -d >> "$LOG" 2>&1

# Wait for feed containers to finish seeding
sleep 120

# Restart ospd-openvas to parse NASLs into redis (~2-3 min for 95K files)
docker compose -f docker-compose.yml -p greenbone-community-edition \
  restart ospd-openvas >> "$LOG" 2>&1

# Wait for VT loading to complete
sleep 180

# Restart gvmd to sync VTs from ospd into PostgreSQL (5-15 min)
docker compose -f docker-compose.yml -p greenbone-community-edition \
  restart gvmd >> "$LOG" 2>&1

echo "$(date): Feed update complete." >> "$LOG"
EOF

sudo chmod +x /opt/greenbone-community-container/feed-update.sh
```

### 10.3 Schedule the cron job

```bash
# Add to root crontab — every Saturday at midnight
(sudo crontab -l 2>/dev/null; echo "0 0 * * 6 /opt/greenbone-community-container/feed-update.sh") | sudo crontab -

# Verify
sudo crontab -l | grep feed
```

### 10.4 Feed update order explained

```
Saturday 00:00  →  Pull feed images (1.4 GB VT image)
                →  Seed data into Docker volumes
                →  Restart ospd-openvas (parse 95K NASLs into redis)
                →  Restart gvmd (sync VTs into PostgreSQL)
Sunday 01:00    →  Wave 1 scans start (with fresh intelligence)
Sunday 05:00    →  Wave 2 scans start
```

> **Rule:** Always update feeds before scans. Never scan with stale intelligence.

---

## 11 — Verify Feed Health

After an update or recovery, confirm the feed loaded properly.

### 11.1 Check NVT count (should be ~95,000+)

```bash
docker exec greenbone-community-edition-ospd-openvas-1 \
  find /var/lib/openvas/plugins/ -name "*.nasl" | wc -l
```

If this shows < 10,000, the feed didn't sync. Re-run the update script.

### 11.2 Check ospd-openvas logs

```bash
docker logs greenbone-community-edition-ospd-openvas-1 --tail 10
```

Healthy: `INFO: Finished loading VTs. The VT cache has been updated from version YYYYMMDD to YYYYMMDD.`

Unhealthy: `ERROR: Updating VTs failed.` → Re-run feed update script.

### 11.3 Check gvmd VT sync

```bash
docker logs greenbone-community-edition-gvmd-1 --tail 10
```

Look for: `MESSAGE: update_nvt_cache_retry: rebuild successful`

### 11.4 Check from GSA web UI

Administration → Feed Status. All feeds should show Current: NVT, SCAP, CERT, GVMD_DATA.

---

## 12 — Resolving Scanner Sync Issues

### Symptoms

- `Administration → Feed Status` shows NVT feed as out of sync
- Error: "Synchronization issue: Could not connect to scanner to get feed info."

### Diagnostic commands

```bash
# Check compose status
docker compose -f /opt/greenbone-community-container/docker-compose.yml \
  -p greenbone-community-edition ps

# Check for ospd socket inside container
docker exec -it greenbone-community-edition-ospd-openvas-1 ls -l /run/ospd/

# List scanners from gvmd
docker exec -it -u gvmd greenbone-community-edition-gvmd-1 gvmd --get-scanners
```

### Expected scanner output

```
08b69003-5fc2-4037-a479-93b440211c73 OpenVAS /run/ospd/ospd-openvas.sock 0 OpenVAS Default
```

### Fix

Restart scanner-related containers in the correct order:

```bash
docker compose -f /opt/greenbone-community-container/docker-compose.yml \
  -p greenbone-community-edition restart gvmd ospd-openvas openvas
```

If still broken, full stack restart:

```bash
cd /opt/greenbone-community-container
docker compose -f docker-compose.yml -p greenbone-community-edition down
docker compose -f docker-compose.yml -p greenbone-community-edition up -d
```

---

## 13 — Recovery After Hard Failure

### 13.1 Start the stack

```bash
cd /opt/greenbone-community-container
docker compose -f docker-compose.yml -p greenbone-community-edition up -d
```

### 13.2 Disable conflicting native services

If the VM previously had a bare-metal GVM install:

```bash
sudo systemctl stop gvmd ospd-openvas 2>/dev/null
sudo systemctl disable gvmd ospd-openvas 2>/dev/null
```

### 13.3 Update the feed (will be stale after restore)

```bash
sudo /opt/greenbone-community-container/feed-update.sh
```

Monitor progress:

```bash
docker logs -f greenbone-community-edition-ospd-openvas-1
docker logs -f greenbone-community-edition-gvmd-1
```

### 13.4 Recovery timeline

| Time | Event |
|------|-------|
| T+0 | `docker compose up -d` — containers start |
| T+2 min | Feed containers finish seeding volumes |
| T+3 min | ospd-openvas begins loading 95K NVTs |
| T+5 min | ospd-openvas finishes: "Finished loading VTs" |
| T+6 min | gvmd detects new VT version, starts DB sync |
| T+15-20 min | gvmd finishes: "rebuild successful" |
| T+20 min | Tasks can be started |

---

## 14 — Adding New Devices

When you add a new VM or device to the lab:

1. Create the `vas_scan` account on the new host (Step 7.1)
2. Add the IP to a target in GSA (Configuration → Targets → edit)
3. If the target is in use (assigned to a running task), you can't edit it directly. Options:
   - Wait for the current scan to finish, then edit
   - Create a new target + task for the new host
   - Delete the task (not the reports), edit the target, recreate the task

> **Gotcha:** GVM won't let you modify a target that's referenced by an active task. Plan target changes between scan windows.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| GSA not reachable on port 9392 | Port bound to 127.0.0.1 only | `sed -i 's/127.0.0.1:9392:80/0.0.0.0:9392:80/' docker-compose.yml` then restart stack |
| "Could not connect to scanner to get feed info" | ospd-openvas socket missing | Restart scanner containers: `restart gvmd ospd-openvas openvas` |
| ospd-openvas not running | Container crash or startup ordering | Full stack `down` then `up -d` |
| Feed status stuck on "Updating" | First-time sync takes time | Wait 10–30 min; feed containers exit after populating (expected) |
| Native `gvmd.service` crash-looping | Conflicts with Docker stack | `systemctl disable gvmd` |
| "Updating VTs failed" in ospd logs | Stale/empty feed volume | Pull latest feed images, restart ospd |
| Tasks stuck "Interrupted" | VTs not synced after restart | Wait for gvmd "rebuild successful", then start tasks |
| NVT count ~3K (should be ~95K) | Feed containers didn't run | `docker compose up -d` (re-runs feeds), restart ospd |
| GSA "Cookie missing or bad" | Session expired | Re-login |
| Scan finds 0 results | Scanner not connected or VTs empty | Check `get_scanners`, verify ospd socket exists |
| Credentialed scan shallow results | vas_scan lacks sudo | Add account to sudo group on target |

---

## Quick Reference

```bash
# Set working directory
export DOWNLOAD_DIR=/opt/greenbone-community-container
cd "$DOWNLOAD_DIR"

# Start stack
docker compose -f docker-compose.yml -p greenbone-community-edition up -d

# Stop stack
docker compose -f docker-compose.yml -p greenbone-community-edition down

# Tail all logs
docker compose -f docker-compose.yml -p greenbone-community-edition logs -f

# Check container status
docker compose -f docker-compose.yml -p greenbone-community-edition ps

# Update admin password (CLI)
docker compose -f docker-compose.yml exec -u gvmd gvmd gvmd --user=admin --new-password='<new-password>'

# List scanners
docker exec -it -u gvmd greenbone-community-edition-gvmd-1 gvmd --get-scanners

# Check ospd socket
docker exec -it greenbone-community-edition-ospd-openvas-1 ls -l /run/ospd/

# Restart scanner-related containers (fix sync issues)
docker compose -f docker-compose.yml -p greenbone-community-edition restart gvmd ospd-openvas openvas

# Pull latest images and recreate
docker compose -f docker-compose.yml -p greenbone-community-edition pull
docker compose -f docker-compose.yml -p greenbone-community-edition up -d

# Manual feed update
sudo /opt/greenbone-community-container/feed-update.sh

# Check feed health (NVT count)
docker exec greenbone-community-edition-ospd-openvas-1 \
  find /var/lib/openvas/plugins/ -name "*.nasl" | wc -l
```

---

## Quirks & Gotchas

- **Feed update order:** ospd-openvas must restart BEFORE gvmd. ospd loads NASLs into memory first, then gvmd pulls from ospd via the socket.
- **Feed containers exit immediately** — by design. They copy data into volumes and exit. Only `gvmd`, `gsa`, `ospd-openvas`, `redis`, `pg-gvm` stay running.
- **GSA form API is brittle** — for automation, use the GMP Unix socket. GSA HTTP proxy adds undocumented mandatory form fields that break between versions.
- **Credentialed scans need sudo** — without it, the scanner can't read package lists, kernel info, or config files. Coverage drops ~60%.
- **Don't scan during feed updates** — schedule feeds Saturday, scans Sunday.
- **Targets in use can't be edited** — GVM locks targets assigned to tasks. Plan host additions between scan windows.
- **`gvm-cli` version mismatch** — host `python-gvm` can be outdated vs container GVM version. Use raw Unix socket with Python instead.

---

Last updated: 2026-07-04
