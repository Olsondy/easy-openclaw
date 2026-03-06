# Bootstrap Config Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** provision 预设最小可运行配置（GLM-4-Flash + tools.exec.host=node + 飞书插件），exec 首次向导补齐飞书 appId/appSecret，自动下发容器热重载，实现开箱即用体验。

**Architecture:** tenant 新增 `model_presets` 表（加密存储 apiKey）和 `wizard_feishu_done` 字段，provision 时注入模型 apiKey 并预装飞书插件；verify 响应携带 `needsBootstrap` 结构，exec 首次弹向导，提交到新接口 `POST /api/licenses/:id/bootstrap-config`，tenant 写入 openclaw.json 触发热重载；exec Settings 页保留飞书配置入口支持随时修改。

**Tech Stack:**
- tenant: Bun + Hono + bun:sqlite（`packages/api`），Biome lint，`bun test`
- exec: React 18 + TypeScript + Zustand + Tauri v2，Vitest，Biome lint
- openclaw: provision-docker.sh（bash）

---

## Task 1: DB Migration — model_presets 表 + 种子数据

**Files:**
- Modify: `openclaw-tenant/packages/api/src/db/schema.ts`
- Test: `openclaw-tenant/packages/api/src/db/schema.test.ts`

**Step 1: 写失败测试**

在 `schema.test.ts` 末尾追加：

```typescript
it("model_presets 表存在且含种子数据", () => {
  const db = getDb();
  const row = db
    .query<{ provider_id: string; model_id: string }, []>(
      "SELECT provider_id, model_id FROM model_presets WHERE provider_id = 'zhipuai'",
    )
    .get();
  expect(row?.provider_id).toBe("zhipuai");
  expect(row?.model_id).toBe("glm-4-flash-250414");
});
```

**Step 2: 运行确认失败**

```bash
cd openclaw-tenant
bun run --cwd packages/api test -- src/db/schema.test.ts
```

预期：FAIL — `model_presets` 不存在

**Step 3: 在 `schema.ts` 的 `SCHEMA_SQL` 末尾追加**

```typescript
  CREATE TABLE IF NOT EXISTS model_presets (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    provider_id  TEXT NOT NULL UNIQUE,
    label        TEXT NOT NULL,
    base_url     TEXT NOT NULL,
    api          TEXT NOT NULL,
    model_id     TEXT NOT NULL,
    api_key_enc  TEXT,
    enabled      INTEGER NOT NULL DEFAULT 1,
    updated_at   TEXT NOT NULL DEFAULT (datetime('now'))
  );

  INSERT OR IGNORE INTO model_presets
    (provider_id, label, base_url, api, model_id, enabled)
  VALUES
    ('zhipuai', 'GLM-4-Flash (智谱AI)',
     'https://open.bigmodel.cn/api/paas/v4/',
     'openai-completions', 'glm-4-flash-250414', 1);
```

**Step 4: 运行确认通过**

```bash
bun run --cwd packages/api test -- src/db/schema.test.ts
```

预期：PASS

**Step 5: Commit**

```bash
git add openclaw-tenant/packages/api/src/db/schema.ts \
        openclaw-tenant/packages/api/src/db/schema.test.ts
git commit -m "feat(tenant/db): add model_presets table with zhipuai seed"
```

---

## Task 2: DB Migration — licenses.wizard_feishu_done 字段

**Files:**
- Modify: `openclaw-tenant/packages/api/src/db/schema.ts`
- Test: `openclaw-tenant/packages/api/src/db/schema.test.ts`

**Step 1: 写失败测试**

```typescript
it("licenses 表含 wizard_feishu_done 字段默认值为 0", () => {
  const db = getDb();
  db.run(
    `INSERT INTO licenses (license_key, gateway_token, gateway_url)
     VALUES ('TEST-WIZARD-FIELD', 'tok', 'wss://test')`,
  );
  const row = db
    .query<{ wizard_feishu_done: number }, []>(
      "SELECT wizard_feishu_done FROM licenses WHERE license_key='TEST-WIZARD-FIELD'",
    )
    .get();
  expect(row?.wizard_feishu_done).toBe(0);
  db.run("DELETE FROM licenses WHERE license_key='TEST-WIZARD-FIELD'");
});
```

**Step 2: 运行确认失败**

```bash
bun run --cwd packages/api test -- src/db/schema.test.ts
```

**Step 3: 在 `SCHEMA_SQL` licenses 表定义末尾（`exec_public_key TEXT` 之后）追加**

```sql
    wizard_feishu_done   INTEGER NOT NULL DEFAULT 0
```

**Step 4: 运行确认通过**

```bash
bun run --cwd packages/api test -- src/db/schema.test.ts
```

**Step 5: Commit**

```bash
git add openclaw-tenant/packages/api/src/db/schema.ts \
        openclaw-tenant/packages/api/src/db/schema.test.ts
git commit -m "feat(tenant/db): add wizard_feishu_done to licenses"
```

---

## Task 3: Crypto 服务（AES-256-GCM）

**Files:**
- Create: `openclaw-tenant/packages/api/src/services/crypto.ts`
- Create: `openclaw-tenant/packages/api/src/services/crypto.test.ts`

**Step 1: 写失败测试**

```typescript
// crypto.test.ts
import { expect, it, describe } from "bun:test";
import { encryptApiKey, decryptApiKey } from "./crypto";

describe("encryptApiKey / decryptApiKey", () => {
  const secret = "a".repeat(32);

  it("加密后可解密还原原文", () => {
    const plain = "sk-test-api-key-12345";
    const enc = encryptApiKey(plain, secret);
    expect(enc).not.toBe(plain);
    expect(decryptApiKey(enc, secret)).toBe(plain);
  });

  it("每次加密产生不同密文（随机 iv）", () => {
    const enc1 = encryptApiKey("same", secret);
    const enc2 = encryptApiKey("same", secret);
    expect(enc1).not.toBe(enc2);
  });

  it("篡改密文后解密抛出错误", () => {
    const enc = encryptApiKey("value", secret);
    const tampered = enc.slice(0, -4) + "xxxx";
    expect(() => decryptApiKey(tampered, secret)).toThrow();
  });

  it("secret 过短时加密抛出错误", () => {
    expect(() => encryptApiKey("value", "short")).toThrow("JWT_SECRET");
  });
});
```

**Step 2: 运行确认失败**

```bash
bun run --cwd packages/api test -- src/services/crypto.test.ts
```

**Step 3: 实现 `crypto.ts`**

```typescript
import { createCipheriv, createDecipheriv, createHash, randomBytes } from "node:crypto";

function deriveKey(secret: string): Buffer {
  if (secret.length < 32) {
    throw new Error("JWT_SECRET must be at least 32 characters for AES-256-GCM encryption");
  }
  return createHash("sha256").update(secret).digest();
}

/** 加密 apiKey，返回 iv:authTag:ciphertext（hex 拼接） */
export function encryptApiKey(plaintext: string, secret: string): string {
  const key = deriveKey(secret);
  const iv = randomBytes(12);
  const cipher = createCipheriv("aes-256-gcm", key, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, "utf8"), cipher.final()]);
  const authTag = cipher.getAuthTag();
  return `${iv.toString("hex")}:${authTag.toString("hex")}:${encrypted.toString("hex")}`;
}

/** 解密，失败直接抛出（不 fallback 明文） */
export function decryptApiKey(encoded: string, secret: string): string {
  const key = deriveKey(secret);
  const parts = encoded.split(":");
  if (parts.length !== 3) throw new Error("Invalid encrypted format");
  const [ivHex, authTagHex, ciphertextHex] = parts;
  const iv = Buffer.from(ivHex, "hex");
  const authTag = Buffer.from(authTagHex, "hex");
  const ciphertext = Buffer.from(ciphertextHex, "hex");
  const decipher = createDecipheriv("aes-256-gcm", key, iv);
  decipher.setAuthTag(authTag);
  return decipher.update(ciphertext).toString("utf8") + decipher.final("utf8");
}
```

**Step 4: 运行确认通过**

```bash
bun run --cwd packages/api test -- src/services/crypto.test.ts
```

**Step 5: Commit**

```bash
git add openclaw-tenant/packages/api/src/services/crypto.ts \
        openclaw-tenant/packages/api/src/services/crypto.test.ts
git commit -m "feat(tenant): add AES-256-GCM crypto service for apiKey storage"
```

---

## Task 4: model_presets 管理接口

**Files:**
- Create: `openclaw-tenant/packages/api/src/routes/model-presets.ts`
- Create: `openclaw-tenant/packages/api/src/routes/model-presets.test.ts`
- Modify: `openclaw-tenant/packages/api/src/index.ts`

**Step 1: 写失败测试**

```typescript
// model-presets.test.ts
import { expect, it, beforeEach, describe } from "bun:test";
import { Hono } from "hono";
import { initDb } from "../db/client";
import modelPresetsRouter from "./model-presets";

const JWT_SECRET = "a".repeat(32);

function buildApp() {
  const app = new Hono();
  app.route("/api/settings/model-presets", modelPresetsRouter);
  return app;
}

beforeEach(() => { initDb(":memory:"); });

describe("GET /api/settings/model-presets", () => {
  it("返回 preset 列表，api_key_enc 用 *** 掩码", async () => {
    const res = await buildApp().request("/api/settings/model-presets", {
      headers: { Authorization: "Bearer valid-admin-token" },
    });
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body.success).toBe(true);
    const preset = body.data.find((p: { provider_id: string }) => p.provider_id === "zhipuai");
    expect(preset).toBeDefined();
    expect(preset.api_key_masked).toBe(false); // 未配置
  });
});

describe("PUT /api/settings/model-presets/:provider_id", () => {
  it("保存加密 apiKey 后，GET 返回 masked=true", async () => {
    const app = buildApp();
    const put = await app.request("/api/settings/model-presets/zhipuai", {
      method: "PUT",
      headers: {
        "Content-Type": "application/json",
        Authorization: "Bearer valid-admin-token",
      },
      body: JSON.stringify({ apiKey: "sk-test-key" }),
    });
    expect(put.status).toBe(200);

    const get = await app.request("/api/settings/model-presets", {
      headers: { Authorization: "Bearer valid-admin-token" },
    });
    const body = await get.json();
    const preset = body.data.find((p: { provider_id: string }) => p.provider_id === "zhipuai");
    expect(preset.api_key_masked).toBe(true);
  });
});
```

**Step 2: 运行确认失败**

```bash
bun run --cwd packages/api test -- src/routes/model-presets.test.ts
```

**Step 3: 实现 `model-presets.ts`**

```typescript
import { Hono } from "hono";
import { getDb } from "../db/client";
import { encryptApiKey } from "../services/crypto";
import { jwtMiddleware } from "../middleware/jwt";

const router = new Hono();
router.use("/*", jwtMiddleware);

router.get("/", (c) => {
  const db = getDb();
  const rows = db
    .query<{
      id: number;
      provider_id: string;
      label: string;
      base_url: string;
      api: string;
      model_id: string;
      api_key_enc: string | null;
      enabled: number;
    }, []>("SELECT id, provider_id, label, base_url, api, model_id, api_key_enc, enabled FROM model_presets")
    .all();

  const data = rows.map((r) => ({
    id: r.id,
    provider_id: r.provider_id,
    label: r.label,
    base_url: r.base_url,
    api: r.api,
    model_id: r.model_id,
    api_key_masked: r.api_key_enc !== null,
    enabled: r.enabled === 1,
  }));

  return c.json({ success: true, data });
});

router.put("/:provider_id", async (c) => {
  const providerId = c.req.param("provider_id");
  const secret = process.env.JWT_SECRET ?? "";

  let body: { apiKey?: string; enabled?: boolean };
  try {
    body = await c.req.json();
  } catch {
    return c.json({ success: false, error: "INVALID_JSON" }, 400);
  }

  const db = getDb();
  const existing = db
    .query<{ id: number }, string>(
      "SELECT id FROM model_presets WHERE provider_id = ?",
    )
    .get(providerId);

  if (!existing) {
    return c.json({ success: false, error: "NOT_FOUND" }, 404);
  }

  if (body.apiKey !== undefined) {
    if (!body.apiKey.trim()) {
      return c.json({ success: false, error: "API_KEY_EMPTY" }, 400);
    }
    const encrypted = encryptApiKey(body.apiKey, secret);
    db.run(
      "UPDATE model_presets SET api_key_enc=?, updated_at=datetime('now') WHERE provider_id=?",
      [encrypted, providerId],
    );
  }

  if (body.enabled !== undefined) {
    db.run(
      "UPDATE model_presets SET enabled=?, updated_at=datetime('now') WHERE provider_id=?",
      [body.enabled ? 1 : 0, providerId],
    );
  }

  return c.json({ success: true });
});

export default router;
```

**Step 4: 在 `index.ts` 注册路由**

在现有路由挂载区域（`app.route("/api/licenses", licenses)` 附近）追加：

```typescript
import modelPresets from "./routes/model-presets.js";
// ...
app.route("/api/settings/model-presets", modelPresets);
```

**Step 5: 运行确认通过**

```bash
bun run --cwd packages/api test -- src/routes/model-presets.test.ts
```

**Step 6: Commit**

```bash
git add openclaw-tenant/packages/api/src/routes/model-presets.ts \
        openclaw-tenant/packages/api/src/routes/model-presets.test.ts \
        openclaw-tenant/packages/api/src/index.ts
git commit -m "feat(tenant): add model_presets API (GET/PUT with encrypted apiKey)"
```

---

## Task 5: patchModelApiKey — provision 后注入模型 apiKey

**Files:**
- Create: `openclaw-tenant/packages/api/src/services/provisioning/patchModelApiKey.ts`
- Create: `openclaw-tenant/packages/api/src/services/provisioning/patchModelApiKey.test.ts`
- Modify: `openclaw-tenant/packages/api/src/services/provisioning/licenseProvisioningService.ts`

**Step 1: 写失败测试**

```typescript
// patchModelApiKey.test.ts
import { expect, it, beforeEach } from "bun:test";
import { join } from "path";
import { mkdir, writeFile, readFile, rm } from "fs/promises";
import { initDb } from "../../db/client";
import { patchModelApiKey } from "./patchModelApiKey";
import { encryptApiKey } from "../crypto";

const TMP = "/tmp/test-patch-model";
const SECRET = "a".repeat(32);

beforeEach(async () => {
  await rm(TMP, { recursive: true, force: true });
  await mkdir(TMP, { recursive: true });
  initDb(":memory:");
});

it("将 model_presets 中的 apiKey 注入到 openclaw.json", async () => {
  const db = (await import("../../db/client")).getDb();
  db.run(
    `INSERT OR IGNORE INTO model_presets (provider_id, label, base_url, api, model_id, api_key_enc, enabled)
     VALUES ('zhipuai', 'GLM', 'https://test/', 'openai-completions', 'glm-4', ?, 1)`,
    [encryptApiKey("sk-live-key", SECRET)],
  );

  const configPath = join(TMP, "openclaw.json");
  await writeFile(
    configPath,
    JSON.stringify({
      gateway: { auth: { token: "tok" } },
      models: { providers: { zhipuai: { baseUrl: "https://test/", models: [] } } },
    }),
  );

  await patchModelApiKey(TMP, SECRET);

  const result = JSON.parse(await readFile(configPath, "utf8"));
  expect(result.models.providers.zhipuai.apiKey).toBe("sk-live-key");
});

it("api_key_enc 为 null 时跳过该 provider", async () => {
  const configPath = join(TMP, "openclaw.json");
  await writeFile(configPath, JSON.stringify({ models: { providers: {} } }));

  await patchModelApiKey(TMP, SECRET);

  const result = JSON.parse(await readFile(configPath, "utf8"));
  expect(result.models?.providers?.zhipuai?.apiKey).toBeUndefined();
});
```

**Step 2: 运行确认失败**

```bash
bun run --cwd packages/api test -- src/services/provisioning/patchModelApiKey.test.ts
```

**Step 3: 实现 `patchModelApiKey.ts`**

```typescript
import { readFile, writeFile } from "fs/promises";
import { join } from "path";
import { getDb } from "../../db/client.js";
import { decryptApiKey } from "../crypto.js";

export async function patchModelApiKey(configDir: string, secret: string): Promise<void> {
  const db = getDb();
  const presets = db
    .query<
      { provider_id: string; api_key_enc: string | null },
      []
    >("SELECT provider_id, api_key_enc FROM model_presets WHERE enabled=1")
    .all();

  const enabledWithKey = presets.filter((p) => p.api_key_enc !== null);
  if (enabledWithKey.length === 0) return;

  const configPath = join(configDir, "openclaw.json");
  let data: Record<string, unknown>;
  try {
    data = JSON.parse(await readFile(configPath, "utf8"));
  } catch {
    return; // 文件不存在，静默跳过
  }

  if (!data.models || typeof data.models !== "object") {
    data.models = {};
  }
  const models = data.models as Record<string, unknown>;
  if (!models.providers || typeof models.providers !== "object") {
    models.providers = {};
  }
  const providers = models.providers as Record<string, Record<string, unknown>>;

  for (const { provider_id, api_key_enc } of enabledWithKey) {
    let apiKey: string;
    try {
      apiKey = decryptApiKey(api_key_enc!, secret);
    } catch {
      // 解密失败：跳过，不记录原文
      console.error(`[patchModelApiKey] failed to decrypt apiKey for provider=${provider_id}`);
      continue;
    }
    if (!providers[provider_id]) {
      providers[provider_id] = {};
    }
    providers[provider_id] = { ...providers[provider_id], apiKey };
  }

  await writeFile(configPath, JSON.stringify(data, null, 2));
}
```

**Step 4: 在 `licenseProvisioningService.ts` 中调用**

在 `runProvisionScript()` 调用之后、`getContainerId()` 之前插入：

```typescript
import { patchModelApiKey } from "./patchModelApiKey.js";
// ...

// provision 脚本完成后注入模型 apiKey
const jwtSecret = process.env.JWT_SECRET ?? "";
await patchModelApiKey(configDir, jwtSecret);
```

**Step 5: 运行确认通过**

```bash
bun run --cwd packages/api test -- src/services/provisioning/patchModelApiKey.test.ts
```

**Step 6: Commit**

```bash
git add openclaw-tenant/packages/api/src/services/provisioning/patchModelApiKey.ts \
        openclaw-tenant/packages/api/src/services/provisioning/patchModelApiKey.test.ts \
        openclaw-tenant/packages/api/src/services/provisioning/licenseProvisioningService.ts
git commit -m "feat(tenant): inject model apiKey into openclaw.json after provision"
```

---

## Task 6: verify.ts — 新增 needsBootstrap

**Files:**
- Modify: `openclaw-tenant/packages/api/src/routes/verify.ts`
- Modify: `openclaw-tenant/packages/api/src/routes/verify.test.ts`

**Step 1: 在 `LicenseRow` 接口追加字段**

```typescript
wizard_feishu_done: number;
```

并在 SELECT 查询字段列表中加入 `wizard_feishu_done`。

**Step 2: 在响应体末尾追加 needsBootstrap**

```typescript
return c.json({
  success: true,
  data: {
    nodeConfig: { ... },
    userProfile: { ... },
    needsBootstrap: {
      feishu: license.wizard_feishu_done === 0,
    },
  },
});
```

**Step 3: 写测试（在 `verify.test.ts` 追加）**

```typescript
it("首次绑定时 needsBootstrap.feishu 为 true", async () => {
  // ... 正常 verify 流程
  const body = await res.json();
  expect(body.data.needsBootstrap.feishu).toBe(true);
});

it("wizard_feishu_done=1 时 needsBootstrap.feishu 为 false", async () => {
  const db = getDb();
  db.run("UPDATE licenses SET wizard_feishu_done=1 WHERE license_key=?", [licenseKey]);
  // 再次 verify
  const body = await res.json();
  expect(body.data.needsBootstrap.feishu).toBe(false);
});
```

**Step 4: 运行确认通过**

```bash
bun run --cwd packages/api test -- src/routes/verify.test.ts
```

**Step 5: Commit**

```bash
git add openclaw-tenant/packages/api/src/routes/verify.ts \
        openclaw-tenant/packages/api/src/routes/verify.test.ts
git commit -m "feat(tenant): add needsBootstrap.feishu to verify response"
```

---

## Task 7: bootstrap-config 接口

**Files:**
- Create: `openclaw-tenant/packages/api/src/routes/bootstrap-config.ts`
- Create: `openclaw-tenant/packages/api/src/routes/bootstrap-config.test.ts`
- Modify: `openclaw-tenant/packages/api/src/index.ts`

**Step 1: 写失败测试**

```typescript
// bootstrap-config.test.ts
import { expect, it, beforeEach, describe } from "bun:test";
import { Hono } from "hono";
import { initDb, getDb } from "../db/client";
import { join } from "path";
import { mkdir, writeFile, readFile, rm } from "fs/promises";
import bootstrapRouter from "./bootstrap-config";

const TMP = "/tmp/test-bootstrap";

async function setup() {
  await rm(TMP, { recursive: true, force: true });
  await mkdir(TMP, { recursive: true });
  initDb(":memory:");
  const db = getDb();
  // 插入测试 license（compose_project + data_dir 指向 TMP）
  db.run(`INSERT INTO licenses
    (license_key, gateway_token, gateway_url, status, provision_status,
     compose_project, data_dir, auth_token, hwid, wizard_feishu_done)
    VALUES
    ('LK-TEST','tok','wss://t','active','ready','proj',?,
     'auth-tok-123','hwid-001', 0)`, [TMP]);
  // 建 config 目录及 openclaw.json
  const configDir = join(TMP, "proj", ".openclaw");
  await mkdir(configDir, { recursive: true });
  await writeFile(join(configDir, "openclaw.json"), JSON.stringify({ gateway: {} }));
  return db.query<{ id: number }, []>("SELECT id FROM licenses WHERE license_key='LK-TEST'").get()!.id;
}

function buildApp() {
  const app = new Hono();
  app.route("/api/licenses", bootstrapRouter);
  return app;
}

describe("POST /api/licenses/:id/bootstrap-config", () => {
  it("写入飞书配置并置 wizard_feishu_done=1", async () => {
    const id = await setup();
    const res = await buildApp().request(`/api/licenses/${id}/bootstrap-config`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        authToken: "auth-tok-123",
        feishu: { appId: "cli_abc123", appSecret: "secret456" },
      }),
    });
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body.data.applied).toContain("feishu");

    const db = getDb();
    const row = db.query<{ wizard_feishu_done: number }, number[]>(
      "SELECT wizard_feishu_done FROM licenses WHERE id=?", [id],
    ).get();
    expect(row?.wizard_feishu_done).toBe(1);

    const configDir = join(TMP, "proj", ".openclaw");
    const cfg = JSON.parse(await readFile(join(configDir, "openclaw.json"), "utf8"));
    expect(cfg.channels.feishu.appId).toBe("cli_abc123");
    // appSecret 不记录原文（测试验证写入即可）
    expect(cfg.channels.feishu.appSecret).toBe("secret456");
  });

  it("authToken 错误时返回 403", async () => {
    const id = await setup();
    const res = await buildApp().request(`/api/licenses/${id}/bootstrap-config`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        authToken: "wrong-token",
        feishu: { appId: "cli_x", appSecret: "s" },
      }),
    });
    expect(res.status).toBe(403);
  });

  it("feishu.appId 为空时返回 400", async () => {
    const id = await setup();
    const res = await buildApp().request(`/api/licenses/${id}/bootstrap-config`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        authToken: "auth-tok-123",
        feishu: { appId: "", appSecret: "s" },
      }),
    });
    expect(res.status).toBe(400);
  });
});
```

**Step 2: 运行确认失败**

```bash
bun run --cwd packages/api test -- src/routes/bootstrap-config.test.ts
```

**Step 3: 实现 `bootstrap-config.ts`**

```typescript
import { Hono } from "hono";
import { readFile, writeFile } from "fs/promises";
import { join } from "path";
import { getDb } from "../db/client.js";
import { buildConfigDir } from "../services/provisioning/nameBuilder.js";

const router = new Hono();

router.post("/:id/bootstrap-config", async (c) => {
  const id = Number(c.req.param("id"));
  if (Number.isNaN(id)) return c.json({ success: false, error: "INVALID_ID" }, 400);

  let body: { authToken?: string; feishu?: { appId?: string; appSecret?: string } };
  try {
    body = await c.req.json();
  } catch {
    return c.json({ success: false, error: "INVALID_JSON" }, 400);
  }

  const db = getDb();
  const license = db
    .query<
      { auth_token: string | null; compose_project: string | null; data_dir: string | null },
      number
    >(
      "SELECT auth_token, compose_project, data_dir FROM licenses WHERE id=? AND status='active'",
    )
    .get(id);

  if (!license) return c.json({ success: false, error: "NOT_FOUND" }, 404);
  if (!body.authToken || body.authToken !== license.auth_token) {
    return c.json({ success: false, error: "UNAUTHORIZED" }, 403);
  }
  if (!license.compose_project || !license.data_dir) {
    return c.json({ success: false, error: "NOT_PROVISIONED" }, 409);
  }

  const applied: string[] = [];

  if (body.feishu) {
    const { appId, appSecret } = body.feishu;
    if (!appId?.trim() || !appSecret?.trim()) {
      return c.json({ success: false, error: "FEISHU_FIELDS_REQUIRED" }, 400);
    }

    const configDir = buildConfigDir(license.data_dir, license.compose_project);
    const configPath = join(configDir, "openclaw.json");

    let cfg: Record<string, unknown>;
    try {
      cfg = JSON.parse(await readFile(configPath, "utf8"));
    } catch {
      return c.json({ success: false, error: "CONFIG_NOT_FOUND" }, 500);
    }

    // 白名单写入：只允许 channels.feishu.{appId,appSecret}
    if (!cfg.channels || typeof cfg.channels !== "object") cfg.channels = {};
    const channels = cfg.channels as Record<string, unknown>;
    channels.feishu = {
      ...(typeof channels.feishu === "object" && channels.feishu !== null ? channels.feishu : {}),
      appId: appId.trim(),
      appSecret: appSecret.trim(),
    };

    await writeFile(configPath, JSON.stringify(cfg, null, 2));

    db.run("UPDATE licenses SET wizard_feishu_done=1 WHERE id=?", [id]);
    applied.push("feishu");
  }

  return c.json({ success: true, data: { applied } });
});

export default router;
```

**Step 4: 在 `index.ts` 注册路由**

```typescript
import bootstrapConfig from "./routes/bootstrap-config.js";
// ...
app.route("/api/licenses", bootstrapConfig);
```

**Step 5: 运行确认通过**

```bash
bun run --cwd packages/api test -- src/routes/bootstrap-config.test.ts
```

**Step 6: 运行全量测试**

```bash
bun run --cwd packages/api test
```

**Step 7: Biome 检查**

```bash
cd openclaw-tenant && bunx biome check
```

**Step 8: Commit**

```bash
git add openclaw-tenant/packages/api/src/routes/bootstrap-config.ts \
        openclaw-tenant/packages/api/src/routes/bootstrap-config.test.ts \
        openclaw-tenant/packages/api/src/index.ts
git commit -m "feat(tenant): add POST /api/licenses/:id/bootstrap-config"
```

---

## Task 8: provision-docker.sh — 基线模板 + 飞书插件安装

**Files:**
- Modify: `openclaw/provision-docker.sh`

**Step 1: 替换 openclaw.json 模板（第 48-81 行 heredoc）**

将 cat heredoc 替换为含 `tools.exec` 和 `models.providers.zhipuai` 的版本：

```bash
cat > "$CONFIG_FILE" <<JSON
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "$OPENCLAW_GATEWAY_BIND",
    "controlUi": {
      "dangerouslyAllowHostHeaderOriginFallback": true
    },
    "auth": {
      "mode": "token",
      "token": "$OPENCLAW_GATEWAY_TOKEN"
    },
    "tailscale": {
      "mode": "off",
      "resetOnExit": false
    },
    "remote": {
      "token": ""
    },
    "nodes": {
      "denyCommands": [
        "camera.snap",
        "camera.clip",
        "screen.record",
        "contacts.add",
        "calendar.add",
        "reminders.add",
        "sms.send"
      ]
    }
  },
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
JSON
```

**Step 2: 在修权步骤之后、`docker compose up` 之前追加飞书插件安装**

```bash
# 安装飞书插件（直接 docker run，在 gateway 启动前，避免 depends_on 约束）
echo "==> Installing feishu plugin"
"$RUNTIME_CMD" run --rm \
  -v "${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw" \
  "$IMAGE_NAME" \
  node dist/index.js plugins install ./extensions/feishu
```

**Step 3: 手动验证（需本地 Docker 环境）**

```bash
cd openclaw
COMPOSE_PROJECT_NAME=test-provision \
OPENCLAW_CONFIG_DIR=/tmp/test-oc-config \
OPENCLAW_WORKSPACE_DIR=/tmp/test-oc-ws \
OPENCLAW_GATEWAY_PORT=19789 \
OPENCLAW_BRIDGE_PORT=19790 \
OPENCLAW_GATEWAY_TOKEN=testtoken123 \
OPENCLAW_GATEWAY_BIND=lan \
bash provision-docker.sh
```

验证：
- `/tmp/test-oc-config/openclaw.json` 含 `tools.exec.host=node` 和 `models.providers.zhipuai`
- `/tmp/test-oc-config/extensions/feishu/` 目录存在（插件已安装）

**Step 4: Commit**

```bash
git add openclaw/provision-docker.sh
git commit -m "feat(provision): add tools.exec, GLM-4-Flash preset and feishu plugin install"
```

---

## Task 9: exec — 类型定义更新

**Files:**
- Modify: `openclaw-exec/src/types/index.ts`
- Modify: `openclaw-exec/src/types/index.test.ts`

**Step 1: 在 `index.ts` 追加 BootstrapNeeds 并更新 VerifyResponse**

```typescript
/** verify 接口返回的向导需求，true 表示该步骤尚未完成 */
export interface BootstrapNeeds {
  feishu?: boolean
}

/** 服务端 verify 接口的返回结构 */
export interface VerifyResponse {
  success: boolean
  data: {
    nodeConfig: NodeRuntimeConfig
    userProfile: UserProfile
    needsBootstrap?: BootstrapNeeds
  }
}
```

**Step 2: 更新测试（`index.test.ts`）**

确认 `VerifyResponse` 类型包含可选的 `needsBootstrap` 字段：

```typescript
it("VerifyResponse 支持 needsBootstrap 字段", () => {
  const resp: VerifyResponse = {
    success: true,
    data: {
      nodeConfig: {
        gatewayWsUrl: "wss://x",
        gatewayWebUI: "http://x",
        gatewayToken: "tok",
        agentId: "aid",
        deviceName: "dev",
      },
      userProfile: { licenseStatus: "Valid", expiryDate: "2027-01-01" },
      needsBootstrap: { feishu: true },
    },
  }
  expect(resp.data.needsBootstrap?.feishu).toBe(true)
})
```

**Step 3: 运行确认通过**

```bash
cd openclaw-exec && npm test
```

**Step 4: Commit**

```bash
git add openclaw-exec/src/types/index.ts \
        openclaw-exec/src/types/index.test.ts
git commit -m "feat(exec): add BootstrapNeeds type to VerifyResponse"
```

---

## Task 10: exec — bootstrap store

**Files:**
- Create: `openclaw-exec/src/store/bootstrap.store.ts`

**Step 1: 创建 `bootstrap.store.ts`**

```typescript
import { create } from 'zustand'
import type { BootstrapNeeds } from '../types'

interface BootstrapState {
  needs: BootstrapNeeds
  wizardOpen: boolean
  setNeeds: (needs: BootstrapNeeds) => void
  openWizard: () => void
  closeWizard: () => void
}

export const useBootstrapStore = create<BootstrapState>((set) => ({
  needs: {},
  wizardOpen: false,
  setNeeds: (needs) => set({ needs, wizardOpen: Object.values(needs).some(Boolean) }),
  openWizard: () => set({ wizardOpen: true }),
  closeWizard: () => set({ wizardOpen: false }),
}))
```

**Step 2: 在 `src/store/index.ts` 导出**

```typescript
export { useBootstrapStore } from './bootstrap.store'
```

**Step 3: Commit**

```bash
git add openclaw-exec/src/store/bootstrap.store.ts \
        openclaw-exec/src/store/index.ts
git commit -m "feat(exec): add bootstrap Zustand store"
```

---

## Task 11: exec — FeishuWizard 组件

**Files:**
- Create: `openclaw-exec/src/components/features/wizard/FeishuWizard.tsx`
- Create: `openclaw-exec/src/components/features/wizard/FeishuWizard.test.tsx`

**Step 1: 写失败测试**

```typescript
// FeishuWizard.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import { FeishuWizard } from './FeishuWizard'

describe('FeishuWizard', () => {
  it('提交按钮在两个字段均填写后才可用', () => {
    render(<FeishuWizard licenseId={1} authToken="tok" onSuccess={vi.fn()} onClose={vi.fn()} />)
    const submitBtn = screen.getByRole('button', { name: /提交/ })
    expect(submitBtn).toBeDisabled()

    fireEvent.change(screen.getByPlaceholderText(/App ID/), { target: { value: 'cli_abc' } })
    expect(submitBtn).toBeDisabled() // 还缺 appSecret

    fireEvent.change(screen.getByPlaceholderText(/App Secret/), { target: { value: 'sec123' } })
    expect(submitBtn).not.toBeDisabled()
  })

  it('提交成功后调用 onSuccess', async () => {
    const onSuccess = vi.fn()
    global.fetch = vi.fn().mockResolvedValue({ ok: true, json: async () => ({ success: true, data: { applied: ['feishu'] } }) })

    render(<FeishuWizard licenseId={1} authToken="tok" onSuccess={onSuccess} onClose={vi.fn()} />)
    fireEvent.change(screen.getByPlaceholderText(/App ID/), { target: { value: 'cli_abc' } })
    fireEvent.change(screen.getByPlaceholderText(/App Secret/), { target: { value: 'sec123' } })
    fireEvent.click(screen.getByRole('button', { name: /提交/ }))

    await waitFor(() => expect(onSuccess).toHaveBeenCalled())
  })
})
```

**Step 2: 运行确认失败**

```bash
cd openclaw-exec && npm test -- FeishuWizard
```

**Step 3: 实现 `FeishuWizard.tsx`**

```tsx
import { useState } from 'react'
import { Loader2, MessageSquare } from 'lucide-react'
import { Button, Card } from '../../ui'
import { useConfigStore } from '../../../store'

interface Props {
  licenseId: number
  authToken: string
  onSuccess: () => void
  onClose: () => void
}

export function FeishuWizard({ licenseId, authToken, onSuccess, onClose }: Props) {
  const { runtimeConfig } = useConfigStore()
  const [appId, setAppId] = useState('')
  const [appSecret, setAppSecret] = useState('')
  const [error, setError] = useState<string | null>(null)
  const [loading, setLoading] = useState(false)

  const canSubmit = appId.trim().length > 0 && appSecret.trim().length > 0

  const tenantBase = runtimeConfig?.tenantUrl ?? ''

  const handleSubmit = async () => {
    if (!canSubmit) return
    setLoading(true)
    setError(null)
    try {
      const res = await fetch(`${tenantBase}/api/licenses/${licenseId}/bootstrap-config`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ authToken, feishu: { appId: appId.trim(), appSecret: appSecret.trim() } }),
      })
      const body = await res.json()
      if (!res.ok || !body.success) {
        setError(body.error ?? '提交失败，请重试')
        return
      }
      onSuccess()
    } catch (e) {
      setError(String(e))
    } finally {
      setLoading(false)
    }
  }

  const inputClass =
    'w-full px-3 py-2 text-sm rounded-lg border border-outline bg-surface text-surface-on focus:outline-none focus:ring-2 focus:ring-primary/50'

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4">
      <Card className="w-full max-w-md">
        <div className="flex items-center gap-2 mb-4">
          <MessageSquare size={16} className="text-primary" />
          <h2 className="text-sm font-semibold text-surface-on">飞书配置向导</h2>
        </div>
        <p className="text-xs text-surface-on-variant mb-4">
          请填写飞书应用的 App ID 和 App Secret，配置完成后即可通过飞书与 AI 交互。
        </p>
        <div className="space-y-3">
          <div>
            <label className="text-xs text-surface-on-variant mb-1 block">App ID</label>
            <input
              value={appId}
              onChange={(e) => setAppId(e.target.value)}
              placeholder="App ID (例: cli_xxxxxxxx)"
              className={inputClass}
              autoComplete="off"
            />
          </div>
          <div>
            <label className="text-xs text-surface-on-variant mb-1 block">App Secret</label>
            <input
              type="password"
              value={appSecret}
              onChange={(e) => setAppSecret(e.target.value)}
              placeholder="App Secret"
              className={inputClass}
              autoComplete="off"
            />
          </div>
          {error && (
            <p className="text-xs text-error-on-container bg-error-container p-2 rounded-lg">{error}</p>
          )}
          <div className="flex gap-2 pt-2">
            <Button variant="ghost" onClick={onClose} disabled={loading} className="flex-1">
              稍后配置
            </Button>
            <Button onClick={handleSubmit} disabled={!canSubmit || loading} className="flex-1">
              {loading ? <Loader2 size={14} className="animate-spin" /> : '提交'}
            </Button>
          </div>
        </div>
      </Card>
    </div>
  )
}
```

**Step 4: 运行确认通过**

```bash
cd openclaw-exec && npm test -- FeishuWizard
```

**Step 5: Commit**

```bash
git add openclaw-exec/src/components/features/wizard/FeishuWizard.tsx \
        openclaw-exec/src/components/features/wizard/FeishuWizard.test.tsx
git commit -m "feat(exec): add FeishuWizard component"
```

---

## Task 12: exec — useNodeConnection 接入 needsBootstrap

**Files:**
- Modify: `openclaw-exec/src/hooks/useNodeConnection.ts`
- Modify: `openclaw-exec/src/store/config.store.ts`

**Step 1: config.store.ts 新增 licenseId + authToken + tenantUrl 存储**

在 `ConfigState` 接口追加：

```typescript
licenseId: number | null
authToken: string | null
tenantUrl: string
setSessionMeta: (meta: { licenseId: number; authToken: string; tenantUrl: string }) => void
```

实现：

```typescript
licenseId: null,
authToken: null,
tenantUrl: '',
setSessionMeta: ({ licenseId, authToken, tenantUrl }) =>
  set({ licenseId, authToken, tenantUrl }),
```

**Step 2: useNodeConnection.ts — 解析 needsBootstrap 并触发向导**

在 verify 成功后追加：

```typescript
import { useBootstrapStore } from '../store'
// ...
const { setNeeds } = useBootstrapStore()
// ...
// verify 成功后
const { nodeConfig, userProfile, needsBootstrap } = result.data
setRuntimeConfig(nodeConfig)
setUserProfile(userProfile)
setSessionMeta({
  licenseId: result.data.licenseId,   // 需服务端 verify 响应携带
  authToken: nodeConfig.authToken,
  tenantUrl: result.data.tenantUrl ?? '',
})
if (needsBootstrap) {
  setNeeds(needsBootstrap)
}
```

> **注意：** 服务端 verify 响应需补充 `licenseId` 和 `tenantUrl` 字段（Task 6 回去补充）。

**Step 3: App.tsx — 挂载向导覆盖层**

```tsx
import { useBootstrapStore, useConfigStore } from './store'
import { FeishuWizard } from './components/features/wizard/FeishuWizard'

// 在 <BrowserRouter> 内部顶层追加：
const { wizardOpen, needs, closeWizard } = useBootstrapStore()
const { licenseId, authToken } = useConfigStore()
const { verifyAndConnect } = useNodeConnection()

// JSX 末尾追加：
{wizardOpen && needs.feishu && licenseId && authToken && (
  <FeishuWizard
    licenseId={licenseId}
    authToken={authToken}
    onSuccess={() => { closeWizard(); verifyAndConnect() }}
    onClose={closeWizard}
  />
)}
```

**Step 4: 运行全量测试**

```bash
cd openclaw-exec && npm test
```

**Step 5: Lint 检查**

```bash
npm run lint
```

**Step 6: Commit**

```bash
git add openclaw-exec/src/hooks/useNodeConnection.ts \
        openclaw-exec/src/store/config.store.ts \
        openclaw-exec/src/App.tsx
git commit -m "feat(exec): wire needsBootstrap to FeishuWizard on verify"
```

---

## Task 13: exec — SettingsPage 飞书配置入口

**Files:**
- Modify: `openclaw-exec/src/pages/SettingsPage.tsx`

**Step 1: 在 SettingsPage 审批规则卡片之前插入飞书配置卡片**

```tsx
import { MessageSquare } from 'lucide-react'
import { useBootstrapStore, useConfigStore } from '../store'

// 在组件内：
const { openWizard } = useBootstrapStore()
const { licenseId } = useConfigStore()

// JSX（审批规则 Card 之前插入）：
{isOnline && licenseId && (
  <Card>
    <div className="flex items-center gap-2 mb-4">
      <MessageSquare size={16} className="text-primary" />
      <h2 className="text-sm font-semibold text-surface-on">飞书配置</h2>
    </div>
    <p className="text-xs text-surface-on-variant mb-3">
      配置飞书 App ID 和 App Secret，以启用飞书消息通道。
    </p>
    <Button variant="outline" onClick={openWizard}>
      配置飞书
    </Button>
  </Card>
)}
```

**Step 2: 运行测试 + lint**

```bash
cd openclaw-exec && npm test && npm run lint
```

**Step 3: Commit**

```bash
git add openclaw-exec/src/pages/SettingsPage.tsx
git commit -m "feat(exec): add persistent feishu config entry in SettingsPage"
```

---

## Task 14: verify.ts 补充 licenseId + tenantUrl

**Files:**
- Modify: `openclaw-tenant/packages/api/src/routes/verify.ts`

**Step 1: 响应体补充 licenseId 和 tenantUrl**

在 `nodeConfig` 对象中追加：

```typescript
nodeConfig: {
  gatewayUrl: license.gateway_url,
  gatewayToken: license.gateway_token,
  agentId,
  deviceName,
  authToken,
  tenantUrl: process.env.TENANT_PUBLIC_URL ?? '',   // 管理员配置的公开地址
},
```

并在响应 `data` 追加：

```typescript
licenseId: license.id,
```

**Step 2: `NodeRuntimeConfig` 类型同步更新**（`openclaw-exec/src/types/index.ts`）

```typescript
export interface NodeRuntimeConfig {
  gatewayWsUrl: string
  gatewayWebUI: string
  gatewayToken: string
  agentId: string
  deviceName: string
  authToken: string        // 新增
  tenantUrl: string        // 新增
}
```

**Step 3: 全量测试**

```bash
# tenant
cd openclaw-tenant && bun run --cwd packages/api test

# exec
cd openclaw-exec && npm test
```

**Step 4: Commit**

```bash
git add openclaw-tenant/packages/api/src/routes/verify.ts \
        openclaw-exec/src/types/index.ts
git commit -m "feat: expose licenseId and tenantUrl in verify response"
```

---

## Task 15: 最终集成验证

**Step 1: 运行 tenant 全量测试**

```bash
cd openclaw-tenant
bun run --cwd packages/api test
bunx biome check
```

预期：全部 PASS，无 lint 错误

**Step 2: 运行 exec 全量测试**

```bash
cd openclaw-exec
npm run check:all
npm test
```

**Step 3: 手动端到端验证清单**

- [ ] provision 后 `openclaw.json` 含 `tools.exec.host=node`、`models.providers.zhipuai`（无 apiKey）
- [ ] `patchModelApiKey` 调用后 `models.providers.zhipuai.apiKey` 已写入
- [ ] `extensions/feishu/` 目录存在于 config 卷中
- [ ] verify 首次返回 `needsBootstrap.feishu: true`
- [ ] exec 自动弹出 FeishuWizard
- [ ] 填写并提交后，容器 `openclaw.json` 含 `channels.feishu.{appId,appSecret}`
- [ ] 再次 verify 返回 `needsBootstrap.feishu: false`，不再弹向导
- [ ] Settings 页"配置飞书"按钮始终可见，点击可重新提交新值

**Step 4: 最终 Commit**

```bash
git add .
git commit -m "feat: bootstrap-config complete — provision preset + exec wizard"
```
