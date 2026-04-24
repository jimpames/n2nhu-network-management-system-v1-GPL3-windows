<div align="center">

24 apr 26   installers and binaries coming soon

demo videos

https://youtu.be/uZA-syheo3E

https://youtu.be/dcGKuFYmUBw


# N2NHU NMS

**A self-hosted remote monitoring and management platform for Windows networks.**
Deploy MSIs, collect inventory, manage accounts, and audit operations across your fleet — from one console, no cloud, no per-endpoint licensing.

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20server-lightgrey.svg)]()
[![Tests](https://img.shields.io/badge/tests-12%20passing-brightgreen.svg)]()
[![Version](https://img.shields.io/badge/version-12.2-blue.svg)]()

</div>

---

## What this is

N2NHU NMS is an open-source alternative to commercial RMMs like NinjaOne, Atera, and ConnectWise Automate for small-to-medium Windows shops. Two Python programs — a server and a lightweight agent — give one administrator centralized control over a fleet of endpoints: push MSI packages, uninstall software, collect hardware and software inventory, manage user accounts, and maintain a full audit trail. Everything runs on your own hardware. Nothing phones home. No subscription.

It's built on a simple TCP protocol, a single SQLite database, and about 7,000 lines of readable Python. One administrator can understand the whole codebase in an afternoon.

## Quickstart

```bash
# Server
git clone https://github.com/n2nhu/nms.git
cd nms/server
pip install -r requirements.txt
python v12-nms-server-PRODUCTION.py

# Client (on each managed endpoint)
cd ../client
pip install cryptography colorama
# Edit nms_client_config.ini → point at your server's host:port
python v12_1-nms-client-PRODUCTION.py
```

The server drops you into an interactive console menu. Agents connect outbound and register automatically. First client should appear within a few seconds under **View Client List**.

Full install and configuration guide in [`docs/NMS_v12_2_User_Manual.docx`](docs/NMS_v12_2_User_Manual.docx).

## Table of contents

- [Why this exists](#why-this-exists)
- [Features](#features)
- [How it compares](#how-it-compares)
- [Architecture](#architecture)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the tests](#running-the-tests)
- [Security](#security)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

## Why this exists

Commercial RMMs start around $3–$5 per endpoint per month. A shop with 100 endpoints pays $4,000–$6,000 per year. For classroom labs, hobbyist fleets, and small IT shops that value data sovereignty, that math doesn't work — and the bundled features (remote desktop, PSA, ticketing) are often things they already solve with other tools.

N2NHU NMS covers the core RMM tasks cleanly: inventory, deploy, uninstall, account management, audit. It does not try to compete with commercial suites on surface feature count. It wins on cost, auditability, and the fact that the operator actually owns their data.

## Features

**Fleet visibility**
- Auto-discovery of every connected client with hostname, IP, MAC, OS, domain
- Real-time online/offline status via heartbeat
- Per-client hardware summary (CPU, RAM, disk)
- Full software inventory via WMIC (name, version, vendor, install date)

**Software management**
- MSI import and one-click deployment to any client
- SHA-256 verified chunked transfer on a dedicated channel
- One-time per-deployment authorization tokens
- Silent software uninstall using the client's own inventory

**Account management**
- Create and delete Windows user accounts remotely
- Domain-joined or local — credentials routed automatically

**Audit & compliance**
- Append-only operations log with exit codes, durations, and timestamps
- SQLite backing store readable by any SQL tool
- Auto-rotating database backups on startup

**Security posture**
- Credentials held in server memory only — never written to disk
- Fernet-encrypted (AES-128-CBC + HMAC-SHA256) credential push to clients
- Framing caps (4 MB per message) against adversarial clients
- Identity-aware session handling — clean reconnect eviction

## How it compares

| Feature | **N2NHU NMS** | NinjaOne | Atera | TacticalRMM | ConnectWise Automate |
|---|:---:|:---:|:---:|:---:|:---:|
| Software/hardware inventory | ✅ | ✅ | ✅ | ✅ | ✅ |
| MSI package deployment | ✅ | ✅ | ✅ | ✅ | ✅ |
| Windows user management | ✅ | ✅ | ✅ | via script | ✅ |
| Operation audit log | ✅ | ✅ | ✅ | ✅ | ✅ |
| Self-hosted, no cloud | ✅ | ❌ | ❌ | ✅ | optional |
| Credentials in memory only | ✅ | ❌ | ❌ | ❌ | ❌ |
| Readable end-to-end by one person | ✅ *(~7k lines)* | ❌ | ❌ | partial | ❌ |
| No licensing fees | ✅ *(GPL)* | ❌ | ❌ | ✅ | ❌ |
| Remote desktop / shell | ❌ | ✅ | ✅ | ✅ *(MeshCentral)* | ✅ |
| Automated patch management | manual via MSI | ✅ | ✅ | ✅ | ✅ |
| Real-time alerting/monitoring | ❌ | ✅ | ✅ | ✅ | ✅ |
| Ticketing / PSA | ❌ | add-on | ✅ | ❌ | ✅ |
| AI-assisted ticketing | ❌ | ✅ | ✅ *(Copilot)* | ❌ | limited |
| **Cost @ 50 endpoints/year** | **$0** | ~$3,200 | $1,140/tech | $0 | custom |
| Deployment time (first endpoint) | ~15 min | ~30 min | ~30 min | 30–60 min | days |

**Choose NMS if:** you run ≤500 Windows endpoints, want no licensing cost, need data sovereignty, and the core RMM operations cover your needs. **Choose a commercial RMM if:** you need real-time alerting, remote desktop, MSP-scale multi-tenancy, or integrated ticketing. Both are honest answers to different situations.

See the [User Manual](docs/NMS_v12_2_User_Manual.docx) for the full feature matrix with 25+ rows.

## Architecture

```
┌────────────────────────┐        TCP :2222 (JSON/newline)      ┌──────────────────┐
│                        │ <──────────────────────────────────> │                  │
│   NMS Server           │        TCP :2223 (chunked transfer)  │   Windows Agent  │
│                        │ <──────────────────────────────────> │                  │
│   ┌──────────────────┐ │                                      │  ┌─────────────┐ │
│   │  Control plane   │ │                                      │  │  Handshake  │ │
│   │  Thread per conn │ │                                      │  │  Heartbeat  │ │
│   └──────────────────┘ │                                      │  │  WMIC scan  │ │
│   ┌──────────────────┐ │                                      │  │  MSI exec   │ │
│   │  Transfer plane  │ │                                      │  │  Cred vault │ │
│   │  SHA-256 chunks  │ │                                      │  └─────────────┘ │
│   └──────────────────┘ │                                      │                  │
│   ┌──────────────────┐ │                                      └──────────────────┘
│   │  SQLite DB       │ │
│   │  ├── clients     │ │
│   │  ├── software    │ │
│   │  ├── operations  │ │
│   │  └── packages    │ │
│   └──────────────────┘ │
│   ┌──────────────────┐ │
│   │  Credential mgr  │ │
│   │  Fernet in-RAM   │ │
│   └──────────────────┘ │
└────────────────────────┘
```

**Control channel (:2222)** — Newline-delimited JSON. Handshakes, heartbeats, inventory reports, operation commands and results. 4 MB per-message cap. Thread-per-client with individual message queues; slow clients don't block others.

**Transfer channel (:2223)** — Length-prefixed JSON handshake + binary chunks. Dedicated to MSI payloads. SHA-256 verified end-to-end. One-time tokens per deployment.

**State** — Single SQLite database. Schema is forward-compatible (CREATE IF NOT EXISTS + ALTER TABLE ADD COLUMN). Auto-backup keeps rolling snapshots.

## Installation

### Requirements

- **Server:** Windows 10/11 or Windows Server 2016+, or Linux with Python 3.10+. ~100 MB disk plus database and package storage.
- **Client:** Windows 7+ or Server 2008 R2+, Python 3.10+, local admin (required for WMIC, MSI install, account operations).

### Server setup

```bash
cd server
pip install -r requirements.txt
# Edit nms_server_config.ini: bind address, ports, credentials policy
python v12-nms-server-PRODUCTION.py
```

On first launch the server prompts for domain and local admin credentials. These live in memory only — a server restart re-prompts. Set `skip_credential_prompt = true` under `[credentials]` for non-interactive starts (inventory and deploy will be disabled until credentials are provided).

### Client setup

```bash
cd client
pip install cryptography colorama
# Edit nms_client_config.ini: server host, port, heartbeat interval
python v12_1-nms-client-PRODUCTION.py
```

For production, install the agent as a Windows service under an administrator account using [NSSM](https://nssm.cc/) or similar. Agents run fine interactively for testing.

## Configuration

All server behavior is controlled by `nms_server_config.ini`:

```ini
[server]
host = 0.0.0.0
port = 2222
transfer_port = 2223

[database]
path = network_management.db
auto_backup = true
backup_dir = backups

[credentials]
skip_credential_prompt = false

[deployables]
packages_dir = deployables
max_package_size_mb = 500

[performance]
max_clients = 1000
client_timeout = 300
```

All client behavior via `nms_client_config.ini`:

```ini
[server]
host = nms.your-company.com
port = 2222

[client]
reconnect_interval = 5
heartbeat_interval = 30
```

See the [User Manual](docs/NMS_v12_2_User_Manual.docx) for every key, default, and purpose.

## Running the tests

```bash
pip install pytest cryptography colorama
python -m pytest tests/ -v
```

Expected output: **12 passed in ~13 seconds.** Tests launch the real server and agent as threads inside pytest, exercise the wire protocol on ephemeral ports, and assert outcomes. No mocks, no subprocess orchestration.

Coverage:
- Server boot on configured ports
- Transfer port honors INI (regression against the old hardcoded-2223 bug)
- Client registration and system info
- Reconnect cleanly evicts stale session (regression against silent record clobber)
- Heartbeat round-trip updates server-side state
- Hard disconnect detected within 3 seconds
- Inventory report writes to SQLite correctly
- Server survives garbage JSON on handshake
- Server survives oversized payloads without memory exhaustion

## Security

**Threat model.** NMS assumes a trusted network between server and clients — LAN or VPN-bridged. It is not yet designed for direct exposure to the open Internet.

**What's protected:**
- Credentials are held server-side in memory only, Fernet-encrypted when pushed to clients, and never persisted to disk
- MSI package integrity via SHA-256 verification before installation
- Per-deployment one-time transfer tokens
- 4 MB per-message framing cap defends against adversarial or buggy clients

**What's not protected in v12.2:**
- Control-channel traffic is plaintext JSON. TLS is planned for v13 and will be opt-in via config.
- Client IDs derive from MAC address. Network-level isolation is the current protection against impersonation.
- No built-in rate limiting on handshakes. Use firewall rules.

**Recommended deployment:** behind a corporate firewall, on the same LAN or reachable only through a VPN. Administrator credentials pushed to clients should be minimally scoped — a dedicated local admin account per fleet, not your personal domain admin.

Report security issues privately to `security@n2nhu.example` (replace with your real contact before publishing). Please do not open public issues for security-sensitive findings.

## Roadmap

**v12.3** — Pcap-style operation logging, health endpoint for monitoring integration, structured JSON logs.

**v13** — TLS for the control and transfer channels with mutual authentication, rate limiting, hardening for Internet-exposed deployments.

**v13.x** — Pure-PowerShell inventory path (to replace WMIC, which Microsoft has deprecated on newer Windows builds). Cross-platform agent (Linux/macOS inventory + package deployment using apt/brew).

**v14** — Optional Web UI alongside the existing console menu. REST API. Multi-operator support with per-operator audit trails.

Long-term, non-goals:
- We will not add ticketing / PSA. Integrate with an existing helpdesk if needed.
- We will not add TLS interception or SSL-inspecting proxy features. Modern traffic is encrypted end-to-end and that's a good thing.
- We will not add "application categorization" / L7 firewall features to the NMS. Use a proper firewall alongside it.

## Contributing

Pull requests and issues welcome. Before submitting:

1. Run `python -m pytest tests/ -v` — all tests must pass.
2. For new behavior, add a test. The integration harness in `tests/test_nms_integration.py` shows the pattern; use `NMSTestEnv` to spin up server + client in-process.
3. Keep changes focused. One bug or one feature per PR.
4. Match the existing coding style (no linter config shipped; just read the surrounding code).

Areas where help is especially welcome:

- **TLS implementation for v13** — wrap the control and transfer sockets with `ssl.wrap_socket` and add cert/key config.
- **Non-Windows clients** — a Linux agent that reports installed packages via `dpkg` or `rpm` and can run a shell-script "deployable".
- **Web UI** — a Flask or FastAPI companion that mirrors the console menu. The existing architecture is already a clean server library; a web frontend is mostly presentation work.

## License

GPLv3. See [`LICENSE`](LICENSE) for the full text.

The short version: you can use, modify, and redistribute this software freely, including for commercial purposes. If you distribute a modified version or a derivative work, it must also be GPLv3 and you must provide the source.

---

<div align="center">

**Built by [J P Ames](https://github.com/jpames) at N2NHU Labs, Newburgh NY.**
Part of a broader applied algebraic design theory project — the book is forthcoming.

If this saves you the cost of an RMM subscription, consider opening an issue to say hello.

</div>
