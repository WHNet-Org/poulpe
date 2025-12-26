# Poulpe

**Cross-platform remote access gateway for SSH, RDP, VNC, and web applications.**

Poulpe is a stateful remote access platform that runs entirely in the browser. Connect to servers, desktops, and web applications through a unified interface with persistent sessions that survive tab closures and device switches.

## Key Features

- **Protocol Support**: SSH, RDP, VNC, and web application proxying—all rendered in the browser
- **Cross-Platform Agent**: Single Go binary (Tentacle) runs on Linux, Windows, and macOS with no external dependencies
- **Stateful Sessions**: Close your tab, switch devices, resume exactly where you left off
- **Multi-Tenancy**: Organizations, workspaces, and folder hierarchies for access segregation
- **Multi-OIDC**: Email-based identity provider discovery with per-organization SSO configuration
- **Fine-Grained RBAC**: Groups, roles, and permissions down to individual connections
- **VS Code-Inspired UI**: Familiar panels, tabs, and tree navigation

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
│                         Poulpe Server (Go)                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ REST API │  │   Auth   │  │   RBAC   │  │  Events  │             │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│       └─────────────┴─────────────┴─────────────┘                    │
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
│            Screen Capture: DXGI (Win) / X11 (Linux)                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Components

**Poulpe Server** (`cmd/poulpe-server/`)
- REST API + WebSocket for browser clients
- gRPC for Tentacle agents
- Multi-OIDC authentication with email-based provider discovery
- PostgreSQL for state, Redis for clustering
- Session routing and WebSocket relay

**Tentacle Agent** (`cmd/tentacle/`)
- Connects to targets via SSH, RDP, VNC, or local screen capture
- Maintains persistent connections and buffers output
- Rust FFI bridge for RDP (IronRDP) and screen capture (DXGI/X11)
- Reports health metrics back to server

**Web Frontend** (`web/`)
- React SPA with VS Code-inspired layout
- xterm.js for terminal rendering
- Binary protocol handling for RDP/VNC frames
- Real-time WebSocket events for session health

## Protocol Implementation

| Protocol | Implementation | Notes |
|----------|---------------|-------|
| SSH | Go `x/crypto/ssh` | Pure Go, no dependencies |
| RDP | IronRDP (Rust) | FFI bridge, CredSSP auth |
| VNC | RFB (Go) | Native implementation |
| Host | Rust + OS APIs | DXGI (Windows), X11 (Linux) |
| Web | Chromium + VNC | Docker container orchestration (Kubernetes coming soon) |

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

**Authorization**
- Hierarchical folder permissions with inheritance
- Grant or deny actions: `view`, `connect`, `edit`, `delete`, `manage`
- Assignable to users, groups, or roles
- Priority-based evaluation for conflict resolution

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

**Linux** ✅ Fully supported
- X11 display capture via `scrap` crate
- Input injection via `enigo`/xdotool

**macOS** ⚠️ Remote protocols only
- SSH, RDP, and VNC protocols work (pure Go/Rust, no OS dependencies)
- **Host protocol not implemented** — no local screen capture/remote control
- The architecture is designed for macOS Host support (Screen Capture framework + Accessibility APIs), but we lack access to Apple hardware for development and testing
- Contributions welcome!

No external dependencies beyond OS-provided APIs.

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

# Run migrations
make migrate-up

# Start server
./bin/poulpe-server
```

### Tentacle

```bash
# Build tentacle (includes Rust bridge)
make build-tentacle

# Run with registration token
./bin/tentacle --server your-server:9090 --token <registration-token>
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
| `JWT_SECRET` | Secret for JWT signing |
| `CREDENTIAL_KEK` | Key encryption key for credentials |
| `POULPE_WEB_DIR` | Path to built frontend |

See `.env.example` for full configuration options.

## Why Poulpe?

| Feature | Poulpe | Apache Guacamole | Teleport | Cloudflare Access |
|---------|--------|------------------|----------|-------------------|
| **Protocols** | SSH, RDP, VNC, Web | SSH, RDP, VNC | SSH, K8s, DB, Windows* | SSH, Web |
| **Stateful sessions** | Yes (survive disconnects) | No | No | No |
| **Self-hosted** | Yes | Yes | Yes/Cloud | Cloud only |
| **Multi-tenant** | Native | Manual | Enterprise | Enterprise |
| **OIDC discovery** | Per-email domain | Single provider | Single provider | Native |
| **Agent dependencies** | Tentacle (single binary) | guacd + libs | teleport binary | cloudflared |
| **Cross-platform agent** | Linux, Windows, macOS | Linux only (guacd) | Linux, Windows, macOS | Linux, Windows, macOS |

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
