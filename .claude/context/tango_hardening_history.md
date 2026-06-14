# Tango 二进制加固工程 — 迭代记录 (Iteration History)

> 本文件从 [tango_hardening.md](./tango_hardening.md) 拆分而来，收录 V0.18 起逐版本迭代细节、版本依赖关系与各版本补丁代码片段。
> 当前发布状态、后续规划、验证标准与必过用例请回主文件查看。

---

**迭代路线**: V0.18 (加壳+加密, Done) → V0.19 (平台绑定, Done) → V0.20 (MAC 密钥融合+暗桩, Done) → V0.21 (代码混淆+反调试增强, Done) → V0.22 (授权密码学绑定+完整性, Done) → V0.23 (运行期强化, Done) → V0.24 (pid_max 钳位, Done) → V0.25 (静态分析对抗, Done) → V0.26 (Loader 完整性保护, Done) → V0.27 (密钥派生硬化+完整性加固, Done) → V0.28 (HMAC 种子推导+诱饵哨兵, Done) → V0.29 (Guardian 自愈+init 注册, Done) → V0.30 (arm96server 数据域融合, Done) → V0.31 (空壳检测+延时下毒, Done) → V0.32 (双向互保, Done) → V0.33 (多客户构建 + CLI 合约 + 最小镜像集成, Done) → V0.34 (arm96-only 交付与冷启动收口, Done) → V0.35 (源码模块名去 Tango 化, Done for main/1.0) → V0.36 (Core 去 Tango 品牌指纹, Done) → V0.37 (迁移追溯+打包格式, Done) → V0.38 (PID 追踪替代 SEIZE, Done) → V0.39 (Core 混淆填充 30KB, Done) → V0.40 (arm96server 加固, Done) → V0.41 (Loader 混淆覆盖扩展, Done) → V0.42 (过期节拍收敛, Done) → V0.44 (验收/文档口径收敛, Done) → V0.45 (VC 交付版本号推进, Archived) → V0.48 (关键攻击面快速封口, Done) → V0.49 (风险门禁收口, Done) → V0.50 (持续认证链, Planned) → V0.51 (对抗强化, In Progress Optional) → V0.52 (最小改动封口, Done) → V0.53（硬件绑定敏感信息去明文 + 抗补丁收口, Done） → V0.54（D1 多点见证并入解密链, Done） → V0.55（关键代码域并入解密链，封堵离线重签绕过, Implemented / Validation In Progress） → V0.56（service UID Android release fallback, Deployed Candidate） → V0.57（构建期预翻译 opt-in 试点, Historical / Paused on `debug/20260509_arm96_new_dbg_v0.57`） → V0.57→V0.58（安装时预翻译交付链、六件套打包、arm96d phase0 binfmt 接管, Current Candidate） → V0.62.0（arm96server native 反 trace / 反 hook 增强, Done） → V0.64.0（SIGSTOP 互检 + Action Token 去常量化 + 残留 IOC 收口, Partially Done） → V0.65.0（libarm96 Stub 注入机制, Done） → V0.66.0（Shadow Core Guard 暗线授权兜底, Implemented） → V0.67.0（Full Layout Repack / RX Gap Expansion, Done / Mainline Verified） → V0.67+（CAS 路径收口 / challenge-response, Planned）

**版本依赖关系**:
```
V0.29 (历史起点)
  │
  ├── V0.30 (数据域融合)        ← 独立可发布，改 loader.c + encrypt_core.py + make.sh
  │     │
  │     └── V0.31 (延时下毒)    ← 依赖 V0.30（区分“二进制不存在” vs “空壳替换”）
  │
  └── V0.32 (双向互保)          ← Done，依赖 V0.29（spawn_guardian 返回 pid_t + 主循环监控）
      │
    └── V0.33 (多客户构建/CLI/集成) ← Done，基于 V0.32 做 profile、参数契约与最小镜像路径收敛
        │
        └── V0.34 (loader 改名 arm96) ← Done，已完成交付名称、binfmt 解释器、镜像与验收入口收敛
            │
            └── V0.35 (源码模块名去 Tango 化) ← Done for main/1.0；源码目录、脚本名与文档中的兼容映射已收口，目录级迁移留给 `main/2.0`
                │
                └── V0.36 (Core 去 Tango 品牌指纹) ← Done，main.py 新增 _wipe_tango_identifiers()
                      │
                      └── V0.37 (迁移追溯+打包格式) ← Done，arm96_migration_trace.md + package.sh 改 arm96-V0.Y.Z
                            │
                            └── V0.38 (PID 追踪替代 SEIZE) ← Done；arm96server.c 全量重构 scan_and_track
                                  │
                                    └── V0.39 (Core 混淆填充) ← Done，改 main.py + encrypt_core.py，loader.c 无改动
                                        │
                                        └── V0.40 (arm96server 加固) ← Done；arm96server OLLVM + 字符串加密 + .text hash token
                                              │
                                            └── V0.41 (Loader 混淆覆盖扩展) ← Done；仅改 loader.c + version.txt
                                                │
                                                └── V0.42 (过期节拍收敛) ← Done
                                                    │
                                                    └── V0.44 (验收/文档口径收敛) ← Done
                                                        │
                                                        └── V0.45 (VC 交付版本号推进) ← Archived
                                                            │
                                                            └── V0.48 (关键数据域完整性 + release fallback 收口) ← Done
                                                                │
                                                                └── V0.49 (输入指纹门禁 + 前镜像断言 + 白名单核验) ← Current Release
                                                                    │
                                                                    └── V0.50 (loader↔arm96server 周期 challenge) ← Planned
                                                                        │
                                                                        └── V0.51 (keyed-tag + 错误结果模式) ← Planned Optional
```

    **迭代约定**: 每个版本一个独立会话，避免上下文过长。已完成链路按顺序 V0.29 → V0.30 → V0.31 → V0.32 → V0.33 → V0.34 → V0.35 → V0.36 → V0.37 → V0.38 → V0.39 → V0.40 → V0.41 → V0.42 → V0.44 → V0.45 → V0.48 → V0.49 → V0.52 → V0.53 → V0.54 → V0.55。V0.56 为 service UID fallback 候选；V0.57 的 host/build-time 试点保留为历史调试分支；当前 main/1.0 实际交付候选已推进到 V0.59.0，并以 `arm96j` / `arm96d` / `arm96_sidecar_gen` 安装时预翻译链和六件套打包为主线，完整 mustpass 仍待对新候选复跑。

#### V0.18: 加壳 + 加密 (Core Protection)
> **目标**: 磁盘上永不落明文 core，内存加载执行。
> **状态**: **Done (2026-02-23)**

- [x] **UPX 压缩**: 管线已实现 (`ENABLE_UPX=1`)，但默认关闭。目标设备 CIX P1 CS8180 CPU 的 BTI (Branch Target Identification) 与 UPX 4.2.2 ARM64 runtime stub 不兼容（SIGILL/exit 132）。待 UPX 适配 BTI landing pad 后可重新启用。
- [x] **对称加密**: 密钥 = `MD5(LICENSEE + "|" + EXPIRE_EPOCH + "|" + CHECK_INTERVAL)`；同参数→同产物（可复现），换参数→全局不同。
- [x] **memfd 执行**: loader `memfd_create("jit-cache")` → 读加密 libarm96 → 内存解密 → `fexecve(memfd)` → 磁盘永无明文。
- [x] **密钥分片**: gen/enc_key.h 中拆成 4 个 uint32 常量 + XOR 因子，编译后散布在 loader .text 段，静态分析无法直接提取。
- [x] **Loader 自保护**: loader 为 static-linked + stripped；UPX 受 BTI 限制暂不可用（同上），静态链接本身已有一定保护。
- [x] Build pipeline: `main.py patch` → `UPX --best --force` → `strip magic` → `encrypt(key)` → 产出 libarm96（密文）。

#### V0.19: 平台绑定 (Platform Binding)
> **目标**: 二进制锁定到目标硬件平台（CIX P1 CS8180 + Android 10），拷贝到其他平台无法运行。企业授权模式——同一构建可部署到同平台所有设备。
> **状态**: **Done (2026-02-23)**

- [x] **L1: CPU Part (MIDR_EL1)** — 纯 inline asm `mrs MIDR_EL1`，提取 Part 字段 [15:4]，与 `PLATFORM_CPU_PARTS` 配置的白名单比对。CIX P1 = {0xD81 (Cortex-X4), 0xD80 (Cortex-A720)}。不可伪造（硬件寄存器）。
- [x] **L2: ISA 特征 (ID_AA64ISAR0_EL1)** — 纯 inline asm `mrs ID_AA64ISAR0_EL1`，检查 SM3 [31:28] ≠ 0 且 SM4 [35:32] ≠ 0。CIX P1 独有特征，区分于 Qualcomm/MediaTek。不可伪造。
- [x] **L3a: Android 版本** — 直接读取 `/system/build.prop`（无 getprop/popen），匹配 `ro.build.version.release=10`。
- [x] **L3b: Device-Tree 平台标识** — 读取 `/proc/device-tree/compatible`，验证包含 `cix` + `sky1` 关键字。DTB 级标识，精度 A+。
- [x] **SIGILL 安全** — 安装 SIGILL handler：若 MRS 在非目标平台触发 trap，直接 `_exit(0)` 静默退出。
- [x] **构建配置** — version.txt 新增 `PLATFORM_BINDING=1`, `PLATFORM_CPU_PARTS`, `PLATFORM_REQUIRE_SM3SM4`, `PLATFORM_DT_KEYWORDS`, `PLATFORM_ANDROID_RELEASE`。
- [x] **零 libc 敏感依赖** — L1/L2 使用纯 inline asm（不依赖任何 libc），L3a/L3b 使用直接 open/read/close（内核 syscall，无 getprop/popen 等可被劫持的通道）。
- [x] **密钥融合 (Key Fusion)** — AES 解密密钥与 `ID_AA64ISAR0_EL1` 硬件寄存器值 XOR 融合。构建时 `encrypt_core.py` 用 `bound_key = base_key ⊕ {ISAR0, ISAR0}` 加密 core；运行时 loader 重建 base_key 后 MRS 读取 ISAR0 并 XOR。非目标硬件产生错误密钥 → AES 解密产出垃圾 → fexecve 失败。**无任何 `_exit(0)` 可被 NOP**，失败点在数据域而非控制流。
- [x] **ISAR0 不入二进制** — `PLATFORM_ISAR0` 仅写入 `gen/version.py`（供 `encrypt_core.py` 读取），不写入 `gen/version.h`，二进制 `.rodata` 中不包含该值。
- [x] **L4: MAC OUI 绑定** — 比对设备网络接口 MAC 地址的 OUI（前 3 字节）与构建时嵌入的 salted FNV-1a hash。OUI 明文及 salt 不出现在二进制 `.rodata` 中，仅 64-bit hash 以常量形式编译进 loader。支持多 OUI（逗号分隔）。
- [x] **MAC 三路获取** — 三种独立方法获取设备 MAC，任一命中即通过：
  - Method A (sysfs scan): `getdents64(/sys/class/net/)` 遍历所有接口，读取 `address` 文件，跳过 lo/全零 MAC
  - Method B (sysfs explicit): 依次尝试 wlan0, eth0, rmnet0, rmnet1 的 `/sys/class/net/<iface>/address`
  - Method C (netlink): `RTM_GETLINK` + `IFLA_ADDRESS` 解析，直接从内核获取，不依赖 sysfs 挂载
- [x] **MAC OUI 加密流程** — `encrypt_core.py` 新增 `--mac-ouis` 参数；`_fnv1a_64()` 以 LICENSEE 的 FNV-1a hash 为 salt 计算 OUI hash；输出到 `gen/enc_key.h` 的 `MAC_OUI_SALT`, `MAC_OUI_HASH_COUNT`, `MAC_OUI_HASH_0` 等宏。
- [x] **MAC OUI 不入 version.h** — `PLATFORM_MAC_OUIS` 从 version.txt 读取但 **跳过** version.h 生成（与 ISAR0 同理），仅通过 `--mac-ouis` 传给 encrypt_core.py。

#### V0.20: MAC 密钥融合 + 延迟暗桩 (MAC Key Fusion + Delayed Poison)
> **目标**: 将 MAC OUI 从 gate-check（可被 NOP）升级为 data-domain 防御；错误 MAC 导致解密失败 + 内存投毒，崩溃点不可复现。
> **状态**: **Done (2026-02-23)**

**Prong 1 — MAC 密钥融合**:
- [x] **MAC hash 融入 AES 密钥**: `triple_key = base_key ⊕ {ISAR0, ISAR0} ⊕ {MAC_OUI_HASH, MAC_OUI_HASH}`（64-bit hash 各重复一次填满 128-bit）。构建时 encrypt_core.py `apply_platform_fusion()` 加密用三重 XOR；运行时 loader `reconstruct_key()` 先 ISAR0 XOR 再 `mac_get_oui_hash()` XOR 后重建。错误 MAC → 错误密钥 → 解密垃圾 → fexecve 失败。无控制流可 NOP。
- [x] **enc_key.h 仍嵌入 base_key 分片**（未融合），不暴露 bound_key / triple_key。
- [x] **正确设备验证**: ISAR0+MAC 双因子融合后 TC1-TC5 全绿（2026-02-23 验证通过）。

**Prong 2 — 延迟投毒 (Delayed Poison)**:
- [x] **投毒种子**: 若 MAC OUI hash 不匹配，以 `seed = PID ^ time ^ buf_addr ^ __jit_reloc_base` 为 LCG PRNG 种子（Knuth multiplier）。
- [x] **scatter-corrupt**: 在 AES 解密完成后、fexecve 前，按 PRNG 随机偏移覆写解密缓冲区中 32 × 8 字节（共 256 B 随机散射写入）。
- [x] **伪装变量**: 局部 `static volatile uint64_t __jit_reloc_base` 存储投毒种子，跨重启积累熵；IDA 中看似 JIT 基地址。
- [x] **不可复现**: PID/time 混合使每次崩溃地址不同，阻止攻击者通过 diff 定位投毒逻辑。

#### V0.21: 代码混淆 + 反调试增强 (Obfuscation + Anti-Debug+)
> **目标**: 提升 loader 逆向分析成本，保护密钥派生/解密/反调试等核心逻辑。
> **状态**: **Done (2026-02-25)**

- [x] **OLLVM 集成**: `libObfusPass.so` 已构建（`hardening/obfus_pass/build/`）；函数混淆策略：
  - `reconstruct_key()`、`antidebug_early_check()` → `OBFUS_CFF_SUB`（CFF + SUB，最强密度）
  - `platform_check()`、`mac_get_oui_hash()`、`platform_check_cpu_part()` → `OBFUS_CFF`
  - `platform_check_isa_features()`、`antidebug_quick_exit_if_traced()` → `OBFUS_SUB`
  - GCC fallback：所有以上函数保持 `__attribute__((noinline))`，防止内联暴露。
  - 注：V0.21 未启用 BCF；V0.25 已新增 `ObfusBCFPass` 并在 `reconstruct_key`/`antidebug_early_check` 上启用 `OBFUS_CFF_BCF_SUB`。
- [x] **构建开关**: `OBFUSCATE_LOADER=1`（version.txt/version.h）；`make.sh` V0.21 路径：自动检测 `clang-14` + `libObfusPass.so` → 使用 `-fpass-plugin` 编译，plugin 不存在时尝试 `build_obfus_pass.sh` 自动构建，均失败则 fallback 到 gcc。
- [ ] **Core 内部注入 (可选)**: 利用 jump-table patch 产生的 ~148B 死代码 (0x44614–0x446A7) 注入二次校验 shellcode。（仍为可选，未实现）
- [x] **Build pipeline**: `混淆编译 loader.c` → `link static`（UPX 因目标 CPU BTI 不兼容仍禁用）；混淆在构建层已生效。
- [x] **反调试增强**: `ANTI_DEBUG_TRACERPID_TASKS=1`（线程级 TracerPid 扫描，任意线程被 trace 即触发）；`ANTI_DEBUG_ACTION_SIGKILL=1`（binfmt 路径命中时改用 SIGKILL，较 `_exit(0)` 更强硬）。
- [x] **运行期 watchdog 配置**: `ANTI_DEBUG_WATCHDOG=1`，`ANTI_DEBUG_WATCHDOG_INTERVAL=5`（version.txt + version.h 已就绪）；V0.21 时 loader.c 中 watchdog 实现保留在 `#if 0`；**V0.23 起由 arm96server 接管，V0.38 起改为 PID tracking + TracerPid 轮询**，timer-based 巡检方案不再需要。

#### V0.22: 授权密码学绑定 + 完整性加固 (Authorization Crypto Binding + Integrity)
> **目标**: 将授权到期执行从可修改的运行时常量升级为密码学约束，同时修补已知的密钥清零漏洞和解密完整性缺失，使安全评级达到 **A-**。
> **状态**: **Done (2026-02-25)**

**核心问题**: 当前 `arm_expiry_timer()` 的 `EXPIRE_EPOCH` 是运行时可修改的常量（尽管已拆分为 HI/LO 并加混淆），kill timer 与 AES 解密密钥完全解耦——攻击者修改定时器即可获得永久运行权，无需破解密码学。

**P0 — 授权到期融入密钥 (Expiry Key Fusion)**:
- [x] **EXPIRE_EPOCH 融入 AES 密钥**: 构建时 `encrypt_core.py` 将 `EXPIRE_EPOCH` 也作为密钥融合因子：
  ```
  quad_key = base_key ⊕ {ISAR0,ISAR0} ⊕ {MAC_HASH,MAC_HASH} ⊕ {EXPIRE_MASK,EXPIRE_MASK}
  ```
  其中 `EXPIRE_MASK = ~EXPIRE_EPOCH & 0xFFFFFFFF`（位取反防止 epoch=0 时 mask 为零），使修改 loader 中的 EXPIRE 常量后 → 密钥错误 → AES 解密产出垃圾 → core 无法运行。**授权期执行从控制流域转移到数据域。**
- [x] **loader.c `reconstruct_key()` 追加 EXPIRE 因子**: 运行时最后一步 XOR `get_expire_epoch()` 派生的掩码；无条件启用（`#if PLATFORM_BINDING` 之外），即使 `PLATFORM_BINDING=0` 也生效。
- [x] **EXPIRE_MASK 不入 `.rodata`**: `EXPIRE_MASK` 是运行期从 `EXPIRE_EPOCH_LO/HI` 推导的临时变量，不写入 `gen/version.h`，二进制无独立可 patch 的掩码常量。

**P0 — 密钥清零修复 (Key Erasure Fix)**:
- [x] **`secure_zero` 替换 `memset`**: `decrypt_core_to_memfd()` 中 `memset(key, 0, 16)` 和 `memset(iv, 0, 16)` 改为 `secure_zero()`。`secure_zero` 宏定义在 loader.c 头部：API≥23/glibc≥2.25 映射 `explicit_bzero`，否则用 `volatile uint8_t *` 循环填零（编译器不可 DCE）。
- [x] IV 方向：评估后**跳过 IV XOR**：CBC 下 key 已错时 IV 效果边际为零；GCM 升级后 nonce 由 auth tag 保护，IV XOR 直接作废，不值得增加死代码。

**P0 — 解密完整性 (AES-CBC → AES-GCM)**:
- [x] **升级为 AES-128-GCM**: GCM 模式自带 128-bit 认证标签（GHASH），解密时若密钥错误或密文被篡改，Tag 验证直接失败，无需额外 HMAC。相较 CBC+PKCS7，GCM 失败点更明确，错误密钥无法产生「碰巧合法的 padding」。
- [x] **密文格式升级**: `TGENC018` → `TGENC022`，格式：`magic(8B) | orig_size(4B) | nonce(12B) | tag(16B) | ciphertext`；`ENC_HDR_SIZE` = 40B。
- [x] **`aes128.h` 新增 `aes128_gcm_decrypt()`**: 纯 C 实现，无外部依赖：forward AES 块加密（`aes128_encrypt_block`）+ GF(2¹²⁸) 乘法（`gcm_gf_mul`，无查表，纯位移）+ CTR 解密 + GHASH tag 生成 + 常数时间比较；tag 不符时自动清零 `*pt`，约 200 行。
- [x] **encrypt_core.py 对应更新**: 优先用 `cryptography.AESGCM`，回退 `PyCryptodome.MODE_GCM`；nonce 每次 `os.urandom(12)` 随机生成（构建时）；`MAGIC = b"TGENC022"`；`aes_cbc_encrypt_stdlib` 保留但不再调用。

**P1 — 信息泄漏收敛 (.rodata 字符串混淆)**:
- [x] **`LICENSEE`/`EXPIRE_DATE_STR` 不再作为字符串字面量写入 `version.h`**: `make.sh` 新增 XOR(0x5A) 编码 helper `_xor_c_array()`，生成 `LICENSEE_ENC`/`EXPIRE_DATE_ENC` 字节数组宏及 `STR_XOR_KEY`；明文仅保留在 `version.py` 供 `encrypt_core.py` 读取。
- [x] **`print_version()` 运行时栈上解码**: 两个花括号作用域分别解码 LICENSEE 和 EXPIRE，`printf` 之后立即 `secure_zero` 清栈；`-V` 输出格式完全不变（TC1 合规）；`strings tango_translator | grep -i cheersu` 无输出。
- [x] 合规确认：TC1 要求 3 行固定格式，方案保持格式不变，仅消除 `.rodata` 明文暴露，不影响必过用例。

#### V0.23: 运行期强化 (Runtime Hardening)
> **目标**: 覆盖运行中动态分析路径，补全代码保护，使安全评级达到 **A**。
> **状态**: **Stable (2026-02-26)**

- [x] **运行期 watchdog 激活 (P0)**: SELinux/seccomp 兼容性分析通过；移除 `loader.c` 中的 `#if 0`，`kill_traced_app_processes()` 保留为 live 代码（`__attribute__((unused))`）供将来计时器方案复用。`antidebug_watchdog_loop`/`start_antidebug_watchdog` 的 double-fork daemon 逻辑已重新包入 `#if 0`（产生可见进程，不可接受）；`main()` 中无 `start_antidebug_watchdog` 调用。验证：`zygote_secondary=running`，`ps | grep tango` 无常驻 daemon，TC1-TC3 全绿，binfmt flags=POC。
  - 后续方向：timer-based 方案需找到一个在 execv 后仍可回调 loader 代码的机制（如 libarm96 内注入，或在下次 zygote restart 时扫描）。
- [x] **Frida/gadget 注入检测 (P1)**: `antidebug_scan_maps()` 扫描 `/proc/self/maps`，检测 `frida`、`gadget`、`xposed`、`substrate`、`lsposed` 等特征字符串；命中则静默 `_exit(0)`。验证：mmap libfrida-gadget.so → DETECTED，真实 Frida 16.6.6 attach → 命中。
- [x] **arm96server 反调试守护进程 (P2)**: 独立 daemon (`/system/bin/arm96server`)，由 loader 在 is_zygote 路径中 double-fork 启动。功能：
    - V0.23 初版使用 `PTRACE_SEIZE + PTRACE_O_EXITKILL`；V0.38 起改为 PID tracking，避免信号拦截导致的 APP ANR。
    - 定期扫描 `/proc`，将 zygote_secondary 子进程（UID 10000-19999）加入 tracked PID 列表。
    - 发现 tracked app 的 `TracerPid > 0` 时杀 tracer + app；同时扫描并杀掉 frida/strace/gdbserver 等工具进程。
    - arm96server 被杀 → guardian 读取 `TRACKED_PID_FILE`，SIGKILL 全部 tracked app（cascade 等价清理）。
  - 自保护：`PTRACE_TRACEME` + `sigprocmask(全阻塞)` 防止被 attach
  - Zygote 检测：`/proc/pid/exe` 包含 "memfd"（区分 32 位 zygote 与 64 位 app_process64）
  - 新增文件：`hardening/src/arm96server.c`（~300 行），修改 `loader.c`/`make.sh`/`install.sh`/`package.sh`
    - 验证：TC-008（guardian tracked-PID 级联）、TC-009（attach 后被拒绝或扫描窗口内清理）、TC-010（arm96server 自保护 EPERM）全绿
    - **V0.38 后信号策略**: app 进程不再被 ptrace seize，因此没有 signal-delivery-stop 需要转发；当前实现已移除 app tracee 转发路径，仅保留 signalfd 作为静默 scan-loop pacing。
  - **已知环境约束**: 容器 `kernel.pid_max=4194304` 时，32 位 bionic libc 的 `pthread_mutex_t` 仅用 16 位存储 owner TID，PID > 65535 触发 abort（详见踩坑记录 #7）。需在部署时或 loader 启动时将 `pid_max` 钳位到 65536（计划纳入 V0.24）。

#### V0.24: pid_max 钳位 + 环境兼容 (PID Clamping + Environment Compat)
> **目标**: 解决容器环境 `pid_max=4194304` 导致 32 位进程 PID > 65535 时 bionic abort 的问题。
> **状态**: **Stable (2026-02-27)**

- [x] **pid_max 钳位**: loader 在 `is_zygote` 路径中、exec core 之前写入 `/proc/sys/kernel/pid_max=65536`。每次 zygote 启动自动执行。实现为 `open("/proc/sys/kernel/pid_max", O_WRONLY)` + `write("65536\n")` + `close()`，约 6 行 C 代码，位于 `kill_orphaned_app_processes()` 之前。
- [x] **影响评估**: pid_max=65536 对 64 位进程无副作用（容器典型进程数 < 500，PID 空间充足）。容器内 `/proc/sys/kernel/pid_max` 独立可写，不影响宿主。

#### 验证结果 (2026-02-27, V0.24)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.24" / "for Cheersu Technology Co" / "to 20260401-11:00:00" |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 无效参数静默 | ✅ | 输出 0 字节 |
| TC5 | 32 位应用运行 | ✅ | com.dragon.read 正常启动运行，pid_max 自动钳位后无 abort |
| TC8 | EXITKILL 级联 | ✅ | kill -9 arm96server → dragon.read 被内核 SIGKILL |
| TC9 | strace 附加 app | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| TC10 | strace 附加 arm96server | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| binfmt | flags = POC | ✅ | 以 POC 为准 |
| pid_max | 自动 clamp 65536 | ✅ | loader is_zygote 路径自动写入；4194304 → 65536 端到端验证通过 |

**pid_max 自动钳位 (V0.24)**: loader 在 is_zygote 路径中自动将 `pid_max` 从任意值钳位到 65536，无需手动操作。已验证：设置 pid_max=4194304 → 重启 zygote → loader 自动写 65536 → zygote 正常启动 → 应用正常运行。

#### V0.25: 静态分析对抗 (Static Analysis Countermeasures)
> **目标**: 消除 `strings`/IDA 的静态分析价值，提升逆向成本。
> **状态**: **Stable**

- [x] **P0 — 辅助字符串运行时 XOR 解密**: `make.sh` 新增 14 条辅助字符串的 XOR(0x5A) 编码（`_str_entries` 数组循环调用 `_xor_c_array`），生成 `STR_<name>_ENC` / `STR_<name>_ENC_LEN` 字节数组宏到 `version.h`。`loader.c` 新增 `_xor_decode()` inline helper，在各使用点声明 `static const uint8_t _enc[] = STR_XXX_ENC; char _buf[STR_XXX_ENC_LEN+1];` 并在使用后 `secure_zero(_buf)`。覆盖字符串：`/system/bin/libarm96`、`/system/bin/arm96server`、`arm96server`（comm）、`--exe`、`ANDROID_SOCKET_`、`/proc/sys/kernel/pid_max`、`jit-cache`、`tango_translator`、以及 6 个 frida 检测 needle（frida/gadget/gum-js-loop/xposed/substrate/lsposed）。`#define CORE_PATH` / `ARM96SERVER_PATH` 删除，`exec_core()` 签名改为接受 `core_path` 参数。**验收**: `strings tango_translator | grep -iE 'libarm96|arm96server|ANDROID_SOCKET'` 无输出。
- [x] **P1 — BCF (虚假控制流)**: `ObfusPass.cpp` 新增 `ObfusBCFPass`（~100 行）：`canBCFBlock()` 资格检查（跳过 EH pad / return / unreachable / 小块）→ `createOpaquePredicate()` 生成 `((x*(x-1)) & 1) == 0`（volatile alloca 阻止 LLVM 优化器常数折叠）→ `runBCF()` 对每个候选 BB clone + trampoline + opaque branch 重定向前驱，最多处理 8 个 BB/函数控制膨胀率。Pass 注册顺序：CFF → BCF → SUB。新增 `OBFUS_CFF_BCF_SUB` 属性（`annotate("obfus_cff,obfus_bcf,obfus_sub")`），应用到 `reconstruct_key()` 和 `antidebug_early_check()`。
- [x] **P2 — 调试器进程检测**: `arm96server.c` 新增 `scan_and_kill_debuggers()`（~60 行）：遍历 `/proc/<pid>/cmdline`，提取 argv[0] 的 basename 与 needle 数组比对（gdbserver / lldb-server / strace / frida-server / frida-inject / frida-gadget / frida），命中 `kill(pid, SIGKILL)`。集成到 daemon 主循环 `scan_and_track()` 之后调用。

#### V0.26: Loader 完整性保护 (Loader Integrity Protection)
> **目标**: 防止 loader 被单独 binary patch 绕过密钥融合/反调试。
> **状态**: **Done (2026-02-27)**

- [x] **P0 — Loader `.text` 段哈希自校验**: 启动时计算 loader 关键段（executable PT_LOAD segment，包含 `.text`）的 SHA-256 哈希，与编译时嵌入的预期值比对；不匹配则静默退出。
  - **SHA-256 实现**: `hardening/src/sha256.h`，纯 C 实现（FIPS 180-4），~170 行，`_sha256_transform` 标记 `noinline` 防止 OLLVM 膨胀 64 轮循环。
  - **Sentinel 结构**: `_text_hash_sentinel[40]` 存储在 `.data` 段（`volatile + __attribute__((used))`）：
    - `[8B]` magic `"TGHASH26"` — 构建脚本定位标记
    - `[32B]` SHA-256 哈希占位（编译后为全零，构建脚本回填真实值）
  - **两阶段构建** (`shell/make.sh`):
    1. 正常编译 loader（哈希占位 = 全零）
    2. Python 脚本解析 ELF64 program headers → 找到 `PT_LOAD + PF_X` 段 → 读取段字节 → `hashlib.sha256()` → 搜索 `TGHASH26` + 32×`\x00` sentinel → binary-patch 32 字节哈希
  - **运行时校验** (`loader.c` — `integrity_check_text()`):
    1. 开 `/proc/self/exe` → 解析 ELF64 phdr → 定位 executable segment
    2. `malloc` + `read` entire segment → `sha256()` → 常数时间比较
    3. 不匹配 → `_exit(0)`（静默，无输出）
    4. 全零占位（未 patch 的 dev 构建）→ 跳过检查
  - **调用时机**: `main()` 最顶部，在 `-V` 解析和反调试之前（所有路径受保护）
  - **混淆保护**: `OBFUS_CFF_BCF_SUB`（CFF + BCF + SUB 三重混淆），使逆向分析者难以定位哈希比较逻辑和 sentinel 地址
  - **关键设计选择**:
    - 哈希覆盖整个 executable PT_LOAD 段（含 `.text` 和同段的 `.rodata`），而非仅 `.text` section — 更鲁棒（section headers 可被 strip）
    - Sentinel 在 `.data` 段（`PF_R|PF_W`），不在 `.text` 段（`PF_R|PF_X`）— 无循环依赖
    - 使用 program headers（非 section headers）— 对 stripped binary 兼容
    - `__attribute__((noinline))` on `sha256()`/`_sha256_transform()` — 防止 OLLVM 膨胀加密循环

#### V0.27: 密钥派生硬化 + 完整性校验加固 + 字符串混淆升级 (Key Derivation + Integrity + String Obfuscation)
> **目标**: 消除完整性校验的绕过路径，消除 sentinel 的可定位性，将字符串混淆从全局单字节 XOR 升级为 per-string LCG keystream。
> **状态**: **Done (2026-02-27)**

**P0-1 — 完整性自校验去后门**:
- [x] **移除全零跳过的逃逸路径**: V0.26 的 sentinel 全零时跳过校验（dev 便利），release 应始终执行。移除运行时 `hash_is_all_zero()` 判断。
- [x] **编译期开关 (DEV_BUILD)**: 保留 `-DINTEGRITY_CHECK_DISABLED` 用于开发构建快速迭代（跳过编译后的 hash 回填步骤）。`make.sh` 中 `DEV_BUILD=1` 自动传入此 flag。

**P0-2 — Sentinel 反定位**:
- [x] **消除 ASCII magic**: V0.26 使用 `"TGHASH26"` 作为 sentinel magic（`strings` 可直接定位），改为 `SHA-256("TANGO_INTEGRITY_V27")[:8]` 的 8 字节非打印序列。
- [x] **make.sh 同步**: 构建脚本搜索模式从 `b"TGHASH26"` 改为 `sha256("TANGO_INTEGRITY_V27")[:8]` + 32 零字节。

**P1-2 — 字符串混淆升级 (Per-string LCG keystream)**:
- [x] **去掉全局单字节 XOR(0x5A)**: V0.25 使用 `STR_XOR_KEY=0x5A` 全局异或，`xortool` 等工具可自动爆破。
- [x] **Per-string 独立种子**: 每条字符串使用 `SHA-256("TANGO_LCG_V27_"+name)[:8]` (little-endian uint64) 作为 LCG PRNG 种子。
- [x] **LCG keystream**: Knuth 乘法器 `6364136223846793005`，增量 `1442695040888963407`，`key_byte = (state >> 33) & 0xFF`。
- [x] **make.sh `_lcg_seed()` + `_xor_c_array_lcg()`**: 生成 `*_ENC` 字节数组 + `*_SEED` uint64 宏到 version.h。
- [x] **loader.c `_xor_decode()` 升级**: 接受 `uint64_t seed` 参数，内部运行 LCG PRNG 产生 per-byte keystream。
- [x] **验收**: `strings tango_translator | grep -iE "libarm96|arm96server|ANDROID_SOCKET|cheersu"` 无输出；`xortool` 无法破解。

#### V0.28: HMAC-SHA256 运行时种子推导 + 诱饵哨兵 (HMAC Seed Derivation + Decoy Sentinels)
> **目标**: 将字符串混淆种子从编译时静态常量升级为运行时 HMAC 推导，消除 `.rodata` 中 seed 常量的暴露；新增诱饵哨兵增加逆向噪声。
> **状态**: **Done (2026-02-28)**

**P0 — HMAC-SHA256 运行时种子推导**:
- [x] **去掉 `*_SEED` 宏**: V0.27 将 per-string seed 作为 `#define XXX_SEED 0x...` 编译进二进制，IDA 可直接定位 16 个 uint64 常量并关联到 LCG 解码逻辑。V0.28 移除所有 seed 宏输出。
- [x] **`_hmac_sha256_u32()` 函数**: 标准 HMAC-SHA256 构造，接受 16 字节 key + 4 字节 uint32 message，输出 32 字节 digest。~30 行纯 C，复用 `sha256.h`。
- [x] **HKDF 索引枚举 (19 项)**: `HKDF_IDX_LICENSEE=0` ~ `HKDF_IDX_DECOY_B=18`，loader.c 和 make.sh 必须一致。
- [x] **`_derive_seed(uint32_t idx)` 函数**: 从 `ENC_KEY_PART0..3 ⊕ ENC_KEY_XOR` 重建 base AES key → `HMAC-SHA256(key, LE_u32(idx))[:8]` → uint64 LE 种子 → 清零中间状态。标记 `OBFUS_CFF_SUB` 混淆保护。
- [x] **make.sh `_lcg_seed_by_idx()`**: `HMAC-SHA256(MD5(LICENSEE|EXPIRE|INTERVAL), LE_u32(idx))[:8]`，与 loader.c 运行时推导一致。
- [x] **17 处调用点迁移**: 所有 `_xor_decode(..., XXX_SEED)` 改为 `_xor_decode(..., _derive_seed(HKDF_IDX_XXX))`。

**P0-2 — 哨兵 magic HMAC 派生**:
- [x] **HMAC-derived sentinel magic**: `HMAC-SHA256(base_key, LE_u32(16))[:8]`，每次部署（不同 licensee/expire/interval 组合）产生不同 magic，防止跨部署 pattern 复用。
- [x] **make.sh 接受 `sentinel_magic_hex` 参数**: 二阶段 patcher 不再自行计算固定 hash，由 make.sh 传入。

**P1 — 诱饵哨兵结构**:
- [x] **`_decoy_sentinel_a[40]`**: HMAC magic (idx=17) + `SHA-256("DECOY_A_" + base_key)` 伪随机 32 字节，模拟已 patch 的合法 sentinel。
- [x] **`_decoy_sentinel_b[40]`**: HMAC magic (idx=18) + 32 零字节，模拟未 patch 的真实 sentinel（与编译后、回填前的真实 sentinel 布局完全相同）。
- [x] **make.sh 生成宏**: `SENTINEL_MAGIC_BYTES`、`DECOY_A_MAGIC_BYTES`、`DECOY_A_HASH_BYTES`、`DECOY_B_MAGIC_BYTES`。
- [x] **对抗效果**: 攻击者搜索 "8 字节 magic + 32 零字节" 会命中 decoy_b 而非真实 sentinel；搜索所有 40 字节结构会得到 3 个结果（1 真 + 2 假），增加分析复杂度。

#### 验证结果 (2026-02-28, V0.28)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.28" / "for Cheersu Technology Co" / 授权有效期 |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 无效参数静默 | ✅ | 输出 0 字节 |
| TC4 | 授权期内 reboot | ✅ | zygote/system_server 正常 |
| TC5 | 32 位应用运行 | ✅ | com.dragon.read 正常启动运行 |
| TC8 | EXITKILL 级联 | ✅ | kill -9 arm96server → 所有 tracee 被内核 SIGKILL |
| TC9 | Frida/strace 附加 app | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| TC10 | strace 附加 arm96server | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| binfmt | flags = POC | ✅ | 以 POC 为准 |
| strings | 静态分析对抗 | ✅ | `strings` 无 libarm96/arm96server/cheersu/ANDROID_SOCKET 等敏感字符串 |

设备 192.168.11.35:5004，TC-001~TC-010 全绿。


#### V0.29: arm96server Guardian 自愈 + init 注册 (Guardian Self-Revival + init Service)
> **目标**: arm96server 被杀后自动复活；.rc 被删后自动修复；init 级重启保障。解决"杀掉 arm96server 后新 app 不受保护"的已知弱点。
> **状态**: **Done (2026-03-01)**

**背景 — 已知弱点分析**:

arm96server 当前承载 3 项业务（全部是反调试/反逆向，零授权逻辑）：
1. **PTRACE_SEIZE 进程锁定** — 对 UID 10000-19999 的 32 位 app 执行 SEIZE + EXITKILL
2. **反附加自保护** — PTRACE_TRACEME + PR_SET_DUMPABLE=0 + 全信号阻塞
3. **调试器猎杀** — 扫描 /proc cmdline 发现 frida/gdb/strace → SIGKILL

手动杀掉 arm96server：
- 授权不受任何影响（setitimer 在 zygote 进程，与 arm96server 无关）
- 所有已 SEIZE 的 32 位 app 立即全灭（EXITKILL 内核语义）
- 之后新启动的 app 完全不受反调试保护（arm96server 无自动重启机制）

**攻击场景覆盖**:

| # | 攻击手段 | V0.28 结果 | V0.29 防御 |
|---|---|---|---|
| 3 | 只删 .rc，运行时 kill arm96server | ⚠️ 无保护 | guardian 复活 + .rc 自修复 |
| 6 | 删 .rc + 运行时 kill arm96server | ⚠️ 无保护 | guardian 复活 + .rc 自修复 |
| 7 | 同时 kill main + guardian + 删 .rc | ⚠️ 依赖重启 | init .rc 保障（若 .rc 已修复）|

**P0 — Guardian 自愈守护**:
- [x] **`spawn_guardian()` 函数**: arm96server `main()` 启动时 fork 子进程，伪装进程名 `hwservicemanager`（`PR_SET_NAME` + `argv[0]` 覆写），独立 `PTRACE_TRACEME` 自保护 + `PR_SET_DUMPABLE=0` + 全信号阻塞（`sigfillset` + `sigprocmask`）
- [x] **监控逻辑**: guardian `sleep(2)` 循环，`kill(main_pid, 0)` 检测 arm96server 主进程存活；不存在 → 调用 `repair_rc()` → `fork` + `execl` 重启 arm96server → guardian 自身退出（新 arm96server 会 fork 新 guardian）
- [x] **killall 对抗**: guardian 进程名 `hwservicemanager`，`killall arm96server` 无法杀掉 guardian

**P1 — .rc 自修复**:
- [x] **`repair_rc()` 函数**: 检查 `/system/etc/init/arm96server.rc` 是否存在（`access()`）；不存在则 `open` + `write` 重建（best-effort，/system 可能只读）
- [x] **.rc 内容**: `service arm96server /system/bin/arm96server` + `class core` + `user root` + `group root` + `critical` + `seclabel u:r:su:s0`
- [x] **调用时机**: guardian 复活 arm96server 之前执行

**P2 — init service 注册**:
- [x] **新增 `init/arm96server.rc`**: 标准 Android init service 定义（6 行）
- [x] **`res/install.sh` 更新**: 新增 `deploy_arm96server_rc()` 函数，heredoc 写入 .rc 到 `/system/etc/init/`
- [x] **`res/uninstall.sh` 更新**: 卸载时 `rm -f /system/bin/arm96server` + `rm -f /system/etc/init/arm96server.rc`
- [x] **与 loader 兼容**: loader `launch_arm96server()` 已有单例检查；init 先启动 → loader 扫到已存在 → skip

**改动清单**:

| 文件 | 改动 | 实际行数 |
|---|---|---|
| `hardening/src/arm96server.c` | 新增 `spawn_guardian()` + `repair_rc()` + main 调用 | +131 行 (622→753) |
| 新增 `init/arm96server.rc` | init service 定义 | +7 行 |
| `res/install.sh` | 新增 `deploy_arm96server_rc()` | +16 行 (153→169) |
| `res/uninstall.sh` | 移除 arm96server + arm96server.rc | +2 行 (54→56) |

**验收标准**:
- [x] kill -9 arm96server → 2~3 秒内自动复活（guardian 拉起）
- [x] 删 .rc + kill -9 arm96server → 自动复活 + .rc 恢复
- [x] `killall -9 arm96server` → guardian 存活（伪装名不匹配）→ 复活 main
- [x] 必过用例 TC-001~TC-010 全绿

#### 验证结果 (2026-03-01, V0.29)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.29" / "for Cheersu Technology Co" / 授权有效期 |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 无效参数静默 | ✅ | 输出 0 字节 |
| TC4 | 授权期内 reboot | ✅ | zygote/system_server 正常 |
| TC5 | 32 位应用运行 | ✅ | com.dragon.read 正常启动运行 |
| TC8 | EXITKILL 级联 | ✅ | kill -9 arm96server → 所有 tracee 被内核 SIGKILL |
| TC9 | Frida/strace 附加 app | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| TC10 | strace 附加 arm96server | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| Guardian | kill -9 → 自动复活 | ✅ | guardian 检测主进程死亡 → repair_rc + re-exec，~3s 恢复 |
| Guardian | 删 .rc + kill -9 → .rc 自修复 | ✅ | .rc 被重建后 arm96server 复活 |
| Guardian | killall -9 → guardian 存活 | ✅ | 伪装名 hwservicemanager，killall 无法命中 |
| binfmt | flags = POC | ✅ | 以 POC 为准 |

设备 192.168.11.35:5004，TC-001~TC-010 + Guardian 三场景全绿。

#### V0.30: arm96server 数据域融合 (arm96server Token Key Fusion)
> **目标**: 将 arm96server 二进制的存在性融入 AES 密钥派生链（第 5 层 XOR），删除二进制 → 解密失败 → 32 位 app 完全无法启动。**无任何 if/else 分支可 NOP**。
> **状态**: **Done (2026-03-01)**
> **前置**: V0.28（V0.29 不是硬前置，但建议先完成）

**设计原理**:

现有 AES 密钥已有 4 层融合：
```
quad_key = base_key
  ⊕ {ISAR0, ISAR0}           # L1: CPU 硬件绑定 (V0.19)
  ⊕ {MAC_HASH, MAC_HASH}     # L2: 网卡 MAC OUI 绑定 (V0.20)
  ⊕ {EXPIRE_MASK, EXPIRE_MASK} # L3: 授权到期绑定 (V0.22)
  ⊕ {TOKEN[0:8], TOKEN[8:16]}  # L4: arm96server 存在性绑定 (V0.30) ← 新增
```

**攻击场景覆盖**:

| # | 攻击手段 | V0.29 结果 | V0.30 防御 |
|---|---|---|---|
| 2 | 删除 arm96server 二进制 | ⚠️ 无 arm96server 但 app 仍可运行 | ✅ 解密失败，app 无法启动 |
| 4 | 删 .rc + 删二进制 | ⚠️ 同上 | ✅ 解密失败 |
| 8 | kill 全部 + 删 .rc + 删二进制 | ⚠️ 下次重启 app 仍可运行 | ✅ 解密失败 |

**P0 — Token 生成 (构建时)**:
- [x] **`encrypt_core.py` 生成 16 字节令牌**: `token = HMAC-SHA256(base_key, LE_u32(HKDF_IDX_ARM96SRV_TOKEN))[:16]`
- [x] **追加到 arm96server 尾部**: 构建完 arm96server 后，将 token 写入二进制末尾
- [x] **加密 core 时融合**: `quint_key = quad_key ⊕ {token[0:8], token[8:16]}`

**P0 — Token 校验 (运行时)**:
- [x] **`reconstruct_key()` 新增第 5 层**: 读 arm96server 尾部 16 字节 → XOR 融合到 key
- [x] **路径复用**: arm96server 路径通过已有的 `HKDF_IDX_STR_ARM96SRV_PATH` 解码（不引入新字符串）
- [x] **失败模式**: 二进制不存在 → token 全零 → 密钥错误 → AES-GCM tag 验证失败 → 解密输出清零 → fexecve 失败
- [x] **HKDF 索引**: `HKDF_IDX_ARM96SRV_TOKEN = 19`（新增，紧跟 DECOY_B=18）

**改动清单**:

| 文件 | 改动 | 行数估算 |
|---|---|---|
| `hardening/src/loader.c` | `reconstruct_key()` 新增第 5 层 XOR；HKDF 枚举新增 idx=19 | +45 行 |
| `hardening/src/encrypt_core.py` | 生成 token、追加到 arm96server、加密时 XOR 融合；新增 `--arm96server` 参数 | +45 行 |
| `make.sh` | 构建顺序调整：先编译 arm96server → encrypt_core.py 追加 token + 加密 core | +10 行 |

**验收标准**:
- [x] 正常部署 → TC-001~TC-010 全绿
- [x] 篡改 arm96server 尾部 16 字节 token → 32 位 zygote 无法启动（AES-GCM tag 验证失败）
- [x] 恢复正确 arm96server → 系统恢复正常，32 位 app 正常运行
- [ ] 仅替换 arm96server 前 N-16 字节（保留 token）→ app 正常（token 正确）

#### 验证结果 (2026-03-02, V0.30)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.30" / "for Cheersu Technology Co" / 授权有效期 |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 无效参数静默 | ✅ | 输出 0 字节 |
| TC4 | binfmt flags = POC | ✅ | 以 POC 为准 |
| TC5 | 32 位应用运行 | ✅ | com.dragon.read (PID forked from zygote) 正常启动运行 |
| TC8 | EXITKILL 级联 | ✅ | kill -9 arm96server → dragon.read 立即消失，arm96server 自动重启 |
| TC9 | strace 附加 32-bit app | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| TC10 | strace 附加 arm96server | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| V0.30 | Token 篡改 → zygote 消失 | ✅ | 覆写尾部 16B 为全零 → 32-bit zygote 消失 |
| V0.30 | Token 恢复 → 系统正常 | ✅ | 恢复正确 arm96server → zygote + app 正常 |

设备 192.168.11.35:5004，TC-001~TC-010 + V0.30 Token 篡改/恢复验证全绿。

#### V0.31: 空壳检测 + 延时下毒 (Hollow Shell Detection + Delayed Poison)
> **目标**: arm96server 被替换为空壳（保留正确 token 但不做 PTRACE_SEIZE）→ loader 检测 → 缩短杀时器 → 系统表面正常但持续不稳定，攻击者难以关联到 arm96server。
> **状态**: **Done (2026-03-02)**
> **前置**: V0.30（需区分"二进制不存在"和"二进制是空壳"——V0.30 处理前者，V0.31 处理后者）

**攻击场景**:

攻击者保留 arm96server 二进制不动（或仅 patch 内部逻辑使 PTRACE_SEIZE 不执行），V0.30 的 token 融合不触发（token 正确），但反调试保护实际失效。

**P0 — TRACEME 探测 + 延时下毒**:
- [x] **`verify_arm96server_alive()` 函数**: loader `launch_arm96server()` 之后、`exec_core()` 之前（仅 is_zygote 路径），usleep 200ms 等待启动
- [x] **探测逻辑**:
  1. 扫描 /proc 找到 arm96server comm 的 PID
  2. `ptrace(PTRACE_ATTACH, srv_pid)` 尝试附加
  3. 成功 = 假的（没做 TRACEME）→ 立即 DETACH → 标记 `poisoned=1`
  4. EPERM = 真的（已有 tracer = init via TRACEME）→ 正常
  5. 进程不存在 → 标记 `poisoned=1`
- [x] **下毒动作**: `poisoned=1` 时，将 `setitimer` 的 interval 缩短为 `15 + (getpid() % 31)` 秒（随机 15-45s）
- [x] **攻击者体验**: 系统看起来正常启动，32 位 app 能跑，但每 15-45 秒 zygote 被 SIGALRM 杀死 → app 频繁闪退 → "系统不稳定"而非"arm96server 检测失败"
- [x] **OBFUS_CFF_SUB 混淆**: `verify_arm96server_alive()` 标记 `OBFUS_CFF_SUB` 混淆保护

**改动清单**:

| 文件 | 改动 | 实际行数 |
|---|---|---|
| `hardening/src/loader.c` | 新增 `verify_arm96server_alive()` (~65行) + is_zygote 路径调用 + poison timer (~10行) | +75 行 |

**验收标准**:
- [x] 替换 arm96server 为空壳二进制（保留正确 token，不做 TRACEME）→ 15-45 秒内 zygote 被 SIGALRM 杀死；循环重启
- [x] 正常 arm96server → 不触发下毒，行为不变（60s 监控 PID 不变）
- [x] 必过用例 TC-001~TC-010 全绿

#### 验证结果 (2026-03-02, V0.31)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.30" / "for Cheersu Technology Co" / 授权有效期 |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 无效参数静默 | ✅ | 输出 0 字节 |
| TC4 | binfmt flags = POC | ✅ | 以 POC 为准 |
| TC8 | EXITKILL 级联 | ✅ | kill -9 arm96server → tracee 被内核 SIGKILL |
| TC9 | strace 附加 32-bit app | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| TC10 | strace 附加 arm96server | ✅ | `PTRACE_SEIZE: Operation not permitted` (TracerPid=1) |
| V0.31 | 空壳 arm96server → 下毒 | ✅ | 替换为无 TRACEME 的空壳二进制（保留 token）→ zygote 每 15-45s 被杀，PID 循环变化 |
| V0.31 | 正常 arm96server → 无下毒 | ✅ | 恢复正常 arm96server → 60s 内 zygote PID 不变 |

设备 192.168.11.35:5004，TC-001~TC-010 + V0.31 空壳检测/正常验证全绿。

#### V0.32: Guardian 双向互保 (Mutual Guard)
> **目标**: arm96server ↔ guardian 双向监控，提升同时杀掉两者的攻击门槛。
> **状态**: **Done (2026-03-02)**
> **前置**: V0.29

**评估记录 (2026-03-02)**:
- ~~**主进程名伪装 — 已评估，不实施**~~：V0.23~V0.31 已构成完整防御闭环（附加→EPERM，杀掉→复活，删除→解密失败，空壳→下毒），进程可见但不可被利用。进程名伪装属于 security by obscurity，且 `/proc/*/exe` 仍暴露真实路径，PPID=1 可区分伪装内核线程。额外代码复杂度（loader comm 扫描需改 exe 匹配）不值得。
- ~~**guardian 伪装**~~ — V0.29 已实现（`hwservicemanager`）
- ~~**killall 对抗**~~ — V0.29 已实现

**P0 — 双向互保**:
- [x] **`spawn_guardian()` 返回值改为 `pid_t`**: 返回 guardian 子进程 PID（fork 失败返回 0），原 `void` 无法传递 PID
- [x] **main 保存 guardian_pid**: `pid_t guardian_pid = spawn_guardian(argc, argv);` 在 self-protection 之前调用
- [x] **main 监控 guardian**: 主循环 slow-path（`scan_and_kill_debuggers()` 之后）新增 `kill(guardian_pid, 0)` 检测；ESRCH → `guardian_pid = spawn_guardian(argc, argv)` 重建
- [x] **攻击门槛**: 必须同一瞬间杀掉两个不同进程名的进程（arm96server + hwservicemanager），单杀任一方均在一个 scan interval 内被对方重建

**改动清单**:

| 文件 | 改动 | 实际行数 |
|---|---|---|
| `hardening/src/arm96server.c` | `spawn_guardian()` 返回值 void→pid_t + main 保存 PID + 主循环 guardian 存活检测 | +7 行 (753→760) |
| `hardening/src/version.txt` | CUSTOM_VERSION=V0.31 → V0.32 | 1 行 |

**验收标准**:
- [x] kill -9 guardian → 主进程在下一轮 scan 中检测到 ESRCH 并重建新 guardian（PID 变化确认）
- [x] kill -9 主进程 → guardian 复活主进程（V0.29 不退化，PID 变化确认）
- [x] 必过用例 TC-001~TC-010 全绿

#### 验证结果 (2026-03-02, V0.32)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.32" / "for Cheersu Technology Co" / 授权有效期 |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 无效参数静默 | ✅ | 输出 0 字节 |
| TC4 | binfmt flags = POC | ✅ | 以 POC 为准 |
| TC5 | 32 位应用运行 | ✅ | com.dragon.read 正常启动运行 |
| TC8 | EXITKILL 级联 | ✅ | kill -9 arm96server → tracee 被内核 SIGKILL，guardian 复活 arm96server |
| TC9 | strace 附加 32-bit app | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| TC10 | strace 附加 arm96server | ✅ | `PTRACE_SEIZE: Operation not permitted` |
| V0.32 | kill -9 guardian → 重建 | ✅ | arm96server 检测 ESRCH → spawn 新 guardian (PID 4814→4907) |
| V0.32 | kill -9 arm96server → 复活 | ✅ | guardian 检测死亡 → re-exec，新实例 spawn 新 guardian (PID 4813→4913) |

设备 192.168.11.35:5004，TC-001~TC-010 + V0.32 双向互保验证全绿。

#### V0.33: 多客户构建 + CLI 合约调整 + 最小镜像集成
> **目标**: 在 V0.32 安全基线不变的前提下，补齐多客户定制编译链、拆分版本/授权信息输出接口，并把镜像集成收敛到最小可验证路径。
> **状态**: **Done (2026-03-16)**
> **前置**: V0.32

**P0 — 多客户 profile 构建**:
- [x] `shell/make.sh` 新增 `--build <target>`，从 `hardening/src/profiles/<target>.txt` 合并覆盖 `version.txt`
- [x] 构建产物改为 profile 感知输出：`build/out/<target>/bin/tango_translator`
- [x] 新增/收敛 profile：
    - `vc.txt`: `LICENSEE=vclusters`，`PLATFORM_DT_KEYWORDS=cix,sky1`，`PLATFORM_MAC_BINDING=1`
    - `cs.txt`: `LICENSEE=Cheersu Technology Co`，当前沿用 `vc` 的 OUI 绑定策略
    - `myt.txt`: `LICENSEE=moyunteng`，`PLATFORM_DT_KEYWORDS=myt,p1`，`PLATFORM_MAC_BINDING=0` 以适配 Docker 虚拟 MAC 环境

**P1 — CLI 合约调整**:
- [x] loader 将原 `-V` 语义拆分为：
    - `-v`: 只输出 1 行版本号，如 `V0.33`
    - `-i`: 只输出 2 行授权主体与到期时间
- [x] `verify_mustpass.sh`、`res/install.sh`、源码侧 `src/tango_translator_reimpl/main.cpp` 同步更新到新 CLI 合约
- [x] 设备侧临时推送验证通过：`-v` 1 行、`-i` 2 行、`-V` 0 字节、`-h` 0 字节

**P2 — 打包与镜像集成收敛**:
- [x] `shell/package.sh` 改为从 `build/last_build.env` 读取 `BUILD_TARGET/OUT_BIN/VERSION`，按 profile 打包 `arm96-<version>-<target>.tgz`
- [x] `res/install.sh` 修复 `arm96server.rc` 的 here-doc 写入兼容性，改为 `printf` 直写，适配 Android `sh`
- [x] `prebuilts/Android.mk` 从 3 个 prebuilt 收敛为只保留 `tango_translator`
- [x] `tango.mk` 最小镜像集成仅保留 `tango_translator` 与 `hello_world_arm32`，不再打包 `tangolic` 和 `tango_pretranslator`

**改动清单**:

| 文件 | 关键改动 |
|---|---|
| `shell/make.sh` | 新增 `--build <target>`，profile 合并覆盖，profile 感知输出目录 |
| `shell/package.sh` | 从构建缓存读取 target/version/source bin，按 profile 打包 |
| `hardening/src/profiles/{vc,cs,myt}.txt` | 新增/完善多客户授权与平台绑定配置 |
| `hardening/src/loader.c` | `-V` 拆分为 `-v` + `-i` |
| `verify_mustpass.sh` | 必过用例改用 `-v`/`-i` 新契约 |
| `res/install.sh` | 安装后自检改用 `-v`；`arm96server.rc` 写入改为 `printf` |
| `prebuilts/Android.mk` | 移除 `tangolic`/`tango_pretranslator` prebuilt |
| `tango.mk` | 最小镜像集成仅保留 `tango_translator` |

**验证结果 (2026-03-16, V0.33)**:
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| CLI-1 | `tango_translator -v` | ✅ | 输出 1 行版本号 |
| CLI-2 | `tango_translator -i` | ✅ | 输出 2 行授权信息 |
| CLI-3 | `tango_translator -V` | ✅ | 输出 0 字节 |
| CLI-4 | `tango_translator -h` | ✅ | 输出 0 字节 |
| IMG-1 | 最小镜像集成移除 `tangolic`/`tango_pretranslator` | ✅ | 仅保留 `tango_translator` 路径验证通过 |
| DEV-1 | 手动 `make.sh` + 安装 + 整机重启 | ✅ | 用户实机确认“正常” |

当前版本建议以 **V0.34** 作为后续 profile 构建、设备部署与验收文档的基线版本。

#### V0.34: arm96-only 交付与冷启动收口（已完成）
> **目标**: 将当前 loader 交付名从 `tango_translator` 收敛为 `arm96`，进一步消除对外显式的 Tango 命名，同时保持 split 模型、CLI 合约、binfmt 启动链和现有授权/反调试语义不变。
> **状态**: **Done (2026-03-17)**
> **前置**: V0.33

**实现结果**:
- 交付入口、镜像集成和文档口径已统一到 `arm96`
- `tango.mk`/`tango.rc`/`tango-debug.rc`/`tango.te` 已分别收敛为 `arm96.mk`/`arm96.rc`/`arm96-debug.rc`/`arm96.te`
- 设备与宿主验收入口均已切换到 `arm96 -v/-i/-h`
- `init` 注册前会显式清理 `arm96` 与 `tango_translator` 两个旧 binfmt 节点，规避宿主残留规则导致的冷启动失败

**验收结果 (2026-03-17)**:
- `verify_mustpass.sh`: `11 PASS / 0 FAIL / 0 SKIP`
- `verify_mustpass_host.sh`: `ALL HOST-ORCHESTRATED TESTS PASSED`
- 宿主真实部署链冷启动恢复：`sys.boot_completed=1`，`zygote_secondary=running`
- 产物侧只保留 `arm96`、`libarm96`、`arm96server`，不再交付 `/system/bin/tango_translator`

#### V0.35: 源码模块名去 Tango 化（main/1.0 范围已完成）
> **目标**: 在 V0.34 已完成交付层去 Tango 化的基础上，继续收敛源码目录、构建脚本名、文档中的 translator 模块命名，降低后续协作中的“交付名已是 arm96、源码名仍是 tango_translator_reimpl”的混淆。
> **状态**: **Done for main/1.0 (2026-03-17)**
> **前置**: V0.34

**分支边界（必须遵守）**:
- `android10/vendor/hello/arm96/src/` 是 C++ 源码主线区域，对应 `arm96_translator_reimpl` 重构主任务。
- 源码主线在 `main/2.0` 分支持续迭代；`main/1.0` 分支当前只允许做交付链、hardening、验收链和公共文档层的命名收敛。
- 因此，`main/1.0` 上 **暂不对** `src/tango_translator_reimpl/` 做物理目录改名、源码搬迁或大规模源码内引用改写。
- 当前阶段允许记录的“命名迁移”仅限公共文档与构建兼容层：源码目录仍保留 `src/tango_translator_reimpl/`，源码模块主名记为 `arm96_translator_reimpl`，并保留 `tango_translator_reimpl` 兼容目标。
- 后续切回 `main/2.0` 继续主线时，应以这里的兼容映射为线索，而不是假设 `src/` 已在 `main/1.0` 完成实体迁移。

**本轮完成情况（main/1.0）**:
- 已完成公共文档、构建辅助脚本、README、测试入口的命名收敛与兼容映射记录。
- 已建立 `arm96_translator_reimpl` 主模块名，并保留 `tango_translator_reimpl` 兼容目标。
- 已完成新旧模块构建验证，确认不影响当前交付链。
- 未完成且不应在 `main/1.0` 完成的部分：`src/tango_translator_reimpl/` 物理目录改名、源码搬迁、源码内大规模引用改写；这些动作留给 `main/2.0`。

**核心范围**:
- `src/tango_translator_reimpl/` 目录及对应引用
- 运行时构建脚本命名继续收敛到 `arm96_*`
- 相关说明文档、路径示例、调试说明
- 在不破坏现有 Android 模块名依赖的前提下，分层评估“文件/目录名”与“模块名”是否需要同步收敛

**建议顺序**:
1. 先清点源码目录名、脚本名和文档锚点的影响面
2. 再决定 Android 模块名是否保留兼容层，避免一次性改动过大
3. 最后补一轮镜像构建与必过验收，确认“源码名变化”不影响交付名与运行时行为

**P2 — 镜像与构建系统切换**:
- `package.sh` 默认打包 `arm96`
- `res/install.sh` / `res/uninstall.sh` 以 `arm96` 为主路径执行备份与恢复
- `prebuilts/Android.mk`、`tango.mk` 切到 `arm96` 模块/文件名
- `verify_mustpass_host.sh` 作为最终验收入口同步改用 `arm96`

**P3 — 去兼容别名（可选）**:
- 当所有设备、镜像、binfmt 和验收链已稳定后，再决定是否彻底移除 `/system/bin/tango_translator`
- 若移除旧名，必须补做一次全链路回归：安装、binfmt、生效、reboot、过期后 reboot、卸载回滚

**关键风险**:
- **binfmt 失配**: 解释器路径改了但 register/rule 未同步，32 位 app 会直接无法启动
- **stock/rollback 混乱**: 旧版 stock 备份仍叫 `tango_translator.stock`，若迁移策略不清晰，卸载时可能恢复到错误路径
- **文档/验收分叉**: 只改产物名，不同步 `verify_mustpass*.sh` 和文档，会导致“代码已改名、验收仍跑旧名”
- **兼容窗口缺失**: 若直接一次性删掉旧名，当前设备和镜像侧已有规则可能全部失效，排障成本高

**V0.34 验收标准（规划）**:
- `arm96 -v` 只输出 1 行版本号，`arm96 -i` 只输出 2 行授权信息
- binfmt 规则指向新 loader 后，32 位 app 仍可正常启动
- `verify_mustpass.sh` 与 `verify_mustpass_host.sh` 全量通过
- 授权过期后的 kill loop 与 reboot 行为与 V0.33 一致
- 若保留兼容别名：`tango_translator -v/-i` 在兼容期内行为与 `arm96` 等价
- 若移除兼容别名：卸载/回滚路径仍能恢复到上一个稳定版本

**建议实现顺序**:
1. 先做脚本/文档层的命名抽象与兼容方案确认
2. 再做安装包与镜像产物双写验证
3. 最后切 binfmt 与最终验收入口

#### V0.36: Core 二进制去 Tango 品牌指纹 (Brand Wipe)
> **目标**: 擦除 core 二进制 `.rodata` 中所有 `tango`/`Tango`/`TANGO` 品牌字符串，消除二进制中的来源指纹，同时保留运行必需的功能性引用。
> **状态**: **Done (2026-03-28)**
> **前置**: V0.35
> **commits**: `4b03611` (补提代码) → `0bfeccf` (去掉大部分 tango 字段) → `46b1537` (V0.36)

**核心改动 — `_wipe_tango_identifiers()`**:
- 在 `main.py` 的 `patch_binary()` 末尾新增 `_wipe_tango_identifiers(f)` 调用
- 线性扫描整个 core 二进制，检测所有 `tango` / `Tango` / `TANGO` 子串
- 匹配后等长替换为 `sys__` / `Sys__` / `SYS__`（保持偏移不变）
- **白名单跳过**（功能性字符串，改动会导致运行时故障）：
  - `/dev/tango*` — 内核设备节点路径（`open()` 目标）
  - `*.tango` — JIT 持久化缓存文件扩展名
  - `tango_prop` — SELinux 属性上下文标签
  - `tango uid` — UID 日志消息

**实测数据**: 128 处匹配 → 117 处擦除 + 11 处功能性保留

**改动清单**:

| 文件 | 改动 | commit |
|---|---|---|
| `hardening/src/main.py` | 新增 `_wipe_tango_identifiers()` 函数 + 在 `patch_binary()` 末尾调用 | `0bfeccf` |
| `hardening/src/version.txt` | `CUSTOM_VERSION=V0.36` | `46b1537` |
| `README.md`, `build_*.sh`, `hardening/README.md`, `test/test_integration.sh` | 补提代码（构建脚本与文档对齐） | `4b03611` |

**验收**: 必过用例全绿；`strings core_patched | grep -ciE 'tango'` 仅剩白名单中的功能性引用。

#### V0.37: 迁移追溯文档与打包格式统一
> **目标**: 补充 legacy tango 移除 → arm96 新增的完整迁移追溯记录；prebuilt 三件套对齐；打包版本号格式统一为 `arm96-V0.Y.Z`。
> **状态**: **Done (2026-03-29)**
> **前置**: V0.36
> **commits**: `89c4d9c` (迁移追溯文档) → `baffe3c` (V0.37) → `7593e80` (package.sh 格式) → `fefdb1a` (V0.37.0)

**P0 — 迁移追溯文档** (`89c4d9c`):
- 新增 `arm96_migration_trace.md`：记录本轮先移除 legacy tango、再新增 arm96 的完整迁移路径
- 更新 `debug.md`、`hardening/README.md`：加入追溯入口
- prebuilts 更新为 arm96 / libarm96 / arm96server 三件套
- `prebuilts/Android.mk` 同步改用 arm96 模块名
- `verify_mustpass.sh` / `verify_mustpass_host.sh` 版本号升级到 V0.37
- `sync_active_prebuilts.sh` 与 `test/Android.bp` 对齐

**P1 — 打包版本号格式** (`7593e80`):
- `package.sh` 改动：
  - 版本号来源：始终从 `hardening/src/version.txt` 的 `CUSTOM_VERSION` 读取
  - 包名：从 `tango_reimpl` 改为 `arm96`
  - 版本格式：从 `VERSION[-TARGET]` 改为直接使用 `CUSTOM_VERSION`（如 `V0.37.0`）
  - 输出产物名：`arm96-V0.37.0.tgz`
- prebuilts 同步更新

**改动清单**:

| 文件 | 改动 | commit |
|---|---|---|
| `arm96_migration_trace.md` | 新增迁移追溯文档 | `89c4d9c` |
| `README.md`, `debug.md`, `hardening/README.md` | 文档追溯入口 | `89c4d9c` |
| `prebuilts/arm96`, `prebuilts/libarm96`, `prebuilts/arm96server` | 二进制更新 | `89c4d9c`, `7593e80` |
| `prebuilts/Android.mk` | 模块名对齐 arm96 | `89c4d9c` |
| `verify_mustpass.sh`, `verify_mustpass_host.sh` | 版本号 → V0.37 | `89c4d9c` |
| `shell/package.sh` | 包名 `arm96` + 版本格式 `V0.Y.Z` | `7593e80` |
| `hardening/src/version.txt` | `CUSTOM_VERSION=V0.37` → `V0.37.0` | `baffe3c`, `fefdb1a` |

**验收**: 必过用例全绿；`package.sh` 输出 `arm96-V0.37.0.tgz`。

#### V0.38: arm96server PID 追踪替代 PTRACE_SEIZE
> **目标**: 将 arm96server 的 32 位应用监控机制从 `PTRACE_SEIZE` 改为 PID 列表追踪 + `TracerPid` 轮询检测，消除 SEIZE 拦截信号导致的应用 ANR 问题（QQ `libSecShell.so` 等使用 ptrace 反调试的应用）。
> **状态**: **Done (2026-03-30) — Superseded by V0.44 验收口径收敛**
> **前置**: V0.37
> **commits**: `f17530c` (PID 追踪重构) → `94be040` (V0.38.0)

**背景 — PTRACE_SEIZE 的问题**:
`PTRACE_SEIZE` 会拦截所有发送给 tracee 的信号，导致 signal-delivery-stop。尽管 V0.32 通过 `reap_tracees()` 两阶段转发做了缓解（Pass 1 waitpid 正常转发 + Pass 2 GETSIGINFO 探测卡住的 tracee），但某些应用内部使用 ptrace 反调试（如 QQ `libSecShell.so`），SEIZE 与之冲突导致 ANR。

**核心重构 — `scan_and_track()` 改为 PID 追踪**:

| 维度 | V0.37 (PTRACE_SEIZE) | V0.38 (PID tracking) |
|------|----------------------|----------------------|
| 进程发现 | `/proc` 扫描 + `PTRACE_SEIZE` | `/proc` 扫描 + 加入 `g_tracked[]` |
| 信号干扰 | ✅ 拦截所有信号 → ANR 风险 | ❌ 无信号干扰 |
| 调试器检测 | TracerPid 检查（排除自身） | TracerPid > 0 即 kill（不区分自身） |
| 级联语义 | `PTRACE_O_EXITKILL` 内核自动级联 | guardian 读 `TRACKED_PID_FILE` 手动 kill tracked apps |
| 扫描间隔 | 5s (可配) | 3s (硬编码) |
| 信号转发 | `reap_tracees()` 两阶段 (~80 行) | 无 app tracee，移除转发路径 |

**新增数据结构与函数**:

| 函数/结构 | 作用 |
|---|---|
| `g_tracked[MAX_TRACKED]` / `g_tracked_count` | PID 追踪列表（替代 `g_seized[]`） |
| `is_tracked()` / `add_tracked()` / `prune_tracked()` | 追踪集合维护 |
| `prune_tracked()` | 移除已退出的 PID（`kill(pid, 0)` + `ESRCH`） |
| `write_tracked_pids()` | 持久化 PID 列表到 `TRACKED_PID_FILE`，供 guardian failover |
| `kill_all_tracked()` | Guardian cascade：遍历内存列表 + PID 文件全部 SIGKILL |

**`scan_and_track()` 新流程**:
1. 扫描 `/proc`，发现 app UID + zygote_secondary 子进程
2. ~~`ptrace(PTRACE_SEIZE, pid, ...)`~~ → `add_tracked(pid)`
3. 新增 TracerPid 巡检：遍历 `g_tracked[]`，任何 TracerPid > 0 → kill(tracer) + kill(app)
4. `write_tracked_pids()` 持久化列表

**Guardian tracked-PID cascade**:
- `spawn_guardian()` 检测到 main arm96server 死亡时，调用 `kill_all_tracked()` 杀死所有被追踪的 32 位应用
- `kill_all_tracked()` 双重来源：内存 `g_tracked[]` + 磁盘 `TRACKED_PID_FILE`（覆盖 guardian 自身无内存数据的场景）

**改动清单**:

| 文件 | 改动 | commit |
|---|---|---|
| `hardening/src/arm96server.c` | 全量重构：PID 追踪替代 PTRACE_SEIZE (~+150/-100 行) | `f17530c` |
| `README.md`, `debug.md` | 文档同步 | `f17530c` |
| `prebuilts/arm96`, `prebuilts/libarm96`, `prebuilts/arm96server` | 二进制更新 | `f17530c` |
| `hardening/src/version.txt` | `CUSTOM_VERSION=V0.38.0` | `94be040` |

**验收标准**:
- [x] 必过用例 TC-001~TC-010 全绿
- [x] QQ 等含 `libSecShell.so` 的应用不再 ANR
- [x] `kill arm96server` → guardian 杀死所有被追踪应用（tracked-PID cascade）
- [x] frida/strace 附加 32 位应用 → TracerPid 检测 → SIGKILL

#### V0.39: Core 混淆填充 30KB (Obfuscated Padding + Fake Crypto)
> **目标**: 在 libarm96 的 ELF 尾部追加 30KB 混淆数据块，拉开与原始 tango_translator 的文件大小差异，同时让逆向者误以为存在附加的加密/许可证校验逻辑。追加数据在 AES-GCM 加密保护内，运行时被 `original_size` 截断丢弃，不影响功能。
> **状态**: **Done**
> **前置**: V0.38

**设计原理**:
- 文件大小差异 ~30KB，无法直接关联到原始二进制
- 混淆内容设计为多层假加密/假许可证结构，增加逆向噪声
- 追加数据与 ELF 一起被 AES-GCM 加密，磁盘上不可读不可篡改

**核心架构 — 追加块（30720 字节 = 30KB）**:

```
Offset    Size    Content
0x0000    4       Magic: "A96D" (encrypt_core.py 检测标记)
0x0004    4       uint32 LE: elf_original_size (追加前的文件大小)
0x0008    4       uint32 LE: block_total_size = 30720
0x000C    4       uint32 LE: block_version = 1
0x0010    2048    Layer 1: 心经全文 UTF-8 (~260B 原文)
                  XOR 分片为 16 chunk, 每 chunk 128B
                  每 chunk 前置 4B 假 HMAC tag
                  间隔填充伪 TLV 条目
0x0810    8192    Layer 2: 伪 AES-CBC 密文块
                  16B 假 IV + 8160B os.urandom + 16B 假 tag
0x2810    4096    Layer 3: 伪 DER 结构
                  假 RSA-2048 公钥 (270B DER) + 假 Ed25519 公钥 (44B)
                  假 X.509 证书片段 + 假签名体 (256B)
                  XOR 编码的 URL 格式假字符串
0x3810    6144    Layer 4: 伪 License TLV
                  假 UUID, 假 timestamp, 假 interval
                  假 feature flags bitmask
                  假 HMAC-SHA256 校验值散布
0x4E10    ~6400   Layer 5: 高熵随机填充
                  os.urandom 凑满 30720 字节
```

**Layer 1 核心内容 — 心经全文**:
- 《般若波罗蜜多心经》UTF-8 全文 (~260 字节)
- XOR 分片为 16 chunk 散布在 2048 字节空间内
- 每个 chunk 前有 4 字节假 HMAC prefix，模拟认证数据结构
- 逆向者解码后看到佛经文本，增加心理层面的混淆

**encrypt_core.py 适配**:

```python
# encrypt_file() 加密前，检测文件尾部的追加块:
#   从文件末尾回溯，检查 offset=elf_size 处是否为 "A96D" magic
#   → 是: original_size = struct.unpack("<I", plaintext[elf_size+4:elf_size+8])
#   → 否: original_size = len(plaintext) (向后兼容)
```

**改动清单**:

| 文件 | 改动 | 复杂度 |
|---|---|---|
| `hardening/src/main.py` | `patch_binary()` 末尾追加 30KB 混淆块 | 中 |
| `hardening/src/encrypt_core.py` | 检测 `A96D` magic → 取 `elf_original_size` 作为加密 header 的 `original_size` | 低 |
| `hardening/src/version.txt` | 可选：新增 `PADDING_SIZE_KB=30` 配置项 | 低 |
| `hardening/src/loader.c` | **无需修改** — `pt_len = original_size` 自动截断追加数据 | - |

**验收标准**:
- [x] `ls -la build/out/bin/libarm96` 大小比 `tango_translator.cheersu` 大 ~30KB
- [x] 必过用例 TC-001~TC-010 全绿
- [x] hexdump 加密前中间产物确认 `A96D` magic 和心经分片存在
- [x] 设备端 32 位应用正常启动运行

#### V0.40: arm96server 加固 (arm96server Hardening)
> **目标**: arm96server 当前为明文 ELF，对静态分析零抵抗。本版本对 arm96server 实施三层加固：OLLVM 代码混淆、字符串加密、.text 完整性融入 token。patch arm96server .text 后 token 失效 → AES 解密失败 → 32 位应用无法启动。
> **状态**: **Done**
> **前置**: V0.39

**背景 — 当前 arm96server 的安全弱点**:

| 维度 | arm96 (loader) | libarm96 (core) | arm96server |
|------|---------------|-----------------|-------------|
| 磁盘加密 | ❌ 明文 | ✅ AES-128-GCM | ❌ 明文 |
| .text 完整性 | ✅ SHA-256 sentinel | ✅ (在加密内) | ❌ 无 |
| 字符串加密 | ✅ LCG-XOR + HMAC | ✅ (在加密内) | ❌ 全部明文 |
| 代码混淆 | ✅ OLLVM CFF+SUB | ✅ (在加密内) | ❌ 无 |
| strip 符号 | ✅ | ✅ | ✅ (仅此一项) |

**攻击路径（当前可行）**: IDA 打开 arm96server → 定位 `scan_and_kill_debuggers()` 中的 `kill()` → NOP → 保留尾部 16 字节 token 不动 → `adb push` 替换 → 反调试完全失效，解密不受影响。

**P0 — OLLVM 代码混淆**:
- [x] **编译切换**: `make.sh` 中 arm96server 编译改用 clang-14 + OLLVM (`-fpass-plugin=libObfusPass.so`)
- [x] **函数混淆策略**:

| 函数 | 混淆属性 | 理由 |
|------|---------|------|
| `scan_and_kill_debuggers()` | `OBFUS_CFF_SUB` | 核心反调试，最高优先级保护 |
| `scan_and_track()` | `OBFUS_CFF_SUB` | TracerPid 检测 + PID tracking |
| `spawn_guardian()` | `OBFUS_CFF` | 守护逻辑 + 进程名伪装 |
| `kill_all_tracked()` | `OBFUS_SUB` | Guardian tracked-PID cascade 语义 |
| `main()` | `OBFUS_CFF` | 入口 + 自保护 (TRACEME/信号阻塞) |
| `repair_rc()` | 无 | 低安全价值 |

- [x] **BTI 注意**: 初始版本**不启用 BCF**（仅 CFF+SUB），避免 arm96server 上的 BTI 兼容性风险

**P1 — 字符串加密**:
- [x] **方案**: 与 loader V0.27 一致——编译时 LCG seed + 加密字节数组，运行时栈上解码 + `secure_zero`
- [x] **不使用 HMAC 运行时派生**（与 loader V0.28 不同）——arm96server 不持有 AES base_key，改为 `make.sh` 预计算 seed 写入 `gen/arm96server_strings.h`
- [x] **新增 HKDF 索引 20~31**:

| 索引 | 名称 | 字符串 |
|------|------|--------|
| 20 | `HKDF_IDX_SRV_ARM96SERVER` | `"arm96server"` |
| 21 | `HKDF_IDX_SRV_HWSERVMGR` | `"hwservicemanager"` |
| 22 | `HKDF_IDX_SRV_RC_PATH` | `"/system/etc/init/arm96server.rc"` |
| 23 | `HKDF_IDX_SRV_PID_FILE` | `"/data/local/tmp/.arm96_tracked_pids"` |
| 24 | `HKDF_IDX_SRV_GDBSERVER` | `"gdbserver"` |
| 25 | `HKDF_IDX_SRV_LLDB` | `"lldb-server"` |
| 26 | `HKDF_IDX_SRV_STRACE` | `"strace"` |
| 27 | `HKDF_IDX_SRV_FRIDA_SERVER` | `"frida-server"` |
| 28 | `HKDF_IDX_SRV_FRIDA_INJECT` | `"frida-inject"` |
| 29 | `HKDF_IDX_SRV_FRIDA_GADGET` | `"frida-gadget"` |
| 30 | `HKDF_IDX_SRV_FRIDA` | `"frida"` |
| 31 | `HKDF_IDX_SRV_RC_CONTENT` | RC 文件内容 |

- [x] **验收**: `strings arm96server | grep -iE "arm96server|hwservice|frida|strace|gdbserver"` 无输出

**P2 — .text hash 融入 token（双端 XOR 消去）**:
- [x] **当前弱点**: V0.30 的 token = `HMAC(base_key, 19)[:16]`，追加到 arm96server 尾部。loader 读尾部 16 字节 XOR 到 key。**攻击者仅 patch .text 保留尾部 → token 不变 → 解密仍成功**。

- [x] **V0.40 升级——构建端 (encrypt_core.py)**:
```python
# 1. 解析 arm96server ELF, 提取 executable PT_LOAD 段
text_hash = sha256(arm96server_text_section)[:16]
# 2. 原始 token（与 V0.30 相同的推导）
raw_token = HMAC(base_key, 19)[:16]
# 3. 存储到 arm96server 尾部的是 XOR 混合值
stored_token = raw_token ^ text_hash
# 4. 加密 libarm96 仍使用 raw_token（不变）
```

- [x] **V0.40 升级——运行端 (loader.c)**:
```c
// reconstruct_key() 的 token 融合段:
// 新增: 计算 arm96server 的 .text SHA-256
uint8_t srv_text_hash[32];
sha256_elf_executable_segment("/system/bin/arm96server", srv_text_hash);

// 读取尾部 16 字节 (= stored_token = raw_token ^ build_text_hash)
uint8_t tail[16];  // ... 现有逻辑 ...

// V0.40 新增: XOR text hash 消去
for (int i = 0; i < 16; i++)
    tail[i] ^= srv_text_hash[i];
// .text 未改时: tail = (raw ^ build_hash) ^ build_hash = raw_token ✅
// .text 被 patch: tail = (raw ^ build_hash) ^ patched_hash ≠ raw_token ❌

secure_zero(srv_text_hash, 32);
// 后续: key ^= tail (原有逻辑不变)
```

- [x] **数学证明**:
  - .text 未改: `fusion = (raw ⊕ build_hash) ⊕ build_hash = raw` → 正确密钥 ✅
  - .text 被 patch: `fusion = (raw ⊕ build_hash) ⊕ patched_hash ≠ raw` → 错误密钥 → AES-GCM 失败 ✅
  - 攻击者修正 tail: 需要 `raw_token` → 需要 `base_key` → 无法获取 ✅

**改动清单**:

| 文件 | 改动 | 复杂度 |
|---|---|---|
| `hardening/src/arm96server.c` | 混淆属性标注 + 字符串加密基础设施 (`_srv_xor_decode` + 引用改写) | 中 |
| `shell/make.sh` | arm96server 编译加 OLLVM flags + 生成 `gen/arm96server_strings.h` | 中 |
| `hardening/src/encrypt_core.py` | `generate_arm96server_token()` 改为含 .text hash 的双端 XOR 方案 | 低 |
| `hardening/src/loader.c` | `reconstruct_key()` token 融合段新增 .text hash XOR 消去 (~50 行) | 中 |

**版本内实施顺序**:
1. P0 (OLLVM 混淆) → 编译、验证必过用例
2. P1 (字符串加密) → 编译、`strings` 验证、必过用例
3. P2 (.text hash 融入 token) → 完整构建链、必过用例 + 篡改验证

**验收标准**:
- [x] 必过用例 TC-001~TC-010 全绿
- [x] `strings arm96server` 无敏感字符串
- [x] IDA 打开 arm96server：关键函数 CFG 被 CFF 拍平，无法直接识别 kill() 调用
- [x] patch arm96server 任意 .text 字节（保留尾部 token）→ 32 位 zygote 无法启动（AES-GCM 失败）
- [x] 恢复正确 arm96server → 系统恢复正常

**攻击场景覆盖更新**:

| # | 攻击手段 | V0.38 结果 | V0.40 防御 |
|---|---|---|---|
| A1 | IDA 静态分析 arm96server | ⚠️ 明文，逻辑清晰 | ✅ OLLVM 混淆 + 字符串加密 |
| A2 | NOP 掉 kill() 保留 token | ⚠️ 解密正常，反调试失效 | ✅ .text hash 融入 token → 解密失败 |
| A3 | `strings` 提取调试工具名单 | ⚠️ 7 个名字明文可见 | ✅ 运行时解码，`strings` 无输出 |
| A4 | 删除 arm96server | ✅ V0.30 已覆盖（token 全零） | ✅ 不变 |
| A5 | 替换为空壳（保留 token） | ✅ V0.31 已覆盖（TRACEME 探测） | ✅ 不变 + .text hash 不匹配 |

#### V0.41: Loader OLLVM 混淆覆盖扩展 (Loader Obfuscation Coverage Extension)
> **目标**: loader (arm96) 已于 V0.21 启用 OLLVM 混淆，但仅覆盖 11 个关键函数。本版本将覆盖范围扩展至 21 个函数，重点补全解密链路 (`decrypt_core_to_memfd`) 和运行期防护函数。
> **状态**: **Done (2026-04-01)**
> **前置**: V0.40
> **改动范围**: 仅 `loader.c` + `version.txt`（make.sh / encrypt_core.py 无改动）

**背景 — 当前混淆覆盖审计 (V0.40)**:

已混淆 (11 个):
| 函数 | 混淆属性 | 版本 |
|------|---------|------|
| `reconstruct_key` | CFF+BCF+SUB | V0.21 |
| `integrity_check_text` | CFF+BCF+SUB | V0.26 |
| `antidebug_early_check` | CFF+BCF+SUB | V0.21 |
| `_derive_seed` | CFF+SUB | V0.28 |
| `antidebug_scan_maps` | CFF+SUB | V0.23 |
| `arm_expiry_timer` | CFF+SUB | V0.22 |
| `verify_arm96server_alive` | CFF+SUB | V0.31 |
| `platform_check` | CFF | V0.19 |
| `platform_check_cpu_part` | CFF | V0.19 |
| `platform_check_isa_features` | SUB | V0.19 |
| `antidebug_quick_exit_if_traced` | SUB | V0.21 |

**V0.41 新增混淆 (10 个)**:
| 函数 | 混淆属性 | 理由 |
|------|---------|------|
| `decrypt_core_to_memfd` | **CFF+BCF+SUB** | 核心中的核心：密钥使用、AES-GCM 解密、memfd_create、fexecve 全在此函数 |
| `exec_core` | **CFF+SUB** | fexecve 前最后一站，构造 argv 传递给 core |
| `launch_arm96server` | **CFF+SUB** | 暴露 arm96server 启动方式和路径 |
| `kill_traced_app_processes` | **CFF+SUB** | /proc 遍历的反调试检测逻辑 |
| `kill_orphaned_app_processes` | **CFF+SUB** | 启动时孤儿进程扫描策略 |
| `antidebug_watchdog_loop` | **CFF+SUB** | watchdog 扫描循环，暴露扫描间隔和策略 |
| `platform_check_mac_oui` | **CFF** | MAC OUI gate-check 逻辑（含 hash 比较） |
| `mac_oui_scan_netlink` | **CFF** | netlink RTM_GETLINK 协议解析 |
| `antidebug_trigger` | **SUB** | 反调试触发时的杀进程动作 |
| `start_antidebug_watchdog` | **SUB** | fork + 伪装 comm 名 |

**不改动的函数**（太小或无安全价值）:
- `sys_memfd_create` (3 行 syscall wrapper)
- `parse_fd_from_path` / `argv0_is_self` / `debug_log_argv` (辅助解析)
- `read_tracer_pid` / `any_task_traced` / `read_uid_and_tracerpid_of_pid` (简单 /proc 读取)
- `platform_sigill_exit` (1 行 `_exit(0)`)
- `print_version` / `print_version_info` / `sanitize_fds` / `is_android_socket_fd` (无安全价值)
- `fnv1a_salted` / `mac_oui_match` / `parse_mac_oui` / `mac_oui_check_sysfs_iface` / `mac_oui_scan_sysfs` / `mac_oui_probe_explicit` / `mac_get_first_oui` (纯算术/已被调用者保护)
- `platform_check_android_release` / `platform_check_dt_compatible` (简单字符串匹配)

**预估影响**:
| 指标 | V0.40 | V0.41 预估 |
|------|-------|-----------|
| arm96 大小 | 575 KB | ~680-720 KB |
| 间接分支 (`br xN`) | 102 | ~170+ |
| 编译时间 | ~8s | ~12s |
| 运行性能 | 基线 | 无感知差异（新增函数均为启动期一次性调用） |

**风险**:
- `decrypt_core_to_memfd` 约 150 行，CFF+BCF+SUB 三重混淆后 .text 膨胀约 8-12 倍。若编译器栈溢出或生成质量差，降级为 CFF+SUB
- 编译后需重新计算 .text SHA-256 hash（既有 make.sh 流程自动完成）

**验收标准**:
- [x] 编译日志 `[V0.21] OBFUSCATE_LOADER=1 → clang-14 + ObfusPass plugin`
- [x] `objdump` 间接分支数 108（V0.40 为 102，+6；3 个函数在 `#if 0` 块内未编译）
- [x] 必过用例 TC-000 ~ TC-010 全绿
- [x] zygote / zygote_secondary 稳定运行（无重启循环）

#### 已完结 (Archived)
- [x] ~~**运行期反调试巡检**~~ — **Superseded by arm96server (V0.23)**
- [x] **反调试增强 (Anti-Debug+)**: `TRACERPID_TASKS` + `ACTION_SIGKILL` 已在 V0.21 实现，进一步强化已纳入 V0.23。
- [x] ~~**去 Watchdog 化**~~ — **Done (V0.16)**

### 0.19 版本 (Platform Binding)
*   **状态**: **Stable (2026-02-23)**
*   **基础**: 继承 0.18 所有特性（AES-128-CBC 加密 core + memfd 内存执行 + 密钥分片）。

#### 平台绑定 — 五层检查

**为什么？** 企业授权模式下，一次构建需要部署到同平台（CIX P1 CS8180）的所有设备。相比 per-device HWID 绑定，平台绑定更适合批量部署。同时需防止二进制被复制到其他 ARM64 平台（Qualcomm/MediaTek 等）运行。

**设计原则**:
-   **最小化 libc 依赖**: L1/L2 使用纯 inline asm 读取 ARM64 系统寄存器，不可被 LD_PRELOAD 或 libc hook 劫持（loader 为静态链接，但仍减少攻击面）。L4 使用直接 syscall（open/read/socket/sendto/recvmsg），不依赖 libc 高层封装。
-   **纵深防御**: 5 层独立检查，任一失败即静默退出。攻击者需同时绕过所有层。
-   **静默失败**: 不输出错误信息，不暴露检查逻辑的存在。`_exit(0)` 退出。

**五层检查**:

| 层级 | 目标寄存器/文件 | 检查内容 | 可伪造性 |
|:-----|:----------------|:---------|:---------|
| L1 | `MIDR_EL1` (MRS 指令) | CPU Part ∈ {0xD81, 0xD80} | 不可伪造 (硬件) |
| L2 | `ID_AA64ISAR0_EL1` (MRS 指令) | SM3 ≠ 0 && SM4 ≠ 0 | 不可伪造 (硬件) |
| L3a | `/system/build.prop` | `ro.build.version.release=10` | 中等 (需 root) |
| L3b | `/proc/device-tree/compatible` | 包含 "cix" 和 "sky1" | 高可信 (DTB) |
| L4 | 网络接口 MAC OUI | salted FNV-1a hash 匹配 | 中等 (root 可改 MAC，但 OUI 不在二进制中) |

**配置项** (version.txt):
```
PLATFORM_BINDING=1
PLATFORM_CPU_PARTS=0xD81,0xD80
PLATFORM_REQUIRE_SM3SM4=1
PLATFORM_DT_KEYWORDS=cix,sky1
PLATFORM_ANDROID_RELEASE=10
PLATFORM_ISAR0=0x0021111110212120
PLATFORM_MAC_OUIS=f0:d7:af
```

**实现** (`loader.c`):
-   `platform_sigill_exit()` — SIGILL 处理器，MRS trap → `_exit(0)`
-   `platform_check_cpu_part()` — 解析 `PLATFORM_CPU_PARTS` 字符串，比对 MIDR_EL1 Part 字段 [15:4]
-   `platform_check_isa_features()` — 读取 ID_AA64ISAR0_EL1，检查 SM3[31:28]/SM4[35:32] 位域
-   `platform_check_android_release()` — 直接读 `/system/build.prop`，匹配版本字符串（`_PB_XSTR()` 宏将数字 define 转为字符串）
-   `platform_check_dt_compatible()` — 读 `/proc/device-tree/compatible`（null 分隔符→空格），校验平台关键字
-   `platform_check_mac_oui()` — MAC OUI 三路获取 + salted FNV-1a hash 比对：
    -   `fnv1a_salted(oui_bytes, 3, MAC_OUI_SALT)` 计算设备 OUI hash
    -   `mac_oui_scan_sysfs()` — Method A: getdents64 遍历 `/sys/class/net/`，读各接口 `address` 文件
    -   `mac_oui_probe_explicit()` — Method B: 依次尝试 wlan0/eth0/rmnet0/rmnet1
    -   `mac_oui_scan_netlink()` — Method C: netlink RTM_GETLINK + IFLA_ADDRESS 解析
    -   仅当 `MAC_OUI_HASH_COUNT > 0` 时编译（`#ifdef MAC_OUI_HASH_COUNT`）
-   `platform_check()` — 主函数：安装 SIGILL handler → L1 → L2 → 恢复 handler → L3a → L3b → L4(MAC OUI)

**调用时机**: `main()` 中 `antidebug_early_check()` 之后、`exec_core()` 之前（仅 binfmt 执行路径；`-V`/`-h` 不触发）。

#### 密钥融合 — 密码学绑定

**为什么？** 四层门卫检查是纯控制流（`if (!check()) _exit(0)`），攻击者只需 NOP 一条 `BL platform_check` 即可绕过。密钥融合将硬件特征融入 AES 解密密钥，失败点从控制流转移到数据域——**不存在可被 NOP 的指令**。

**机制**:
-   构建时: `encrypt_core.py` 读取 `PLATFORM_ISAR0`（目标硬件的 `ID_AA64ISAR0_EL1` 值），计算 `bound_key = base_key ⊕ {ISAR0_LE, ISAR0_LE}` 和 `bound_iv = base_iv ⊕ {~ISAR0_LE, ~ISAR0_LE}`，用 bound_key/iv 加密 core。
-   `enc_key.h` 嵌入 **base_key** 分片（未融合的），不是 bound_key。
-   运行时: `reconstruct_key()` 重建 base_key，然后 `MRS ID_AA64ISAR0_EL1` 读取实际硬件值，XOR 得到 bound_key。
-   正确硬件: ISAR0 匹配 → bound_key 正确 → AES 解密成功 → fexecve 运行。
-   错误硬件: ISAR0 不同 → bound_key 错误 → AES 产出垃圾 → fexecve 失败。

**与门卫检查的关系**: 四层门卫检查作为“早期快速失败”保留，提供友好的静默退出行为。即使攻击者 NOP 掉门卫检查，密钥融合仍然阻止解密。

**安全评估**:
| 维度 | 仅门卫检查 | + 密钥融合 | + MAC OUI |
|:-----|:-----------|:---------|:---------|
| 绕过难度 | 1 条 NOP (~5 min) | 需理解密钥派生 + 获取 ISAR0 | 需额外识别 FNV-1a hash 算法 + 确定目标 OUI |
| 失败域 | 控制流 (可 NOP) | 数据域 (不可 NOP) | 控制流 (gate-check, 可 NOP) |
| 攻击者需要 | IDA + 1 个 patch | 逆向密钥派生 + 确定目标 ISAR0 + 重新加密 | IDA + NOP L4 check 或伪造 MAC |

### 0.11 版本 (Split Model + Watchdog + Panic Fix)
*   **状态**: **Verified (2026-02-17)**
*   **基础**: 继承 0.10 所有特性。

#### 架构变更 — Split 模型

**为什么要拆分？** 原始 `tango_translator` 是 Rust 编译的单体二进制，我们无法在其内部增加 C 逻辑（如 FD 清理、watchdog 守护进程）。拆成 loader + core 后：
-   **loader** (`/system/bin/tango_translator`, ~727KB, 静态链接 aarch64 C 程序) — 我们完全控制的代码，可自由添加启动时逻辑。
-   **core** (`/system/bin/libarm96`, ~1.1MB, 经 patch 的原始翻译器二进制) — 纯二进制 patch，不改架构。V0.17 起由 `tango_core` 改名为 `libarm96`，消除与 Tango 的名称关联。
-   binfmt_misc 匹配 ARM32 ELF → 触发 loader → loader 做预处理 → exec core。
-   `binfmt_misc flags` 以 `POC` 为默认/正确口径（必须包含 `P`）。`O` = open-binary 模式（传递被执行文件的 FD 而非路径）；`C` = credentials（保持 setuid 等能力）；`P` = preserve-argv0（在 argv 前面插入原始程序名）。V0.11 不支持 P（会导致 zygote 崩溃），V0.12 通过 argv 重构解决了这个问题。

#### 新增特性详解

##### 1. version.txt 配置中心
**文件**: `hardening/src/version.txt`
**为什么？** 之前所有授权参数（expire epoch、licensee、interval 等）散落在 make.sh 和 main.py 的硬编码中，改一个参数要改多处，容易遗漏不同步。
**机制**:
-   `version.txt` 是纯 `KEY=VALUE` 格式，支持 `#` 注释。
-   `make.sh`（实际是 `shell/make.sh`，根目录 `make.sh` 是符号链接）读取 version.txt，用 `tr -d '\r'` 去除 Windows 换行符（因为 Y: 盘映射到 Linux 共享文件夹，Windows 编辑器可能引入 CR），生成：
    -   `gen/version.h` — C 宏定义，供 loader.c `#include`
    -   `gen/version.py` — Python 常量，供 main.py `from gen.version import ...`
-   环境变量 `TANGO_LICENSE_EXPIRE_EPOCH` / `TANGO_LICENSE_EXPIRE_IN_SEC` / `TANGO_LICENSE_CHECK_INTERVAL` 可覆盖 version.txt 默认值。`EXPIRE_IN_SEC` 与 `EXPIRE_EPOCH` 互斥（同时传入时 make.sh 会报错）。
-   `make_5min.sh` 就是设 `EXPIRE_IN_SEC=300` + `CHECK_INTERVAL=180` 后 exec make.sh。

**注意**: `make.sh` 是从根目录到 `shell/make.sh` 的符号链接。脚本内使用 `REPO_ROOT="$(cd "$SCRIPT_DIR/.." && pwd)"` 从 `shell/` 向上一级定位仓库根。如果把 make.sh 改为普通文件，这个路径计算就会出错。

##### 2. FD 白名单合规 (loader.c)
**为什么？** binfmt_misc 的 `O` flag 会额外传入一个指向被执行 ARM32 ELF 的 open FD（如 `/dev/fd/3` → `/system/bin/app_process32`）。Android zygote 有严格的 FD 白名单检查（`frameworks/base` 中的 `FileDescriptorInfo`），非白名单 FD 会触发 `JNI FatalError: Not whitelisted`，进程 SIGABRT。

**实现** (`loader.c` — V0.12 简化版):
```c
// V0.12: binfmt fd 不再需要特殊处理。
//   loader 通过 --exe 传路径给 libarm96（core 自行 open），
//   因此 binfmt fd 与其他非必要 FD 一样被 FD_CLOEXEC。
//
// sanitize_fds():
//   遍历 /proc/self/fd/ 目录，对所有 fd > 2 的文件描述符：
//   1) 检查是否是 ANDROID_SOCKET_* 环境变量指向的 FD — 如果是则保留（zygote 监听端口）。
//   2) 否则设置 FD_CLOEXEC — exec 后自动关闭，不会泄漏到 core 进程。
//   这样既满足了 zygote 的白名单要求，又不会丢失 zygote socket。
//
// (V0.11 的 resolve_and_close_binfmt_fd() 已移除 — 不再需要显式 close+readlink)
```

##### 3. -v / -h 合规 (loader.c)
**为什么？** 当前必过用例要求：`-v` 只输出 1 行版本号，`-i` 输出 2 行授权信息，`-h` 和无参数必须静默。这些逻辑在 loader 中处理，不传递给 core。
```
arm96 -v  →  "V0.45.0\n"
arm96 -i  →  "for Cheersu Technology Co\nto 20260701-19:00:00\n"
arm96 -h  →  (空, exit 0)
arm96     →  (空, exit 0)
arm96 <arg1> ...  →  exec libarm96 --exe <orig_prog> -- <arg1> ...
```
`-i` 中的日期字符串由 `gen/version.h` 的 `EXPIRE_DATE_STR` 宏提供（make.sh 在生成 version.h 时用 `date` 命令格式化 epoch）。

##### 3.1 反调试策略 (loader.c)
反调试逻辑仅在 **loader** 中实现（split 模型），并通过 `hardening/src/version.txt` 生成到 `gen/version.h`。

**触发面分层**：
1.  `arm96 -v`（用户态探测）
    *   先执行 **quick-exit**：只读取 `TracerPid`（可选 task 级 TracerPid），若被 trace/strace，则直接 `_exit(0)`。
    *   目的：针对 `strace arm96 -v`，直接静默退出，不跑后续任何逻辑/输出。
2.  binfmt_misc 真正执行链路（系统态执行）
    *   进入 argv 重构/exec core 之前执行启动早期检查：
        *   `prctl(PR_SET_DUMPABLE, 0)`（best-effort）
        *   `TracerPid`（可选 task 级）
        *   可选 `prctl(PR_SET_PTRACER, 0)`（best-effort，失败不影响运行）
        *   可选 `ptrace(PTRACE_TRACEME)`（默认关闭；需验证误杀风险）
    *   命中后静默退出（不输出任何信息）。

**关键开关（version.txt）**：
- `ANTI_DEBUG_ENABLE`：总开关。
- `ANTI_DEBUG_TRACERPID` / `ANTI_DEBUG_TRACERPID_TASKS`：TracerPid 检测强度（task 级更强）。
- `ANTI_DEBUG_PR_SET_PTRACER`：best-effort 限制未来 ptrace attach。
- `ANTI_DEBUG_PTRACE_TRACEME`：可选激进策略（默认建议关闭）。
- `ANTI_DEBUG_ACTION_SIGKILL`：仅影响 binfmt 早期检查命中时的动作（`_exit(0)` vs `SIGKILL`）。
    *   `-v` quick-exit 固定 `_exit(0)`，不使用 SIGKILL（避免额外可观测行为）。
- `ANTI_DEBUG_WATCHDOG*`：运行期反调试巡检（扫 app UID 进程并 kill traced）作为增强手段；实现可能处于禁用/不启用状态，需以当前 loader.c 为准。

##### 4. Watchdog 守护进程 (loader.c)
**为什么？** 原始 core 有内置的 kill timer（`sub_444A0` 中设置 POSIX timer），但我们的 sig bypass 补丁 (0x441E0/0x441F8 NOP + force jump) 同时禁用了这个 timer 的设置逻辑。因此需要一个外部机制来实现过期自杀。

**为什么用 double-fork？** 第一版用简单 `fork()` 创建子进程，但该子进程会被 zygote 的进程管理杀掉。double-fork 模式：parent → child (立即 exit) → grandchild (被 init 收养)。grandchild 的 ppid=1，独立于 zygote 进程树，不会被杀。

**实现**:
```c
// start_kill_watchdog():
//   1) fork() → child
//   2) child: fork() → grandchild, 然后 _exit(0)
//   3) parent: waitpid(child) 回收中间进程
//   4) grandchild: setsid() 脱离终端，调用 watchdog_loop()

// watchdog_loop(zygote_pid, interval, expire_epoch):
//   - 写 /data/local/tmp/tango_watchdog.log 记录启动状态（调试用）
//   - 计算距过期的秒数，sleep 到过期时刻
//   - 死循环: kill_children_of(zygote_pid); sleep(interval)

// kill_children_of(zygote_pid):
//   - 扫描 /proc/*/stat，解析第 4 字段 (ppid)
//   - 如果 ppid == zygote_pid 且该进程不是 watchdog 自身 → SIGKILL
//   - 这会杀掉所有由 zygote32 孵化的 32 位 app 进程
```

##### 5. [V12] 跳转表 Panic 修复 (main.py)
**为什么？** 这是导致"reboot 后连不上"的根本原因。sig bypass 补丁 (0x441E0/0x441F8) 只绕过了 `sub_44080` 内的**签名验证**逻辑，但 `sub_44080` 的**返回值**仍然会反映授权状态（0=无授权, 1=有效, 2=过期, 3=无效CPU, 4=无效授权）。调用链：
```
sub_57A24 (翻译器入口)
  → sub_44470 (授权分发)
      → sub_44080 (解密+验签+判定) → 返回 result code
      → 读跳转表 0xE55F0[result] → BR X10 (跳到对应处理分支)
```
跳转表位于 `.rodata` 段 0xE55F0，5 个 int32 相对偏移（基地址 0x44490）：
```
[0] = 0x10  → 0x444A0 (timer setup, 安全路径: 设置 14400s POSIX timer 后继续执行)
[1] = 0x30  → 0x444C0 (valid license, 正常执行)
[2] = 0x184 → 0x44614 (expired → panic "License expired on ...")  ← 崩溃源
[3] = 0x1D0 → 0x44660 (invalid CPU → panic)
[4] = 0x1E8 → 0x44678 (invalid license → panic)
```
过期时 `sub_44080` 返回 2 → 跳转表跳到 0x44614 → 构造 "License expired on 2026-..." 字符串 → `sub_1B440` (Rust panic!) → 进程终止。zygote_secondary 死亡 → system_server 无法连接 32 位 zygote socket → `java.io.IOException: No such file or directory`。

**修复**: 将 entry 2/3/4 全部改写为 0x10（与 entry 0 相同），过期时走 timer setup 路径而非 panic。timer setup 会设定一个 POSIX timer，但因为我们已经有独立的 watchdog 守护进程负责杀进程，所以这个 timer 实际上是无害的"空转"。
```python
# main.py 中的补丁代码:
safe_offset = struct.pack("<I", 0x10)
for idx in (2, 3, 4):
    f.seek(0xE55F0 + idx * 4)
    f.write(safe_offset)
```

##### 6. [V12] Kill-interval 常量补丁 (main.py)
**为什么？** 原始 core 在 timer setup 路径中硬编码了 `v3 + 14400`（即当前时间 + 14400 秒后触发 timer）。偏移 0xE1F02 处存储的 4 字节 LE uint32 就是这个 14400 常量。覆写为 `LICENSE_CHECK_INTERVAL` 使其与 watchdog 的 interval 一致（虽然 timer 路径现在是"空转"，但保持一致性以防未来需要）。
```python
f.seek(0xE1F02)
f.write(struct.pack("<I", LICENSE_CHECK_INTERVAL))
```

#### 构建与打包改进详解

##### package.sh — 加入 core (libarm96)
**为什么？** 之前 package.sh 只打包 `build/out/bin/tango_translator`（loader），缺少 core。用户部署后只有 loader 没有 core，loader exec core 时找不到文件，但不会报明显错误——binfmt_misc 直接回退到执行原始未 patch 的 tango_translator.orig，其中没有任何补丁，授权过期直接 panic。
```bash
# 新增逻辑:
SRC_CORE="$(dirname "$SRC_BIN")/libarm96"
if [[ -f "$SRC_CORE" ]]; then
    cp -f "$SRC_CORE" "$CACHE_DIR/$NAME/bin/libarm96"
fi
```

##### res/install.sh — 部署 core (libarm96) + stock 备份
**为什么？**
1.  `release_new_version()` 原来只拷贝 `$TMPDIR/bin/tango_translator`。即使 tgz 里有 core，也不会被安装到 `/system/bin/`。新增对 `$TMPDIR/bin/libarm96` 的检测和拷贝。
2.  `backup_original_version()` 增加 `.stock` 备份。因为反复安装会覆盖 `.orig`，但 `.stock` 只在首次安装时创建，保留"真正的原始版本"用于回滚。
3.  `remove_old_version()` 增加 `rm -f /system/bin/libarm96`（及旧名 `tango_core`）。卸载时如果不清理 core，下次安装新版时可能残留旧 core。

#### 关键偏移量速查表
| 偏移 | 大小 | 用途 | 所在段 | 补丁版本 |
| :--- | :--- | :--- | :--- | :--- |
| 0x210 | 20B | BuildID (SHA1, 动态生成) | `.note.gnu.build-id` | V8 |
| 0x10FB60+264 | 8B LE | License timestamp (UTC epoch) | `.data` | V8 |
| 0x441E0 | 4B | Sig bypass: `TBNZ W0,#0 → NOP` (1F2003D5) | `.text` | V8 |
| 0x441F8 | 4B | Sig bypass: `CBZ X8 → B` (09000014, 强制跳转到 valid 路径) | `.text` | V8 |
| 0xE1F02 | 4B LE | Kill interval 常量 (原值 14400 = 0x3840) | `.rodata` | V11 |
| 0xE55F0 | 20B | 授权跳转表 (5×int32, entry 2/3/4 改为 0x10) | `.rodata` | V11 |
| 0xE57BA | ~30B | Fake error 1 ("Checksum Mismatch.........") | `.rodata` | V9 |
| 0xE57FF | ~15B | Fake error 2 ("Memory Error   ") | `.rodata` | V9 |
| 0xE977E | 6B | Version string (e.g. "V0.12 ") | `.rodata` | V8 |
| 0xE97A6 | 38B | Copyright string | `.rodata` | V8 |
| 0xE97D6 | 37B | Kernel label (伪装) | `.rodata` | V8 |
| 0xEAF40 | 6B | Version string 副本 | `.rodata` | V8 |

#### 已知踩坑记录

1.  **binfmt_misc `P` flag 导致 zygote 崩溃 (V0.11 已修复于 V0.12)**: `P` = preserve-argv[0]，使 argv 布局变为 `[orig_program, interpreter, /dev/fd/N, args...]`。V0.11 直传 argv 给 core 会导致 `ClassNotFoundException: -Xzygote`（根因：`execv()` 丢失 `AT_EXECFD` → core 进入 normal arg parsing → 把 `-Xzygote` 当成类名）。V0.12 通过构造 `--exe <orig_prog> -- <args...>` 绕过此问题。
2.  **make.sh 是符号链接**: `vendor/hello/arm96/make.sh → shell/make.sh`。`REPO_ROOT` 用 `$(cd "$SCRIPT_DIR/.." && pwd)` 从 `shell/` 上溯。如果改为普通文件，路径计算会错。
3.  **Windows CR 在 version.txt**: Linux shell 的 `declare` 命令无法处理 `KEY=VALUE\r`。make.sh 中用 `tr -d '\r'` 去除。
4.  **FD 泄漏导致 "Not whitelisted"**: binfmt_misc 的 `O` flag 传入的 FD 必须在 exec core 前关闭，否则 zygote 的 FD 白名单检查会 abort。
5.  **watchdog 简单 fork 会被杀**: zygote 进程管理会清理不在预期中的子进程。必须用 double-fork 使 watchdog reparent 到 init(PID 1)。
6.  **sig bypass ≠ 跳转表 bypass**: 0x441E0/0x441F8 只绕过签名校验函数内部的两个分支，但 `sub_44080` 的返回值仍为 2（expired），跳转表仍会跳到 panic。两者是独立的代码路径，必须同时修补。
7.  **容器 pid_max 过大导致 32 位 app 闪退 (V0.23 发现)**: 目标容器环境 `kernel.pid_max=4194304`（默认值），32 位 bionic libc 的 `pthread_mutex_t` 结构体仅用 16 位存储 owner TID（`__pthread_mutex_t.__private[0]` 中的 bits [31:16]），PID > 65535 时触发运行时 abort：`"Limited by the size of pthread_mutex_t, 32 bit bionic libc only accepts pid <= 65535, but current pid is XXXXXX"`。此问题在 zygote_secondary 本身和其 fork 出的所有 32 位 APP 进程中均可触发。**修复方法**：在 loader 的 is_zygote 路径中、exec core 之前写入 `/proc/sys/kernel/pid_max=65536`（需 root 权限，zygote 以 root 运行满足）。临时 workaround：`adb shell "echo 65536 > /proc/sys/kernel/pid_max"` + 重启 zygote。

#### 验证结果 (2026-02-20, V0.12)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.12" / "for Cheersu Technology Co" / "to 20260401-19:00:00" |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 有效授权 reboot | ✅ | zygote/system_server/webview_zygote 全正常，crash log 零条 |
| TC5 | 过期自杀 | ✅ | 5 分钟版: ~180s 首杀，~360s 二杀 |
| TC6 | 过期 reboot | ✅ | crash log 零条，zygote_secondary socket 可达，webview_zygote 正在翻译 app_process32 |
| binfmt | flags = POC | ✅ | 以 POC 为准 |

### 0.16 版本 (setitimer 替换 watchdog + 孤儿清理)
*   **状态**: **Verified (2026-02-23)**
*   **基础**: 继承 0.12 所有特性。

#### 问题背景 — 外部 watchdog 的缺陷

V0.11 引入的 double-fork watchdog daemon 存在以下问题：
-   **进程暴露**: `ps -ef` 可直接观测到 `tango_watchdog` 进程。
-   **单点故障**: watchdog 被 kill 后授权策略彻底失效。
-   **复杂度**: fork/setsid/double-fork 逻辑 ~120 行，维护成本高。

#### 新方案 — setitimer(ITIMER_REAL) + 孤儿清理

**核心原理**:
1.  **布防**: loader 在 `execv(libarm96)` 前调用 `setitimer(ITIMER_REAL)`，设置内核定时器。
2.  **存活**: POSIX 保证 `setitimer` 跨 `execv` 存活。libarm96 继承了定时器但完全不感知。
3.  **自杀**: 定时器到期 → 内核发 SIGALRM → `SIG_DFL = terminate` → zygote 进程终止。
4.  **重启**: Android init 检测 zygote 退出 → 自动重启 → loader 重新执行 → 重新布防 → 循环。

**孤儿清理** (`kill_orphaned_app_processes`):
-   SIGALRM 只杀 zygote 本身，fork 出的 32 位 APP 被 reparent 到 init (PPID=1)。
-   每次 zygote 重启时，若已过期，loader 扫描 `/proc`，SIGKILL 所有 PPID=1 且 UID ∈ [10000, 19999] 的孤儿进程。
-   64 位 APP 不受影响（其父进程是 zygote64，PPID≠1）。

**定时器参数**:
-   未过期: `it_value = EXPIRE_EPOCH - now`（精确到期时刻触发）
-   已过期: `it_value = interval`（避免紧密重启循环）
-   `it_interval = LICENSE_CHECK_INTERVAL`（周期重复）

**对比**:
| 维度 | V0.11 watchdog | V0.16 setitimer |
|------|------|------|
| 额外进程 | ❌ double-fork daemon (ps 可见) | ✅ 零 |
| 抗强杀 | ❌ kill PID 即可绕过 | ✅ 内核定时器无法从用户态单独杀 |
| 代码量 | ~120 行 | ~60 行 (arm_expiry_timer + kill_orphaned) |

**代码变更** (`loader.c`):
-   删除: `kill_children_of()`, `watchdog_loop()`, `start_kill_watchdog()` (~120 LOC)
-   新增: `arm_expiry_timer()` (~15 LOC), `kill_orphaned_app_processes()` (~45 LOC)
-   仅对 `--zygote` 进程布防，普通 32 位 APP 不受影响。

#### 验证结果 (2026-02-23, V0.16)
| 用例 | 测试内容 | 结果 | 备注 |
| :--- | :--- | :--- | :--- |
| TC1 | `-V` 输出 3 行 | ✅ | "V0.16" / "for Cheersu Technology Co" / "to 20260401-11:00:00" |
| TC2 | `-h` / 无参静默 | ✅ | 输出 0 字节 |
| TC3 | 无效参数静默 | ✅ | 输出 0 字节 |
| TC4 | 授权期内 zygote 正常 | ✅ | zygote PID 正常，零 tango_watchdog 进程 |
| TC5 | 32 位应用运行 | ✅ | com.dragon.read 正常启动 |
| TC6 | 过期自杀 (5min) | ✅ | SIGALRM 杀 zygote + 孤儿清理杀 APP，180s 精确间隔 |
| TC7 | 过期重启 | ✅ | 系统正常启动 |
| 零进程 | ps 无额外进程 | ✅ | `grep tango_watchdog` 结果为空 |

### 0.12 版本 (POC binfmt 兼容 + argv 重构)
*   **状态**: **Verified (2026-02-20)**
*   **基础**: 继承 0.11 所有特性。

#### 问题背景 — AT_EXECFD 丢失

在容器环境中，binfmt_misc 使用 `POC` flags（带 `P` = preserve-argv0）。V0.11 的 loader 直接将 binfmt argv 原样传给 core，会导致 zygote_secondary 崩溃：

```
java.lang.ClassNotFoundException: -Xzygote
```

**根因分析 (IDA 逆向)**:
1.  `execv()` 替换进程映像后，内核注入的 `AT_EXECFD` 辅助向量条目丢失。
2.  core (原始 Rust 二进制) 在 `sub_8A3C0` 中读取 `AT_EXECFD` → 存入全局变量 `dword_222318`。
3.  主函数 `sub_6FB68` 在 0x6FC48 处：`CBZ W24, loc_6FD08` — 如果 `AT_EXECFD` 为 0，跳到 **normal arg parsing** 分支。
4.  Normal mode 将 argv 作为翻译器自身的命令行参数解析 → 遇到 `-Xzygote` 尝试当作类名加载 → `ClassNotFoundException`。

**IDA 关键发现**:
| 地址 | 内容 | 说明 |
| :--- | :--- | :--- |
| 0x222318 | `dword_222318` | AT_EXECFD 全局存储 (在 `sub_8A3C0` 中设置) |
| 0x6FC48 | `CBZ W24, loc_6FD08` | AT_EXECFD == 0 时进入 normal arg parsing |
| 0x6FE40 | Case 3 of jump table | `--exe EXE` 参数解析入口 |
| 0x711BC | Case 45 of jump table | `--` (stop parsing) 参数解析入口 |
| 0xE9328 | String | "Missing executable path when invoked from binfmt_misc. Make sure the 'P' and 'O' flags are enabled" |

#### 修复方案 — argv 重构

不再将 binfmt argv 直接传递给 core，而是构造 **normal mode 兼容的 argv**：

```
libarm96 --exe <original_program_path> -- <original_args...>
```

其中：
*   `--exe`: 告知 core 待翻译的 ARM32 二进制路径（由 core 自行 open）
*   `--`: 停止 flag 解析，后续参数原样传给被翻译的程序

**loader.c argv 构建逻辑**:
```c
// 检测 binfmt 布局 (扫描 /dev/fd/N 位置)
//   POC: argv = [orig_name,  interpreter, /dev/fd/N, original_args...]  → devfd_idx=2

// 确定 orig_prog:
//   POC: argv[0] 就是原始程序名 (P flag 保留)

// 构造 new_argv:
//   [CORE_PATH, "--exe", orig_prog, "--",  original_args..., NULL]
execv(CORE_PATH, new_argv);
```

#### sanitize_fds 简化

binfmt fd 不再需要特殊保留（core 通过 `--exe` 路径自行打开文件），因此 `sanitize_fds()` 统一对所有非 stdio、非 `ANDROID_SOCKET_*` 的 FD 设置 `FD_CLOEXEC`。

#### package.sh 优化

移除 Python 依赖（删除 `version.py`），版本号直接由 `sed` 从 `version.txt` 解析。

### 0.10 版本 (Historical Candidate)
*   **状态**: **Baseline-only / 历史候选**
*   **基础**: 继承 0.9 所有特性 (混淆+加固)。
*   **新增特性**:
    1.  **滚动授权**: 授权期限动态设定为“构建时间 + 3周”，强制定期更新。
    2.  **动态指纹**: BuildID 基于版本与过期时间动态生成 SHA1，消除静态特征。
*   **说明**: 该版本属于历史候选；当前反调试策略以 split loader 的 TracerPid/PRCTL/可选 ptrace 为主（见 0.12+）。
*   **版本标识**: V1.10, Licensee: Terminator T-X。

> 备注：0.10 候选产物不依赖/注入 `stub.s/stub.bin`；当前主线已将该能力重新规划到 V0.65.0，目标是作为 `libarm96` 原始 ELF 注入 payload，而非 loader 侧逻辑。

### 0.10 反调试注入实验 (Anti-Debug Injection Experiment)
*   **状态**: **待开发 / 实验项**
*   **说明**: 该实验包含入口注入/ptrace 对抗等探索；**未纳入发布版本** (0.8–0.14)。当前发布版本优先采用 split loader 的反调试策略，不依赖入口注入 stub。
*   **注入技术**: Hook Entry Point -> Code Cave + Shellcode (`stub.bin`)。
*   **反调试逻辑**:
    1.  执行 `ptrace(PTRACE_TRACEME, 0, 0, 0)`。
    2.  检测返回值：若失败则 `exit_group(1)`。
    3.  若成功，恢复入口指令并跳转继续执行。
*   **验证要点**: 在设备上使用 `strace` 附加时，进程可能立即退出；需评估该行为是否可接受，以及是否影响系统启动稳定性。

### 0.9 版本 (Obfuscated Release)
*   **状态**: **已验证 / Verified (2026-02-11)**
*   **基础**: 继承 0.8 所有特性。
*   **混淆策略**: 替换敏感报错信息。
*   **特性**:
    1.  **伪造错误**: 遇到 License 问题时，报错 "Memory Error"。
    2.  **版本迭代**: V0.9, Licensee: Terminator T-1000。


### 0.8 版本 (Baseline Release)
*   **基础**: 灵活可选。通常基于 **V5 (永久版)** 进行加固，也可基于 V7 (测试版) 用于验证。
*   **可变性**: 加固仅针对字符串和报错逻辑，**不锁定** 授权期限偏移量 (`0x10fc68`)。加固后仍可修改到期时间。
*   **加固项**:
    1.  **字符串抹除**: 覆盖 "License expired..." 等关键报错。
    2.  **版本号伪装**: 修改版本字符串，混淆视听。

### 探测数据 (Reconnaissance Data) - 2026-02-10
通过对 `tango_translator.cheersu` 的二进制扫描，确认以下关键信息偏移量：

1.  **敏感字符串 (Panic Messages)**:
    *   `"evaluation period exceeded"` @Offset **0xE57BA**
    *   `"License expired"` @Offset **0xE57FF**
    *   *策略*: 将这些区域用 `0x00` (NULL) 覆盖，使 Panic 输出为空白。

2.  **授权数据区**:
    *   License Start: `0x10FB60`
    *   Timestamp Field: `0x10FC68` (License Start + 264)
    *   *策略*: 加固脚本**必须避开**此区域，或提供 API 修改此区域，确保授权期限的可控性。

3.  **代码段 (Code Patch)**:
    *   Signature Check 1: `0x441E0` (NOP)
    *   Signature Check 2: `0x441F8` (Force Jump)
    *   *策略*: V8 脚本将集成这些代码修改，形成“过签+加固”的一体化方案。

### 操作指南
当前仓库推荐通过 `vendor/hello/arm96` 下的 `./make.sh` 一键产出候选二进制；如需单独验证 patch，可直接调用 hardening 入口。

```bash
# 一键构建（推荐）
./make.sh

# 直接运行 hardening 入口（用于单独验证 patch 行为）
python3 hardening/src/main.py --input prebuilts/tango_translator --output build/out/bin/tango_translator

# 如需切换到 3 个月授权版（cheersu），可显式指定输入：
# python3 hardening/src/main.py --input prebuilts/tango_translator.cheersu --output build/out/bin/tango_translator
```
