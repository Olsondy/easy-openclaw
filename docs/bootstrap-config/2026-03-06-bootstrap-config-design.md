# Bootstrap Config 设计文档

> 日期：2026-03-06
> 状态：已审批，待实现

## 概述

provision 预设最小可运行配置 + exec 首次引导向导补齐用户私有配置，实现"安装 exec → 输入 license → 向导填 2 项 → 自动下发容器 → 热重载 → 立即可用"的完整体验。

---

## 一、整体架构与数据流

```
管理员
  → Settings UI → PUT /api/settings/model-presets/:provider_id
  → DB: model_presets（apiKey AES-256-GCM 加密存储）

License 创建
  → licenseProvisioningService
      ├── runProvisionScript()
      │     ├── 写基线 openclaw.json（gateway + tools.exec + models 结构体，不含 apiKey）
      │     ├── docker run --rm <image> node dist/index.js plugins install ./extensions/feishu
      │     └── docker compose up -d openclaw-gateway
      └── patchModelApiKey()
            ├── 查 model_presets WHERE enabled=1
            ├── 解密 api_key_enc
            └── patch openclaw.json → models.providers.<id>.apiKey

用户安装 exec
  → POST /api/verify
  → 返回 nodeConfig + needsBootstrap: { feishu: true/false }

  → (feishu: true) exec 弹出飞书配置向导
      └── 用户填 appId + appSecret
  → POST /api/licenses/:id/bootstrap-config（authToken 鉴权）
      ├── 白名单字段校验
      ├── 写入 channels.feishu.{appId, appSecret} 到 openclaw.json
      ├── 置 wizard_feishu_done = 1
      └── 飞书热重载自动生效（channels.feishu 是 hot 规则）

  → exec 自动重连 Gateway → 可用

用户手动重配置（已完成向导后）
  → exec Settings 页 → "飞书配置" 入口（始终可见）
  → 同 POST /api/licenses/:id/bootstrap-config（幂等）
  → 覆盖写入，热重载生效，wizard_feishu_done 保持 1
```

---

## 二、DB Schema 变更

### 2.1 新表 `model_presets`

```sql
CREATE TABLE IF NOT EXISTS model_presets (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  provider_id  TEXT NOT NULL UNIQUE,
  label        TEXT NOT NULL,
  base_url     TEXT NOT NULL,
  api          TEXT NOT NULL,        -- openclaw api type，e.g. "openai-completions"
  model_id     TEXT NOT NULL,
  api_key_enc  TEXT,                 -- AES-256-GCM 加密，null 表示未配置
  enabled      INTEGER NOT NULL DEFAULT 1,
  updated_at   TEXT NOT NULL DEFAULT (datetime('now'))
);

-- 预置种子数据
INSERT OR IGNORE INTO model_presets
  (provider_id, label, base_url, api, model_id, enabled)
VALUES
  ('zhipuai', 'GLM-4-Flash (智谱AI)',
   'https://open.bigmodel.cn/api/paas/v4/',
   'openai-completions', 'glm-4-flash-250414', 1);
```

### 2.2 `licenses` 表新增字段

```sql
ALTER TABLE licenses ADD COLUMN wizard_feishu_done INTEGER NOT NULL DEFAULT 0;
-- 后续扩展示例（model 向导上线时再加）：
-- ALTER TABLE licenses ADD COLUMN wizard_model_done INTEGER NOT NULL DEFAULT 0;
```

`wizard_*_done` 字段含义：
- `0` = 该步骤未完成，verify 返回对应 needsBootstrap key 为 true
- `1` = 已完成，verify 不再触发该步骤向导

---

## 三、加密方案

模块：`packages/api/src/services/crypto.ts`

- 算法：AES-256-GCM
- 密钥：`SHA-256(JWT_SECRET)` → 32 bytes
- 存储格式：`<iv_hex>:<authTag_hex>:<ciphertext_hex>`

```typescript
encryptApiKey(plaintext: string, secret: string): string
decryptApiKey(encoded: string, secret: string): string
```

安全约束：
- apiKey 原文永不出现在日志、响应体、错误信息
- 解密失败直接 throw，不 fallback 明文
- `JWT_SECRET` 长度 < 32 字符时启动时报错退出

---

## 四、provision-docker.sh 改动

### 4.1 openclaw.json 基线模板新增字段

```json
{
  "gateway": { "...existing..." },
  "tools": {
    "exec": {
      "host": "node",
      "security": "allowlist",
      "ask": "on-miss"
    }
  },
  "models": {
    "providers": {
      "zhipuai": {
        "baseUrl": "https://open.bigmodel.cn/api/paas/v4/",
        "api": "openai-completions",
        "models": [
          {
            "id": "glm-4-flash-250414",
            "name": "GLM-4-Flash (Free)",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

注：`apiKey` 不在模板里，由 `patchModelApiKey()` 在 provision 后注入。

### 4.2 新增插件安装步骤（在 gateway 启动前）

```bash
# 安装飞书插件（直接 docker run，避免 depends_on 约束）
echo "==> Installing feishu plugin"
"$RUNTIME_CMD" run --rm \
  -v "${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw" \
  "$IMAGE_NAME" \
  node dist/index.js plugins install ./extensions/feishu
```

执行顺序：chown 修权 → **安装飞书插件** → docker compose up gateway

---

## 五、后端 API 变更（openclaw-tenant）

### 5.1 `licenseProvisioningService` 新增 `patchModelApiKey()`

```typescript
async function patchModelApiKey(configDir: string): Promise<void>
```

- 查 `model_presets WHERE enabled=1`
- 跳过 `api_key_enc IS NULL` 的行
- 解密后 patch 到 `data.models.providers[provider_id].apiKey`
- 写回 openclaw.json
- 全程不记录 apiKey 原文

在 `runProvisionScript()` 完成、gateway 启动之前调用。

### 5.2 `POST /api/verify` 响应变更

新增 `needsBootstrap` 字段：

```typescript
// 判断逻辑
const needsBootstrap = {
  feishu: license.wizard_feishu_done === 0,
};

// 响应
return c.json({
  success: true,
  data: {
    nodeConfig: { ... },
    userProfile: { ... },
    needsBootstrap,   // { feishu: true/false }
  },
});
```

### 5.3 新接口 `POST /api/licenses/:id/bootstrap-config`

**鉴权**：请求体携带 `authToken`，与 `licenses.auth_token` 比对（防越权）

**请求体**：
```typescript
{
  authToken: string;
  feishu?: {
    appId: string;      // 非空字符串
    appSecret: string;  // 非空字符串
  };
}
```

**白名单写入字段**（仅允许这两个路径，禁止任意 JSON）：
- `channels.feishu.appId`
- `channels.feishu.appSecret`

**处理流程**：
1. 校验 authToken → 403 if mismatch
2. 校验 feishu.appId / appSecret 非空 → 400
3. 读取 openclaw.json → 深合并白名单字段 → 写回
4. `UPDATE licenses SET wizard_feishu_done=1 WHERE id=?`
5. 返回 `{ success: true, data: { applied: ["feishu"] } }`

**热重载**：`channels.feishu` 是 hot 规则（restart-channel:feishu），写文件后 gateway 自动重载，无需容器重启。

### 5.4 新增 model_presets 管理接口

```
GET  /api/settings/model-presets          → 列出所有 preset（api_key_enc 不返回原文，用 "***" 掩码）
PUT  /api/settings/model-presets/:id      → 更新 apiKey（加密后存储）
```

两个接口均需 `jwtMiddleware` 鉴权（管理员专用）。

---

## 六、exec 端改动

### 6.1 首次向导

verify 返回后检查 `needsBootstrap`：

```
needsBootstrap.feishu === true
  → 显示飞书配置向导（全屏或 Modal）
  → 字段：appId（text）、appSecret（password）
  → 本地校验：两者均非空
  → 提交 POST /api/licenses/:id/bootstrap-config
  → 成功 → 关闭向导 → 重连 Gateway WebSocket
```

### 6.2 Settings 页新增飞书配置入口

**位置**：exec Settings 页面，始终可见（不依赖 wizard_feishu_done 状态）

**行为**：
- 点击 → 弹出与首次向导相同的配置表单
- 提交同一个 `POST /api/licenses/:id/bootstrap-config` 接口（幂等）
- 成功 → 提示"配置已更新，飞书热重载中…"
- `wizard_feishu_done` 保持 1，不重置

---

## 七、安全要求

| 项目 | 要求 |
|------|------|
| apiKey 存储 | AES-256-GCM 加密，原文不落库 |
| apiKey 日志 | 全链路脱敏，不出现在任何 log |
| bootstrap-config 鉴权 | authToken 与 license 绑定，防越权改他人容器 |
| 写入字段 | 严格白名单，禁止任意 JSON 注入 |
| model_presets 接口 | jwtMiddleware 管理员专用，返回值掩码 |
| JWT_SECRET | 启动时校验长度 ≥ 32 字符 |

---

## 八、后续扩展（不在本期实现）

- `wizard_model_done` 字段 + model 配置向导
- `needsBootstrap.model` key
- model_presets 支持多条（多 provider）
