# Vulnerability Scanning: Greenbone/OpenVAS Lab Program

## Overview

This document summarizes a Greenbone Community Edition deployment used to run recurring vulnerability scans across a mixed home-lab fleet. It is written for a portfolio audience, focusing on architecture, scan strategy, authenticated scanning, feed management, and lessons learned from operating the system.

The goal of the service is to provide repeatable visibility into patch exposure and configuration risk across hypervisors, Linux systems, workstations, IoT devices, and network-adjacent services.

## Environment

| Field | Value |
|-------|-------|
| System | VMVULN01 |
| IP Address | 10.0.0.139 |
| OS | Debian 12 |
| Runtime | Docker Compose |
| Scanner | Greenbone Community Edition |
| GVM | 26.8.0 (DB revision 262) |
| GSA | 24.9.0 |
| Feed Baseline | 24.10 release, approximately 95K NVTs |
| Web UI | `http://10.0.0.139:9392` |

## Architecture

```text
+----------------------------------------------------------------+
| VMVULN01 - 10.0.0.139                                          |
| Docker Compose: greenbone-community-edition                    |
|                                                                |
|  +----------+    +----------+    +------------------------+    |
|  |   GSA    |--->|  gvmd    |--->|   ospd-openvas         |    |
|  | :9392->80|    | (socket) |    |   scanner engine       |    |
|  +----------+    +----------+    +------------------------+    |
|                       |                    |                   |
|                       v                    v                   |
|                +----------+        +----------+                |
|                |  pg-gvm  |        |  redis   |                |
|                | (PG 13)  |        | VT cache |                |
|                +----------+        +----------+                |
|                                                                |
| Feed volumes populated by run-once containers:                 |
| vulnerability-tests, notus-data, scap-data, cert-bund-data,    |
| dfn-cert-data, data-objects, report-formats, gpg-data          |
+----------------------------------------------------------------+
         |
         | Scans via network using SSH and TCP probes
         v
+----------------------------------------------+
| Scan Targets                                  |
| - 7 Proxmox hypervisors (.140-.147)           |
| - 16 Linux VMs and containers (.10, .123-.139)|
| - 21 IoT/workstation devices                  |
|   (gateway, APs, cameras, desktops, NAS)      |
+----------------------------------------------+
```

The scanner runs as a containerized Greenbone stack. GSA provides the web interface, `gvmd` owns scan/task state and reporting, `ospd-openvas` performs the scanning work, PostgreSQL stores scanner metadata, and Redis supports vulnerability-test caching.

## Scan Targets

| Target | Hosts | Credential Pattern | Notes |
|--------|-------|--------------------|-------|
| Proxmox | 10.0.0.140-147 | Linux service account | 7 hypervisors |
| Linux VMs/Containers | 10.0.0.10, 10.0.0.123-139 | Linux service account | 16 systems |
| IoT and Workstations | 10.0.0.1, 10.0.0.4-7, 10.0.0.33-35, 10.0.0.38-44, 10.0.0.96-97, 10.0.0.100, 10.0.0.128, 10.0.0.131, 10.0.0.143 | Linux service account where supported | 21 mixed devices |
| Full Subnet Discovery | 10.0.0.0/24 | None | Inventory and discovery only |

## Service Account Pattern

Authenticated Linux scans use a dedicated `vas_scan` account. The account exists on Linux targets and is stored in the password manager as a named service credential.

| Attribute | Pattern |
|-----------|---------|
| Username | `vas_scan` |
| Purpose | Credentialed vulnerability scanning |
| Authentication secret | `<stored in password manager>` |
| Privilege model | Sudo-enabled for local security checks |
| Scope | Linux hosts where authenticated scanning is appropriate |

The account improves scan quality because local package, kernel, and configuration checks produce better findings than remote banner inspection alone. In practice, removing sudo from the account significantly reduces finding depth.

## Scan Scheduling Strategy

| Task | Target | Config | Schedule |
|------|--------|--------|----------|
| Linux VMs/Containers | Linux VMs and containers | Full and Fast | Weekly Wave 1, Sunday 01:00 ET |
| IoT and Workstations | Mixed devices | Full and Fast | Weekly Wave 1, Sunday 01:00 ET |
| Proxmox Hosts | Hypervisors | Full and Fast | Weekly Wave 2, Sunday 02:30 ET |

Scans are intentionally split into waves. The lab runs on a 1 Gbps network, and vulnerability scans can create noisy bursts of TCP probing and authenticated checks. Splitting hypervisors from the rest of the fleet reduces contention and makes it easier to reason about scan impact.

## Feed Management

Greenbone depends on multiple feed data sets: vulnerability tests, Notus data, SCAP data, CERT advisories, report formats, data objects, and GPG metadata. The feed update process is treated as part of scanner health, not as a background detail.

Operational pattern:

1. Pull updated feed containers.
2. Start the stack so run-once containers populate the shared volumes.
3. Restart `ospd-openvas` so it loads the current NASL vulnerability tests.
4. Restart `gvmd` after `ospd-openvas` has loaded tests, allowing task and report metadata to synchronize cleanly.
5. Validate that vulnerability-test count and scanner logs match expected health.

The ordering matters: `ospd-openvas` needs to load vulnerability tests before `gvmd` can reliably synchronize scanner metadata.

## Reporting Pipeline

```text
OpenVAS scan results
        |
        v
XML report export
        |
        v
Cribl HTTP input
        |
        v
Graylog structured events
```

Reports are exported into the logging pipeline so vulnerability data can be searched alongside other infrastructure events. This turns scanner output from a standalone UI artifact into part of the broader observability system.

## Design Decisions

- **Containerized Greenbone stack:** Containers reduce dependency drift and make recovery cleaner than a mixed native install.
- **Credentialed scans for Linux systems:** Authenticated checks provide higher-quality findings than remote-only scanning.
- **Wave-based scheduling:** The scan plan accounts for network load and operational blast radius.
- **Separate discovery target:** Full-subnet discovery is useful for inventory without attaching it to a heavy recurring vulnerability task.
- **Structured export path:** Shipping findings to Graylog makes scan results visible outside the scanner UI.

## Adding New Devices

New devices are added by placing them into the appropriate target group, assigning the service account where supported, and reflecting the device's scanner coverage in the discovery database. Devices that do not support Linux-style authenticated checks can still participate in network discovery or remote probing.

One practical limitation is that Greenbone targets already attached to active tasks may not be editable. In that case, the cleaner approach is often to create a replacement target and task rather than mutating a target while it is in use.

## Upgrade and Recovery Approach

The upgrade approach is conservative:

1. Snapshot or otherwise protect the VM before major scanner changes.
2. Update the Greenbone container images and feed containers together.
3. Let feed volumes repopulate before validating scan task health.
4. Confirm that `ospd-openvas` and `gvmd` agree on vulnerability-test state.
5. Restart interrupted tasks only after feed synchronization finishes.

After a hard failure, the key lesson is to avoid running native Greenbone services beside the Docker stack. A restored host may have stale native services enabled, and those can conflict with containerized `gvmd` over the same socket paths.

## Quirks and Gotchas

- **Docker Compose and native services can conflict:** A native `gvmd.service` running beside the container stack can break socket ownership and scanner state.
- **Feed update order matters:** Restart `ospd-openvas` before `gvmd` so vulnerability tests load before metadata sync.
- **The GSA form API is awkward for automation:** The GMP Unix socket is more reliable for scripted task work than reverse-engineering web form fields.
- **Credentialed scans need sudo:** Without sudo, many local checks cannot inspect package, kernel, and configuration state.
- **Scans generate network noise:** Wave scheduling keeps scans from overwhelming the lab network or making unrelated troubleshooting harder.

## What This Demonstrates

This deployment shows more than "installing a scanner." It demonstrates vulnerability-management workflow design: asset grouping, credential strategy, feed lifecycle, scan scheduling, report export, and recovery from real operational failure modes.

*Sanitized for public portfolio use. Last updated from source runbook: 2026-07-01.*
