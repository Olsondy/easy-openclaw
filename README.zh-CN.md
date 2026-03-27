<div align="center">
  <h1>Easy OpenClaw</h1>
  <p>围绕 OpenClaw 构建的私有化部署工具链</p>
</div>

![License](https://img.shields.io/badge/License-MIT-green)
![Node](https://img.shields.io/badge/Node.js-%3E%3D18-brightgreen?logo=node.js)
![Bun](https://img.shields.io/badge/Runtime-Bun-black?logo=bun)
![Tauri](https://img.shields.io/badge/Tauri-2.0-24C8DB?logo=tauri)
![Rust](https://img.shields.io/badge/Rust-1.80+-F46623?logo=rust)
![Svelte](https://img.shields.io/badge/UI-Svelte%205-ff3e00?logo=svelte)

---

## 简介

**Easy OpenClaw** 是一套基于 [OpenClaw](https://github.com/openclaw/openclaw) 开源项目构建的**多租户私有化部署平台**。

官方 OpenClaw 仅支持单用户自托管（一台机器跑一个实例），Easy OpenClaw 则在其之上构建了一套完整的控制平面，实现「**一套服务端，N 个用户各自独享一个隔离的 OpenClaw 实例**」的多租户模式。

| 组件 | 定位 | 说明 |
|------|------|---------|
| **openclaw** | 数据平面 · 隔离运行时实例 | 官方开源自托管版，每个用户独享一个 Docker 容器 |
| **openclaw-mate** | 客户端 · 桌面节点 | Tauri 2 + React 桌面应用，连接用户专属的 openclaw 实例 |
| **openclaw-tenant** | 控制平面 · 管理后台 | Hono + Svelte 5，负责 License 管理、Docker 编排与授权鉴权 |

子项目仓库地址：

- `openclaw`: <https://github.com/Olsondy/openclaw>
- `openclaw-mate`: <https://github.com/Olsondy/openclaw-mate>
- `openclaw-tenant`: <https://github.com/Olsondy/openclaw-tenant>

---

## 📦 项目结构

```text
easy-openclaw/
├── openclaw/          # 自托管的 OpenClaw 核心服务（基于官方开源修改）
├── openclaw-mate/     # 桌面端执行节点（Tauri 2 + React 18）
└── openclaw-tenant/   # 授权与租户管理后台（Hono API + Svelte 5）
```

---

## 🏗️ 架构概览

### 部署架构（多租户）

每个用户（License）对应一个独立的 openclaw Docker 容器和一个 exec 桌面客户端，彼此完全隔离。openclaw-tenant 作为统一控制平面，管理所有实例的生命周期。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         服务端（自托管）                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   openclaw-tenant（控制平面）                  │   │
│  │   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐  │   │
│  │   │  Hono API    │    │  Svelte 管理  │    │  SQLite DB  │  │   │
│  │   │  :3000       │    │  UI (admin)  │    │  licenses   │  │   │
│  │   └──────────────┘    └──────────────┘    └─────────────┘  │   │
│  └─────┬──────────────────────────────────────────────────────┘   │
│        │ 创建 License → provision-docker.sh / provision-podman.sh  │
│        │ 每个 License 独立一个容器，端口隔离                          │
│        ↓                                                           │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐           │
│  │  openclaw   │    │  openclaw   │    │  openclaw   │           │
│  │  Gateway A  │    │  Gateway B  │    │  Gateway C  │    ...    │
│  │  :18800     │    │  :18801     │    │  :18802     │           │
│  │  (Docker)   │    │  (Docker)   │    │  (Docker)   │           │
│  └──────▲──────┘    └──────▲──────┘    └──────▲──────┘           │
│         │ WebSocket         │ WebSocket         │ WebSocket        │
└─────────┼───────────────────┼───────────────────┼─────────────────┘
          │                   │                   │
   ┌──────┴──────┐     ┌──────┴──────┐     ┌──────┴──────┐
   │ openclaw-   │     │ openclaw-   │     │ openclaw-   │
   │ exec 用户A  │     │ exec 用户B  │     │ exec 用户C  │
   │ 桌面客户端   │     │ 桌面客户端   │     │ 桌面客户端   │
   └─────────────┘     └─────────────┘     └─────────────┘
         ↑                    ↑                    ↑
         └────────────────────┴────────────────────┘
              首次激活时 POST /api/verify → tenant
              返回专属 gatewayUrl + gatewayToken
```

> **1 个 License = 1 个 Docker 容器 = 1 个 exec 客户端**，三者一一对应，互相隔离。

---

### 服务架构（单实例视角）

以单个用户为视角，三个组件的内部交互关系：

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                            服务端 / 管理侧 (Self-Hosted)                     ║
║                                                                              ║
║  ┌────────────────────────┐       ┌──────────────────────────────────────┐  ║
║  │    openclaw-tenant     │       │  openclaw（用户专属 Docker 实例）       │  ║
║  │                        │       │                                      │  ║
║  │  [Svelte 5 Admin UI]   │ 创建  │  ┌─────────────┐  ┌──────────────┐  │  ║
║  │  [Hono REST API]       │──────►│  │   Gateway   │  │  AI Agent    │  │  ║
║  │  [SQLite + 授权引擎]    │ Docker│  │  WebSocket  │  │  任务调度器  │  │  ║
║  │                        │ 实例  │  │  (wss://)   │  │              │  │  ║
║  └──────────┬─────────────┘       │  └──────┬──────┘  └──────────────┘  │  ║
║             │                     │         │ 下发 node.invoke 指令        │  ║
║             │ ① POST /api/verify  └─────────┼──────────────────────────┘  ║
║             │   返回 nodeConfig             │                              ║
╚═════════════╪═════════════════════════════╪══════════════════════════════╝
              │                             │
              │                             │ ② wss:// 长连接
              │                             │   connect 握手帧 (role=node)
              ▼                             ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║                           客户端设备 / 终端侧 (Client)                        ║
║                                                                              ║
║  ┌───────────────────────────────────────────────────────────────────────┐  ║
║  │                      openclaw-mate（桌面节点）                          │  ║
║  │  ┌─────────────────────────────────────────────┐                     │  ║
║  │  │           React UI（系统托盘 / 任务监控面板）   │                     │  ║
║  │  └───────────────────┬─────────────────────────┘                     │  ║
║  │                  Tauri IPC                                            │  ║
║  │  ┌───────────────────┴─────────────────────────┐                     │  ║
║  │  │                  Rust Core                  │                     │  ║
║  │  │  ├── auth_client  → POST /api/verify         │                     │  ║
║  │  │  ├── config       → 持久化 config.json        │                     │  ║
║  │  │  └── ws_client    → wss:// Gateway 长连接    │                     │  ║
║  │  │       ├── 发送 connect 握手帧                 │                     │  ║
║  │  │       ├── 接收 node.invoke → 执行本地任务     │                     │  ║
║  │  │       └── 回传 TaskResult → Gateway          │                     │  ║
║  │  └─────────────────────────────────────────────┘                     │  ║
║  │                      ↕ stdin / stdout IPC                             │  ║
║  │  ┌─────────────────────────────────────────────┐                     │  ║
║  │  │              Sidecar（Node.js 子进程）         │                     │  ║
║  │  │  处理高阶任务：browser / system / vision       │                     │  ║
║  │  └─────────────────────────────────────────────┘                     │  ║
║  └───────────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## 🔄 关键交互时序

### ① 管理员创建许可证（License Provisioning）

```
管理员（Svelte UI）
  → POST /api/auth/login               # JWT 认证
  → POST /api/licenses                 # 创建许可证（可选 ownerTag / expiryDate / tokenTtlDays / hostIp / baseDomain）
      ↓
  [Tenant 后台 Worker 异步执行]
      ├── 从 settings 读取全局默认并写入 license 快照（runtime_provider/runtime_dir/data_dir）
      ├── 分配 gateway_port / bridge_port
      ├── 按 license 快照执行 docker/podman provision 脚本，拉起独立 openclaw 实例
      ├── 读取实例生成的 openclaw.json（含 gateway.auth.token）
      ├── [可选] 若 license.nginx_host 存在，写入 Nginx 配置并刷新反向代理
      └── 更新 provision_status: pending → running → ready
```

### ② 用户首次激活（Verify & HWID 绑定）

```
openclaw-mate 桌面端启动，用户输入 licenseKey
  → [Rust auth_client] POST /api/verify { hwid, licenseKey, deviceName, publicKey }
      ↓
  Tenant API 校验：
      ✓ licenseKey 有效 & 未撤销 & 未过期
      ✓ provision_status = ready（否则 409）
      ✓ 首次调用：绑定 HWID，激活 license
      ↓
  返回 nodeConfig：
      { gatewayUrl, gatewayToken, agentId, licenseId }
      ↓
  exec 将 nodeConfig 持久化至本地 config.json
```

### ③ 建立 Gateway 长连接（WebSocket 握手）

```
openclaw-mate [Rust ws_client]
  → 读取 config.json（gatewayUrl + gatewayToken）
  → 建立 wss:// 长连接至对应 openclaw Gateway 实例
  → 发送 connect 握手帧：
      { type: "req", method: "connect", role: "node", scopes: ["node.execute"] }
  → Gateway 响应握手成功 → emit ws:connected 事件至 React UI
```

### ④ 云端下发任务并执行

```
openclaw AI Agent（服务端）
  → Gateway 推送 node.invoke { command, args }
      ↓
  exec ws_client 接收后：
      ├── 简单命令（system.run）→ Rust 直接执行本地 shell
      └── 高阶任务（browser / vision）→ stdin IPC 转发至 Sidecar 子进程
              Sidecar [Node.js] 执行后 → stdout IPC 回传结果
      ↓
  exec 将 TaskResult 回传 Gateway → AI Agent 收到执行结果
  exec 同时 emit ws:task_result 事件 → React UI 更新任务监控面板
```

### ⑤ Bootstrap 配置向导（飞书 + 模型 API）

```
exec 首次 verify 后若 needsBootstrap.feishu=true
  → 弹出飞书向导，提交 appId/appSecret 到：
    POST /api/licenses/{licenseId}/bootstrap-config
    { licenseKey, hwid, feishu: { appId, appSecret } }

exec 设置页另提供“模型 API 配置向导”手动入口：
  → 提交 modelAuth 到同一接口：
    { licenseKey, hwid, modelAuth: { providerId, providerLabel, baseUrl, api, modelId, modelName, apiKey } }
  → tenant 按规则合并 models.json（同 id 替换，不同 id 追加）
  → 同步更新 openclaw.json 默认主模型并执行 chown + 容器重启
  → modelAuth 覆盖值仅写文件即时生效，不持久化在 tenant 服务数据库
```

### ⑥ gatewayToken 自动轮换

```
Token 到期（超过 token_ttl_days）后，下次 POST /api/verify 时：
  → Tenant 自动生成新 gatewayToken
  → 更新数据库 gateway_token / token_expires_at
  → 同步写入对应 openclaw 实例的 openclaw.json
  （对用户完全透明，无需管理员干预）
```

---

## ✨ 核心功能

### openclaw-mate（桌面节点）

- **跨平台支持**：Windows / macOS / Linux（基于 Tauri 2）
- **实时任务接收**：WebSocket 长连接，低延迟接收云端指令
- **本地任务执行**：支持 `system.run`（shell）、`browser`（浏览器自动化）、`vision`（视觉识别）
- **原生系统集成**：系统托盘静默运行、原生通知、本地数据持久化
- **Sidecar 扩展**：Node.js 子进程通过 stdin/stdout IPC 处理高阶自动化任务

### openclaw-tenant（授权管理后台）

- **许可证管理**：创建、撤销、设置到期时间与 Token 轮换周期
- **HWID 设备绑定**：首次激活后锁定物理设备，防止许可证共享
- **Docker 异步编排**：自动 Compose up 拉起 openclaw 实例，追踪 `pending → running → ready → failed` 状态
- **多租户 Token 轮换**：每个 License 独立 gatewayToken，定期自动轮换

---

## 🛠️ 技术栈

| 组件 | 技术 |
|------|------|
| **openclaw** | 官方开源自托管，Node.js / TypeScript |
| **openclaw-mate 前端** | React 18 · React Router v6 · Zustand · Tailwind CSS · Vite |
| **openclaw-mate 核心层** | Tauri v2 · Rust · Tokio · Tokio-Tungstenite · Reqwest |
| **openclaw-mate Sidecar** | Node.js · TypeScript（browser / system / vision 模块） |
| **openclaw-tenant 后端** | Hono · SQLite（bun:sqlite）· JWT（HS256）· bcrypt |
| **openclaw-tenant 前端** | Svelte 5 (Runes) · TailwindCSS v4 · Vite |
| **运行时** | Bun（tenant） · Node.js ≥ 18（exec 构建） · Rust 稳定版 |

---

## 🚀 快速开始

### 前置依赖

| 工具 | 用途 | 版本要求 |
|------|------|----------|
| [Node.js](https://nodejs.org/) | openclaw-mate 构建 | ≥ 18.x |
| [Bun](https://bun.sh/) | openclaw-tenant 运行时 | 最新稳定版 |
| [Rust + Cargo](https://rustup.rs/) | Tauri 核心层编译 | 最新稳定版 |
| Docker | openclaw 实例编排 | ≥ 24.x |

---

### 1. 启动 openclaw-tenant（授权管理后台）

```bash
cd openclaw-tenant

# 安装依赖
bun install

# 配置环境变量
cp .env.example .env
# 编辑 .env，设置 JWT_SECRET、ADMIN_USER、ADMIN_PASS 等关键变量

# 开发模式启动
bun run dev:api    # Terminal 1：启动 Hono API（默认 :3000）
bun run dev:ui     # Terminal 2：启动 Svelte 管理 UI（默认 :5173）
```

> 进入管理 UI 后，创建 License 并等待 `provision_status` 变为 `ready`，即可获得 `licenseKey`。

---

### 2. 启动 openclaw-mate（桌面节点）

```bash
cd openclaw-mate

# 安装前端依赖
npm install

# 开发模式（拉起 Vite + Tauri 桌面窗口）
npm run tauri:dev

# 生产构建
npm run tauri:build
```

> 启动后，在设置页面填入 `licenseKey`，完成激活即可连接至对应 openclaw Gateway。

---

## ⚙️ 环境配置

openclaw-tenant 关键环境变量（参见 `.env.example`）：

| 变量 | 说明 |
|------|------|
| `JWT_SECRET` | JWT 签名密钥，生产环境必须设置强密钥 |
| `ADMIN_USER` / `ADMIN_PASS` | 管理员账号（默认 `admin` / `admin123`，**生产请务必修改**） |
| `OPENCLAW_HOST_IP` | settings 的默认 host_ip（仅首次初始化 settings 时使用） |
| `OPENCLAW_RUNTIME_DIR` | settings 的默认 runtime_dir（仅首次初始化 settings 时使用） |
| `OPENCLAW_DATA_DIR` | settings 的默认 data_dir（仅首次初始化 settings 时使用） |
| `OPENCLAW_GATEWAY_PORT_START/END` | settings 的默认 Gateway 端口范围（仅首次初始化 settings 时使用） |
| `OPENCLAW_BASE_DOMAIN` | settings 的默认 base_domain（仅首次初始化 settings 时使用） |
| `NGINX_SITE_DIR` / `NGINX_RELOAD_CMD` | 域名模式下 Nginx 配置写入与重载命令 |

### Settings 与 License 快照策略

1. `settings`：全局默认模板（可在 UI `Settings` 页面修改）。
2. `license`：创建时固化一份“生效快照”（`runtime_provider/runtime_dir/data_dir/nginx_host` 等）。
3. 域名优先级：`POST /api/licenses` 的 `baseDomain` > `settings.base_domain` > 空（IP:端口模式）。
4. provisioning 运行时以 license 行内值为准，避免后续改全局设置影响已发 license。

---

## 📂 文档索引

- [认证流程](./openclaw-tenant/docs/AUTHENTICATION.md)
- [后端 API 规范](./openclaw-tenant/docs/BACKEND_API.md)
- [许可证编排引擎](./openclaw-tenant/docs/LICENSE_PROVISIONING.md)
- [UI 设计规范](./openclaw-tenant/docs/UI_DESIGN.md)
- [环境变量说明](./openclaw-tenant/docs/ENVIRONMENT.md)

---

## 🔧 工作空间仓库策略

当前工作空间 **不再使用 Git submodule** 跟踪子项目。
根仓库只负责编排与文档，不提交子项目源码。

### 1. 首次拉取

```bash
git clone <easy-openclaw-repo-url>
cd easy-openclaw

# 在工作区根目录单独拉取子项目
git clone <openclaw-repo-url> openclaw
git clone <openclaw-mate-repo-url> openclaw-mate
git clone <openclaw-tenant-repo-url> openclaw-tenant
```

子项目地址请使用上面的“子项目仓库地址”列表。

---

## 🤝 参与贡献

欢迎任何形式的贡献！请先阅读各子项目根目录下的 `AGENTS.md` 了解代码规范。

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/your-feature`)
3. 提交修改 (`git commit -m 'feat: add your feature'`)
4. 推送分支 (`git push origin feature/your-feature`)
5. 发起 Pull Request

---

## 📄 许可证

本项目采用 [MIT License](LICENSE) 许可协议。
