# Device Identity 安全加固 — 实施完成报告

## 改动总览

### Tenant 侧（Node.js/TypeScript）

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `db/schema.ts` | MODIFY | `licenses` 表新增 `exec_public_key TEXT` 字段 |
| `services/provisioning/pairingWriter.ts` | NEW | `deriveDeviceId`（SHA-256 算法，与 openclaw 一致）+ `writePairedJson` + `writePairingIfReady` |
| `routes/verify.ts` | MODIFY | 接收可选 `publicKey`，verify 成功后存入 DB 并调用 `writePairingIfReady`（时机 ②） |
| `services/provisioning/licenseProvisioningService.ts` | MODIFY | provision 成功后调用 `writePairingIfReady`（时机 ①） |
| `services/provisioning/pairingWriter.test.ts` | NEW | 10 个测试覆盖所有边界场景 |

### Exec 侧（Rust / Tauri）

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `Cargo.toml` | MODIFY | 新增 `ed25519-dalek`、`sha2`、`rand`、`base64`、`hex` 依赖 |
| `src/device_identity.rs` | NEW | 密钥对生成/持久化、`derive_device_id`（SHA-256）、`sign_payload`（ed25519 签名） |
| `src/auth_client.rs` | MODIFY | `AuthRequest` 新增可选 `public_key` 字段，`check_auth` command 接收并传递 |
| `src/ws_client.rs` | MODIFY | `DeviceInfo` 新增 `publicKey/signature/signedAt/nonce` 字段；`connect_gateway` 接收密钥参数并生成签名 |
| `src/main.rs` | MODIFY | 注册 `device_identity` 模块、新增 `get_device_identity` command、setup 时预初始化密钥对 |

---

## 验证结果

### Tenant 测试

```
bun test src/services/provisioning/pairingWriter.test.ts
✔ 10 pass / 0 fail
```

覆盖场景：`deriveDeviceId` 幂等性、SHA-256 格式、不同输入差异、无效输入返回 null；`writePairedJson` 格式正确性、幂等跳过；`writePairingIfReady` 的三种 no-op 场景和正常写入。

### Rust 编译

```
cargo check
Finished `dev` profile [unoptimized + debuginfo] target(s)
```

0 error，0 warning（与本次改动相关）。

---

## 关键设计决策

**双时机触发 `writePairingIfReady`**：
- **时机 ①**（provision 完成时）：Docker 容器启动后立即尝试写入，此时 exec 可能尚未 verify，函数内部检查 `exec_public_key IS NOT NULL` 后 no-op
- **时机 ②**（verify 成功时）：exec 首次激活并上报 publicKey 后立即补写

两个时机互为补充，无论先后顺序都能正确触发写入。

---

## 已知预存问题（与本次改动无关）

`licenseProvisioningService.test.ts` 中 `sets provision_status to ready on success` 测试失败：原因是测试 mock 的 `Bun.spawn` 返回 `new Response(...)` 作为 stdout，`getContainerName()` 读取后得到 `[object Response]` 而不是实际容器名称。这是测试本身的 bug，在本次改动前就已存在，不影响生产代码。

---

## 下一步建议

前端（React）需要调用 `get_device_identity` Tauri command 获取 `public_key_raw`，在 verify 时传入 `publicKey` 字段，在 `connect_gateway` 时传入 `public_key_raw` 和 `private_key_raw`（后者只需获取，不需要展示给用户）。
