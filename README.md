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

**Easy OpenClaw** 是一套围绕 [OpenClaw](https://github.com/openclaw/openclaw) 开源项目构建的私有化部署工具链。

- **openclaw**：基于官方开源版本修改并自托管的 AI Agent 核心服务，提供 Gateway WebSocket 服务和 AI 任务调度能力。
- **openclaw-exec**：运行在终端设备上的跨平台桌面节点（Tauri 2 + React），负责接收并执行云端下发的自动化任务。
- **openclaw-tenant**：私有化授权管理后台（Hono + Svelte 5），负责许可证颁发、HWID 设备绑定和 Docker 实例自动编排。

---

## 📦 项目结构

```text
easy-openclaw/
├── openclaw/          # 自托管的 OpenClaw 核心服务（基于官方开源修改）
├── openclaw-exec/     # 桌面端执行节点（Tauri 2 + React 18）
└── openclaw-tenant/   # 授权与租户管理后台（Hono API + Svelte 5）
```

---

## 🏗️ 整体架构

三个组件分为**服务端**与**客户端**两侧，通过 HTTP REST 和 WebSocket 长连接协作。

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                            服务端 / 管理侧 (Self-Hosted)                     ║
║                                                                              ║
║  ┌────────────────────────┐       ┌──────────────────────────────────────┐  ║
║  │    openclaw-tenant     │       │     openclaw（自托管核心服务）          │  ║
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
              │                             │   握手帧 (role=node)
              ▼                             ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║                           客户端设备 / 终端侧 (Client)                        ║
║                                                                              ║
║  ┌───────────────────────────────────────────────────────────────────────┐  ║
║  │                      openclaw-exec（桌面节点）                          │  ║
║  │                                                                       │  ║
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
  → POST /api/licenses                 # 创建许可证（可选 ownerTag / 到期日 / HWID）
      ↓
  [Tenant 后台 Worker 异步执行]
      ├── 分配 gateway_port / bridge_port
      ├── 执行 Docker Compose up，拉起一个独立的 openclaw 实例
      ├── 读取实例生成的 openclaw.json（含 gateway.auth.token）
      ├── [可选] 写入 Nginx 配置并刷新反向代理
      └── 更新 provision_status: pending → running → ready
```

### ② 用户首次激活（Verify & HWID 绑定）

```
openclaw-exec 桌面端启动，用户输入 licenseKey
  → [Rust auth_client] POST /api/verify { hwid, licenseKey, deviceName }
      ↓
  Tenant API 校验：
      ✓ licenseKey 有效 & 未撤销 & 未过期
      ✓ provision_status = ready（否则 409）
      ✓ 首次调用：绑定 HWID，激活 license
      ↓
  返回 nodeConfig：
      { gatewayUrl, gatewayToken, agentId, authToken }
      ↓
  exec 将 nodeConfig 持久化至本地 config.json
```

### ③ 建立 Gateway 长连接（WebSocket 握手）

```
openclaw-exec [Rust ws_client]
  → 读取 config.json（gatewayUrl + authToken）
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

### ⑤ authToken 自动轮换

```
Token 到期（超过 token_ttl_days）后，下次 POST /api/verify 时：
  → Tenant 自动生成新 authToken
  → 更新数据库 auth_token / token_expires_at
  → 同步写入对应 openclaw 实例的 openclaw.json
  （对用户完全透明，无需管理员干预）
```

---

## ✨ 核心功能

### openclaw-exec（桌面节点）

- **跨平台支持**：Windows / macOS / Linux（基于 Tauri 2）
- **实时任务接收**：WebSocket 长连接，低延迟接收云端指令
- **本地任务执行**：支持 `system.run`（shell）、`browser`（浏览器自动化）、`vision`（视觉识别）
- **原生系统集成**：系统托盘静默运行、原生通知、本地数据持久化
- **Sidecar 扩展**：Node.js 子进程通过 stdin/stdout IPC 处理高阶自动化任务

### openclaw-tenant（授权管理后台）

- **许可证管理**：创建、撤销、设置到期时间与 Token 轮换周期
- **HWID 设备绑定**：首次激活后锁定物理设备，防止许可证共享
- **Docker 异步编排**：自动 Compose up 拉起 openclaw 实例，追踪 `pending → running → ready → failed` 状态
- **多租户 Token 缓存**：每个 License 独立 authToken，定期自动轮换

---

## 🛠️ 技术栈

| 组件 | 技术 |
|------|------|
| **openclaw** | 官方开源自托管，Node.js / TypeScript |
| **openclaw-exec 前端** | React 18 · React Router v6 · Zustand · Tailwind CSS · Vite |
| **openclaw-exec 核心层** | Tauri v2 · Rust · Tokio · Tokio-Tungstenite · Reqwest |
| **openclaw-exec Sidecar** | Node.js · TypeScript（browser / system / vision 模块） |
| **openclaw-tenant 后端** | Hono · SQLite（bun:sqlite）· JWT（HS256）· bcrypt |
| **openclaw-tenant 前端** | Svelte 5 (Runes) · TailwindCSS v4 · Vite |
| **运行时** | Bun（tenant） · Node.js ≥ 18（exec 构建） · Rust 稳定版 |

---

## 🚀 快速开始

### 前置依赖

| 工具 | 用途 | 版本要求 |
|------|------|----------|
| [Node.js](https://nodejs.org/) | openclaw-exec 构建 | ≥ 18.x |
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

### 2. 启动 openclaw-exec（桌面节点）

```bash
cd openclaw-exec

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
| `OPENCLAW_HOST_IP` | 宿主机 IP，用于生成节点接入地址 |
| `OPENCLAW_RUNTIME_DIR` | openclaw 实例运行目录 |
| `OPENCLAW_DATA_DIR` | openclaw 数据持久化目录 |
| `OPENCLAW_GATEWAY_PORT_START/END` | Gateway 端口分配范围 |
| `OPENCLAW_BASE_DOMAIN` | （可选）启用域名模式，自动配置 Nginx 反代 |

---

## 📂 文档索引

- [认证流程](./openclaw-tenant/docs/AUTHENTICATION.md)
- [后端 API 规范](./openclaw-tenant/docs/BACKEND_API.md)
- [许可证编排引擎](./openclaw-tenant/docs/LICENSE_PROVISIONING.md)
- [UI 设计规范](./openclaw-tenant/docs/UI_DESIGN.md)
- [环境变量说明](./openclaw-tenant/docs/ENVIRONMENT.md)

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
