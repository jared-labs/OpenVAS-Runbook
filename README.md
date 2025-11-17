# Deploy VMVULN01 — Greenbone Community Edition Vulnerability Scanner

Single-VM deployment of the Greenbone Community Edition (OpenVAS) using Docker CE on Ubuntu Server 24.04, running as `vmvuln01` on Proxmox. Copy/paste friendly.

> Guardrail: **vmvuln01 is scanner-only.** Prometheus, Grafana, and other monitoring components live on `vmmon01`, not here.  
> Guardrail: **GSA is HTTP-only on port 9392.** Keep this VM on your internal LAN or behind a reverse proxy / VPN. Do **not** expose it directly to the Internet.  
> Assumes: A working Proxmox VE cluster and a flat homelab network (e.g. `10.0.0.0/24`).

---

## 0) VM definition in Proxmox (before OS install)

Create a new VM on your Proxmox cluster:

- **Name:** `VMVULN01`
- **Guest OS:** Linux → Ubuntu → 24.04 (or “Other 5.x+ Linux”)
- **CPU:** 2 vCPUs (4+ recommended)
- **RAM:** 4 GB minimum (8 GB recommended for larger scans)
- **Disk:** 40–60 GB on `local-lvm` (or your thin pool)
- **BIOS / Machine:** `OVMF (UEFI)` or `SeaBIOS` (your cluster standard), `q35` if you prefer modern chipset
- **Network:** VirtIO on `vmbr0` (flat LAN)
- **QEMU Guest Agent:** Enabled (tick “Qemu Agent” checkbox)

Install **Ubuntu Server 24.04 (minimal)** inside this VM.

---

## 1) Base OS prep & hostname (Ubuntu 24.04)

Log in as your normal sudo user on `vmvuln01`.

### 1.1 Update packages & basic tools

    sudo apt update
    sudo apt -y full-upgrade

    # Helpful base packages
    sudo apt install -y qemu-guest-agent vim curl ca-certificates gnupg lsb-release

    # Enable guest agent for Proxmox
    sudo systemctl enable --now qemu-guest-agent

Reboot if the kernel was updated:

    sudo reboot

### 1.2 Ensure hostname and /etc/hosts are correct

After reboot:

    hostnamectl set-hostname vmvuln01

Fix the `127.0.1.1` line in `/etc/hosts` so it matches the hostname:

    sudo sed -i 's/^127\.0\.1\.1.*/127.0.1.1\tvmvuln01/' /etc/hosts
    grep '^127\.0\.1\.1' /etc/hosts
    hostnamectl

> If you changed the hostname after install, log out and back in so your shell prompt matches.

---

## 2) Install Docker CE (official Docker repository)

All container work is done with **Docker Engine + docker compose plugin**, installed from the official Docker repo (not `docker.io`).

### 2.1 Remove conflicting Docker packages (if any)

    for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
      sudo apt remove -y "$pkg" 2>/dev/null || true
    done

### 2.2 Add the Docker APT repository

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

### 2.3 Install Docker Engine + compose plugin

    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

Enable and verify:

    sudo systemctl enable --now docker
    sudo systemctl status docker --no-pager

Quick test:

    sudo docker run --rm hello-world

### 2.4 Add your user to the docker group

    sudo usermod -aG docker "$USER"

Log out and log back in (or open a new SSH session), then confirm:

    id | sed 's/ /\n/g' | grep docker || echo "docker group NOT present in this session"

---

## 3) Greenbone Community Containers under /opt

We’ll follow the official Greenbone container docs, but pin everything under `/opt/greenbone-community-container` and expose GSA on all interfaces.

### 3.1 Create working directory and export DOWNLOAD_DIR

    sudo mkdir -p /opt/greenbone-community-container
    sudo chown "$USER":"$USER" /opt/greenbone-community-container

    export DOWNLOAD_DIR=/opt/greenbone-community-container
    cd "$DOWNLOAD_DIR"

(Optional) Add this to your `~/.bashrc` so it’s always available:

    echo 'export DOWNLOAD_DIR=/opt/greenbone-community-container' >> ~/.bashrc

### 3.2 Download the docker-compose.yml via curl (not git clone)

Use the official “curl download” method (no git):

    curl -f -O -L https://greenbone.github.io/docs/latest/_static/docker-compose.yml \
      --output-dir "$DOWNLOAD_DIR"

Verify you have the file:

    ls -l "$DOWNLOAD_DIR"/docker-compose.yml

### 3.3 Expose GSA on all interfaces (0.0.0.0)

By default, the `gsa` service only binds to `127.0.0.1:9392`, which is fine for localhost-only but not for LAN access.

Edit the `gsa` ports section in `docker-compose.yml` and replace:

- `127.0.0.1:9392:80` → `0.0.0.0:9392:80`

Automated one-liner:

    cd "$DOWNLOAD_DIR"
    sed -i 's/127\.0\.0\.1:9392:80/0.0.0.0:9392:80/' docker-compose.yml
    grep -n '9392:80' docker-compose.yml

> Security note: This exposes GSA on all interfaces of the host. Keep `vmvuln01` on a trusted LAN and/or behind a reverse proxy / VPN.

### 3.4 Pull images and start the Greenbone stack

Pull all images:

    docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition pull

Start containers in the background:

    docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition up -d

Watch logs (Ctrl+C to stop):

    docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition logs -f

> First startup can take a while. Several “feed” containers will **exit** after populating data volumes (that’s expected). The long-lived services you care about are: `gvmd`, `gsa`, `ospd-openvas`, `redis-server`, `pg-gvm`, `openvasd`.

### 3.5 Confirm container and port status

Check compose status:

    docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition ps

You should see `State` = `running` for at least:

- `redis-server`
- `pg-gvm`
- `gvmd`
- `gsa`
- `ospd-openvas`
- `openvasd`
- `gvm-tools` (may be idle/short-lived depending on use)

Confirm port 9392 is listening:

    sudo ss -ltnp | grep 9392 || echo "No listener on 9392!"

Typical output should show a `docker-proxy` / `dockerd` listener bound to `0.0.0.0:9392`.

---

## 4) Web login & admin account

### 4.1 Accessing GSA (HTTP, not HTTPS)

Once the feeds have mostly populated (give it a few minutes), open:

- `http://<vmvuln01-ip>:9392/`  
  or, if you have DNS:  
- `http://vmvuln01.lan:9392/`

> **Important:** This is **plain HTTP**, not HTTPS. The browser “Not secure” warning is expected. For remote access, put a TLS-terminating reverse proxy (e.g. Nginx, Traefik, Caddy) in front of it.

### 4.2 Default admin credentials & changing password

Current Greenbone community containers create a default admin user:

- **Username:** `admin`  
- **Password:** `admin`

Change this immediately via the GSA web UI **or** via CLI:

    cd "$DOWNLOAD_DIR"
    docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" \
      exec -u gvmd gvmd gvmd --user=admin --new-password='<new-strong-password>'

> Quote the password in single quotes if it contains `$` or other special characters.

### 4.3 (If needed) Retrieve / verify admin password from logs

If future container versions switch to a generated password, you can inspect the `gvmd` logs:

    cd "$DOWNLOAD_DIR"
    docker compose -f "$DOWNLOAD_DIR/docker-compose.yml" -p greenbone-community-edition logs gvmd | grep -i 'password' || echo "No password line found"

Right now this typically just confirms that the default `admin/admin` account exists, but this pattern is here if the behavior changes.

---

## 5) Keep VMVULN01 scanner-only (no Prometheus / Grafana)

In this homelab stack, **all monitoring lives on `vmmon01`**. `vmvuln01` should not run Prometheus, Grafana, or other monitoring daemons.

If you previously experimented with Prometheus/Grafana on this VM, clean it up:

### 5.1 Disable stray monitoring services (if installed)

    sudo systemctl disable --now prometheus prometheus-node-exporter alertmanager grafana-server 2>/dev/null || true

### 5.2 Remove monitoring packages (optional cleanup)

    sudo apt remove -y prometheus prometheus-node-exporter alertmanager grafana grafana-enterprise grafana-agent 2>/dev/null || true
    sudo apt autoremove -y

Verify nothing obvious is listening that shouldn’t be:

    sudo ss -ltnp

---

## 6) Resolving Scanner Sync Issues (ospd-openvas & Feed Status)

This section captures the real-world troubleshooting steps from when the NVT feed showed a synchronization error and the scanner socket was missing.

### 6.1 A) Detecting the issue

Symptoms in GSA:

- `Administration → Feed Status` showed the **NVT feed** as **out of sync**.
- Error message on the Feed Status page:

  > `Synchronization issue: Could not connect to scanner to get feed info.`

Container-level check:

- `ospd-openvas` was **not running** in `docker compose ps`, or it was stuck / not providing a socket.

### 6.2 B) Commands we ran

All commands are executed on `vmvuln01`.

**1) Check compose status and confirm `ospd-openvas` state**

    sudo docker compose -f /opt/greenbone-community-container/docker-compose.yml -p greenbone-community-edition ps

Look for the `ospd-openvas` row and verify the `State`.

**2) Check for the ospd-openvas socket inside the container**

First, get a shell inside the `ospd-openvas` container:

    sudo docker exec -it greenbone-community-edition-ospd-openvas-1 bash

Then, inside the container:

    ls -l /run/ospd/ || echo "no /run/ospd directory"

If the `/run/ospd` directory or the `ospd-openvas.sock` file is missing, gvmd cannot talk to the scanner.

**3) List scanners from gvmd**

Use the `gvmd` user inside the `gvmd` container:

    sudo docker exec -it -u gvmd greenbone-community-edition-gvmd-1 gvmd --get-scanners

If the scanner is missing or has the wrong socket path, feed / sync status will fail.

### 6.3 C) Expected scanner output

Once things were healthy, `gvmd --get-scanners` showed this entry:

    08b69003-5fc2-4037-a479-93b440211c73 OpenVAS /run/ospd/ospd-openvas.sock 0 OpenVAS Default

- The UUID `08b69003-5fc2-4037-a479-93b440211c73` is the built-in OpenVAS scanner.
- Socket path: `/run/ospd/ospd-openvas.sock`
- Name: `OpenVAS Default`

### 6.4 D) Fix applied

The fix was to restart the critical scanner-related containers in the correct order so the ospd socket and gvmd connection came up cleanly.

From the host:

    sudo docker compose -f /opt/greenbone-community-container/docker-compose.yml \
      -p greenbone-community-edition restart gvmd ospd-openvas openvas

Then:

1. Re-check the socket inside `ospd-openvas`:

       sudo docker exec -it greenbone-community-edition-ospd-openvas-1 ls -l /run/ospd/ || echo "no /run/ospd directory"

   You should see `ospd-openvas.sock` present and owned by the expected user.

2. Re-check scanners from `gvmd`:

       sudo docker exec -it -u gvmd greenbone-community-edition-gvmd-1 gvmd --get-scanners

   Confirm you see:

       08b69003-5fc2-4037-a479-93b440211c73 OpenVAS /run/ospd/ospd-openvas.sock 0 OpenVAS Default

3. Go back to GSA → `Administration → Feed Status` and refresh the page.

   - The NVT feed should now show a valid status and no “Could not connect to scanner” error.

### 6.5 E) Final validation for scanner & feeds

From the **GSA web UI**:

1. Go to **Administration → Feed Status** and ensure:
   - **NVT**
   - **SCAP**
   - **CERT**
   - **GVMD_DATA**

   all show **Status = Current** (or at least “Updating” without scanner errors).

2. Go to **Configuration → Scanners**:
   - Verify the `OpenVAS Default` scanner is present and has a green status.

3. Create a quick test Target (e.g. `vmvuln01` itself) and a simple Task:
   - Start the Task.
   - Confirm it moves through states (Queued → Running → Done) without “No scanner” errors.

### 6.6 Why this happens & how to update safely in the future

**Root cause (likely):**

- `ospd-openvas` wasn’t fully ready, or its socket volume wasn’t available, when `gvmd` tried to connect.
- After an update or first-time feed sync, container startup ordering can occasionally leave `gvmd` without a valid scanner socket, which surfaces as:
  - “Could not connect to scanner to get feed info” on the **Feed Status** page.
  - Missing or incorrect scanner entry from `gvmd --get-scanners`.

**Gentle recovery pattern:**

- Restart just the scanner-related containers (what we did above):

      sudo docker compose -f /opt/greenbone-community-container/docker-compose.yml \
        -p greenbone-community-edition restart gvmd ospd-openvas openvas

- If things are still weird, a full stack restart is safe:

      cd /opt/greenbone-community-container
      docker compose -f docker-compose.yml -p greenbone-community-edition down
      docker compose -f docker-compose.yml -p greenbone-community-edition up -d

**Re-running updates in the future:**

To refresh container images and ensure feeds/containers are current:

    cd /opt/greenbone-community-container

    # Pull latest images
    docker compose -f docker-compose.yml -p greenbone-community-edition pull

    # Recreate containers
    docker compose -f docker-compose.yml -p greenbone-community-edition up -d

After any update:

- Re-check **Feed Status** in GSA.
- If you see scanner connection errors, apply the restart pattern above.

---

## 7) Final validation checklist

Use this as a quick “is vmvuln01 healthy?” checklist.

### 7.1 Host & Docker sanity

On `vmvuln01`:

    hostnamectl
    ip -4 addr show

    sudo systemctl status docker --no-pager

    cd /opt/greenbone-community-container
    docker compose -f docker-compose.yml -p greenbone-community-edition ps

You should see `State = running` for the key services mentioned earlier.

Check the listener:

    sudo ss -ltnp | grep 9392 || echo "No GSA listener on 9392!"

### 7.2 Web UI

From a browser on your LAN:

- Open: `http://vmvuln01.lan:9392/` or `http://<vmvuln01-ip>:9392/`
- Confirm the **Greenbone Security Assistant (GSA)** login page loads.
- Log in with your `admin` account (with the updated strong password).

In GSA:

- `Administration → Feed Status`:
  - NVT, SCAP, CERT, GVMD_DATA all show **Current**.
- `Configuration → Scanners`:
  - `OpenVAS Default` scanner present, no error badges.

Run a sample scan:

1. Create a Target (e.g. `vmvuln01` or a lab host).
2. Create a Task using the default scanner.
3. Start the Task and verify it completes with results.

If all of the above are true, **VMVULN01 is ready for regular vulnerability scanning.**

---

## 8) Handy operational commands

All assume:

- Working dir: `/opt/greenbone-community-container`
- Env var: `export DOWNLOAD_DIR=/opt/greenbone-community-container`

**Start stack (if stopped):**

    cd /opt/greenbone-community-container
    docker compose -f docker-compose.yml -p greenbone-community-edition up -d

**Stop stack:**

    cd /opt/greenbone-community-container
    docker compose -f docker-compose.yml -p greenbone-community-edition down

**Tail all logs:**

    cd /opt/greenbone-community-container
    docker compose -f docker-compose.yml -p greenbone-community-edition logs -f

**Update admin password (CLI):**

    cd /opt/greenbone-community-container
    docker compose -f docker-compose.yml \
      exec -u gvmd gvmd gvmd --user=admin --new-password='<new-strong-password>'

**List scanners:**

    sudo docker exec -it -u gvmd greenbone-community-edition-gvmd-1 gvmd --get-scanners

**Check ospd socket inside scanner container:**

    sudo docker exec -it greenbone-community-edition-ospd-openvas-1 ls -l /run/ospd/ || echo "no /run/ospd directory"

Keep this file at:

- `docs/homelab/vuln-scanner-vmvuln01.md`

and update it whenever you adjust the `vmvuln01` build, feeds, or scanner workflow.
