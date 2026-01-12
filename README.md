# Poulpe

**Cross-platform remote access gateway for SSH, RDP, VNC, and web applications.**

Poulpe is a stateful remote access platform that runs entirely in the browser. Connect to servers, desktops, and web applications through a unified interface with persistent sessions that survive tab closures and device switches.

## Key Features

- **Protocol Support**: SSH, RDP, VNC, and web application proxying—all rendered in the browser
- **Cross-Platform Agent**: Single Go binary (Tentacle) runs on Linux, Windows, and macOS with no external dependencies
- **Stateful Sessions**: Close your tab, switch devices, resume exactly where you left off
- **Multi-Tenancy**: Organizations, workspaces, and folder hierarchies for access segregation
- **Multi-OIDC**: Email-based identity provider discovery with per-organization SSO configuration
- **ReBAC Authorization**: Fine-grained relationship-based access control with hierarchical inheritance (powered by OpenFGA)
- **VS Code-Inspired UI**: Familiar panels, tabs, and tree navigation
- **Poulpe Support**: One-click remote assistance with mandatory user consent
- **Kubernetes-Native**: Production-ready Helm charts with clustering and HA support

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Browser (SPA)                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  Terminal   │  │  RDP View   │  │  VNC View   │  │  Web Proxy  │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         └────────────────┴────────────────┴────────────────┘        │
│                              WebSocket                               │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
┌─────────────────────────────────┴───────────────────────────────────┐
│                    Poulpe Server Cluster (Go)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ REST API │  │   Auth   │  │  ReBAC   │  │  Events  │             │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│       └─────────────┴─────────────┴─────────────┘                    │
│            Redis (Cluster Coordination) + PostgreSQL                 │
│                              gRPC                                    │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
┌─────────────────────────────────┴───────────────────────────────────┐
│                        Tentacle Agent (Go)                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │   SSH    │  │   RDP    │  │   VNC    │  │   Host   │             │
│  │ (stdlib) │  │(IronRDP) │  │  (RFB)   │  │ (Rust)   │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
│                        Rust Bridge (FFI)                             │
│     Screen Capture: DXGI (Win) / X11 (Linux) / AVFoundation (Mac)    │
└─────────────────────────────────────────────────────────────────────┘
```

### Components

**Poulpe Server** (`cmd/poulpe-server/`)
- REST API + WebSocket for browser clients
- gRPC for Tentacle agents
- Multi-OIDC authentication with email-based provider discovery
- PostgreSQL for state, Redis for clustering
- Session routing and WebSocket relay
- Embedded OpenFGA authorization engine

**Tentacle Agent** (`cmd/tentacle/`)
- Connects to targets via SSH, RDP, VNC, or local screen capture
- Maintains persistent connections and buffers output
- Rust FFI bridge for RDP (IronRDP) and screen capture (DXGI/X11/AVFoundation)
- Reports health metrics back to server
- Multiple operational modes: standard, service (Windows), and embedded (Poulpe support)

**Poulpe Support** (`cmd/tentacle-support/`)
- Pre-configured binary for one-click remote assistance
- Cross-platform GUI (Fyne toolkit)
- Mandatory user consent with configurable timeout
- Auto-generated sessions with expiration

**CLI Tools** (`cmd/poulpectl/`, `cmd/gateway/`)
- `poulpectl`: Server management, migrations, organization management
- `gateway`: Native RDP/SSH client gateway (RD Gateway protocol)

**Web Frontend** (`web/`)
- React SPA with VS Code-inspired layout
- xterm.js for terminal rendering
- Binary protocol handling for RDP/VNC frames
- Real-time WebSocket events for session health

## Binaries

| Binary | Description |
|--------|-------------|
| `poulpe-server` | Central coordination server |
| `tentacle` | Full-featured agent (all protocols + service mode) |
| `tentacle-host` | Lightweight agent (host protocol only) |
| `tentacle-support` | Poulpe support with GUI (embedded mode) |
| `poulpectl` | CLI management tool (migrations, orgs, authz) |
| `gateway` | Native RDP/SSH client gateway |
| `poulpe-updater` | Windows self-update helper |

## Protocol Implementation

| Protocol | Implementation | Notes |
|----------|---------------|-------|
| SSH | Go `x/crypto/ssh` | Pure Go, no dependencies |
| RDP | IronRDP (Rust) | FFI bridge, CredSSP auth |
| VNC | RFB (Go) | Native implementation |
| Host | Rust + OS APIs | DXGI (Windows), X11 (Linux), AVFoundation (macOS) |
| Web | Chromium + VNC | Container orchestration (Docker Compose or Kubernetes) |

All protocol handling is baked into the binaries. No external tools like `xfreerdp`, `ssh`, or `x11vnc` required.

## Stateful Session Design

Sessions live on the Tentacle, not in the browser:

1. **Connection Persistence**: Tentacle keeps SSH/RDP/VNC connections open
2. **Output Buffering**: Terminal output stored in circular buffer (64KB); video sessions store last frame
3. **Session Replay**: Reconnecting clients receive buffered terminal output or last video frame
4. **Device Hopping**: Start on laptop, continue on tablet
5. **Lock Management**: Workspace locks prevent concurrent conflicts

## Security Model

**Authentication**
- OIDC-only (no local passwords)
- Per-organization identity provider configuration
- Email domain-based provider discovery
- Session tracking with revocation support

**Authorization (ReBAC)**
- Relationship-Based Access Control powered by embedded OpenFGA
- Hierarchical permissions with inheritance (organization → folder → resource)
- Actions: `view`, `connect`, `edit`, `delete`, `manage`, `view_secrets`
- Assignable to users or OIDC groups
- Computed permissions via relationship graph traversal

**Encryption**
- Credentials encrypted at rest (AES-256-GCM)
- TLS for all network communication
- OIDC client secrets encrypted in database

## Multi-Tenancy

```
Platform
└── Organizations (tenants)
    ├── OIDC Configurations
    ├── Users & Groups
    ├── Folders (ltree hierarchy)
    │   ├── Connections
    │   └── Credentials
    ├── Tentacles
    └── Workspaces
        └── Sessions
```

- Organizations are isolated tenants
- Users can belong to multiple organizations
- Workspaces provide per-user session isolation within an org
- Platform admins manage cross-org resources

## Cross-Platform Support

The Tentacle agent achieves true cross-platform support:

**Windows** ✅ Fully supported
- Service mode with per-user session agents
- DXGI Desktop Duplication for screen capture
- Windows.Graphics.Capture API (modern)
- UAC elevation helper for admin operations

**Linux** ✅ Fully supported
- X11 display capture via `scrap` crate
- Input injection via `enigo`/xdotool

**macOS** ✅ Fully supported
- SSH, RDP, and VNC protocols (pure Go/Rust)
- Host protocol via AVFoundation screen capture
- Input injection via Core Graphics

No external dependencies beyond OS-provided APIs.

## Poulpe Support (Remote Assistance)

Poulpe includes a one-click remote assistance feature for help desk scenarios:

1. **Generate Download Link**: Support agent creates a time-limited support session in the UI
2. **User Downloads Binary**: End user downloads a pre-configured `tentacle-support` binary
3. **Consent Required**: Binary displays a GUI requiring explicit user consent before screen sharing
4. **Session Expires**: Support sessions automatically expire and clean up

Features:
- Cross-platform GUI (Windows, Linux, macOS) using Fyne toolkit
- Mandatory consent with configurable timeout (default: 30 seconds)
- Real-time status display showing connection state and session info
- Automatic binary cleanup after session ends

## Production Deployment

### Kubernetes (Recommended)

Helm charts are provided in `deployments/helm/`:

```bash
# Add repo and install
helm repo add poulpe https://your-registry/helm/stable
helm install poulpe poulpe/poulpe \
  --set secrets.jwtSecret=$(openssl rand -base64 32) \
  --set secrets.credentialKek=$(openssl rand -base64 32) \
  --set postgresql.auth.password=your-secure-password \
  --set appUrl=https://poulpe.example.com
```

Key capabilities:
- **StatefulSet deployment** with stable pod identities
- **Redis-based cluster coordination** for multi-server deployments
- **Affinity routing** for WebSocket session stickiness
- **Migration jobs** as Helm hooks
- **PostgreSQL HA** with failover support
- **Redis Cluster/Sentinel** modes for HA

See `deployments/helm/poulpe/README.md` for full configuration options.

### Clustering

Poulpe uses a cluster-first architecture. Even single-server deployments run as 1-node clusters:

- **Server Registry**: Each server registers with Redis and sends heartbeats
- **Tentacle Assignment**: Tentacles are assigned to specific servers; sessions route accordingly
- **Dead Server Detection**: Servers missing heartbeats are cleaned up; tentacles reassigned
- **Health Monitoring**: Servers terminate if Redis/PostgreSQL connectivity is lost (prevents split-brain)

Configuration:
```bash
POULPE_CLUSTER_ENABLED=true
POULPE_CLUSTER_SERVER_ID=poulpe-1
POULPE_CLUSTER_SERVER_ADDR=poulpe-1:8080
POULPE_CLUSTER_HEARTBEAT_INTERVAL=5s
POULPE_CLUSTER_DEAD_THRESHOLD=15s
```

## Quick Start

### Prerequisites
- Go 1.21+
- Node.js 18+
- PostgreSQL 14+
- Redis 6+
- Rust (for Tentacle with Host/RDP support)

### Server

```bash
# Clone and build
git clone https://github.com/WHNet-Org/poulpe.git
cd poulpe
make build

# Configure
cp .env.example .env
# Edit .env with your database and OIDC settings

# Run migrations (includes OpenFGA schema)
./bin/poulpectl migrate up

# Start server
./bin/poulpe-server
```

### Tentacle

```bash
# Build tentacle (includes Rust bridge)
make build-tentacle

# Run with registration token
./bin/tentacle --server your-server:9090 --token <registration-token>

# Or install as Windows service
./bin/tentacle service install --server your-server:9090 --token <registration-token>
```

### Frontend

```bash
cd web
npm install
npm run dev
```

## Configuration

Key environment variables:

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `JWT_SECRET` | Secret for JWT signing |
| `CREDENTIAL_KEK` | Key encryption key for credentials |
| `POULPE_WEB_DIR` | Path to built frontend |
| `POULPE_CLUSTER_ENABLED` | Enable multi-server clustering |
| `POULPE_SUPPORT_ENABLED` | Enable Poulpe support feature |

See `.env.example` for full configuration options including:
- PostgreSQL HA (multi-host failover, read replicas)
- Redis Cluster and Sentinel modes
- OpenFGA authorization tuning
- Rate limiting
- Gateway configuration

## CLI Management

`poulpectl` provides administrative commands:

```bash
# Migrations
poulpectl migrate up        # Apply all pending migrations
poulpectl migrate down      # Rollback last migration
poulpectl migrate status    # Show migration status

# OpenFGA
poulpectl openfga migrate   # Run OpenFGA migrations only
poulpectl openfga sync      # Rebuild OpenFGA state from database

# Organizations
poulpectl org list                          # List organizations
poulpectl org members <org>                 # List members
poulpectl org set-role <org> <email> admin  # Change role
poulpectl org add-member <org> <email> member
poulpectl org remove-member <org> <email>
```

## Why Poulpe?

| Feature | Poulpe | Apache Guacamole | Teleport | Cloudflare Access |
|---------|--------|------------------|----------|-------------------|
| **Protocols** | SSH, RDP, VNC, Web | SSH, RDP, VNC | SSH, K8s, DB, Windows* | SSH, Web |
| **Stateful sessions** | Yes (survive disconnects) | No | No | No |
| **Self-hosted** | Yes | Yes | Yes/Cloud | Cloud only |
| **Multi-tenant** | Native | Manual | Enterprise | Enterprise |
| **OIDC discovery** | Per-email domain | Single provider | Single provider | Native |
| **Authorization** | ReBAC (OpenFGA) | Role-based | Role-based | Policy-based |
| **Agent dependencies** | Tentacle (single binary) | guacd + libs | teleport binary | cloudflared |
| **Cross-platform agent** | Linux, Windows, macOS | Linux only (guacd) | Linux, Windows, macOS | Linux, Windows, macOS |
| **Poulpe Support** | Built-in GUI | No | No | No |
| **Kubernetes native** | Helm charts included | Community charts | Yes | N/A |

*Teleport's Windows access uses a custom protocol over WebSocket, not native RDP. No VNC support.

**Guacamole** requires `guacd` (C daemon) with system libraries for each protocol. Poulpe's Tentacle is a single binary with all protocols baked in.

**Teleport** is excellent for infrastructure access (SSH, K8s, databases) but doesn't support VNC and uses a proprietary protocol for Windows desktops. Poulpe uses native RDP via IronRDP and supports persistent sessions.

**Cloudflare Access** requires their network. Poulpe is fully self-hosted.

## Non-Goals

Poulpe is not intended to:
- Replace full desktop management suites (MDM, RMM)
- Act as a VPN or network overlay
- Provide file transfer or clipboard sync across sessions (yet)
- Be a drop-in replacement for SaaS remote support tools

Poulpe focuses on secure, stateful, browser-based access to existing systems.

## Threat Model

Poulpe assumes:
- The Poulpe server is trusted infrastructure
- Tentacles authenticate using short-lived registration tokens
- End-user browsers are untrusted
- All protocol traffic is encrypted (TLS for gRPC, WSS for WebSocket)

Poulpe does not attempt to protect against:
- Compromised hosts running Tentacle
- Malicious administrators within an organization

## Built With

Poulpe was architected and engineered by humans, with implementation accelerated by [Claude Code](https://claude.ai/code).

The architecture decisions—stateful session design, cross-platform capture strategy, multi-tenant security model, protocol routing, and OIDC orchestration—are human work. The AI helped translate those designs into code faster.

We believe authorship is about the decisions you make, not the keystrokes. The hard parts of building Poulpe were figuring out *what* to build and *why*—tradeoffs that tools cannot reason about.

## License

This project is licensed under the **GNU Affero General Public License v3.0** (AGPL-3.0).

This means you can use, modify, and distribute Poulpe freely. If you modify Poulpe and run it as a network service, you must make your source code available to users of that service.

### Internal Use

Organizations may fork and modify Poulpe for internal use without publishing changes, as long as the modified system is not exposed to non-employees over a network. Running Poulpe as a hosted service for customers or third parties requires publishing the modified source code under the AGPL.

See [LICENSE](LICENSE) for details.

---

**Poulpe** — French for octopus. Many arms, one brain.
