# Device Identity 安全加固实施计划

通过引入 ed25519 非对称设备身份，将 openclaw-clawmate 接入 Gateway 的认证从"仅凭 token"升级为"设备私钥签名 + token 双重验证"，同时由 tenant 预写 pairing 文件，保持全程自动化、无需手动 approve。

## 需用户确认的事项

> [!IMPORTANT]
> **时序问题**：exec 的 `publicKey` 需要在容器已启动后才能传给 tenant（用户安装 exec 才能生成密钥）。当前流程是：管理员先在 tenant 创建 License → Docker 容器启动 → 用户安装 exec → 首次 verify 时上报 publicKey。
>
> **推荐方案**：两个时机都触发写入，互相补充，保证无论先后顺序都能生效：
> - **provision 完成时**：调用 `writePairingIfReady()`，若 publicKey 尚未到位则 no-op
> - **verify 成功时**：也调用 `writePairingIfReady()`，此时 publicKey 已到位，写入 pairing 文件
>
> **请确认此方案是否符合你的实际使用流程？**

> [!NOTE]
> 需要先确认 openclaw 源码中 `deriveDeviceIdFromPublicKey` 的具体算法实现，以保证 tenant 侧计算的 `deviceId` 与 Gateway 验证时一致。

---

## 改动说明

### 模块一：openclaw-clawmate（Rust）

#### [MODIFY] [config.rs](../../openclaw-clawmate/src-tauri/src/config.rs)

扩展 `NodeConfig`，新增三个设备身份字段；新增 `ensure_device_identity()` 函数（首次调用生成 ed25519 密钥对并持久化）。

#### [MODIFY] [auth_client.rs](../../openclaw-clawmate/src-tauri/src/auth_client.rs)

`AuthRequest` 新增 `public_key: String` 字段，verify 时一并上报。

#### [MODIFY] [ws_client.rs](../../openclaw-clawmate/src-tauri/src/ws_client.rs)

connect 握手帧新增 `device` 字段：监听 `connect.challenge` 事件获取 nonce，用私钥签名后附加 `{ id, publicKey, signature, signedAt, nonce }`。

---

### 模块二：openclaw-tenant

#### [MODIFY] 数据库 schema（migration）

```sql
ALTER TABLE licenses ADD COLUMN exec_public_key TEXT;
```

#### [MODIFY] [verify.ts](../../openclaw-tenant/packages/api/src/routes/verify.ts)

- 接收可选的 `publicKey` 字段（向后兼容，无则忽略）
- verify 成功后存入 `licenses.exec_public_key`，并调用 `writePairingIfReady(licenseId)`

#### [NEW] pairingWriter.ts

新建 `packages/api/src/services/provisioning/pairingWriter.ts`：

- `deriveDeviceId(publicKeyRaw)` — 与 openclaw 源码一致的算法
- `writePairedJson(configDir, deviceId, publicKey)` — 写入 `{configDir}/devices/paired.json`
- `writePairingIfReady(licenseId)` — 查询 DB，满足 `exec_public_key IS NOT NULL AND provision_status = 'ready'` 时写入，否则 no-op

#### [MODIFY] [licenseProvisioningService.ts](../../openclaw-tenant/packages/api/src/services/provisioning/licenseProvisioningService.ts)

provision 成功后调用 `writePairingIfReady(licenseId)`。

---

## 验证计划

### 自动化测试

**运行命令**（在 `openclaw-tenant/packages/api` 目录下）：

```bash
bun test
```

**新增**：`pairingWriter.test.ts`

| 场景 | 断言 |
|------|------|
| `deriveDeviceId` 固定输入 | 固定 deviceId 输出 |
| publicKey 为空时 `writePairingIfReady` | 不创建文件（no-op） |
| provision_status 非 ready 时 | 不创建文件（no-op） |
| 满足条件时 | 生成格式正确的 `devices/paired.json` |

**扩展**：`verify.test.ts` 新增场景：携带 `publicKey` 的请求能正确写入数据库。

### 手动验证

> [!NOTE]
> 以下步骤需要真实环境，用于端到端验证。

1. 启动 tenant（`bun run dev:api`）
2. 创建 License，等待 `provision_status = ready`
3. 启动 exec，检查 `config.json` 中已生成 `device_id` 和 `public_key_raw`
4. exec 执行 verify，确认：
   - DB `licenses.exec_public_key` 已更新
   - 对应 `{configDir}/devices/paired.json` 文件已生成，且 `deviceId` 与 exec 本地值一致
5. exec 重新连接 Gateway，观察 Gateway 日志中**不出现** `pairing required`，连接直接成功
