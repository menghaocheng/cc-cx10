# arm96 远程授权与吊销框架设计

> 日期：2026-05-05  
> 适用范围：arm96 / libarm96 / arm96service / arm96server 云手机交付链  
> 目标：设计一套可商用、可吊销、可灰度、可多客户扩展的远程授权框架，降低纯本地二进制加固被静态 patch 后长期绕过的风险。

---

## 1. 背景与硬约束

当前 arm96 hardening 体系已经包含本地完整性、自校验、AES-GCM 加密、memfd 执行、反调试、arm96server PID tracking、平台绑定与过期自杀策略。但已有对抗分析表明：只要攻击者能离线修改 `/system/bin/arm96` 并重算本地校验值，纯本地保护仍然存在结构性上限。

本项目的主要使用场景是云手机，安全边界与普通 Android App 不同：

1. 客户天然容易获得 Android guest root 权限。
2. Android guest 内可见的硬件信息、系统属性、设备树、MAC、sysfs 节点都可能被修改、hook 或回滚。
3. 当前条件下宿主控制面暂时无法参与授权闭环。
4. EEPROM 不是所有此芯平台都具备，当前只有 vc 平台有较稳定的 EEPROM 槽位。
5. 此芯客户众多，客户定制可能改变 DT compatible、build.prop、驱动节点等软特征；例如 myt 将设备树中的 `cix` 改为 `myt`。
6. 授权开放原则是尽可能小，但不能因为未知合法客户变体导致大量误伤。
7. 当前已有 `arm96service`，定位是协助授权，现有业务较轻，适合作为远程授权客户端与本地授权仲裁器。

因此，本设计不把 guest 内的硬件信息作为根信任源，而是采用：

- 服务端短期租约。
- arm96service 本地授权代理。
- 服务端签名 token。
- capability 参与 libarm96 解密链。
- 可吊销 revocation epoch。
- 分层证据与服务端 policy。
- 有界离线授权。

---

## 2. 设计目标

### 2.1 安全目标

1. **抗本地时间炸弹 patch**  
   授权有效性不再只由本地时间常量和本地控制流决定；没有远程租约材料时，arm96 无法正常完成核心解密或执行。

2. **支持远程吊销**  
   服务端可以按客户、授权、设备、实例、token 粒度吊销授权。吊销后，最长在一个租约 TTL 内失效；在线情况下可以更快传播。

3. **减少硬件绑定误伤**  
   DT、build.prop、MAC 等软特征不作为单点硬拒绝条件。未知客户变体进入服务端 policy 归类，而不是要求重新发版。

4. **提升静态 patch 成本**  
   攻击者不能只 patch `arm96` 的一个 BL/NOP 或本地哈希值就长期运行；必须同时绕过 arm96service、IPC、租约签名、解密材料、续租与吊销状态。

5. **保留离线可用窗口**  
   在网络短时不可达时，允许使用服务端预签的有限离线 token；同时明确离线吊销不可做到强保证。

### 2.2 商用目标

1. 支持多客户、多平台、多 profile。
2. 支持灰度策略与临时观察授权。
3. 支持售后排障，授权失败可观测但不泄露敏感材料。
4. 支持按订单、客户、设备、实例进行审计。
5. 支持后续接入宿主、TEE、Keymaster、TPM 或控制面增强，但第一版不依赖它们。

---

## 3. 非目标与现实边界

在宿主无法参与、TEE 不可用、客户有 guest root 的前提下，本框架不能承诺绝对不可绕过。

明确边界：

1. root 可以 patch `arm96service`。
2. root 可以 hook 本地 IPC。
3. root 可以 dump 进程内存。
4. root 可以回滚本地文件状态。
5. root 可以尝试重放旧 token。
6. 离线模式无法强实时吊销。

因此，第一阶段安全目标不是“密码学意义上不可绕过”，而是把攻击从“单文件静态 patch”提升为“多进程、多协议、多状态、多密钥材料同步绕过”，并为后续宿主/TEE 接入留下接口。

---

## 4. 总体架构

```text
                       +-------------------------+
                       |   License Control Plane |
                       |-------------------------|
                       | Tenant / Order / Policy |
                       | Device Registry         |
                       | Revocation Management   |
                       | Audit / Risk Engine     |
                       +------------+------------+
                                    |
                                    | HTTPS / mTLS optional
                                    v
                       +-------------------------+
                       |      License Server     |
                       |-------------------------|
                       | Lease Issue/Renew       |
                       | Token Signing           |
                       | Policy Decision         |
                       | Revocation Epoch        |
                       +------------+------------+
                                    |
                                    | signed lease token
                                    v
+------------------+      IPC       +-------------------------+
| arm96 loader     +--------------->|      arm96service       |
|------------------| capability     |-------------------------|
| verify token     |<---------------+ Remote lease client     |
| request cap      |                | Local cap issuer        |
| reconstruct key  |                | Lease/offline cache     |
| decrypt libarm96 |                | Revocation state        |
+---------+--------+                | Evidence collector      |
          |                         +------------+------------+
          |                                      |
          v                                      | local status / revoke
+------------------+                            v
| libarm96 memfd   |                  +-------------------------+
| translated core  |                  | arm96server             |
+------------------+                  |-------------------------|
                                      | PID tracking            |
                                      | anti-debug scan         |
                                      | revoked cascade kill    |
                                      +-------------------------+
```

### 4.1 角色职责

| 组件 | 职责 | 信任级别 |
|---|---|---|
| License Server | 签发、续租、吊销、策略判定 | 服务端可信 |
| Control Plane | 客户、订单、设备、策略管理 | 服务端可信 |
| arm96service | 远程租约客户端、本地 capability 签发、离线缓存 | guest 内高价值目标，不作为根信任 |
| arm96 loader | 验证 token，向 service 取 capability，混入解密链 | guest 内高价值目标 |
| libarm96 | 加密核心，运行时 memfd 解密执行 | 被保护资产 |
| arm96server | PID tracking、反调试、吊销后级联清理 | 本地防护执行体 |

---

## 5. 授权模型

### 5.1 核心概念

| 概念 | 含义 |
|---|---|
| tenant | 客户主体 |
| license | 一份商业授权，可绑定客户、设备数、并发数、功能集 |
| device | 服务端登记的设备或设备族实例 |
| instance | 云手机实例，可能频繁创建销毁 |
| policy | 服务端授权规则集合，如 `vc_eeprom_v1`、`cx_generic_v1`、`myt_observed_v1` |
| lease | 服务端签发的短期授权租约 |
| capability | arm96service 基于 lease 给 arm96 单次执行签发的本地能力票据 |
| revocation_epoch | 吊销版本号，服务端递增后旧 token 失效 |

### 5.2 授权链路

```text
1. arm96service 启动
2. 采集 evidence
3. 请求 License Server 签发 lease
4. License Server 根据 policy 判定 allow/deny/observe
5. arm96service 缓存 signed lease
6. arm96 loader 被 binfmt_misc 拉起
7. arm96 loader 向 arm96service 请求 capability
8. arm96service 校验 lease 状态，返回 per-process capability
9. arm96 loader 将 capability 混入 libarm96 解密 key
10. 解密成功后 memfd 执行 libarm96
11. arm96service 周期续租；吊销后通知 arm96server 清理 tracked app
```

---

## 6. 设备证据分层

### 6.1 证据等级

| 等级 | 示例 | 用法 |
|---|---|---|
| 强证据 | 服务端登记设备密钥、未来 TEE/Keymaster/宿主签名、vc EEPROM 多槽位 hash | 可作为 policy 主判定 |
| 中证据 | ISAR0/MIDR、内核补丁特征、vbmeta digest、内核版本族、驱动节点组合 | 参与风险评分和 profile 分类 |
| 弱证据 | DT compatible、build.prop、ro.*、MAC、Android release、文件路径 | 仅作辅助，不单点硬拒绝 |

### 6.2 当前阶段可采集证据

第一版宿主不可参与，因此建议采集但不硬依赖：

```json
{
  "arm96_version": "V0.56.0",
  "arm96_sha256": "...",
  "libarm96_sha256": "...",
  "arm96service_sha256": "...",
  "boot_id": "...",
  "kernel_release": "...",
  "kernel_version": "...",
  "android_release": "10",
  "dt_compatible_hash": "...",
  "build_prop_digest": "...",
  "isar0": "0x...",
  "midr_parts": ["0xd81", "0xd80"],
  "eeprom_present": true,
  "eeprom_digest": "optional",
  "mac_oui_digest": "weak",
  "binfmt_flags": "POC"
}
```

### 6.3 多客户未知变体处理

不能在本地写死 `cix`、`myt`、`sky1` 作为硬门禁。推荐服务端策略：

1. 强证据匹配：签正式租约。
2. 中证据匹配、弱证据变化：签短 TTL 观察租约。
3. 弱证据未知但订单关系可信：签 limited 租约，触发后台审核。
4. 证据与已登记设备冲突：拒绝并进入风控。
5. 同一 license 在多个差异过大的 evidence 上并发出现：降级或吊销。

---

## 7. Token 与密码学设计

### 7.1 服务端签名 token

建议使用 Ed25519。原因：实现简单、签名短、验证快、常量时间库成熟。也可选择 ECDSA P-256 以便兼容企业 HSM。

arm96 / arm96service 内置服务端公钥集合：

```text
pubkey_id = 1
algorithm = Ed25519
public_key = 32 bytes
not_before / not_after for key rotation
```

服务端 token 使用 canonical JSON、CBOR 或 protobuf 编码。推荐 CBOR/COSE 或 protobuf + detached signature，避免 JSON canonicalization 坑。

### 7.2 Lease Token 字段

```json
{
  "typ": "arm96-lease",
  "ver": 1,
  "kid": "prod-ed25519-2026-01",
  "tenant_id": "tenant_xxx",
  "license_id": "lic_xxx",
  "device_id": "dev_xxx",
  "instance_id": "inst_xxx",
  "policy_id": "cx_generic_v1",
  "lease_id": "lease_xxx",
  "token_serial": "tok_xxx",
  "not_before": 1777950000,
  "expires_at": 1777951800,
  "renew_after": 1777950900,
  "revocation_epoch": 18,
  "features": ["translate", "watchdog", "anti_debug"],
  "mode": "online",
  "risk_level": "normal",
  "evidence_digest": "sha256:...",
  "lease_secret_wrap": "base64url:...",
  "offline_bundle_digest": "optional",
  "sig": "base64url:..."
}
```

### 7.3 Lease Secret

`lease_secret` 是服务端为短期租约生成的随机 128/256-bit 材料。它不应以明文长期落盘。

用途：

1. arm96service 用它签发本地 capability。
2. arm96 loader 将 capability 或其派生值混入 `reconstruct_key()`。
3. 服务端可通过停止续租让新 lease_secret 不再发放。

在没有 TEE/宿主密钥时，`lease_secret_wrap` 的保护能力有限。第一版可采用：

- token 内包含加密后的 lease_secret。
- arm96service 使用构建期内置客户公钥/对称材料解封。
- 或者服务端直接返回明文 lease_secret 给 HTTPS 通道，arm96service 仅内存保存。

安全评价：明文返回在 guest root 下可被 dump，但仍能阻断“离线静态 patch 后长期运行”的低成本攻击。后续若接入宿主/TEE，可将 `lease_secret_wrap` 改为硬件密钥解封。

### 7.4 Local Capability

arm96 loader 每次执行时，向 arm96service 请求 capability。

请求：

```json
{
  "typ": "cap-request",
  "ver": 1,
  "pid": 1234,
  "ppid": 1,
  "proc_starttime": "...",
  "boot_id": "...",
  "arm96_text_hash16": "...",
  "libarm96_digest": "...",
  "nonce": "16-32 random bytes",
  "argv_mode": "binfmt",
  "timestamp": 1777950123
}
```

响应：

```json
{
  "typ": "cap-response",
  "ver": 1,
  "lease_id": "lease_xxx",
  "expires_at": 1777951800,
  "revocation_epoch": 18,
  "capability": "HMAC-SHA256(lease_secret, context)",
  "policy_flags": 123,
  "service_nonce": "..."
}
```

HMAC 上下文建议包含：

```text
pid | proc_starttime | boot_id | nonce | arm96_text_hash16 |
libarm96_digest | expires_at | revocation_epoch | policy_id
```

arm96 loader 侧派生：

```text
cap_mask = SHA256("arm96-cap-v1" | capability | nonce | policy_id)[:16]
final_key = local_fused_key XOR cap_mask
```

注意：不要让 arm96service 只返回布尔 allow。必须返回影响解密的数据域材料。

---

## 8. 远程服务接口

### 8.1 API 列表

| API | 方法 | 说明 |
|---|---|---|
| `/v1/lease/issue` | POST | 首次签发租约 |
| `/v1/lease/renew` | POST | 续租 |
| `/v1/lease/status` | POST | 查询当前租约状态 |
| `/v1/revoke/check` | POST | 拉取吊销 epoch 与吊销列表摘要 |
| `/v1/evidence/report` | POST | 上报设备证据与异常 |
| `/v1/offline/bundle` | POST | 获取离线 token 包 |
| `/v1/policy/get` | POST | 拉取策略元数据 |

### 8.2 issueLease 请求

```json
{
  "client_id": "arm96service",
  "client_version": "V0.56.0",
  "license_id": "lic_xxx",
  "tenant_hint": "optional",
  "instance_hint": "optional",
  "nonce": "base64url",
  "evidence": { },
  "supported_algorithms": ["Ed25519", "HMAC-SHA256"],
  "supported_features": ["cap-v1", "offline-bundle-v1"]
}
```

### 8.3 issueLease 响应

```json
{
  "decision": "allow",
  "reason_code": "OK",
  "server_time": 1777950123,
  "lease_token": "base64url(cbor/protobuf + sig)",
  "retry_after_sec": 0,
  "offline_allowed": true,
  "telemetry_level": "normal"
}
```

`decision` 取值：

| 值 | 语义 |
|---|---|
| `allow` | 正式放行 |
| `observe` | 观察放行，短 TTL，需后台归类 |
| `limited` | 限制放行，功能降级或更短 TTL |
| `deny` | 拒绝 |
| `revoked` | 已吊销 |
| `upgrade_required` | 客户端版本过低 |

---

## 9. 吊销设计

### 9.1 吊销粒度

| 粒度 | 场景 |
|---|---|
| tenant | 整个客户违约或停止服务 |
| license | 某份授权到期、欠费、违规 |
| device | 单台设备异常或克隆 |
| instance | 某个云手机实例被封禁 |
| token_serial | 单个 token 泄露或异常 |
| policy | 某类策略错误，需要下线 |
| public key | 签名密钥轮换或泄露 |

### 9.2 Revocation Epoch

服务端为每个作用域维护递增 epoch：

```text
global_epoch
tenant_epoch[tenant_id]
license_epoch[license_id]
device_epoch[device_id]
policy_epoch[policy_id]
```

lease token 内携带签发时看到的 epoch。arm96service 续租或 check revoke 时，如果服务端 epoch 更高，则本地 token 失效。

### 9.3 吊销传播

在线：

1. 服务端标记 revoked，递增 epoch。
2. arm96service 下一次 renew/check 收到 revoked。
3. arm96service 切换本地状态为 revoked。
4. arm96service 拒绝 capability 请求。
5. arm96server kill tracked PIDs，必要时重启 zygote_secondary。

离线：

1. 已签发 token 在 TTL 内仍可能有效。
2. 离线 token 包耗尽后 fail closed。
3. 本地不能保证实时吊销。

### 9.4 本地 revoked 行为

revoked 后建议动作：

1. 停止签发 capability。
2. 删除内存 lease_secret。
3. 清理本地 lease cache。
4. 通知 arm96server kill tracked 32 位进程。
5. 触发 zygote_secondary restart 或等待下一次 binfmt 失败。
6. 写入最小错误码，便于售后定位，不输出用户可见说明。

---

## 10. 离线授权设计

### 10.1 原则

离线授权只能解决可用性，不能提供强吊销。商业策略必须明确：

- 严格模式：离线 5-15 分钟后失效。
- 标准模式：离线 30-120 分钟 grace。
- 企业模式：服务端预签 24 小时以内 token 包。

### 10.2 Offline Bundle

服务端可签发 token 队列：

```json
{
  "typ": "arm96-offline-bundle",
  "bundle_id": "ob_xxx",
  "license_id": "lic_xxx",
  "not_before": 1777950000,
  "not_after": 1778036400,
  "slot_sec": 1800,
  "tokens": [
    {"slot": 0, "expires_at": 1777951800, "lease_token": "..."},
    {"slot": 1, "expires_at": 1777953600, "lease_token": "..."}
  ],
  "max_clock_skew_sec": 300,
  "sig": "..."
}
```

### 10.3 抗回滚增强

guest root 下本地状态可回滚，因此只能做成本提升：

1. token 绑定 `boot_id` 和 arm96service session。
2. token 按 slot 单向消费。
3. 消费记录多点冗余存储并带 MAC。
4. 检测系统时间大幅回退时进入 limited 模式。
5. 离线 token 只允许短窗口，不提供长期授权。
6. 离线期间不更新高权限策略。

---

## 11. arm96service 本地设计

### 11.1 子模块

| 模块 | 职责 |
|---|---|
| LeaseClient | HTTPS 请求 issue/renew/status |
| TokenVerifier | 验证服务端签名、公钥轮换、token 时间窗 |
| EvidenceCollector | 采集 guest 内证据 |
| PolicyCache | 缓存服务端 policy 元数据 |
| CapabilityIssuer | 为 arm96 签发本地 capability |
| RevocationMonitor | 续租失败/epoch 提升后切换 revoked |
| OfflineManager | 管理离线 token 包 |
| AuditReporter | 异常和遥测上报 |
| IPCServer | 提供本地 Unix socket/Binder 接口 |

### 11.2 本地 IPC

优先选择 Unix domain socket：

- Android 10 可用。
- C/C++ loader 易接入。
- 可用 peer credential 获取对端 pid/uid。
- 比 HTTP localhost 少一层解析攻击面。

socket 路径建议：

```text
/dev/socket/arm96service
```

或：

```text
/data/local/tmp/.arm96service.sock
```

生产更推荐 init 创建 `/dev/socket` 并设置 SELinux/权限。若当前 SELinux 约束复杂，MVP 可先使用 `/data/local/tmp`，后续收紧。

### 11.3 IPC 命令

| 命令 | 说明 |
|---|---|
| `GET_CAPABILITY` | arm96 请求单次 capability |
| `GET_STATUS` | arm96server 查询授权状态 |
| `REPORT_FAULT` | loader/server 上报反调试或完整性异常 |
| `FORCE_RENEW` | 调试/运维触发续租 |
| `CONSUME_OFFLINE_SLOT` | 离线模式消费 token |

### 11.4 service 自保护

在 guest root 下不能完全防 patch，但仍应做基础保护：

1. 进程名混淆与 watchdog。
2. `PR_SET_DUMPABLE=0`。
3. TracerPid 自检。
4. 自身 `.text` hash 与签名校验。
5. 与 arm96server 互相监督。
6. service 死亡后 arm96 拿不到 capability，默认 fail closed。
7. lease_secret 仅内存保存，收到 revoke 后立即清零。

---

## 12. arm96 loader 改造

### 12.1 启动路径

当前 loader 主路径：

```text
integrity_check_text
integrity_check_critical_data
anti-debug
platform_check
reconstruct_key
decrypt_core_to_memfd
fexecve
```

新增授权路径：

```text
integrity_check_text
integrity_check_critical_data
anti-debug
collect minimal context
request capability from arm96service
verify capability envelope
reconstruct_key with cap_mask
decrypt_core_to_memfd
fexecve
```

### 12.2 key fusion

新增一层：

```text
base_key = local key fragments
local_fused_key = base_key XOR ISAR0 XOR MAC_HASH XOR EXPIRE_MASK XOR D1 XOR server_token XOR loader_text_hash16
cap_mask = SHA256("arm96-cap-v1" | capability | nonce | policy_id)[:16]
final_key = local_fused_key XOR cap_mask
```

构建端 `encrypt_core.py` 必须使用同样 cap 派生模型。第一版可选择每个 license/profile 固定一个 `lease_profile_secret`，服务端 lease_secret 与它派生出 cap_mask。更理想的是每次 lease 使用不同随机 secret，但这需要服务端与构建/打包之间设计 key wrapping。

### 12.3 fail 语义

拿不到 capability 时：

1. `-v` / `-i` 可保留基本输出，便于售后，但不要泄露授权细节。
2. binfmt 翻译路径必须 fail closed。
3. 失败应静默或写受控错误码，不输出提示文本。

---

## 13. arm96server 联动

arm96server 保留现有 PID tracking / debugger scan / guardian cascade，并新增授权状态联动：

1. 周期查询 arm96service 授权状态。
2. 状态为 revoked/expired/deny 时，kill tracked PIDs。
3. arm96service 不可达时进入 grace 计数。
4. grace 超时后执行 fail closed。
5. 检测到 debugger/fault 时上报 arm96service，由 service 决定是否降级或触发远程风控。

---

## 14. 服务端微服务架构

### 14.1 服务拆分

| 服务 | 职责 |
|---|---|
| API Gateway | 鉴权、限流、TLS、路由 |
| License Service | license 生命周期、订单绑定、额度 |
| Device Registry | device/instance/evidence 管理 |
| Policy Engine | allow/observe/limited/deny 判定 |
| Lease Service | issue/renew/status |
| Signing Service | token 签名，保护私钥 |
| Revocation Service | 吊销状态、epoch、黑名单 |
| Risk Engine | 克隆检测、异常频率、证据漂移 |
| Audit Service | 操作日志、授权日志、合规留存 |
| Admin Console | 商务/运维/售后管理 |

### 14.2 数据库核心表

```text
tenants(id, name, status, created_at)
licenses(id, tenant_id, product, max_instances, expires_at, status)
devices(id, tenant_id, first_seen, last_seen, status, policy_id)
device_evidence(id, device_id, evidence_digest, evidence_json, created_at)
instances(id, device_id, license_id, status, last_seen)
policies(id, name, version, rule_json, status)
leases(id, license_id, device_id, instance_id, expires_at, status, token_serial)
revocations(id, scope_type, scope_id, epoch, reason, created_at)
audit_logs(id, actor, action, target, detail_json, created_at)
risk_events(id, severity, type, subject, detail_json, created_at)
```

### 14.3 高可用要求

商用建议：

1. Lease Service 和 Revocation Service 多副本部署。
2. Signing Service 私钥隔离，最好接入 KMS/HSM。
3. Redis 缓存 policy 与 revocation epoch。
4. 数据库主从或多可用区。
5. API 全链路超时小于 2 秒。
6. 客户端续租使用指数退避，避免网络抖动造成雪崩。
7. 服务端支持按 region 部署，减少云手机跨地域延迟。

---

## 15. Policy Engine 设计

### 15.1 策略输入

```json
{
  "license": {"status": "active", "expires_at": 1782903600},
  "tenant": {"status": "active"},
  "device": {"known": true, "status": "active"},
  "instance": {"concurrency": 12, "limit": 20},
  "evidence": { },
  "risk": {"score": 18, "events": []},
  "revocation": {"epoch": 18}
}
```

### 15.2 策略输出

```json
{
  "decision": "allow",
  "policy_id": "cx_generic_v1",
  "ttl_sec": 1800,
  "offline_allowed": true,
  "offline_horizon_sec": 86400,
  "features": ["translate", "watchdog"],
  "risk_level": "normal",
  "reason_code": "OK"
}
```

### 15.3 推荐策略

| 场景 | 决策 | TTL |
|---|---|---|
| vc EEPROM 多槽位匹配，license 正常 | allow | 30 min |
| cx/myt 未知 DT 但其它中证据稳定 | observe | 5-10 min |
| 同 license 多地并发异常 | limited/deny | 1-5 min 或拒绝 |
| 证据突然大幅漂移 | observe/limited | 5 min |
| license 到期或欠费 | revoked | 0 |
| 客户手动吊销 | revoked | 0 |

---

## 16. 可观测性与审计

### 16.1 客户端事件

arm96service 上报：

1. lease issue/renew 成功失败。
2. evidence_digest 变化。
3. arm96 hash 不匹配。
4. service 被重启频率异常。
5. capability 请求频率异常。
6. revoked 状态传播完成。
7. offline token 消费。
8. anti-debug/fault stamp 事件。

### 16.2 服务端指标

```text
lease_issue_qps
lease_renew_qps
lease_deny_rate
lease_observe_rate
revocation_count
token_verify_failure_count
evidence_drift_count
duplicate_license_concurrency
offline_bundle_issue_count
client_version_distribution
```

### 16.3 售后错误码

错误码应可诊断但不泄露敏感判断逻辑：

| 错误码 | 含义 |
|---|---|
| E5601 | service unavailable |
| E5602 | no valid lease |
| E5603 | token signature invalid |
| E5604 | lease expired |
| E5605 | revoked |
| E5606 | capability denied |
| E5607 | evidence rejected |
| E5608 | offline bundle exhausted |
| E5609 | client version denied |

---

## 17. 安全加固建议

### 17.1 客户端

1. arm96service 与 arm96 使用二进制协议，避免文本协议易被脚本重放。
2. 每次 capability 请求使用 nonce，禁止重放。
3. capability 绑定 pid、starttime、boot_id。
4. arm96service 不持久化明文 lease_secret。
5. arm96 loader 不接受过期或过长 TTL 的 capability。
6. arm96service 和 arm96server 互相探活。
7. 远程租约状态参与数据域 key 派生，而非布尔判定。
8. release 包中保留本地完整性校验，作为防低成本 patch 层。

### 17.2 服务端

1. 签名私钥隔离到 KMS/HSM 或最小权限 Signing Service。
2. token 带 `kid`，支持公钥轮换。
3. token serial 入库，支持精确吊销。
4. 所有授权决策可审计。
5. 对同 license 的多地、多 evidence 并发做风控。
6. 支持按客户灰度 policy。
7. 禁止服务端返回过长 TTL。

---

## 18. 迁移路线

### V0.56：远程授权 MVP

目标：引入 arm96service lease 客户端和本地 capability，但不大改现有 hardening 链。

范围：

1. arm96service 支持 issue/renew lease。
2. arm96service 支持 GET_CAPABILITY IPC。
3. arm96 loader 请求 capability。
4. capability 混入 key 派生。
5. 服务端提供最小 issue/renew/revoke API。
6. DT/build.prop 从硬拒绝改为 evidence 上报。

验收：

1. 无 lease 时 binfmt 翻译失败。
2. 有有效 lease 时正常启动 32 位 app。
3. 服务器吊销后，下一次 renew 失败并停止新翻译。
4. arm96service 死亡后，新 arm96 无法拿到 capability。
5. mustpass 主链不回退。

### V0.57：吊销与离线包

目标：补齐商用吊销与离线 grace。

范围：

1. Revocation epoch。
2. Offline bundle。
3. arm96server revoked cascade。
4. 服务端 audit/risk 最小闭环。
5. 客户/设备/policy 管理接口。

验收：

1. 按 license/device/tenant 吊销生效。
2. 离线 token 到期后 fail closed。
3. 离线短时网络抖动不误杀。
4. evidence 变化进入 observe，而不是本地误伤。

### V0.58+：强信任根接入

目标：逐步接入更高可信度材料。

候选：

1. 宿主控制面签名。
2. Keymaster/TEE attestation。
3. AVB/vbmeta digest。
4. vc EEPROM 多槽位强绑定。
5. KMS/HSM 签名服务。

---

## 19. MVP 实施清单

### 客户端

1. 新增 `arm96service` 授权配置：server URL、公钥、license_id、超时、TTL 上限。
2. 新增 HTTPS 客户端与 token verifier。
3. 新增本地 IPC server。
4. 新增 lease cache，优先内存，必要时加密落盘。
5. arm96 loader 新增 IPC client。
6. `reconstruct_key()` 增加 capability mask。
7. `encrypt_core.py` 增加 capability 派生对称逻辑。
8. arm96server 增加授权状态查询和 revoked kill。

### 服务端

1. `POST /v1/lease/issue`。
2. `POST /v1/lease/renew`。
3. `POST /v1/revoke/check`。
4. Ed25519 signing key。
5. license/device/policy 基础表。
6. 简单 admin CLI 或 SQL 管理吊销。

### 验收用例

1. 正常在线授权。
2. 服务器不可达 grace。
3. 无 token fail closed。
4. token 过期 fail closed。
5. 签名错误拒绝。
6. 吊销后续租失败。
7. arm96service 死亡后新翻译失败。
8. patch 本地时间常量不延长授权。
9. 未知 DT 变体进入 observe。
10. vc EEPROM 存在时进入强 profile。

---

## 20. 推荐默认参数

| 参数 | 默认值 | 说明 |
|---|---:|---|
| lease TTL | 1800s | 标准 30 分钟 |
| renew ahead | 300s | 到期前 5 分钟续租 |
| high-risk TTL | 300s | 高风险客户 5 分钟 |
| offline standard grace | 3600s | 标准离线 1 小时 |
| offline enterprise max | 86400s | 企业最长 24 小时，需单独开通 |
| service request timeout | 2s | 单次请求超时 |
| renew retry backoff | 5s -> 300s | 指数退避 |
| capability max age | 10s | arm96 本地 capability 短有效 |
| nonce length | 16/32B | 随机 nonce |
| signature alg | Ed25519 | 可替换 P-256 |

---

## 21. 结论

在当前约束下，最现实的商用方案是：

1. 把授权判断迁移到服务端。
2. 让 arm96service 承载远程租约、吊销、离线包、本地 capability。
3. 让 capability 参与 libarm96 解密链，而不是只做 allow/deny 布尔判断。
4. 把 DT、build.prop、MAC 等软特征降级为 evidence，不做本地硬拒绝。
5. 用短 TTL 和 revocation epoch 实现可吊销。
6. 用 observe/limited 策略解决未知合法客户误伤。
7. 为未来宿主、TEE、Keymaster、EEPROM 强绑定预留接口。

这套框架不能在无宿主/无 TEE 且 guest root 的条件下达到绝对不可绕过，但它能把当前纯本地二进制二次加固的结构性缺陷明显后移：攻击者不再只面对一个可重算 hash 的本地文件，而要同时绕过服务端签名租约、arm96service 状态机、本地 capability、短期 TTL、吊销 epoch、arm96server 级联与核心解密材料。
