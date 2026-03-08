<div align="center">
  <h1>Easy OpenClaw</h1>
  <p>Multi-tenant private deployment toolkit built on top of OpenClaw</p>
</div>

![License](https://img.shields.io/badge/License-MIT-green)
![Node](https://img.shields.io/badge/Node.js-%3E%3D18-brightgreen?logo=node.js)
![Bun](https://img.shields.io/badge/Runtime-Bun-black?logo=bun)
![Tauri](https://img.shields.io/badge/Tauri-2.0-24C8DB?logo=tauri)
![Rust](https://img.shields.io/badge/Rust-1.80+-F46623?logo=rust)
![Svelte](https://img.shields.io/badge/UI-Svelte%205-ff3e00?logo=svelte)

[中文文档](./README.zh-CN.md)

---

## Overview

**Easy OpenClaw** is a multi-tenant private deployment platform built on the [OpenClaw](https://github.com/openclaw/openclaw) open-source project.

The official OpenClaw only supports single-user self-hosting (one instance per machine). Easy OpenClaw adds a full control plane on top, enabling **one server to serve N users, each with their own isolated OpenClaw instance**.

| Component | Role | Description |
|-----------|------|-------------|
| **openclaw** | Data plane · tenant instance | Official open-source self-hosted gateway, one Docker/Podman container per user |
| **openclaw-mate** | Client · desktop node | Tauri 2 + React desktop app, connects to the user's dedicated openclaw instance |
| **openclaw-tenant** | Control plane · admin backend | Hono + Svelte 5, manages Licenses, container orchestration, and authentication |

---

## 📦 Repository Structure

```text
easy-openclaw/
├── openclaw/          # OpenClaw core service (modified from official upstream)
├── openclaw-mate/     # Desktop execution node (Tauri 2 + React 18)
└── openclaw-tenant/   # License & tenant management backend (Hono API + Svelte 5)
```

---

## 🏗️ Architecture

### Deployment Architecture (Multi-tenant)

Each user (License) maps to one isolated openclaw container and one exec desktop client. `openclaw-tenant` is the unified control plane managing all instance lifecycles.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Server (Self-Hosted)                        │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  openclaw-tenant (Control Plane)             │   │
│  │   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐  │   │
│  │   │  Hono API    │    │  Svelte 5    │    │  SQLite DB  │  │   │
│  │   │  :3000       │    │  Admin UI    │    │  licenses   │  │   │
│  │   └──────────────┘    └──────────────┘    └─────────────┘  │   │
│  └─────┬──────────────────────────────────────────────────────┘   │
│        │ Create License → provision-docker.sh / provision-podman.sh│
│        │ Each License gets an isolated container + port            │
│        ↓                                                           │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐           │
│  │  openclaw   │    │  openclaw   │    │  openclaw   │           │
│  │  Gateway A  │    │  Gateway B  │    │  Gateway C  │    ...    │
│  │  :18800     │    │  :18801     │    │  :18802     │           │
│  └──────▲──────┘    └──────▲──────┘    └──────▲──────┘           │
│         │ WebSocket         │ WebSocket         │ WebSocket        │
└─────────┼───────────────────┼───────────────────┼─────────────────┘
          │                   │                   │
   ┌──────┴──────┐     ┌──────┴──────┐     ┌──────┴──────┐
   │ openclaw-   │     │ openclaw-   │     │ openclaw-   │
   │ exec User A │     │ exec User B │     │ exec User C │
   │ (Desktop)   │     │ (Desktop)   │     │ (Desktop)   │
   └─────────────┘     └─────────────┘     └─────────────┘
         ↑                    ↑                    ↑
         └────────────────────┴────────────────────┘
              On first activation: POST /api/verify → tenant
              Returns dedicated gatewayUrl + gatewayToken
```

> **1 License = 1 Container = 1 exec client** — fully isolated, one-to-one mapping.

---

### Service Architecture (Single Tenant View)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                          Server / Admin Side (Self-Hosted)                   ║
║                                                                              ║
║  ┌────────────────────────┐       ┌──────────────────────────────────────┐  ║
║  │    openclaw-tenant     │       │  openclaw (User's Docker Instance)   │  ║
║  │                        │       │                                      │  ║
║  │  [Svelte 5 Admin UI]   │ spawn │  ┌─────────────┐  ┌──────────────┐  │  ║
║  │  [Hono REST API]       │──────►│  │   Gateway   │  │  AI Agent    │  │  ║
║  │  [SQLite + Auth]       │ Docker│  │  WebSocket  │  │  Scheduler   │  │  ║
║  │                        │       │  │  (wss://)   │  │              │  │  ║
║  └──────────┬─────────────┘       │  └──────┬──────┘  └──────────────┘  │  ║
║             │                     │         │ push node.invoke            │  ║
║             │ ① POST /api/verify  └─────────┼──────────────────────────┘  ║
║             │   returns nodeConfig          │                              ║
╚═════════════╪═════════════════════════════╪══════════════════════════════╝
              │                             │
              │                             │ ② wss:// persistent connection
              │                             │   connect frame (role=node)
              ▼                             ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║                           Client Device (End User)                           ║
║                                                                              ║
║  ┌───────────────────────────────────────────────────────────────────────┐  ║
║  │                       openclaw-mate (Desktop Node)                    │  ║
║  │  ┌─────────────────────────────────────────────┐                     │  ║
║  │  │        React UI (System Tray / Task Monitor) │                     │  ║
║  │  └───────────────────┬─────────────────────────┘                     │  ║
║  │                  Tauri IPC                                            │  ║
║  │  ┌───────────────────┴─────────────────────────┐                     │  ║
║  │  │                  Rust Core                  │                     │  ║
║  │  │  ├── auth_client  → POST /api/verify         │                     │  ║
║  │  │  ├── device_identity → ed25519 key + sign    │                     │  ║
║  │  │  ├── config       → persist config.json      │                     │  ║
║  │  │  └── ws_client    → wss:// Gateway           │                     │  ║
║  │  │       ├── send connect frame                 │                     │  ║
║  │  │       ├── receive node.invoke → execute      │                     │  ║
║  │  │       └── return TaskResult → Gateway        │                     │  ║
║  │  └─────────────────────────────────────────────┘                     │  ║
║  │                      ↕ stdin / stdout IPC                             │  ║
║  │  ┌─────────────────────────────────────────────┐                     │  ║
║  │  │     Sidecar (Node.js subprocess)            │                     │  ║
║  │  │  browser / system / vision automation       │                     │  ║
║  │  └─────────────────────────────────────────────┘                     │  ║
║  └───────────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## 🔄 Key Interaction Flows

### ① Admin Creates a License (Provisioning)

```
Admin (Svelte UI)
  → POST /api/auth/login               # JWT auth
  → POST /api/licenses                 # Create license (optional: ownerTag / expiryDate / baseDomain)
      ↓
  [Tenant background worker]
      ├── Read settings defaults → snapshot into license (runtime_provider / runtime_dir / data_dir)
      ├── Allocate gateway_port / bridge_port
      ├── Run provision-docker.sh or provision-podman.sh → start dedicated openclaw container
      ├── Read generated openclaw.json (contains gateway.auth.token)
      ├── [Optional] Write Nginx config if license.nginx_host is set
      └── Update provision_status: pending → running → ready
```

### ② User Activates License (Verify & HWID Binding)

```
openclaw-mate starts, user enters licenseKey
  → [Rust auth_client] POST /api/verify { hwid, licenseKey, deviceName, publicKey }
      ↓
  Tenant validates:
      ✓ licenseKey valid, not revoked, not expired
      ✓ provision_status = ready (else 409)
      ✓ First call: bind HWID, activate license
      ↓
  Returns nodeConfig + needsBootstrap:
      { gatewayUrl, gatewayToken, agentId, licenseId }
      { needsBootstrap: { feishu: true/false } }
      ↓
  exec stores nodeConfig in memory (not persisted)
  If needsBootstrap.feishu = true → show Bootstrap Wizard
```

### ③ Establish Gateway Connection (WebSocket Handshake)

```
[Rust ws_client]
  → Read config.json (gatewayUrl + gatewayToken)
  → Open wss:// persistent connection to dedicated openclaw Gateway
  → Send connect frame:
      { role: "node", scopes: ["node.execute"], device: { publicKey, signature } }
  → Gateway verifies deviceId against paired.json
  → Connection established → React UI shows online status
```

### ④ Task Dispatch & Execution

```
openclaw AI Agent (server-side)
  → Gateway pushes node.invoke { command, args }
      ↓
  exec ws_client receives:
      ├── Simple commands (system.run) → Rust executes local shell directly
      └── Advanced tasks (browser / vision) → stdin IPC → Sidecar subprocess (Node.js)
                                              → stdout IPC returns result
      ↓
  exec returns TaskResult to Gateway → AI Agent receives execution feedback
```

### ⑤ Bootstrap Config Wizard (First Launch)

```
exec detects needsBootstrap.feishu = true
  → Show FeishuWizard modal (appId + appSecret)
      ↓
  User submits
  → POST {TENANT_API_BASE}/api/licenses/{licenseId}/bootstrap-config
      { licenseKey, hwid, feishu: { appId, appSecret } }
      ↓
  Tenant writes to openclaw.json in container config dir:
      channels.feishu.appId / channels.feishu.appSecret
  → openclaw hot-reloads feishu channel (no restart needed)
  → Sets wizard_feishu_done = 1 in DB
      ↓
  exec calls verifyAndConnect() again
  → needsBootstrap.feishu = false, wizard dismissed
  → Feishu channel active immediately

  exec Settings page always shows "飞书配置" entry for reconfiguration
  exec Settings page also provides a manual "Model API Wizard" entry:
    POST /api/licenses/{licenseId}/bootstrap-config
      { licenseKey, hwid, modelAuth: { providerId, providerLabel, baseUrl, api, modelId, modelName, apiKey } }
    modelAuth writes files only (no tenant DB persistence)
```

### ⑥ gatewayToken Auto-Rotation

```
After token expires (> token_ttl_days), on next POST /api/verify:
  → Tenant generates new gatewayToken
  → Updates DB gateway_token / token_expires_at
  → Syncs new token to openclaw.json of the instance
  (Transparent to the user — no manual intervention required)
```

---

## ✨ Features

### openclaw-mate (Desktop Node)

- **Cross-platform**: Windows / macOS / Linux (Tauri 2)
- **Real-time task reception**: WebSocket long connection, low-latency command delivery
- **Local task execution**: `system.run` (shell), `browser` (browser automation), `vision` (visual recognition)
- **Native OS integration**: system tray, native notifications, local data persistence
- **Device identity**: ed25519 key pair generated locally, deviceId derived via SHA-256
- **Sidecar extension**: Node.js subprocess handles advanced automation via stdin/stdout IPC

### openclaw-tenant (Auth & Management Backend)

- **License management**: create, revoke, set expiry and token rotation period
- **HWID device binding**: locks to a physical device on first activation
- **Container orchestration**: async Docker/Podman provisioning, tracks `pending → running → ready → failed`
- **Baseline config pre-seeding**: provision script writes gateway, tools, models, agents, commands defaults into `openclaw.json` — no interactive wizard needed
- **Model API key injection**: `model_presets` table stores AES-256-GCM encrypted API keys; injected into `openclaw.json` after provisioning
- **Bootstrap config API**: exec wizard submits Feishu credentials via `POST /api/licenses/:id/bootstrap-config`; written to container config and hot-reloaded
- **Runtime auto-detection**: detects Docker or Podman via socket file at startup
- **Settings UI**: runtime provider, port ranges, base domain configurable via admin UI
- **Multi-tenant token rotation**: per-license gatewayToken with automatic rotation

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| **openclaw** | Official open-source, Node.js / TypeScript |
| **openclaw-mate frontend** | React 18 · React Router v6 · Zustand · Tailwind CSS · Vite |
| **openclaw-mate core** | Tauri v2 · Rust · Tokio · Tokio-Tungstenite · ed25519-dalek |
| **openclaw-mate sidecar** | Node.js · TypeScript (browser / system / vision) |
| **openclaw-tenant backend** | Hono · SQLite (bun:sqlite) · JWT (HS256) · bcrypt |
| **openclaw-tenant frontend** | Svelte 5 (Runes) · TailwindCSS v4 · Vite |
| **Runtime** | Bun (tenant) · Node.js ≥ 18 (exec build) · Rust stable |

---

## 🚀 Getting Started

### Prerequisites

| Tool | Purpose | Version |
|------|---------|---------|
| [Node.js](https://nodejs.org/) | openclaw-mate build | ≥ 18.x |
| [Bun](https://bun.sh/) | openclaw-tenant runtime | latest stable |
| [Rust + Cargo](https://rustup.rs/) | Tauri core compilation | latest stable |
| Docker or Podman | openclaw instance orchestration | Docker ≥ 24.x |

### 1. Start openclaw-tenant (Admin Backend)

```bash
cd openclaw-tenant

# Install dependencies
bun install

# Configure environment
cp .env.example .env
# Edit .env: set JWT_SECRET, ADMIN_USER, ADMIN_PASS, and runtime paths

# Dev mode
bun run dev:api    # Terminal 1: Hono API (default :3000)
bun run dev:ui     # Terminal 2: Svelte Admin UI (default :5173)
```

> After entering the admin UI, create a License and wait for `provision_status` to become `ready`.

### 2. Start openclaw-mate (Desktop Node)

```bash
cd openclaw-mate

# Install frontend dependencies
npm install

# Dev mode (Vite + Tauri desktop window)
npm run tauri:dev

# Production build
npm run tauri:build
```

> Enter the `licenseKey` in the settings page to activate and connect to your dedicated openclaw Gateway.

---

## ⚙️ Environment Variables

Key variables for `openclaw-tenant` (see `.env.example`):

| Variable | Description |
|----------|-------------|
| `JWT_SECRET` | JWT signing secret — use a strong value in production |
| `ADMIN_USER` / `ADMIN_PASS` | Admin credentials (default `admin` / `admin123` — **change in production**) |
| `OPENCLAW_HOST_IP` | Default host_ip for settings (used only on first-time init) |
| `OPENCLAW_RUNTIME_DIR` | Default runtime_dir for settings (used only on first-time init) |
| `OPENCLAW_DATA_DIR` | Default data_dir for settings (used only on first-time init) |
| `OPENCLAW_GATEWAY_PORT_START/END` | Default gateway port range for settings |
| `OPENCLAW_BASE_DOMAIN` | Default base_domain for settings (enables Nginx subdomain mode) |
| `TENANT_PUBLIC_URL` | Public URL of this tenant API (e.g. `https://tenant.example.com`) — used for admin UI and documentation references |
| `NGINX_SITE_DIR` / `NGINX_RELOAD_CMD` | Nginx config path and reload command for domain mode |

### Settings & License Snapshot Strategy

1. **`settings` table**: global defaults template (editable via Admin UI Settings page).
2. **`license` row**: snapshots the runtime config at creation time (`runtime_provider`, `runtime_dir`, `data_dir`, `nginx_host`).
3. Domain priority: `POST /api/licenses` `baseDomain` > `settings.base_domain` > none (IP:port mode).
4. Provisioning always uses the license-level snapshot — changing global settings does not affect already-issued licenses.

---

## 📂 Documentation Index

- [Authentication Flow](./openclaw-tenant/docs/AUTHENTICATION.md)
- [Backend API Reference](./openclaw-tenant/docs/BACKEND_API.md)
- [License Provisioning Engine](./openclaw-tenant/docs/LICENSE_PROVISIONING.md)
- [UI Design Spec](./openclaw-tenant/docs/UI_DESIGN.md)
- [Environment Variables](./openclaw-tenant/docs/ENVIRONMENT.md)

---

## 🔧 Git Submodule Workflow

This repository uses a **superproject + submodules** structure:

- `openclaw/`
- `openclaw-mate/`
- `openclaw-tenant/`

### Initial Clone

```bash
git clone --recurse-submodules <easy-openclaw-repo-url>
```

If you already cloned the root repo:

```bash
git submodule update --init --recursive
```

### Committing Changes to a Submodule

Example: modifying `openclaw-mate/`

```bash
# Step 1: commit and push inside the submodule
cd openclaw-mate
git add .
git commit -m "feat: your change"
git push origin <your-branch>

# Step 2: update the submodule pointer in the root repo
cd ..
git add openclaw-mate
git commit -m "chore: bump openclaw-mate submodule"
git push origin main
```

> Always push the submodule **before** pushing the root repo. The root repo stores a commit SHA pointer, not the source code itself.

### Pull Latest Submodule Content

```bash
git pull
git submodule update --init --recursive
```

---

## 🤝 Contributing

Contributions are welcome! Please read the `AGENTS.md` in each subproject root for code conventions.

1. Fork this repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Commit your changes (`git commit -m 'feat: add your feature'`)
4. Push to the branch (`git push origin feature/your-feature`)
5. Open a Pull Request

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
