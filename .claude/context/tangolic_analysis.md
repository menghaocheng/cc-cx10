# Tangolic 原生二进制逆向分析

> **分析日期**: 2026-02-28
> **分析对象**: `/system/bin/tangolic` (设备端) + `prebuilts/tangolic` (仓库)
> **分析工具**: IDA Pro MCP Server + adb 设备端验证

---

## 1. 基础信息

| 属性 | 值 |
|---|---|
| 格式 | ELF 64-bit LSB aarch64, 静态链接, stripped |
| 大小 | ~2.1 MB (设备 2,149,136 字节; 仓库 2,149,232 字节, 差 96 字节) |
| 版本 | **D1.0.2** |
| 开发商 | Amanieu Systems Ltd |
| 编译语言 | **Rust + C 混合** |
| 编译器/工具链 | Rust nightly-2023-04-26 (x86_64 交叉编译到 aarch64) |
| 设备路径 | `/system/bin/tangolic` |
| 启动方式 | 手动 / init service（当前设备未配置自启动） |
| 版权 | Copyright (c) 2023 Amanieu Systems Ltd |

---

## 2. 架构：Multi-Call 合体二进制

tangolic **不是**独立的授权守护进程，而是一个 **multi-call 合体二进制**，包含两套完整代码。
根据 `argv[0]` 的文件名决定运行模式。

| 偏移范围 | 内容 | 语言 | 占比 |
|---|---|---|---|
| `0x00000` – `~0xF7000` (~1 MB) | tango_translator 翻译器核心 | Rust | ~50% |
| `~0x15F000` – `~0x20C000` (~1 MB) | tangolic 授权守护进程 + 静态链接 libcurl/libssl | C | ~50% |

**证据**：
- 翻译器部分包含完整的 Rust 标准库路径 (`/cargo/registry/...`)、JIT 编译器 (`dbt::*`)、syscall 翻译表
- 守护进程部分包含 `main.c`、`logger.c`、`le_decode.c` 等 C 源文件路径
- 两组代码共享同一个 ELF 入口点 (`start @ 0x400418`)，通过 `argv[0]` 分发

---

## 3. tangolic 守护进程功能

### 3.1 命令行接口

```
tangolic [options] <license file> [<backup license file>]

optinos:                    # 注意: 原文拼错 "options" 为 "optinos"
    -v : increase verbosity level by 1.
    -s [path] : use specified socket for talking to translator
```

运行时输出：
```
[INFO ] Tango license daemon D1.0.2 starts in production mode ...
[INFO ] read and parse license file : <path>
```

### 3.2 启动流程

1. 解析命令行参数（license 文件路径 + 可选 backup）
2. 读取 P12 证书文件 → 提取 X.509 证书
3. 提取证书到期日期 → 判断是否过期
4. 提取证书序列号
5. 提取 OU 字段 → Base64 解码 → AES-256-CBC 解密 → 得到 LEData 结构
6. 验证 OS 类型和 CPU 类型
7. 生成 session_id
8. 启动 translator IPC socket 线程
9. 进入主循环：定期与远程授权服务器通信

---

## 4. 授权体系关键参数

### 4.1 硬编码密钥

| 用途 | 值 |
|---|---|
| P12 密码 | `hb2FDGQ7XmYuAtyMkPkvFPBqNwrshdDxL53Nb5ap` |
| OU 解密密钥 (Base64) | `3HvbgxZGvv6WeB2EAhFd2z3gWu4mpWTubYFhwHgr` |
| OU 解密密钥 (解码后) | 32 字节 AES-256 密钥 |

### 4.2 授权服务器

| 属性 | 值 |
|---|---|
| URL | `https://license.amanieusystems.com/api/1.0/check_license` |
| 协议 | HTTPS POST |
| 认证 | P12 客户端证书双向认证 (mTLS) |
| SSL 验证 | 已禁用 (`CURLOPT_SSL_VERIFYPEER = 0`, `CURLOPT_SSL_VERIFYHOST = 0`) |

### 4.3 内嵌 CA 证书

二进制内嵌了自签名 Root CA：
- **Subject**: ASL Root CA, Amanieu Systems Ltd, England, GB
- **有效期**: 2019-04-04 ~ 2119-04-10 (100 年)
- **邮箱**: info@amanieusystems.com
- **密钥**: RSA 4096-bit

### 4.4 OU 字段解密流程

```
P12 证书 → X509 → Subject → 提取 /OU=.../ 字段
→ Base64 decode (得到 33 字节密文)
→ AES-256-CBC 解密 (key=BASE64("3Hvbgx..."), IV=证书序列号前 16 字节)
→ 20 字节明文 → LEData 结构
```

---

## 5. LEData 数据结构

```c
// 从 OU 字段解密后得到, 共 20 字节
struct LEData {
    uint16_t random;           // offset 0: 随机数 (防止相同参数生成相同密文)
    uint8_t  mode;             // offset 2: 授权模式
    uint8_t  os_type;          // offset 3: 操作系统类型
    struct {
        uint8_t  implementer;  // ARM implementer code
        uint16_t partno;       // ARM part number
    } cpu_type[4];             // offset 4: 最多 4 种允许的 CPU 类型 (3×4=12 字节)
    // 剩余 4 字节: 对齐/保留
};
```

### 5.1 授权模式 (mode)

| 值 | 名称 | 行为 |
|---|---|---|
| 0 | DISABLED | 不启动守护进程 |
| 1 | EVALUATION | 时间限制评估版, 到期后翻译器拒绝启动 |
| 2 | PERMISSIVE | 宽松模式, 忽略服务器 BAD 响应 |
| 3 | STRICT | 严格模式, 服务器返回 BAD 则立即设状态为 BAD |

### 5.2 OS 类型 (os_type)

| 值 | 名称 |
|---|---|
| 0 / 1 | Linux |
| 1 / 2 | Android |
| 2 | All (兼容全部) |

> 注：reimpl 源码定义 0=linux/1=android/2=all，原生日志显示 "1: linux, 2: android"，需以实际行为为准。

---

## 6. 服务器通信

### 6.1 POST 上报字段

```
session_id=<UUID>
&os_type=<int>
&num_cpu=<int>
&cpu_implementer=0x<hex>
&cpu_partno=0x<hex>
&translator_version=<version>_<buildid>
&certificate_serial=<string>
&transaction_time=YYYY:MM:DD-HH:MM:SS
&random_seed=<hex>
```

### 6.2 服务器响应格式

```
<status_code>;<message>
```

- status_code = 1 → GOOD (授权有效)
- status_code = 3 → BAD (授权无效)

### 6.3 通信策略

- 启动后立即与服务器通信一次
- 之后定期休眠 N 秒后再次通信 (具体 interval 从代码中确定)
- 通信失败累计超过 `MAX_COMMUNICATION_ERRORS` (reimpl 定义为 20) 次后，重置为正常间隔

---

## 7. IPC：与 tango_translator 的通信

### 7.1 机制

- **协议**: Unix domain socket (SOCK_STREAM)
- **命名方式**: Abstract socket (以 `\0` 开头，或指定路径 via `-s` 参数)
- **默认名称**: `tango_translator` (abstract namespace)
- **方向**: translator → 连接 tangolic → 查询授权状态
- **数据**: 授权状态码 (GOOD=1, BAD=3)

### 7.2 工作流

1. tangolic 启动后创建 Unix socket 并 listen
2. 每个 tango_translator 进程通过 socket 连接 tangolic
3. tangolic 将当前授权状态推送给 translator
4. translator 据此决定是否允许 ARM32 二进制继续运行

---

## 8. tango_translator 中的内嵌授权逻辑

翻译器自身（与 tangolic 守护进程无关）也有内嵌的离线授权验证：

| 组件 | 机制 |
|---|---|
| ECDSA 签名验证 | 使用 `ecdsa-0.14.8` Rust crate，离线校验 |
| 授权源文件 | `src/tango/license.rs` |
| 评估期限制 | `timer_create` + `timer_settime` 定时器，到期后翻译器退出 |
| 状态消息 | `"Time-limited evaluation version"`, `"Invalid license"`, `"License expired on ..."` |
| 环境变量/属性 | `tango.debug`, `tango.log_dir`, `tango.use_pretrans`, `tango.jit_dump_dir` |

---

## 9. 联网能力总结

| 组件 | 联网能力 | 说明 |
|---|---|---|
| tango_translator | **无** | 纯离线, ECDSA 签名校验, 无 URL/域名引用 |
| tangolic 守护进程 | **有** | 完整 libcurl + TLS, 定期 HTTPS POST 到 license.amanieusystems.com |
| tangolic_reimpl | **有** | 使用系统 libcurl, 复刻相同逻辑 |
| tango_reimpl (我们的方案) | **无** | 完全离线, 本地授权控制 |

---

## 10. 文件对照

| 文件 | 大小 | SHA256 前 8 位 | 说明 |
|---|---|---|---|
| `prebuilts/tango_translator.cheersu` | 1,115,288 B | `2E4F3BD9...` | 原始翻译器 (Cheersu 授权版) |
| `prebuilts/tangolic` | 2,149,232 B | `5CAA2F8E...` | 仓库里的 tangolic (含翻译器+守护进程) |
| 设备 `/system/bin/tangolic` | 2,149,136 B | `BDD0D99A...` | 设备端 tangolic (差 96 字节, 可能 strip 差异) |
| `prebuilts/tangolic_reimpl` | 33,304 B | — | 我们的 reimpl 版本 (纯守护进程, 动态链接) |
| `prebuilts/tango_translator` | 1,115,288 B | `7F8C20F6...` | 另一版翻译器 (hash 不同于 cheersu) |

---

## 11. C 源文件结构 (从字符串推断)

| 源文件 | 功能 |
|---|---|
| `main.c` | 主循环: 参数解析 → 证书加载 → 服务器通信循环 → IPC |
| `logger.c` | 日志系统 (PANIC/ERROR/WARN/INFO/DEBUG/VRBSE 6 级) + Android logd socket |
| `le_decode.c` | OU 字段 Base64 解码 + AES-256-CBC 解密 → LEData 提取 |

---

## 12. 关键断言/检查点 (可用于定位函数)

```c
// le_decode.c
key_len == 32                    // AES 密钥必须 32 字节
strlen(data) < 100               // OU 数据长度限制
decoded_ciphertext_len == 33     // Base64 解码后必须 33 字节

// main.c
len < 1000                       // POST 数据长度限制
le_data.mode != LE_MODE_DISABLED // 模式不能为 DISABLED
```

---

## 13. init 配置 (arm96.rc)

设备上的 `/system/etc/init/arm96.rc` **仅配置 binfmt_misc**，不启动 tangolic：

```rc
on late-fs
    mount binfmt_misc none /proc/sys/fs/binfmt_misc
    write /proc/sys/fs/binfmt_misc/arm96 -1
    write /proc/sys/fs/binfmt_misc/tango_translator -1
    write /proc/sys/fs/binfmt_misc/register ":arm96:M::\x7fELF\x01\x01...:/system/bin/arm96:POC"

on property:tango.enabled=1
    write /proc/sys/fs/binfmt_misc/arm96 1

on property:tango.enabled=0
    write /proc/sys/fs/binfmt_misc/tango_translator 0
```

这意味着在当前部署中，tangolic 守护进程需要额外配置才能自启动（或由其他 init 脚本触发）。

---

## 14. 对 reimpl 方案的启示

1. **tangolic_reimpl 已正确复刻核心逻辑**：P12 解析、OU 解密、IPC socket、服务器通信
2. **我们的 tango_reimpl 完全离线**：不需要联网，不需要授权服务器，不需要 tangolic 守护进程
3. **multi-call 设计无需复制**：reimpl 使用独立的 loader + core + watchdog 架构
4. **注意 `optinos` 拼写错误**：原生二进制确实如此，非 reimpl 的 bug
5. **OS 类型编码有歧义**：reimpl 源码与原生日志的值映射不一致，需实测确认
