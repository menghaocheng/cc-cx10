# security-eval

## 用途
用于让模型对当前二进制加固方案做结构化安全评估。

## 输入
粘贴当前版本的加固方案背景、约束与目标。

## 输出格式
按密码学安全性、逆向对抗、运行时防护、局限性、改进优先级与总体评级输出。

---

# 安全评估开场白 (Security Evaluation Prompt)

> 将以下内容完整粘贴到任意 AI 模型的对话中即可。

---

## Prompt 正文

请以**信息安全专家**的身份，对以下"二进制二次加固"防护方案进行**客观、系统的安全评估**。请从密码学安全性、逆向工程对抗能力、运行时防护有效性、方案的科学性与局限性等维度进行分析，并给出整体评级和改进建议。

---

### 一、系统背景

我们有一个 **ARM32→ARM64 二进制翻译器**（类似 ExaGear/QEMU user-mode），部署在 **Android 10 容器**环境中。我们**没有翻译器核心引擎的源码**（第三方 Rust 编写的闭源 ELF），但需要对其进行授权控制和安全加固。

**部署架构 (Split Model)**:
```
/system/bin/tango_translator   ← Loader（我们编写，C 静态链接，~200KB）
/system/bin/libarm96            ← Core 密文（原始翻译引擎，AES-128-GCM 加密，~4MB）
/system/bin/arm96server         ← 反调试守护进程（我们编写，C 静态链接，~100KB）
```

Linux `binfmt_misc` 配置为：所有 ARM32 ELF 由 `tango_translator`（Loader）处理。Loader 负责解密 Core 到内存（`memfd_create` + `fexecve`），磁盘永不落明文。

**目标硬件**: CIX P1 CS8180 SoC（Cortex-X4/A720，支持 SM3/SM4 指令集），单一平台部署。

---

### 二、完整防护层级（V0.18 ~ V0.32，共 15 个迭代版本）

#### L1: 密码学保护层

| 措施 | 说明 |
|---|---|
| **AES-128-GCM 加密 Core** | 密文格式 `TGENC022`：`magic(8B)｜orig_size(4B)｜nonce(12B)｜tag(16B)｜ciphertext`；纯 C 实现（无外部库），常数时间 tag 比较 |
| **5 层 XOR 密钥融合** | `final_key = base_key ⊕ {ISAR0,ISAR0} ⊕ {MAC_HASH,MAC_HASH} ⊕ {EXPIRE_MASK,EXPIRE_MASK} ⊕ {arm96server_token[0:8],token[8:16]}` |
| **密钥分片** | base_key 拆成 4×uint32 + XOR 因子，散布在 Loader .text 段 |
| **密钥清零** | 解密后 `explicit_bzero` / volatile 循环清零，编译器不可 DCE |
| **memfd 内存执行** | `memfd_create("jit-cache")` → 解密 → `fexecve(memfd)`，磁盘无明文 |
| **HMAC-SHA256 种子推导** | 17 条运行时字符串的 LCG 混淆种子由 `HMAC-SHA256(base_key, idx)` 运行时推导，非编译时常量 |
| **散射投毒 (Scatter-Corrupt)** | MAC 不匹配时，解密后随机覆写 32×8B，崩溃地址不可复现 |

**密钥融合因子详解**:
- **L1 ISAR0**: `ID_AA64ISAR0_EL1` 硬件寄存器值（MRS 指令读取，不可伪造）
- **L2 MAC_HASH**: 设备 MAC OUI 的 salted FNV-1a hash（三路获取：sysfs scan / sysfs explicit / netlink）
- **L3 EXPIRE_MASK**: `~EXPIRE_EPOCH & 0xFFFFFFFF`（授权到期时间戳位取反）
- **L4 arm96server_token**: `HMAC-SHA256(base_key, idx=19)[:16]`，追加在 arm96server 二进制尾部

**关键设计**: 任一因子错误 → 密钥错误 → AES-GCM tag 验证失败 → 解密输出清零 → `fexecve` 失败。**无任何 if/else 分支可 NOP**，失败点在数据域而非控制流。

#### L2: 静态分析对抗层

| 措施 | 说明 |
|---|---|
| **字符串擦除** | 原始二进制中 "License expired" 等敏感字符串用 0x00 覆盖 |
| **运行时字符串解密** | 14+3 条字符串运行时栈上解码 + `secure_zero` 清零；`strings` 命令无输出 |
| **Per-string LCG Keystream** | 每条字符串独立 PRNG 种子（HMAC 派生），抗 `xortool` 单字节爆破 |
| **OLLVM 代码混淆** | clang-14 + 自定义 LLVM Pass：CFF（控制流平坦化）+ BCF（虚假控制流，opaque predicate）+ SUB（指令替换）；核心函数 `reconstruct_key`/`antidebug_early_check` 用三重混淆 |
| **符号剥离 + 静态链接** | Loader 和 arm96server 均为 static-linked + stripped |

#### L3: 完整性保护层

| 措施 | 说明 |
|---|---|
| **Loader .text SHA-256 自校验** | 启动时计算 executable PT_LOAD 段的 SHA-256，与编译时回填的期望值比对；不匹配静默退出 |
| **Sentinel 反定位** | sentinel magic 由 `HMAC-SHA256(base_key, idx=16)[:8]` 生成（非打印字节），每次部署唯一 |
| **诱饵哨兵 ×2** | 2 个假 sentinel（布局完全相同），攻击者搜索到 3 个结果（1 真 + 2 假） |
| **AES-GCM tag** | Core 密文自带认证标签，篡改密文 → tag 验证失败 |

#### L4: 运行时反调试层

| 措施 | 说明 |
|---|---|
| **Loader 反调试 (启动时)** | TracerPid 检测（进程级+线程级）、`PR_SET_DUMPABLE=0`、`/proc/self/maps` Frida 特征扫描；命中 `SIGKILL` |
| **arm96server (PTRACE_SEIZE+EXITKILL)** | 独立 daemon，SEIZE 所有 UID 10000-19999 的 32 位 app 进程；被 SEIZE 的进程无法被 frida/strace/gdb attach（EPERM，Linux 单 tracer 规则）|
| **arm96server 级联自毁** | daemon 被杀 → 内核 EXITKILL 语义自动 SIGKILL 所有 tracee |
| **arm96server 自保护** | `PTRACE_TRACEME` + `PR_SET_DUMPABLE=0` + `sigprocmask(全阻塞)` |
| **调试器猎杀** | 扫描 `/proc/<pid>/cmdline`，检测 gdbserver/lldb-server/strace/frida-* → `kill -9` |
| **Guardian 双向互保 (V0.29+V0.32)** | arm96server fork 出 guardian 子进程（伪装名 `hwservicemanager`）；guardian 监控 arm96server → 被杀则复活；arm96server 主循环监控 guardian → 被杀则重建。双方互保，不可同时杀。|
| **init service (.rc)** | `arm96server.rc`（class core, critical），init 级自动重启 |
| **.rc 自修复** | guardian 复活时检查 .rc 是否被删除，不在则重建 |
| **空壳检测 (V0.31)** | Loader 启动后探测 arm96server 是否真正 `PTRACE_TRACEME`；空壳 → 缩短 setitimer 到 15-45s 随机间隔 → 系统持续不稳定但原因难追溯 |

#### L5: 运营/部署层

| 措施 | 说明 |
|---|---|
| **平台绑定 (4 项硬件检查)** | CPU Part (MIDR_EL1) + ISA 特征 (SM3/SM4) + Device-Tree compatible + Android 版本号 |
| **MAC OUI 绑定** | salted FNV-1a hash，OUI 明文不入二进制 |
| **授权到期密码学绑定** | 修改 EXPIRE_EPOCH → 密钥变化 → 解密失败（非 timer 绕过） |
| **setitimer 过期自杀** | `ITIMER_REAL` 跨 `execv` 存活，SIGALRM 终止 zygote；init 自动重启 |
| **pid_max 钳位** | 自动写入 `/proc/sys/kernel/pid_max=65536`，防 32 位 bionic PID>65535 abort |

---

### 三、已知限制（请在评估中验证和扩展）

1. **无真正的秘密**: base_key = MD5(LICENSEE|EXPIRE|INTERVAL)，参数均可从 `-V` 输出和配置推断
2. **无硬件信任根**: 没有使用 TEE/TrustZone 存储密钥
3. **线性 XOR 融合**: 5 层 XOR 是线性运算，理论上可分离
4. **内存明文窗口**: `memfd_create` 后 Core 在内存中以明文存在
5. **KDF 使用 MD5**: 密钥派生用 MD5 而非 HKDF-SHA256
6. **无远程撤销**: 纯离线校验，无法远程吊销授权

---

### 四、请回答以下问题

1. **密码学安全性评估**: 5 层 XOR 密钥融合的安全强度如何？有哪些密码学弱点？
2. **逆向工程对抗评估**: 对于一个有经验的逆向工程师（拥有 IDA Pro + Frida + root 权限），破解此方案的预估工时？关键突破路径是什么？
3. **运行时防护评估**: PTRACE_SEIZE + Guardian 双向互保 + 空壳检测的组合是否有效？有哪些绕过路径？
4. **方案科学性评级**: 相对于商业级方案（Widevine L1、iOS FairPlay、VMProtect），如何定位？
5. **改进优先级**: 如果只能做 3 项改进，应该优先做什么？
6. **总体评级**: 请给出 A-F 的安全评级及理由。

---

*注：此方案的目标不是"绝对不可破解"，而是将破解成本提升到商业上不可行的程度（目标：专业逆向团队 40+ 人时）。请基于这个务实目标进行评估。*
