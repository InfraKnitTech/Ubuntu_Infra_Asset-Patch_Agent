# InfraKnit Ubuntu Agent - Complete Documentation

A Linux (Ubuntu/Debian) endpoint management agent for asset management and fleet/patch management. The agent collects system telemetry, executes software deployment jobs, and reports status to a central server.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Directory Structure](#directory-structure)
6. [Core Components](#core-components)
7. [Data Collection](#data-collection)
8. [Job Execution](#job-execution)
9. [APT Repository Integration](#apt-repository-integration)
10. [Pre-Installation Intelligence](#pre-installation-intelligence)
11. [Disk Space Management](#disk-space-management)
12. [API Endpoints & JSON Formats](#api-endpoints--json-formats)
13. [Security](#security)
14. [Logging](#logging)
15. [Systemd Service](#systemd-service)
16. [Uninstalling](#uninstalling)
17. [Troubleshooting](#troubleshooting)
18. [Windows vs Ubuntu Agent Comparison](#windows-vs-ubuntu-agent-comparison)

---

## Overview

InfraKnit Ubuntu Agent is a lightweight systemd service that:

- **Collects telemetry**: CPU, memory, disk, network metrics
- **Inventories assets**: Hardware specs, installed packages (dpkg), OS info
- **Executes jobs**: Install, uninstall, patch, and run scripts remotely
- **Deploys patches**: Linux .deb patches via per-app APT repositories
- **Reports status**: Real-time job progress with detailed messages
- **Pre-installation checks**: Verifies current state before making changes
- **Disk space verification**: Ensures sufficient space before downloads

### Key Features

| Feature | Description |
|---------|-------------|
| Asset Management | Hardware inventory, software inventory (dpkg), OS info |
| Fleet Management | Remote software deployment (APT, Shell scripts) |
| Patch Management | Linux .deb patches via per-app APT repos + system APT upgrades |
| Service Packs | Bundled packages + patches deployed together in order |
| Real-time Status | Detailed job progress reporting |
| Pre-installation Checks | Version comparison before install/upgrade |
| Disk Space Check | Pre-download verification with configurable thresholds |
| Location Tracking | Physical location assignment from server |
| Heartbeat | Periodic alive signal to server |
| Remote Power Control | Reboot, shutdown, cancel pending operations |
| Offline Queue | Disk-backed job queue for reliability |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        INFRAKNIT UBUNTU AGENT ARCHITECTURE                   │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │   BACKEND       │
                              │   SERVER        │
                              └────────┬────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
            ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
            │   Ingest     │   │   Jobs       │   │  APT Catalog │
            │   Endpoint   │   │   Endpoint   │   │  (Static)    │
            └──────────────┘   └──────────────┘   └──────────────┘
                    ▲                  │                  │
                    │                  ▼                  ▼
┌───────────────────┴──────────────────────────────────────────────────────────┐
│                              UBUNTU AGENT                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         SUPERVISOR (core/supervisor.py)              │    │
│  │                                                                      │    │
│  │   Main control loop that orchestrates all agent operations          │    │
│  │   - Schedules data collection at intervals                          │    │
│  │   - Polls for jobs                                                  │    │
│  │   - Sends heartbeat                                                 │    │
│  │   - Flushes ingest queue                                            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│           │              │              │              │                     │
│           ▼              ▼              ▼              ▼                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │  COLLECTORS  │ │   INGEST     │ │   JOBS       │ │  EXECUTOR    │        │
│  │              │ │   SENDER     │ │   POLLER     │ │              │        │
│  │ - metrics    │ │              │ │              │ │ - precheck   │        │
│  │ - snapshot   │ │ Disk-backed  │ │ Poll server  │ │ - disk_check │        │
│  │ - inventory  │ │ queue for    │ │ for pending  │ │ - apt source │        │
│  │ - hardware   │ │ reliability  │ │ jobs         │ │ - install    │        │
│  │ - os_info    │ │              │ │              │ │ - report     │        │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘        │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         INSTALLERS (executor/installers/)            │    │
│  │                                                                      │    │
│  │   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌───────────┐           │    │
│  │   │  DEB    │   │  APT    │   │ SCRIPT  │   │ UNINSTALL │           │    │
│  │   │ (dpkg)  │   │(apt-get)│   │ (bash)  │   │ (apt-get) │           │    │
│  │   └─────────┘   └─────────┘   └─────────┘   └───────────┘           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
1. COLLECTION FLOW:
   Collectors → Supervisor → Ingest Queue (disk) → Flush → Server /ingest

2. JOB FLOW:
   Server → Poller → Job Queue (disk) → Executor → Status Reports → Server

3. HEARTBEAT FLOW:
   Supervisor → Heartbeat → Server /heartbeat (every 60s)

4. APT INSTALL FLOW (packages):
   Job received → Configure per-app APT source (packages/) → apt-get update → apt-get install

5. APT PATCH FLOW (patches):
   Job received → Configure per-app APT source (patches/) → apt-get update → apt-get install
```

---

## Installation

### Prerequisites

- Ubuntu 20.04+ / Debian 11+ (64-bit)
- Python 3.8+
- Root privileges (for system operations)

### Dependencies

```bash
# System packages
sudo apt-get update
sudo apt-get install -y python3 python3-pip dmidecode

# Python packages
pip3 install psutil requests
```

### Quick Install

```bash
# 1. Clone or copy agent files
sudo mkdir -p /opt/infraknit-agent
sudo cp -r agent-ubuntu/* /opt/infraknit-agent/

# 2. Validate imports before running (RECOMMENDED)
cd /opt/infraknit-agent
python3 validate_agent.py

# 3. Create configuration
sudo mkdir -p /etc/infraknit-agent
sudo nano /etc/infraknit-agent/config.json
# Add: {"server_base": "http://YOUR-SERVER:9000", "master_key": "YOUR_KEY"}

# 4. Install systemd service
sudo tee /etc/systemd/system/infraknit-agent.service > /dev/null <<EOF
[Unit]
Description=InfraKnit Ubuntu Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/infraknit-agent/main.py --daemon
WorkingDirectory=/opt/infraknit-agent
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable infraknit-agent
sudo systemctl start infraknit-agent
```

> **Note**: The agent must run as root (`sudo`) because it creates directories under `/opt/`, `/etc/`, `/var/lib/`, and `/var/log/`, and installs system packages via `apt-get`. If run without root, it falls back to a local `.infraknit-agent/` directory for development purposes only.

### Manual Installation

```bash
# Create directories (requires root)
sudo mkdir -p /opt/infraknit-agent
sudo mkdir -p /etc/infraknit-agent
sudo mkdir -p /var/log/infraknit-agent
sudo mkdir -p /var/lib/infraknit-agent/{state,jobs,packages,scripts}
sudo mkdir -p /var/lib/infraknit-agent/jobs/{queue,running,completed,failed}
sudo mkdir -p /var/lib/infraknit-agent/packages/cache
sudo mkdir -p /var/lib/infraknit-agent/state/{pending_ingest,rollback,backups}
sudo mkdir -p /var/lib/infraknit-agent/scripts/temp

# Copy files
sudo cp -r * /opt/infraknit-agent/
sudo cp config.json /etc/infraknit-agent/

# Set permissions
sudo chmod +x /opt/infraknit-agent/main.py
sudo chown -R root:root /opt/infraknit-agent

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable infraknit-agent
sudo systemctl start infraknit-agent
```

---

## Configuration

### config.json

Located at `/etc/infraknit-agent/config.json` (system) or `./config.json` (development):

```json
{
  "server_base": "https://infraknit.example.com:9000",
  "verify_tls": false,
  "metrics_interval": 60,
  "snapshot_interval": 300,
  "inventory_interval": 86400,
  "master_key": "YOUR_MASTER_KEY",
  "agent_name": "ubuntu-server-01",
  "location": null
}
```

### Configuration Options

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `server_base` | string | required | Backend server URL |
| `verify_tls` | bool | false | Verify SSL certificates |
| `metrics_interval` | int | 60 | Seconds between metrics collection |
| `snapshot_interval` | int | 300 | Seconds between snapshot collection |
| `inventory_interval` | int | 86400 | Seconds between inventory collection |
| `master_key` | string | required | Bootstrap registration key |
| `agent_name` | string | optional | Agent display name |
| `location` | string | auto | Physical location (set by server) |

### Configuration Loading Priority

1. Environment variable: `INFRAKNIT_CONFIG=/path/to/config.json`
2. System config: `/etc/infraknit-agent/config.json`
3. Local config: `./config.json` (development)

---

## Directory Structure

### Source Code

```
agent-ubuntu/
├── main.py                      # Entry point
├── config.json                  # Configuration file (dev)
├── requirements.txt             # Python dependencies
├── uninstall-agent.py           # Full uninstall script
│
├── config/
│   ├── __init__.py
│   └── loader.py                # Config file loader
│
├── core/
│   ├── __init__.py
│   └── supervisor.py            # Main control loop
│
├── identity/
│   ├── __init__.py
│   └── agent_id.py              # Persistent agent ID generation
│
├── security/
│   ├── __init__.py
│   ├── auth_state.py            # Runtime auth state
│   └── bootstrap.py             # First-run bootstrap
│
├── network/
│   ├── __init__.py
│   ├── http_client.py           # HTTP wrapper (get_json, post_json)
│   └── heartbeat.py             # Heartbeat sender
│
├── ingest/
│   ├── __init__.py
│   └── sender.py                # Disk-backed ingest queue
│
├── collectors/
│   ├── __init__.py              # Exports all collectors
│   ├── metrics.py               # CPU, RAM, disk, network (psutil)
│   ├── inventory.py             # Software inventory (dpkg-query)
│   ├── snapshot.py              # Runtime state aggregator
│   ├── hardware.py              # Hardware info (dmidecode)
│   └── os_info.py               # Ubuntu version, kernel, user
│
├── jobs/
│   ├── __init__.py              # Job module exports
│   ├── model.py                 # Job schema normalization
│   ├── store.py                 # Disk-backed job storage
│   ├── state.py                 # Job state transitions + JobState class
│   └── poller.py                # Job polling from server
│
├── executor/
│   ├── __init__.py              # Executor exports
│   ├── engine.py                # Main job execution engine
│   ├── precheck.py              # Pre-install version check (dpkg-query)
│   ├── disk_check.py            # Disk space verification
│   ├── downloader.py            # Package downloader
│   ├── status_reporter.py       # Real-time status updates
│   ├── reporter.py              # Final result reporter
│   ├── system_commands.py       # Reboot/shutdown commands
│   └── installers/
│       ├── __init__.py
│       ├── deb.py               # DEB installer (dpkg -i)
│       ├── apt.py               # APT installer (apt-get)
│       ├── script.py            # Shell script executor
│       └── uninstall.py         # Package uninstaller
│
└── utils/
    ├── __init__.py
    ├── fs.py                    # Directory utilities
    └── exec.py                  # Command execution helpers
```

### Runtime Directories

```
/opt/infraknit-agent/                   # Installation directory
├── main.py                             # Entry point
├── [source files]                      # Agent source code

/etc/infraknit-agent/                   # Configuration directory
├── config.json                         # Main configuration

/var/log/infraknit-agent/               # Log directory
├── agent.log                           # Main log file

/var/lib/infraknit-agent/               # Data directory
├── agent_id                            # Persistent agent UUID
├── token.json                          # Auth token storage
├── bootstrap.json                      # Bootstrap secret
│
├── state/
│   ├── pending_ingest/                 # Queued telemetry payloads
│   ├── rollback/                       # Rollback state files
│   └── backups/                        # File backups
│
├── jobs/
│   ├── queue/                          # Pending jobs (*.json)
│   ├── running/                        # Currently executing
│   ├── completed/                      # Successful jobs
│   └── failed/                         # Failed jobs
│
├── packages/
│   └── cache/                          # Downloaded packages
│       └── {package_id}/               # Per-package directory
│
└── scripts/
    └── temp/                           # Temporary script files
```

---

## Core Components

### Supervisor (`core/supervisor.py`)

The central control loop that orchestrates all agent operations.

### Collection Schedule

| Data Type | Interval | Payload Type |
|-----------|----------|--------------|
| Metrics | 60s | `type: "metrics"` |
| Snapshot | 300s | `type: "snapshot"` |
| Inventory | 86400s | `type: "inventory"` |
| Hardware | 86400s | `type: "hardware"` |
| OS Info | 86400s | `type: "os_info"` |
| Heartbeat | 60s | Direct POST |

### Job Executor (`executor/engine.py`)

Handles all job execution with pre-checks and status reporting:

- **install**: Package installation (APT from per-app repo, Script)
- **uninstall**: Package removal (apt-get remove/purge)
- **patch**: Linux .deb patches via per-app APT repo OR system APT upgrade
- **run_script**: Shell script execution (bash/python)
- **reboot**: System reboot (shutdown -r)
- **shutdown**: System shutdown (shutdown -h)
- **cancel_shutdown**: Cancel pending operation (shutdown -c)
- **collect_inventory**: Force inventory collection
- **agent_update**: Self-update (planned)

---

## Data Collection

### Metrics (`collectors/metrics.py`)

Collected every 60 seconds using `psutil`.

```json
{
  "type": "metrics",
  "agent_id": "uuid",
  "location": "Building A",
  "collected_at": "2025-02-05T10:00:00Z",
  "cpu_percent": 45.2,
  "memory": {
    "total": 17179869184,
    "available": 8589934592,
    "percent": 50.0,
    "used": 8589934592
  },
  "disk": [
    {
      "mountpoint": "/",
      "total": 500107862016,
      "used": 250053931008,
      "free": 250053931008,
      "percent": 50.0
    }
  ],
  "net": {
    "bytes_sent": 1234567890,
    "bytes_recv": 9876543210
  },
  "boot_time": 1706345600.0,
  "load_average": [1.5, 1.2, 0.9]
}
```

### Inventory (`collectors/inventory.py`)

Collected every 24 hours using `dpkg-query`.

```json
{
  "type": "inventory",
  "agent_id": "uuid",
  "location": "Building A",
  "collected_at": "2025-02-05T10:00:00Z",
  "software": [
    {
      "name": "nginx",
      "version": "1.18.0-6ubuntu14.4",
      "install_date": "2025-01-15"
    }
  ]
}
```

### Hardware (`collectors/hardware.py`)

Collected every 24 hours using `dmidecode` and `/sys` filesystem.

```json
{
  "type": "hardware",
  "agent_id": "uuid",
  "location": "Building A",
  "collected_at": "2025-02-05T10:00:00Z",
  "hardware": {
    "system": {
      "manufacturer": "Dell Inc.",
      "model": "PowerEdge R640",
      "serial_number": "ABC1234"
    },
    "cpu": {
      "model": "Intel(R) Xeon(R) Gold 6230 CPU @ 2.10GHz",
      "physical_cores": 20,
      "logical_cores": 40,
      "architecture": "x86_64"
    },
    "memory": { "total_gb": 128 },
    "storage": [
      { "device": "/dev/sda", "model": "SAMSUNG MZ7L3960", "size_gb": 960 }
    ]
  }
}
```

### OS Info (`collectors/os_info.py`)

Collected every 24 hours from `/etc/os-release` and system commands.

```json
{
  "type": "os_info",
  "agent_id": "uuid",
  "location": "Building A",
  "collected_at": "2025-02-05T10:00:00Z",
  "hostname": "ubuntu-server-01",
  "os": {
    "name": "Ubuntu",
    "version": "22.04.3 LTS",
    "codename": "jammy",
    "kernel": "5.15.0-91-generic",
    "architecture": "x86_64"
  },
  "uptime_seconds": 1234567,
  "last_boot": "2025-01-20T08:00:00Z"
}
```

---

## Job Execution

### Job Types

| Type | Description | Installer Used |
|------|-------------|----------------|
| `install` | Install software | APT (per-app repo) or Script |
| `uninstall` | Remove software | APT (apt-get remove/purge) |
| `patch` | Apply patch / upgrade package | APT (per-app repo for .deb patches, system apt-get for upgrades) |
| `run_script` | Execute shell script | Bash/Python interpreter |
| `reboot` | Restart system | shutdown -r |
| `shutdown` | Power off system | shutdown -h |
| `cancel_shutdown` | Cancel pending operation | shutdown -c |
| `collect_inventory` | Force inventory collection | Internal |
| `agent_update` | Update agent itself | Script-based (planned) |

### Job Execution Flow

```
1. JOB RECEIVED
   ├── Report: "Job received: install for 'nginx'"
   └── Move job to: jobs/running/

2. PRE-CHECK (for install/patch)
   ├── Run: dpkg-query -W -f='${Status}|${Version}' nginx
   ├── Check if software is installed
   ├── Get current version
   ├── Compare with target version
   └── Report status:
       ├── "Not installed - will perform fresh install"
       ├── "Version 1.17.0 is outdated - upgrading to 1.18.0"
       └── "Already at target version - skipping"

3. APT SOURCE CONFIGURATION (for per-app repo installs)
   ├── Write /etc/apt/sources.list.d/infraknit-{app_name}.list
   │   Content: deb [trusted=yes arch=amd64] http://server:9000/apt-catalog/ubuntu/{codename}/{content_type}/{app_name} {codename} main
   ├── content_type = "packages" (for install jobs) or "patches" (for patch jobs)
   └── Run targeted apt-get update for that source only

4. INSTALL
   ├── APT: apt-get install -y {package_name}={version}
   ├── Script: bash /tmp/script.sh
   └── Report: "Installing 'nginx' using apt"

5. RESULT
   ├── SUCCESS:
   │   ├── Report: "Successfully installed in 45 seconds"
   │   ├── Move job to: jobs/completed/
   │   └── POST /api/v1/jobs/{job_id}/complete
   │
   └── FAILURE:
       ├── Report: "Installation failed: Exit code 1"
       ├── Move job to: jobs/failed/
       └── POST /api/v1/jobs/{job_id}/complete {status: "failed"}
```

### Status Values

| Status | Description |
|--------|-------------|
| `received` | Job received by agent |
| `checking` | Checking current state |
| `downloading` | Downloading package |
| `installing` | Running installer / configuring APT source |
| `completed` | Successfully completed |
| `failed` | Installation failed |
| `rollback_started` | Starting rollback |
| `rollback_completed` | Rollback succeeded |
| `rollback_failed` | Rollback failed |
| `skipped` | Already at target version |
| `in_progress` | Script execution in progress |
| `rebooting` | System reboot scheduled |
| `shutting_down` | System shutdown scheduled |
| `cancelled` | Pending operation cancelled |

---

## APT Repository Integration

The agent integrates with InfraKnit's per-app per-codename APT repository structure hosted by the backend server.

### Repository Structure (Server-Side)

```
apt-catalog/ubuntu/{codename}/
  ├── packages/{app-name}/          ← Software packages
  │   ├── pool/{file}.deb
  │   └── dists/{codename}/main/binary-amd64/{Packages,Packages.gz,Release}
  └── patches/{patch-name}/         ← Security patches
      ├── pool/{file}.deb
      └── dists/{codename}/main/binary-amd64/{Packages,Packages.gz,Release}
```

### How Package Install Works

1. Backend creates an `install` job with `installer_type=apt`, `ubuntu_codename`, `repo_app_name`
2. Agent receives job via poll
3. Agent writes `/etc/apt/sources.list.d/infraknit-{app_name}.list`:
   ```
   deb [trusted=yes arch=amd64] http://server:9000/apt-catalog/ubuntu/noble/packages/nginx noble main
   ```
4. Agent runs targeted `apt-get update` for that source only
5. Agent runs `apt-get install -y {package_name}` (or `={version}` if specified)

### How Patch Install Works

1. Backend creates a `patch` job with `installer_type=apt`, `ubuntu_codename`, `repo_app_name`
2. Agent receives job via poll
3. Agent writes `/etc/apt/sources.list.d/infraknit-{app_name}.list`:
   ```
   deb [trusted=yes arch=amd64] http://server:9000/apt-catalog/ubuntu/noble/patches/openssl noble main
   ```
4. Agent runs targeted `apt-get update` for that source only
5. Agent runs `apt-get install -y {package_name}` (with the patched version from the repo)

### Supported Ubuntu Codenames

| Codename | Ubuntu Version |
|----------|---------------|
| `focal` | 20.04 LTS |
| `jammy` | 22.04 LTS |
| `noble` | 24.04 LTS |
| `oracular` | 24.10 |
| `plucky` | 25.04 |

---

## Pre-Installation Intelligence

### Version Checking (`executor/precheck.py`)

Before any install/patch operation, the agent checks current state using `dpkg-query`:

```python
status = check_software_status("nginx", target_version="1.18.0")
# Returns:
# {
#   "is_installed": True,
#   "current_version": "1.17.0-6ubuntu14",
#   "display_name": "nginx",
#   "comparison": "older",      # same, older, newer, unknown
#   "action_needed": "upgrade", # none, install, upgrade, downgrade
#   "message": "'nginx' version 1.17.0 is older than target 1.18.0"
# }
```

### Decision Matrix

| Current State | Target Version | Action |
|--------------|----------------|--------|
| Not installed | Any | Fresh install |
| Installed, same version | Same | Skip (already done) |
| Installed, older | Newer | Upgrade |
| Installed, newer | Older | Downgrade (with warning) |
| Unknown version | Any | Install (best effort) |

---

## Disk Space Management

### Pre-Download Check (`executor/disk_check.py`)

Before downloading any package, the agent verifies sufficient disk space:

| Component | Requirement | Notes |
|-----------|-------------|-------|
| Minimum free | 500 MB | Always enforced |
| Download buffer | 2.5x file size | For extraction/temp files |
| Install location | 500 MB | If on different partition |

---

## API Endpoints & JSON Formats

### Endpoints Used by Agent

| Method | Endpoint | Purpose | Interval |
|--------|----------|---------|----------|
| POST | `/api/v1/agents/register_bootstrap` | Initial registration | Once |
| POST | `/api/v1/agents/ingest` | Send telemetry | varies |
| POST | `/api/v1/agents/heartbeat` | Alive signal | 60s |
| GET | `/api/v1/jobs/poll` | Poll for jobs | 30s |
| GET | `/api/v1/packages/{id}/download` | Download package | On-demand |
| POST | `/api/v1/jobs/{id}/status` | Real-time status | During job |
| POST | `/api/v1/jobs/{id}/complete` | Report completion | On-demand |

---

### Bootstrap Registration

**Request:**
```http
POST /api/v1/agents/register_bootstrap
Content-Type: application/json
```
```json
{
  "agent_id": "550e8400-e29b-41d4-a716-446655440000",
  "bootstrap_secret": "abc123def456",
  "master_key": "INFRAKNIT_MASTER_KEY"
}
```

**Response:**
```json
{
  "agent_token": "eyJhbGciOiJIUzI1NiIs...",
  "location": "Building A - Floor 3"
}
```

---

### Heartbeat

**Request:**
```http
POST /api/v1/agents/heartbeat
Authorization: Bearer <token>
Content-Type: application/json
```
```json
{
  "agent_id": "uuid",
  "timestamp": "2025-02-05T10:00:00Z",
  "status": "online",
  "location": "Building A",
  "uptime_seconds": 123456,
  "hostname": "ubuntu-server-01",
  "agent_version": "1.0.0"
}
```

**Response:**
```json
{
  "status": "ok",
  "location": "Building A"
}
```

---

### Job Poll

**Request:**
```http
GET /api/v1/jobs/poll?limit=10
Authorization: Bearer <token>
```

**Response:**
```json
{
  "jobs": [
    {
      "job_id": "7ced81d6-abcd-1234-5678-abcdef123456",
      "job_type": "install",
      "parameters": "{\"service_pack_id\": 1}",
      "package_id": 10,
      "patch_id": null,
      "priority": 5,
      "timeout_seconds": 3600,
      "package_name": "nginx",
      "version": "1.24.0",
      "installer_type": "apt",
      "silent_args": null,
      "product_code": null,
      "target_build": null,
      "target_version": null,
      "delay_seconds": null,
      "reason": null,
      "force": false,
      "reboot_required": false,
      "ubuntu_codename": "noble",
      "repo_app_name": "nginx"
    }
  ]
}
```

#### Job fields by job type:

**install** (package installation):
| Field | Type | Description |
|-------|------|-------------|
| `job_type` | string | `"install"` |
| `package_name` | string | APT package name (e.g., `"nginx"`) |
| `version` | string | Target version (optional) |
| `installer_type` | string | `"apt"` for APT repos, `"sh"` for scripts |
| `ubuntu_codename` | string | `"focal"`, `"jammy"`, `"noble"`, etc. |
| `repo_app_name` | string | App name in APT catalog (e.g., `"nginx"`) |

**patch** (Linux .deb patch via APT or system upgrade):
| Field | Type | Description |
|-------|------|-------------|
| `job_type` | string | `"patch"` or `"install_patch"` (normalized to `"patch"`) |
| `package_name` | string | APT package name (e.g., `"openssl"`) |
| `version` | string | Target version (optional) |
| `installer_type` | string | `"apt"` for .deb patches from APT repo |
| `ubuntu_codename` | string | Ubuntu codename for the patch |
| `repo_app_name` | string | Package name in `patches/` subfolder |

**run_script** (script execution):
| Field | Type | Description |
|-------|------|-------------|
| `job_type` | string | `"run_script"` or `"execute_script"` (normalized to `"run_script"`) |
| `parameters` | string (JSON) | Contains `script_content`, `script_type`, `timeout` |

Parameters JSON for scripts:
```json
{
  "script_content": "#!/bin/bash\napt policy apparmor",
  "script_type": "bash",
  "timeout": 3600
}
```

**reboot** / **shutdown**:
| Field | Type | Description |
|-------|------|-------------|
| `job_type` | string | `"reboot"` or `"shutdown"` |
| `delay_seconds` | int | Delay before operation (default: 60) |
| `reason` | string | Human-readable reason |
| `force` | bool | Force operation |

**uninstall** (package removal):
| Field | Type | Description |
|-------|------|-------------|
| `job_type` | string | `"uninstall"` or `"uninstall_package"` (normalized to `"uninstall"`) |
| `package_name` | string | Package to remove |

---

### Job Status Update (Agent → Server)

**Request:**
```http
POST /api/v1/jobs/{job_id}/status
Authorization: Bearer <token>
Content-Type: application/json
```
```json
{
  "job_id": "7ced81d6-abcd-1234-5678-abcdef123456",
  "status": "installing",
  "message": "Installing 'nginx' using apt",
  "timestamp": "2025-02-05T10:30:45Z",
  "details": {
    "package_name": "nginx",
    "installer_type": "apt"
  }
}
```

---

### Job Completion (Agent → Server)

**Request:**
```http
POST /api/v1/jobs/{job_id}/complete
Authorization: Bearer <token>
Content-Type: application/json
```

**Success:**
```json
{
  "job_id": "7ced81d6-abcd-1234-5678-abcdef123456",
  "job_type": "install",
  "status": "completed",
  "exit_code": 0,
  "stdout": "nginx is already the newest version (1.24.0-2ubuntu7.1).\n",
  "stderr": "",
  "duration_sec": 12,
  "message": "Already at version 1.24.0-2ubuntu7.1"
}
```

**Failure:**
```json
{
  "job_id": "7ced81d6-abcd-1234-5678-abcdef123456",
  "job_type": "install",
  "status": "failed",
  "exit_code": 100,
  "stdout": "",
  "stderr": "E: Unable to locate package nonexistent-pkg",
  "duration_sec": 3,
  "message": "Execution error: apt-get install failed with exit code 100"
}
```

---

### Ingest (Telemetry)

**Request:**
```http
POST /api/v1/agents/ingest
Authorization: Bearer <token>
Content-Type: application/json
```
```json
{
  "type": "metrics",
  "agent_id": "uuid",
  "location": "Building A",
  "collected_at": "2025-02-05T10:00:00Z",
  "cpu_percent": 45.2,
  "memory": { ... },
  "disk": [ ... ],
  "net": { ... }
}
```

---

## Security

### Authentication Flow

```
1. First Run:
   - Generate agent_id (UUID5 based on hostname + MAC)
   - Generate bootstrap_secret (random UUID)
   - Send to server with master_key
   - Receive agent_token (JWT)

2. Token Storage:
   - File: /var/lib/infraknit-agent/token.json
   - Permissions: 600 (root only)

3. Request Authentication:
   - All requests include: Authorization: Bearer <token>
```

### Security Features

| Feature | Implementation |
|---------|----------------|
| Token storage | File with 600 permissions |
| TLS verification | Configurable via `verify_tls` |
| Master key | Required for bootstrap registration |
| Agent identity | UUID5 based on hostname + primary MAC |
| Root execution | Agent runs as root for system access |

---

## Logging

### Log Locations

| File | Purpose |
|------|---------|
| `/var/log/infraknit-agent/agent.log` | Main application log |
| `journalctl -u infraknit-agent` | Systemd journal |

### Log Format

```
2025-02-05 10:30:00 [INFO] Heartbeat sent
2025-02-05 10:30:05 [INFO] [Job abc12345] received: install for 'nginx'
2025-02-05 10:30:06 [INFO] [Job abc12345] checking: 'nginx' is not installed
2025-02-05 10:30:45 [INFO] [Job abc12345] completed: Successfully installed in 39 seconds
```

### Viewing Logs

```bash
# View systemd logs
sudo journalctl -u infraknit-agent -f

# View log file
sudo tail -f /var/log/infraknit-agent/agent.log
```

---

## Systemd Service

### Service Management

```bash
# Start service
sudo systemctl start infraknit-agent

# Stop service
sudo systemctl stop infraknit-agent

# Restart service
sudo systemctl restart infraknit-agent

# Check status
sudo systemctl status infraknit-agent

# Enable auto-start
sudo systemctl enable infraknit-agent

# View logs
sudo journalctl -u infraknit-agent -f
```

---

## Uninstalling

To fully remove the agent and all its data:

```bash
sudo python3 /opt/infraknit-agent/uninstall-agent.py
```

Or if the agent isn't installed in `/opt/`:

```bash
sudo python3 uninstall-agent.py
```

This script will:
- Stop and disable the systemd service
- Remove the systemd unit file
- Delete all agent directories (`/opt/infraknit-agent`, `/etc/infraknit-agent`, `/var/lib/infraknit-agent`, `/var/log/infraknit-agent`)
- Remove InfraKnit APT source files from `/etc/apt/sources.list.d/`
- Run `systemctl daemon-reload`

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Service won't start | Check `journalctl -u infraknit-agent` |
| No data in server | Verify `server_base` URL and network |
| Auth failed | Delete `/var/lib/infraknit-agent/token.json`, restart |
| Jobs not executing | Check `/var/lib/infraknit-agent/jobs/queue/` |
| Permission denied | Ensure agent runs as root (`sudo`) |
| dpkg lock | Wait for other apt/dpkg processes |
| Directories not created | Must run as root: `sudo python3 main.py` |
| APT source 404 | Verify codename matches the uploaded package/patch |

### Debug Mode

```bash
# Run interactively (must be root)
cd /opt/infraknit-agent
sudo python3 main.py
```

### Health Checks

```bash
# Check if running
sudo systemctl is-active infraknit-agent

# Check agent ID
cat /var/lib/infraknit-agent/agent_id

# Check pending jobs
ls -la /var/lib/infraknit-agent/jobs/queue/

# Check disk space
df -h /var/lib/infraknit-agent

# Check APT sources created by agent
ls -la /etc/apt/sources.list.d/infraknit-*
```

---

## Windows vs Ubuntu Agent Comparison

### Platform Comparison

| Aspect | Windows Agent | Ubuntu Agent |
|--------|---------------|--------------|
| **Platform** | Windows 10/11 (64-bit) | Ubuntu 20.04+ / Debian 11+ |
| **Service Type** | Windows Service (pywin32) | Systemd Service |
| **Language** | Python 3.10+ | Python 3.8+ |
| **Install Location** | `C:\Program Files\InfraKnit\` | `/opt/infraknit-agent/` |
| **Config Location** | Same as install | `/etc/infraknit-agent/` |
| **Data Location** | `C:\Program Files\InfraKnit\agent\` | `/var/lib/infraknit-agent/` |
| **Log Location** | `C:\...\agent\logs\` | `/var/log/infraknit-agent/` |

### Package Management Comparison

| Aspect | Windows Agent | Ubuntu Agent |
|--------|---------------|--------------|
| **Package Formats** | EXE, MSI, MSU, PS1 | APT (per-app repo), Shell scripts |
| **Patch Formats** | MSU, CAB, EXE | .deb via per-app APT repo |
| **Installer Tools** | msiexec, PowerShell | apt-get, bash |
| **Version Check** | Registry + WMI | dpkg-query |
| **Uninstall** | msiexec /x, registry | apt-get remove/purge |
| **Silent Install** | /S, /quiet, /qn | DEBIAN_FRONTEND=noninteractive |

### Feature Availability

| Feature | Windows | Ubuntu |
|---------|:-------:|:------:|
| Asset Collection | Yes | Yes |
| Job Execution | Yes | Yes |
| Pre-install Checks | Yes | Yes |
| Disk Space Check | Yes | Yes |
| Status Reporting | Yes | Yes |
| Heartbeat | Yes | Yes |
| Offline Queue | Yes | Yes |
| Remote Reboot/Shutdown | Yes | Yes |
| Per-App APT Repos | N/A | Yes |
| Linux .deb Patches | N/A | Yes |
| Service Packs | Yes | Yes |
| File Rollback | Yes | No (planned) |
| Self-Update | Yes | No (planned) |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-02 | Initial release with full job execution |
| 1.1.0 | 2026-02 | Per-app per-codename APT repository integration, Linux .deb patch support, Service Pack patch items, Script execution fix (JSON parameters parsing) |

---

## License

Proprietary - InfraKnit

---

## Support

For issues and feature requests, contact the InfraKnit team.
