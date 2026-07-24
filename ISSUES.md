# OpenClaw 配置与源码问题记录

> 本文档记录调试过程中发现的问题、解决方案及注意事项。
> 涉及源码修改的部分需要重新构建 Docker 镜像。

---

## 1. `tools.allow` vs `tools.alsoAllow` 配置错误

**问题：** 使用 `tools.allow` 会完全替换 profile 的默认工具集，导致 messaging profile 的所有工具（message、sessions 等）丢失。

**解决：** 改用 `tools.alsoAllow`，在 profile 基础上叠加工具组。

```json
"tools": {
  "profile": "messaging",
  "alsoAllow": ["group:fs", "group:runtime"]
}
```

---

## 2. `models.providers.zai.baseUrl` 尾部斜杠

**问题：** `baseUrl` 末尾带斜杠 `/` 会导致拼接 URL 时出现双斜杠。

**解决：** 去掉尾部斜杠。

```json
"baseUrl": "https://open.bigmodel.cn/api/paas/v4"
```

---

## 3. GLM 模型 `reasoning: true` 导致 400 错误

**问题：** 模型配置中 `reasoning: true` 会让 openclaw 向 ZAI API 注入 `reasoning: { effort: "low" }` 参数，但 GLM 不支持此参数，返回 400。

**解决：** 视觉模型（如 `glm-4.6V-FlashX`）设置 `reasoning: false`。

```json
{
  "id": "glm-4.6V-FlashX",
  "reasoning": false,
  "input": ["text", "image"]
}
```

---

## 4. `exec.host = "node"` 无法回退导致 exec 完全不可用

**问题：** 配置 `tools.exec.host: "node"` 时，若 openclaw-exec 伴侣应用未连接，agent 调用 exec 工具会直接抛出错误：
```
exec host=node requires a paired node (none available).
```
exec 工具完全不可用，无法执行任何命令。

**解决（源码修改）：** 在 `bash-tools.exec.ts` 中增加 fallback 逻辑：节点不可用时自动降级到容器本地执行。

**修改文件：** `openclaw/src/agents/bash-tools.exec.ts`

**修改内容：**

1. 新增 import：
```typescript
import { listNodes } from "./tools/nodes-utils.js";
```

2. 修改 `host === "node"` 分支（原来直接 return，现在先检查节点可用性）：
```typescript
if (host === "node") {
  const availableNodes = await listNodes({});
  if (availableNodes.length > 0) {
    return executeNodeHostCommand({
      command: params.command,
      workdir,
      env,
      requestedEnv: params.env,
      requestedNode: params.node?.trim(),
      boundNode: defaults?.node?.trim(),
      sessionKey: defaults?.sessionKey,
      turnSourceChannel: defaults?.messageProvider,
      turnSourceTo: defaults?.currentChannelId,
      turnSourceAccountId: defaults?.accountId,
      turnSourceThreadId: defaults?.currentThreadTs,
      agentId,
      security,
      ask,
      timeoutSec: params.timeout,
      defaultTimeoutSec,
      approvalRunningNoticeMs,
      warnings,
      notifySessionKey,
      trustedSafeBinDirs,
    });
  }
  // No paired node available: fall back to local execution
  logInfo("exec host=node: no paired node available, falling back to local execution");
  warnings.push("⚠️ No paired node available, running locally in container.");
}
```

> ⚠️ **此修改需要重新构建 Docker 镜像**

---

## 5. `tools.exec.security` 有效值范围

**问题：** `tools.exec.security` 只接受 `"deny"` | `"allowlist"` | `"full"`，设置 `"ask"` 会报配置校验错误。

**解决：** 使用 `"full"` 允许容器内执行所有命令（容器本身是隔离环境）。

```json
"exec": {
  "security": "full"
}
```

---

## 6. Agent MEMORY.md 记录了错误的 workspace 路径

**问题：** Agent 首次启动时将 workspace 路径写入 `MEMORY.md`，后续每次启动读取该文件，导致认为只能访问 `/home/node/.openclaw/workspace`，拒绝访问 media 目录。

**解决：** 手动编辑 `workspace/memory/MEMORY.md`，补充正确信息：

```markdown
- Workspace: /home/node/.openclaw/workspace
- Media files (inbound images): /home/node/.openclaw/media/inbound/
- File access is NOT limited to workspace — tools can read/write any path in the container (tools.fs.workspaceOnly = false)
- 列出目录内容: 用 `exec` 工具运行 `ls /path/` — `read` 工具只能读文件，不能读目录(EISDIR)
```

---

## 7. 飞书图片识别配置

**问题：** 图片保存到 `media/inbound/` 后，agent 没有自动识别图片内容。

**根本原因：** 当主模型支持 vision（`input: ["text", "image"]`），媒体理解流水线会被跳过。需要主模型为纯文本模型，并单独配置图片理解模型。

**解决：**

```json
"agents": {
  "defaults": {
    "model": { "primary": "zai/glm-4.7" }
  }
},
"tools": {
  "media": {
    "image": {
      "models": [
        { "type": "provider", "provider": "zai", "model": "glm-4.6V-FlashX" }
      ]
    }
  }
}
```

流程：`glm-4.6V-FlashX` 描述图片 → 描述文字注入消息上下文 → `glm-4.7` 处理并回复。

---

## 8. workspace/skills 中的 image-recognizer skill 干扰

**问题：** `workspace/skills/image-recognizer/SKILL.md` 引导 agent 尝试用 tesseract 等 OCR 工具手动识别图片，干扰内置媒体理解流水线。

**解决：** 删除该 skill 目录。内置流水线会自动处理。

---

## Docker 重新构建步骤

以下修改涉及源码，需要重新构建镜像：

- **问题 4**（exec fallback 策略）：`openclaw/src/agents/bash-tools.exec.ts`

```bash
# 在项目根目录执行
docker compose build openclaw

# 重新创建容器（配置会自动挂载）
docker compose up -d --force-recreate openclaw
```

> 容器重建后，`openclaw-data` 目录下的配置文件（`openclaw.json`、`MEMORY.md` 等）无需重新配置，数据卷会自动挂载。

---

## 当前生效配置（openclaw.json 关键字段）

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "zai": {
        "baseUrl": "https://open.bigmodel.cn/api/paas/v4",
        "api": "openai-completions",
        "models": [
          {
            "id": "glm-4.7",
            "reasoning": true,
            "input": ["text"]
          },
          {
            "id": "glm-4.6V-FlashX",
            "reasoning": false,
            "input": ["text", "image"]
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": { "primary": "zai/glm-4.7" },
      "workspace": "/home/node/.openclaw/workspace"
    }
  },
  "tools": {
    "profile": "messaging",
    "alsoAllow": ["group:fs", "group:runtime"],
    "exec": {
      "host": "node",
      "security": "full"
    },
    "fs": { "workspaceOnly": false },
    "media": {
      "image": {
        "models": [
          { "type": "provider", "provider": "zai", "model": "glm-4.6V-FlashX" }
        ]
      }
    }
  }
}
```

---

## 9. 飞书群聊“看起来不回复”排障记录（2026-03-09）

### 现象

- 群里发消息，机器人有时不回复。
- 一度出现日志：`did not mention bot`。
- 后续即使配置了 `requireMention=false`，仍偶发“看起来没回复”。

### 根因拆解

这次实际是**两个问题叠加**：

1. **消息门控问题（配置侧）**
   - `requireMention=true` 时，群消息未 @ 会被拦截，日志会出现：
     - `message in group ... did not mention bot`
   - 修复后（`requireMention=false`），非 @ 消息可以进入 agent。

2. **回包网络问题（运行环境侧）**
   - 消息已进入 gateway 并成功 `dispatching to agent`，但回飞书时失败：
     - `final reply failed: Error: connect ECONNREFUSED 198.18.0.21:443`
   - 表现为“看起来机器人没回复”，但并非未处理消息。

### 关键证据

- 非 @ 群消息进入网关并分发（已生效）：
  - `received message ... (group)`
  - `Feishu[default] message in group ...: 在吗`
  - `dispatching to agent (session=agent:feishu-group:...)`
- 同时间窗内存在飞书回包失败：
  - `connect ECONNREFUSED 198.18.0.21:443`

### 修复动作（已执行）

1. 清理多实例干扰，仅保留一个 gateway 容器：
   - 保留：`openclaw-olsond2-2-openclaw-gateway-1`
2. 设置并确认群聊无需 @：
```json5
{
  "channels": {
    "feishu": {
      "requireMention": false,
      "groups": {
        "oc_7c74c455bceb332b6b73a8ff348f7ccb": {
          "requireMention": false
        }
      }
    }
  }
}
```
3. 清理 `groupAllowFrom` 中错误空格（避免切回 allowlist 时踩坑）：
```json5
"groupAllowFrom": ["oc_7c74c455bceb332b6b73a8ff348f7ccb"]
```

### 结论

- `requireMention=false` **是有效的**，但只对“已到达 gateway 的消息”生效。
- 如果日志里没有 `received message ... (group)`，问题在飞书投递侧。
- 如果有 `dispatching to agent` 但无最终回复，优先检查网络/代理链路（尤其 `ECONNREFUSED 198.18.0.21:443`）。

### 建议排查顺序（固定流程）

1. 先确认只跑一个 gateway 实例（避免看错容器日志）。
2. 看入站：是否出现 `received message ... (group)`。
3. 看门控：是否出现 `did not mention bot`。
4. 看回包：是否出现 `final reply failed` / `ECONNREFUSED`。
5. 再回头检查飞书侧权限、事件订阅、机器人接收范围。

---

## 10. 第二个 Agent 显式固化浏览器路径（容器环境）

### 背景

在容器环境中，浏览器真实二进制路径是：

- `/home/node/.cache/ms-playwright/chromium-1208/chrome-linux64/chrome`

虽然 `browser.executablePath` 已配置，但某些回合模型会走 `exec` 侧自检 `chromium` 命令，导致“找不到浏览器”类误判。

### 解决（已生效）

1. 显式设置浏览器默认 profile：

```json5
"browser": {
  "defaultProfile": "openclaw",
  "executablePath": "/home/node/.cache/ms-playwright/chromium-1208/chrome-linux64/chrome",
  "headless": true,
  "noSandbox": true
}
```

2. 对第二个 agent（`feishu-group`）显式固定 exec 到容器本地并补 PATH：

```json5
"agents": {
  "list": [
    { "id": "main" },
    {
      "id": "feishu-group",
      "tools": {
        "exec": {
          "host": "gateway",
          "pathPrepend": [
            "/home/node/.openclaw/bin",
            "/home/node/.cache/ms-playwright/chromium-1208/chrome-linux64"
          ]
        }
      }
    }
  ]
}
```

3. 在容器内增加命令别名包装（便于模型执行 `chromium`/`google-chrome`）：

- `/usr/local/bin/chromium`
- `/usr/local/bin/chromium-browser` -> `chromium`
- `/usr/local/bin/google-chrome` -> `chromium`

包装器内容：

```sh
#!/bin/sh
exec /home/node/.cache/ms-playwright/chromium-1208/chrome-linux64/chrome "$@"
```

验证：

```sh
chromium --version
# Google Chrome for Testing 145.0.7632.6
```

### 注意

- `~/.openclaw/bin` 在某些挂载场景可能是 `noexec`，脚本存在但不可直接执行；因此命令包装建议放在 `/usr/local/bin`。
