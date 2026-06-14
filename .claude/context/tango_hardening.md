# Tango 二进制加固工程 (Binary Hardening)

> **模块类型**: 二进制安全防护 / 逆向对抗 (Post-Link Optimization)
> **核心目标**: 在无源码条件下，对 ISO/ELF 二进制文件进行“黑盒”安全加固，消除逆向路标，提升分析与篡改成本。

---

## 1. 背景与动机

通过对 Tango 原始二进制的分析，我们发现其安全性处于 **C-** 水平。主要问题在于：
*   **明文字符串**: `.rodata` 段包含大量清晰的报错信息（如 "License expired"），直接暴露了核心逻辑的位置。
*   **详细的 Panic 信息**: Rust 的 Panic handler 会打印精确的文件名和行号，相当于直接送给攻击者调试符号。
*   **缺乏对抗**: 没有反调试、反篡改机制。

本模块的目标是在不拥有源代码的前提下，通过对 ELF 二进制文件的直接修改（Patch/Instrumentation），实现安全加固。这既可以用于保护正版二进不受篡改，也可以用于保护我们产生的“修改版”不被轻易分析（黑吃黑）。

## 2. 加固策略 (无源码模式)

我们采用 **纵深防御 (Defense in Depth)** 策略，分阶段实施：

### 2.1 第一阶段：抹除路标 (Anti-Reversing Baseline)
此阶段目标是让 IDA/Strings 等工具失效，无法通过字符串定位逻辑。
*   **字符串擦除 (String Wiping)**: 定位并覆盖敏感字符串（用 `0x00` 填充）。
    *   目标字符串: `"License expired"`, `"Tango binary translator evaluation period exceeded"`, `"Version:"` 等。
*   **符号剥离 (Symbol Stripping)**: 虽然 Release 版已 Strip，但仍需检查是否有残留的调试段。

### 2.2 第二阶段：行为混淆 (Behavior Obfuscation)
此阶段目标是让动态调试和日志分析失效。
*   **静默崩溃 (Silent Panic)**: 修改 Panic 处理函数的入口，使其不再输出 Logcat，而是直接调用 `sys_exit` 或触发 `SIGKILL`。
*   **虚假报错**: 将报错信息修改为误导性内容（例如“Memory Error”掩盖“License Error”）。

### 2.3 第三阶段：环境对抗 (Environment Checks)
利用 ELF 特性注入代码。
*   **反调试 (Anti-Debug)**:
    *   当前实现不依赖 `DT_INIT_ARRAY` 注入；反调试逻辑由 split 模型中的 **loader** (`/system/bin/arm96`, C 静态链接) 负责执行。
    *   目标：
        *   对 `strace/ptrace` 的 **探测/附加**（尤其是 `strace arm96 -v`）做“静默早退”。
        *   对 binfmt_misc 真正执行链路做“启动早期检查”，命中则直接退出，避免继续进入 core。
    *   机制（详见 `vendor/hello/arm96/hardening/src/loader.c` + `hardening/src/version.txt` 配置）：
        *   `-v` quick-exit：若被 trace，则直接 `_exit(0)`，不会输出版本信息，也不会跑后续逻辑。
        *   binfmt 早期检查：`PR_SET_DUMPABLE=0`（best-effort）+ `TracerPid`（可选 task 级）+ 可选 `PR_SET_PTRACER`；命中按策略退出。
        *   `ptrace(PTRACE_TRACEME)`：提供开关（默认关闭），需单独验证 SELinux/seccomp 兼容性与误杀风险。
*   **加壳 (Packing)**:
    *   使用 UPX (需定制 Magic 以防通用脱壳机) 对 `.text` 段进行压缩/加密。

## 3. 实施与迭代计划 (Implementation Roadmap)

### 📂 脚本与资源位置
当前加固入口已统一到 `vendor/hello/arm96` 的 hardening 模块（历史的 `hardening/patchs/` 目录已删除）：
*   **一键入口**: `android10/vendor/hello/arm96/make.sh`
*   **加固入口脚本**: `android10/vendor/hello/arm96/hardening/src/main.py`
*   **加密脚本**: `android10/vendor/hello/arm96/hardening/src/encrypt_core.py`
*   **AES-128 解密头**: `android10/vendor/hello/arm96/hardening/src/aes128.h`
*   **SHA-256 哈希头**: `android10/vendor/hello/arm96/hardening/src/sha256.h`
*   **Shellcode**: `android10/vendor/hello/arm96/hardening/src/stub.s`（V0.65.0 起作为 `libarm96` 原始二进制注入 payload）
*   **编译工具**: `android10/vendor/hello/arm96/hardening/compile_stub.sh`（V0.65.0 起输出 `res/stub.bin` + `res/stub.meta` 并接入注入门禁）

### ✅ 已完成 (Implemented Features)
| 类别 | 项目 | 说明 | 状态 | 版本 |
| :--- | :--- | :--- | :--- | :--- |
| **基础** | 授权/签名 | 永久授权 + 签名绕过 | **Done** | 0.8 |
| **抹除** | 静态特征 | 敏感字符串擦除 (License/Panic) | **Done** | 0.8 |
| **混淆** | 身份伪装 | 定制 Version/Copyright/BuildID | **Done** | 0.8 |
| **混淆** | 行为混淆 | 虚假报错 (Fake "Memory Error") | **Done** | 0.9 |
| **架构** | Split 模型 | loader (tango_translator) + core (libarm96) 分体部署 | **Done** | 0.11 |
| **架构** | version.txt 配置 | 集中管理授权参数，make.sh 生成 gen/version.h + gen/version.py | **Done** | 0.11 |
| **安全** | FD 白名单合规 | 非必要 FD 统一设 CLOEXEC，保留 ANDROID_SOCKET_*；binfmt fd 不再特殊处理 | **Done** | 0.11→0.12 |
| **安全** | 反调试 (Anti-Debug) | loader 分层反调试：`-v` quick-exit(TracerPid) + binfmt 早期检查(PR_SET_DUMPABLE/TracerPid/可选 TRACEME)，命中静默退出 | **Done** | 0.12→0.14 |
| **安全** | 跳转表 Panic 修复 | 0xE55F0 处 entry 2/3/4 重定向到定时器路径，防止授权过期 panic | **Done** | 0.11 |
| **功能** | ~~Watchdog + 过期熔断~~ | ~~double-fork daemon~~ — **已被 V0.16 setitimer 方案替代** | **Superseded** | 0.11→0.16 |
| **功能** | setitimer 过期自杀 | ITIMER_REAL 跨 execv 存活，SIGALRM 默认终止 zygote；init 自动重启；零额外进程 | **Done** | 0.16 |
| **功能** | 孤儿进程清理 | SIGALRM 杀 zygote 后，fork 的 32 位 APP 被 reparent 到 init(PPID=1)；loader 重启时扫 /proc SIGKILL PPID=1+UID 10000-19999 | **Done** | 0.16 |
| **功能** | CLI 合规 | `-v` 输出严格 1 行版本号，`-i` 输出严格 2 行授权信息，`-h` 和无参静默 | **Done** | 0.33 |
| **兼容** | POC binfmt 兼容 | loader 适配 POC argv 布局，构造 `--exe` + `--` argv 绕过 AT_EXECFD 丢失 | **Done** | 0.12 |
| **隐蔽** | Core 改名 | tango_core → libarm96，消除二进制与 Tango 的名称关联 | **Done** | 0.17 |
| **加壳** | UPX 压缩 + 抹 magic | UPX 管线已实现但 **默认关闭**：目标设备 CPU (CIX P1 CS8180) 的 BTI 与 UPX ARM64 runtime stub 不兼容 (SIGILL)；`ENABLE_UPX=1` 可手动开启 | **Disabled** | 0.18 |
| **加密** | ~~AES-128-CBC 加密 core~~ | ~~密钥=MD5(LICENSEE\|EXPIRE\|INTERVAL)，密文格式 `TGENC018`~~ — **已被 V0.22 AES-128-GCM (TGENC022) 替代** | **Superseded** | 0.18→0.22 |
| **加密** | memfd 内存执行 | loader memfd_create→读密文→AES 解密→fexecve；磁盘永不落明文 | **Done** | 0.18 |
| **加密** | 密钥分片 | gen/enc_key.h: 4×uint32 + XOR 因子，编译后散布在 loader .text | **Done** | 0.18 |
| **加密** | Loader 自保护 | loader 为静态链接 + stripped + OLLVM CFF/SUB/BCF 混淆（V0.21 起生效）；UPX 受 BTI 限制暂不可用；V0.41 扩展混淆覆盖至 21 个函数 | **Done** | 0.21→0.41 |
| **安全** | 平台绑定 (Platform Binding) | MRS MIDR_EL1/ID_AA64ISAR0_EL1 + device-tree + build.prop + MAC OUI 五层检查；非 CIX P1/Android 10 静默退出 | **Done** | 0.19 |
| **加密** | 平台密钥融合 (Key Fusion) | AES 密钥与 ID_AA64ISAR0_EL1 硬件寄存器 XOR 融合；非目标硬件产生错误密钥→解密失败，无控制流可 NOP | **Done** | 0.19 |
| **安全** | MAC OUI 绑定 | salted FNV-1a hash 比对，三路获取（sysfs scan / sysfs explicit / netlink RTM_GETLINK），gate-check 层；OUI 不入二进制 | **Done** | 0.19 |
| **加密** | MAC 密钥融合 (Prong 1) | triple_key = base_key ⊕ {ISAR0,ISAR0} ⊕ {MAC_HASH,MAC_HASH}；`reconstruct_key()` 在 ISAR0 XOR 后追加 MAC hash XOR；错误 MAC → 错误密钥 → 解密垃圾，无控制流可 NOP | **Done** | 0.20 |
| **安全** | 延迟投毒 scatter-corrupt (Prong 2) | MAC 不匹配时，AES 解密后 fexecve 前 LCG PRNG 散射覆写 32×8B；seed = PID^time^addr^__jit_reloc_base，不可复现 | **Done** | 0.20 |
| **混淆** | OLLVM 代码混淆 | clang-14 + `-fpass-plugin=libObfusPass.so`；`reconstruct_key`/`antidebug_early_check` 用 CFF+SUB；`platform_check`/`mac_get_oui_hash` 用 CFF；`platform_check_isa_features`/`antidebug_quick_exit_if_traced` 用 SUB；GCC fallback=noinline；`OBFUSCATE_LOADER=1` 开关 | **Done** | 0.21 |
| **安全** | 反调试增强 (Anti-Debug+) | `TRACERPID_TASKS=1`（线程级 TracerPid 扫描）+ `ACTION_SIGKILL=1`（binfmt 路径命中改用 SIGKILL）；运行期 watchdog 配置就绪（version.h `ANTI_DEBUG_WATCHDOG=1`），loader.c 实现已在 V0.23 P0 激活 | **Done** | 0.21→0.23 |
| **安全** | ~~运行期 watchdog~~ | ~~timer-based 巡检方案~~ — **Superseded by arm96server**：V0.23 为 PTRACE_SEIZE，V0.38 起改为 PID tracking + TracerPid 轮询 | **Superseded** | 0.23→0.38 |
| **安全** | /proc/self/maps Frida 检测 | `antidebug_scan_maps()` 扫描 frida/gadget/xposed/substrate/lsposed 特征；命中静默退出 | **Done** | 0.23 |
| **安全** | arm96server PID tracking + guardian cascade | 独立 daemon 追踪 32 位 app PID；发现 TracerPid>0 时杀 tracer + app；daemon 被杀→guardian 读取 `TRACKED_PID_FILE` 杀全部 tracked app；自保护 PTRACE_TRACEME | **Done** | 0.23→0.38 |
| **兼容** | pid_max 钳位 | loader is_zygote 路径自动写 `/proc/sys/kernel/pid_max=65536`，防止 32 位 bionic PID>65535 abort | **Done** | 0.24 |
| **加密** | EXPIRE_EPOCH 密钥融合 (Prong 3) | `quad_key = base_key ⊕ {ISAR0,ISAR0} ⊕ {MAC_HASH,MAC_HASH} ⊕ {EXPIRE_MASK,EXPIRE_MASK}`，`EXPIRE_MASK = ~EXPIRE_EPOCH & 0xFFFFFFFF`；`reconstruct_key()` 最后一步 XOR `get_expire_epoch()` 派生值；无条件启用（不依赖 PLATFORM_BINDING）；patch EXPIRE_EPOCH_LO/HI → 密钥变化 → AES 垃圾 → 无法运行 | **Done** | 0.22 |
| **安全** | 密钥清零修复 | `decrypt_core_to_memfd()` 中 `memset(key/iv)` 改为 `secure_zero()`；`secure_zero` 宏：API≥23/glibc≥2.25 映射到 `explicit_bzero`；否则用 `volatile uint8_t *` 循环填零，编译器不可 DCE | **Done** | 0.22 |
| **加密** | AES-CBC → AES-GCM | `aes128.h` 新增 `aes128_gcm_decrypt()`（纯 C：forward AES 块加密 + GF(2¹²⁸) GHASH + CTR 解密 + 常数时间 tag 比较）；密文格式 `TGENC022`：`magic(8B)｜orig_size(4B)｜nonce(12B)｜tag(16B)｜ct`；`encrypt_core.py` 切换 `AESGCM`，nonce 每次随机；错误密钥 → tag 验证失败 → `*pt` 自动清零 | **Done** | 0.22 |
| **隐蔽** | -V 字符串混淆 (P1-1) | `LICENSEE`/`EXPIRE_DATE_STR` 运行时解码+`secure_zero` 清零；`strings` 无输出；初版 XOR(0x5A)，V0.27 升级 per-string LCG，V0.28 升级 HMAC 种子推导 | **Done** | 0.22→0.28 |
| **隐蔽** | 辅助字符串运行时解密 (P0) | 共 14 条辅助字符串运行时栈上解密 + `secure_zero` 清零；初版 XOR(0x5A)，V0.27 升级 per-string LCG keystream，V0.28 升级 HMAC-SHA256 种子推导；`strings` 无输出 | **Done** | 0.25→0.28 |
| **混淆** | BCF 虚假控制流 (P1) | `ObfusPass.cpp` 新增 `ObfusBCFPass`：opaque predicate `(x*(x-1))&1==0`（volatile alloca 阻止优化器证明）+ `CloneBasicBlock` + trampoline 重定向；最多 8 个 BB/函数控制膨胀率；新增 `OBFUS_CFF_BCF_SUB` 属性并应用到 `reconstruct_key`/`antidebug_early_check` | **Done** | 0.25 |
| **安全** | 调试器进程检测 (P2) | `arm96server.c` 新增 `scan_and_kill_debuggers()`：扫描 `/proc/<pid>/cmdline` 检测 gdbserver/lldb-server/strace/frida-server/frida-inject/frida-gadget/frida，命中 `kill -9`；集成到 daemon scan 循环 | **Done** | 0.25 |
| **安全** | Loader .text 段哈希自校验 (P0) | 启动时 SHA-256 校验 executable PT_LOAD 段；两阶段构建（编译→提取.text→SHA-256→回填）；OBFUS_CFF_BCF_SUB 混淆校验函数；sentinel magic 初版 ASCII `TGHASH26`，V0.27 改为 SHA-256 派生非打印字节，V0.28 改为 HMAC 派生；V0.27 去除全零逃逸路径；V0.28 新增 2 个诱饵哨兵 | **Done** | 0.26→0.28 |
| **安全** | 完整性自校验去后门 (P0-1) | 移除 hash 全零跳过的逃逸路径；release 始终执行 `.text` SHA-256 自校验；仅保留编译期 `-DINTEGRITY_CHECK_DISABLED` 用于 DEV_BUILD | **Done** | 0.27 |
| **安全** | Sentinel 反定位 (P0-2) | `.data` 中 ASCII magic `TGHASH26` 替换为 `SHA-256("TANGO_INTEGRITY_V27")[:8]` 非打印 8 字节；`strings` 无法定位 sentinel | **Done** | 0.27 |
| **隐蔽** | Per-string LCG keystream (P1-2) | 去掉全局 XOR(0x5A)；每条字符串独立 seed `SHA-256("TANGO_LCG_V27_"+name)[:8]`；LCG Knuth PRNG keystream `(state>>33)&0xFF`；抗 `xortool` 单字节爆破 | **Done** | 0.27 |
| **隐蔽** | HMAC-SHA256 运行时种子推导 (P0) | LCG 种子不再以 `#define` 编译进二进制；运行时 `HMAC-SHA256(base_key, LE_u32(idx))[:8]` 从 AES 基础密钥推导；17 处 `_xor_decode` 迁移至 `_derive_seed(HKDF_IDX_*)` | **Done** | 0.28 |
| **安全** | 哨兵 magic HMAC 派生 (P0-2) | sentinel magic 改为 `HMAC-SHA256(base_key, idx=16)[:8]`，每次部署唯一，防跨部署 pattern 复用 | **Done** | 0.28 |
| **安全** | 诱饵哨兵 (P1) | 新增 `_decoy_sentinel_a[40]`（idx=17 + 伪随机 hash）和 `_decoy_sentinel_b[40]`（idx=18 + 全零）；与真实哨兵布局相同，干扰自动化 sentinel 搜索 | **Done** | 0.28 |
| **安全** | arm96server Guardian 自愈 (P0) | fork 伪装名 guardian 子进程，监控 arm96server 存活，被杀后 2 秒内自动 re-exec 复活 | **Done** | 0.29 |
| **安全** | .rc 自修复 (P1) | guardian 复活 arm96server 时检查 init .rc 是否存在，不存在则重建；防删 .rc 后重启失去 init 保护 | **Done** | 0.29 |
| **安全** | init service 注册 (P2) | 新增 `arm96server.rc`，class core, critical；init 级自动重启保障 | **Done** | 0.29 |
| **加密** | arm96server Token 密钥融合 (P0) | arm96server 尾部 16 字节 HMAC 令牌融入 AES 密钥（第 5 层 XOR）；删除二进制 → 解密失败，无控制流可 NOP | **Done** | 0.30 |
| **安全** | 空壳检测 + 延时下毒 (P0) | loader 启动后探测 arm96server 是否真正 PTRACE_TRACEME；空壳 → 缩短 setitimer 到 15-45s 随机间隔 | **Done** | 0.31 |
| **安全** | Guardian 双向互保 (P0) | arm96server 主循环监控 guardian 存活，死亡则重建；spawn_guardian 返回 pid_t | **Done** | 0.32 |
| **工程** | 多客户构建与最小镜像集成 | `make.sh --build <target>` profile 构建；`package.sh` 按 profile 打包；镜像侧仅保留 `tango_translator`，移除 `tangolic`/`tango_pretranslator` | **Done** | 0.33 |
| **工程** | Loader 二进制去 Tango 命名 | `/system/bin/arm96` 已取代 `/system/bin/tango_translator` 作为交付入口，并同步收敛 binfmt / 安装 / 验收 / 镜像路径 | **Done** | 0.34 |
| **工程** | 源码模块名去 Tango 命名 | 公共文档记账、构建兼容层和命名映射收敛完成（main/1.0 范围）；`src/` 目录级迁移留给 `main/2.0` | **Done** | 0.35 |
| **隐蔽** | Core 二进制去 Tango 品牌指纹 | `main.py` 新增 `_wipe_tango_identifiers()`：线性扫描 core `.rodata` 中所有 `tango`/`Tango`/`TANGO`，等长替换为 `sys__`/`Sys__`；白名单跳过 `/dev/tango32`、`.tango` 缓存扩展名、SELinux `tango_prop`、`tango uid` UID 日志；实测 128→11（117 处擦除，11 处功能性保留） | **Done** | 0.36 |
| **工程** | 迁移追溯文档与 prebuilt 对齐 | 新增 `arm96_migration_trace.md` 记录 legacy tango 移除 → arm96 新增的完整追溯；prebuilts 更新为 arm96/libarm96/arm96server 三件套；verify_mustpass*.sh 版本号同步至 V0.37 | **Done** | 0.37 |
| **工程** | 打包版本号格式统一 X.Y.Z | `package.sh` 版本号格式从 `VERSION[-TARGET]` 改为 `arm96-V0.Y.Z`；包名从 `tango_reimpl` 改为 `arm96` | **Done** | 0.37 |
| **安全** | arm96server PID 追踪替代 PTRACE_SEIZE | `scan_and_track()` 采用 PID 列表追踪 + TracerPid 轮询检测；解决 SEIZE 拦截信号导致 QQ libSecShell.so 等应用 ANR；扫描间隔从 5s 硬编码为 3s；guardian 通过 `TRACKED_PID_FILE` 实现 tracked app 级联清理 | **Done** | 0.38 |
| **隐蔽** | Core 混淆填充 (30KB) | `main.py` patch 后追加 30KB 混淆数据块（心经 XOR 分片 + 伪 AES-CBC 密文 + 伪 DER 公钥/签名 + 伪 License TLV + 高熵随机填充）；`encrypt_core.py` 检测 `A96D` magic 取 ELF 真实大小为 `original_size`；loader 无改动（`pt_len = original_size` 自动截断） | **Done** | 0.39 |
| **安全** | arm96server OLLVM 混淆 | arm96server 编译切换 clang-14 + OLLVM CFF+SUB，关键函数 (`scan_and_kill_debuggers`/`scan_and_track`/`spawn_guardian`) 标记混淆属性 | **Done** | 0.40 |
| **隐蔽** | arm96server 字符串加密 | arm96server 明文字符串 (comm/路径/调试工具名) 改为编译时 LCG-XOR 加密 + 运行时栈上解码；`make.sh` 生成 `gen/arm96server_strings.h`；HKDF 索引 20~31 | **Done** | 0.40 |
| **安全** | arm96server .text hash 融入 token | token 生成改为 `stored = HMAC(base_key, 19)[:16] ⊕ SHA-256(.text)[:16]`；loader 运行时计算 arm96server .text hash 做 XOR 消去；.text 任何改动 → token 失效 → 解密失败 | **Done** | 0.40 |
| **混淆** | Loader OLLVM 混淆覆盖扩展 | 扩大 loader.c OLLVM 混淆覆盖范围：新增 10 个高价值函数标记混淆属性（`decrypt_core_to_memfd` CFF+BCF+SUB、`exec_core`/`launch_arm96server`/`kill_traced_app_processes`/`kill_orphaned_app_processes`/`antidebug_watchdog_loop` CFF+SUB、`platform_check_mac_oui`/`mac_oui_scan_netlink` CFF、`antidebug_trigger`/`start_antidebug_watchdog` SUB）；间接分支数从 102 增至 108；仅改 loader.c + version.txt | **Done** | 0.41 |
| **功能** | 过期首次自杀抖动 | 正式版 `LICENSE_CHECK_INTERVAL=1800`；已过期启动时首次 kill 在 interval 后 1/3 窗口内派生（正式版 1200-1800s，5 分钟评估版 120-180s），后续按固定 interval 周期执行 | **Done** | 0.42 |
| **验收** | PID tracking 验收口径收敛 | `verify_mustpass.sh` 按 `TRACKED_PID_FILE`、guardian cascade、TracerPid 扫描窗口验证；必过项不再允许 SKIP 后仍算全绿 | **Done** | 0.44 |
| **安全** | V0.48 关键数据域完整性与验收闭环 | 完整性链路与关键数据域口径收口，发布链维持 mustpass 回归约束 | **Done** | 0.48 |
| **安全** | V0.49 构建门禁与一致性校验 | 输入指纹门禁（E4901）+ patch 前镜像断言（E4902）+ patch 后白名单核验（E4903）+ `orig_size` 一致性校验（E4905）；擦除策略升级为可审计 remap（`.tango→.arm96`、`tango_prop→arm96_prop`、`tango uid→arm96 uid`）并保留 `/dev/tango*` 功能路径 | **Done** | 0.49 |
| **安全** | V0.54 D1 多点见证并入解密链 | D1 witness 融入解密密钥派生；`loader.c` 启动/密钥路径/周期检查三检查点收口，`encrypt_core.py` 保持对称派生，并在 `verify_mustpass_host.sh` 增加 `TC-015` 回归烟测 | **Done** | 0.54 |
| **安全** | V0.55 关键代码域并入解密链 | `loader` 自体可执行段 `text hash16` 融入密钥，构建端同口径融合；`make.sh` 增加 V0.55 static gate，`verify_mustpass_host.sh` 增加 `TC-016` round4 smoke，`release.sh` 增加发布门禁 | **Implemented** | 0.55 |
| **兼容** | V0.56 service UID 平台校验兼容 | `loader` 在 `/system/build.prop` 因 `0600 root:root` 对非 root service UID 不可读时，回退到混淆字符串解码后的 `/system/bin/getprop ro.build.version.release`；HKDF 索引 38/39 对应 `STR_GETPROP_PATH` / `STR_BUILD_RELEASE_PROP`，用于修复 arm96 下 32 位 OMX/CAS init service 启动失败 | **Implemented** | 0.56 |
| **性能/工程** | V0.57 构建期预翻译 opt-in 试点 | 复用 host `tango_pretranslator.x86_64`，以显式 `--output-file-name=*.arm96` 生成 `MX64/v15` 侧车；已在 AOSP debug 分支验证 system/vendor 覆盖与运行链路，但 `com.dragon.read` 未观察到明显体感优化，当前暂停，仅保留为历史调试分支参考 | **Paused / Historical** | 0.57 |
| **工程/功能** | 安装时预翻译交付链 | `frameworks/base` 安装/卸载事件经 `android.os.arm96.Arm96j` → `system_server Arm96Service("arm96j")` → `arm96j.jar` 插件进程 → `arm96d` Binder 服务；`arm96d` 对 `code_paths`、`lib/arm/*.so`、`oat/arm/*.odex` 生成 `.arm96` 侧车 | **Implemented** | 0.57→0.58 |
| **工程** | 六件套产物流与打包收口 | `package.sh` / release patch / 交付 README 已按 `arm96`、`libarm96`、`arm96server`、`arm96_sidecar_gen`、`arm96j.jar`、`arm96d` 六件套收口；`debug_pretranslate_apk.sh` 用于单 APK 调试；`arm96_sidecar_gen` 已加入独立加固 | **Implemented** | 0.57→0.58 |
| **工程** | V0.59 release patch 序列精简 | `res/release/patchs` 当前已精简为 2 个 patch；安装时预翻译相关接线已并入第 2 个 patch，不再拆分独立 install-pretranslate patch；`release.sh` 改为固定打包当前 patch 集与六件套产物 | **Implemented** | 0.59 |
| **兼容** | arm96d phase0 接管 binfmt 注册 | `arm96d --phase0` 负责清理旧 `arm96`/`tango_translator` binfmt 节点并重建 `POC` 规则，交付链不再依赖单独 shell 注册脚本维护解释器状态 | **Implemented** | 0.58 |
| **发布** | V0.67.0 | `CUSTOM_VERSION=V0.67.0`；V0.67 在 V0.66 基础上落地 Full Layout Repack：RW LOAD 文件偏移整体后移 `0x10000`（VA 不变），首个 R-X LOAD 扩展到 `0x114b88`，释放 `0x10204`（约 64.5KB）executable island，stub 迁移到新 island。约束保持：不新增 `PT_LOAD`、不把 RW 改成 RWX、A96D/密文容器与 loader memfd/fexecve 模型不变。构建 `./make.sh --build cx` 通过；设备主链 mustpass `12 PASS / 0 FAIL / 0 SKIP` 通过。 | **Done** | 0.67 |
| **发布** | V0.67.0 Full Layout Repack | 新增 `relayout_v067.py` 独立重排 helper + 静态 verifier，并接入 `main.py` patch pipeline（stub 注入前）；`E4903` 从纯 offset 白名单扩展为"offset + bulk range"结构化核验；`verify_mustpass_host.sh` 新增 TC-017 静态门禁（无新增 PT_LOAD / RW 非 RWX / island 可执行区覆盖校验）。 | **Done / Host+Device Mainline Verified** | 0.67 |
| **后续** | V0.68.0 Scatter Decoy Points | 在 V0.67 RX 扩展岛基础上，分散注入非混淆诱饵攻击点（含授权截止时间/epoch/伪配置键值），并以结构化白名单和静态门禁约束注入窗口，降低攻击者定位真实授权链路的效率；`SCATTER_DECOY_ENABLE=1`，`SCATTER_DECOY_MIN_HITS=4`，`SCATTER_DECOY_MIN_GAP=0x40`。已提交（commit `b38f9cd V0.68.0`），未单独验证设备 mustpass。 | **Committed (2026-05-30)** | 0.68 |
| **进行中** | V0.69.0 VMP-Core (rx-cave) | 在 V0.67 RX island 基础上引入可选的 host-side VMP（虚拟化打包）工具 `res/bin/vmpacker`，对 libarm96 core 内显式 target range 做 rx-cave 注入 + trampoline 跳转，并保持 PT_LOAD/RWX 布局不变（`_verify_vmp_layout_drift` 校验 `E5904`，target 解析校验 `E5906`）；配置：`VMP_CORE_ENABLE=1`，`VMP_CORE_MODE=rx-cave`，`VMP_CORE_TARGETS`（首版仅支持单个显式 range，为空回退 `.text` 启动期默认窗口），`VMP_CORE_STRICT_VERIFY=1`，`VMP_CORE_MIN_FREE=0x1000` 为后续 V0.68 scatter decoy 预留空间；`main.py` patch pipeline 新增约 320 行（VMP 阶段插入 stub 注入与 scatter decoy 之间）。**当前为本地未提交工作树状态（`git status` 显示 `hardening/src/main.py`/`version.txt`/`loader.c`/`arm96server.c`/`encrypt_core.py` 等均为 modified，未 commit）**，2026-06-14 实测 `./make.sh --build cx` 可产出六件套（`VERSION=V0.69.0`），但**用户确认相关功能尚未跑通，仍在 debug 阶段**，未做设备 mustpass 验证。 | **In Progress / Debugging, Uncommitted** | 0.69 |
| **后续** | V0.69+ 独立收口 | CAS `setThreadPoolConfiguration:-25` 与周期 challenge-response 继续独立排期，不与 V0.68/V0.69 改动叠加。 | **Planned** | 0.69+ |

### 🧭 后续规划 (Planned Roadmap, Next Rounds)

> 来源：基于红方 Round1（V0.44）攻击复盘与 V0.49 落地后的收口结果。  
> 说明：V0.48、V0.49 已完成并已并入上方“功能特性概览”；本节仅保留后续版本计划。

#### V0.48（已完成）
> **状态**: **Done (2026-05-03)**

- [x] **关键数据域完整性覆盖扩展**：在 `.text` 自校验之外补齐关键安全数据域篡改检测。
- [x] **Release 禁用明文 core fallback**：release 构建强制走加密容器解密执行链。
- [x] **篡改门禁回归**：关键路径篡改样本纳入验收回归。

#### V0.49（已完成）
> **状态**: **Done / Current Release (2026-05-03)**

- [x] **输入指纹门禁（E4901）**：构建前校验 `build-id + 文件长度窗口 + 关键段hash`。
- [x] **patch 前镜像断言（E4902）**：关键 offset 写入前校验旧值签名。
- [x] **patch 后白名单核验（E4903）**：限制改写偏移与计数范围。
- [x] **容器头一致性校验（E4905）**：强校验 `orig_size <= pt_len` 且与预期 ELF 原长一致。
- [x] **字符串擦除规则收口**：remap 为 `.tango→.arm96`、`tango_prop→arm96_prop`、`tango uid→arm96 uid`，并保留 `/dev/tango*`。

#### V0.50（优先：中代价 / 高价值，补持续认证；原 V0.49 顺延）
> **目标**: 解决“启动期一次性校验”短板（重点针对 W6）。
> **状态**: **Planned (Renumbered from old V0.49)**

- [ ] **loader ↔ arm96server 周期 challenge-response**：引入 nonce + HMAC 周期认证，避免仅靠启动期探测。
- [ ] **失败策略升级为延迟失效链**：认证失败后进入延迟失效 + tracked app 清理路径，降低定位性与一次性绕过收益。

**V0.50 实施前检查清单（Preflight）**
- [ ] 明确认证协议边界：挑战发起方、应答方、超时、重试次数、会话恢复条件。
- [ ] 明确密钥来源与生命周期：是否复用现有派生链、是否区分设备/会话粒度、内存清零策略。
- [ ] 明确通信通道与安全属性：本地 socket/binder 的鉴权、重放防护、消息完整性、防中间人假转发。
- [ ] 明确失效策略与可用性边界：延迟失效窗口、降级行为、对正常业务抖动容忍度。
- [ ] 明确与现有机制协同：避免与 setitimer、guardian cascade、PID tracking 触发顺序冲突。

**V0.50 验收判据（Acceptance Gate）**
- [ ] 正常样本：周期 challenge 在设定窗口内稳定通过，不引入额外 ANR/重启抖动。
- [ ] 单边 patch（仅 loader 或仅 arm96server）导致 challenge 失败：必须进入受控失效链。
- [ ] 消息重放/延迟注入：过期 nonce 或重复应答必须被拒绝。
- [ ] 本地伪造应答进程：未持有正确密钥材料时不得通过认证。
- [ ] 回归不退化：现有 mustpass（含 anti-debug / tracked app 清理链路）保持全绿。

#### V0.51（可选：视对抗强度开启；原 V0.50 顺延）
> **目标**: 进一步提高静态批量 patch 与脚本化绕过成本。
> **状态**: **In Progress (Optional, keyed-tag + 错误结果模式实施中)**

- [x] **反 hook 比较升级为 keyed-tag（首版）**：loader maps 检测与 arm96server debugger 识别增加 keyed-tag 判定路径（默认由 `V051_ENABLE=0` 保持关闭）。
- [x] **判定结果耦合关键翻译路径（首版错误结果模式）**：anti-debug / arm96server 异常可进入延迟失效窗口，并在解密后写入前执行受控数据扰动（有界、可配置）。
- [x] **构建接线**：`encrypt_core.py` 生成 `gen/v051_keyed_tag.h`，`make.sh` 已透传接线。
- [x] **灰度开关**：`version.txt` 新增 `V051_*` 配置，默认 `V051_ENABLE=0`。
- [x] **最小可观测性**：arm96server 命中 debugger/tracer 时写故障戳，loader 消费后进入 fault mode。

**本轮范围约束**
- 仅实施 V0.51（keyed-tag + 错误结果模式），不包含 V0.50 challenge-response。
- 保持 V0.49 E4901/E4902/E4903/E4905 门禁逻辑不回退。
- 不改 guardian / tracked PID 主链语义，只做最小耦合。

**V0.51 实施前检查清单（Preflight）**
- [ ] 明确 keyed-tag 设计：tag 长度、算法选择（HMAC/AEAD）、key 派生索引与版本兼容策略。
- [ ] 明确替换范围：哪些现有明文匹配点迁移到 keyed-tag，避免“新旧逻辑并存”形成旁路。
- [ ] 明确错误结果模式边界：影响哪些翻译路径、失败是否可控、是否会扩大误杀面。
- [ ] 明确可观测性策略：错误结果模式下保留最小诊断信号，避免运维完全失明。
- [ ] 明确性能预算：tag 计算与路径耦合后对冷启动与热点翻译耗时的上限。

**V0.51 验收判据（Acceptance Gate）**
- [ ] 密文随机化/改写不再能稳定关闭反 hook 检测（应导致认证失败或失效链触发）。
- [ ] 仅 patch 单点 abort 逻辑无法恢复稳定运行（仍应进入错误结果或受控失败）。
- [ ] 错误结果模式具备稳定可复现边界：在同类触发条件下表现一致，不出现无界崩溃。
- [ ] 性能回归受控：关键场景开销在预设阈值内（由基线压测定义）。
- [ ] 与 V0.49/V0.50 机制不冲突：完整性、challenge、guardian 链路回归全绿。

#### V0.52（当前规划：最小改动封口）
> **目标**: 先堵住 Round 2 已证实的稳定绕过链，避免“配置回退 + 明文 fallback”再次进入发布包。
> **状态**: **Planned (Ready for implementation)**

- [ ] **P0-1 强制启用 V051 路径**：`version.txt` 置 `V051_ENABLE=1`，保持 `V051_KEYED_TAG_ENABLE=1`、`V051_ERROR_MODE=1`。
- [ ] **P0-2 去除 maps 明文 fallback**：`loader.c` 的 `antidebug_scan_maps()` 仅保留 keyed-tag 判定，删除 `strstr("frida|gadget|...")` 分支。
- [ ] **P0-3 构建门禁 fail-fast**：`shell/make.sh` 增加硬门禁：若 V051 三开关不满足即失败；产物 `arm96` 若出现反 hook 明文关键词即失败。
- [ ] **P0-4 发布门禁复核**：`release.sh` 在打包 `update-bin/arm96` 前重复执行明文关键词检查，防止绕过 make 阶段门禁。
- [ ] **P0-5 mustpass 增补**：`verify_mustpass_host.sh` 增加两条 host 侧用例：
  - `TC-013`：断言 `gen/version.py` 中 V051 三开关为启用态。
  - `TC-014`：断言 `arm96` 二进制不含 `frida/gadget/gum-js-loop/xposed/substrate/lsposed` 明文。

**V0.52 范围约束**
- 不在本版本引入 V0.50 的 challenge-response 全链路（避免扩大变更面）。
- 不修改 guardian / tracked PID 主链语义。
- 仅处理“已证实可稳定复现”的绕过入口与发布门禁缺口。

**V0.52 实施顺序（按风险收敛）**
1. `version.txt` 开关与版本号收口。
2. `loader.c` 删除反 hook 明文 fallback。
3. `shell/make.sh` 加构建门禁（配置 + strings）。
4. `release.sh` 加发布门禁复核。
5. `verify_mustpass_host.sh` 增加 `TC-013/TC-014`。

**V0.52 验收判据（Acceptance Gate）**
- [ ] release 构建默认启用 V051 三开关；任一关闭会阻断构建。
- [ ] `update-bin/arm96` 不出现反 hook 明文关键词。
- [ ] 明文关键词清零类 patch 不再构成稳定绕过（应进入 keyed-tag/fault 链）。
- [ ] 现有 V0.48/V0.49 mustpass 项不回退。

**V0.52 后续增强（不在本次最小范围）**
- [ ] **P1 多点时间判定**：去单点 `cmp/bl` 依赖，至少双点触发失效链。
- [ ] **P1 过期统一进入 fault mode**：降低定点 NOP 的恢复收益。
- [ ] **P2 周期 challenge-response**：并入 V0.50 持续认证链。

#### V0.53（硬件绑定敏感信息去明文 + 抗补丁收口）
> **目标**: 消除硬件绑定路径中的明文敏感信息（如 `cix;myt,p1`、`/proc/device-tree/compatible`、`/system/build.prop`、`ro.build.version.release=`），并提升单点补丁绕过成本。
> **状态**: **Done (2026-05-05)**

- [x] **P0-1 绑定字符串加密接入生成链**（`shell/make.sh`）
  - 已接入并输出：`STR_BUILD_PROP_PATH`、`STR_BUILD_RELEASE_KEY`、`STR_DT_COMPAT_PATH`、`STR_DT_GROUPS`。
  - HKDF 索引与 loader 侧对齐，避免错位解密。

- [x] **P0-2 loader 去硬编码明文**（`hardening/src/loader.c`）
  - `platform_check_android_release()` 与 `platform_check_dt_compatible()` 已改为运行时解密并 `secure_zero()` 清理。
  - 绑定路径相关硬编码明文已移除。

- [x] **P0-3 HKDF 索引显式化**（`hardening/src/loader.c` / 公共头）
  - 已新增 `HKDF_IDX_STR_BUILD_PROP_PATH`、`HKDF_IDX_STR_BUILD_RELEASE_KEY`、`HKDF_IDX_STR_DT_COMPAT_PATH`、`HKDF_IDX_STR_DT_GROUPS` 并完成映射顺延。

- [x] **P0-4 绑定检查抗单点补丁**（`hardening/src/loader.c`）
  - 绑定检查已扩展为多检查点复验，失败统一走 fail gate。

- [x] **P0-5 配置到产物链路去敏**（`hardening/src/version.txt`、`hardening/src/profiles/*.txt`、`hardening/src/gen/version.h`）
  - 配置侧可保留明文维护；`gen/version.h`/最终二进制已去除上述绑定敏感明文。

**V0.53 验收结果（Acceptance Gate）**
- [x] `strings arm96` 不命中：`cix;myt,p1`、`/proc/device-tree/compatible`、`/system/build.prop`、`ro.build.version.release=`。
- [x] 目标设备功能回归通过，非目标设备绑定拒绝保持生效。
- [x] 历史单点 NOP 位点不再构成稳定绕过链。
- [x] mustpass 与既有门禁未回退。

#### V0.54（D1 多点见证并入解密链）
> **目标**: 将 D1 从“单点时间判定”升级为“多点见证参与解密”，降低单点 patch（NOP/常量改写）收益。
> **状态**: **Done (2026-05-05)**

- [x] **P0-1 D1 判定链路收口**（`hardening/src/loader.c`）
  - 已移除直接 `EXPIRE_EPOCH` 比较口，统一走运行时派生链路。

- [x] **P0-2 多点见证采样**（`hardening/src/loader.c`）
  - 已在启动早期、关键执行路径、定时检查路径接入 3 个 witness 采样点。

- [x] **P0-3 见证参与派生链**（`hardening/src/loader.c`、`hardening/src/encrypt_core.py`）
  - D1 witness 已并入密钥派生，构建端与运行端保持对称。

- [x] **P0-4 失败语义统一**（`hardening/src/loader.c`）
  - witness 异常统一走现有 fail gate。

- [x] **P0-5 验收补齐（单点补丁 smoke）**（`verify_mustpass_host.sh`）
  - 已增加 `TC-015`，覆盖 D1 单点补丁扰动验证。

**V0.54 验收结论**
- [x] 单点 NOP/常量改写不再构成稳定绕过。
- [x] witness 失配可进入受控失败链。
- [x] mustpass 主链（anti-debug / PID tracking）未回退。

> **注（2026-05-05 Round4 对抗复盘）**: 红方已在 V0.54 发布包上验证“P1~P5 关键位点补丁 + P6/P7 D4 重算回填”可稳定绕过；D1 并入解密链有效抬升门槛，但不足以单独阻断离线重签链。

#### V0.55（关键代码域并入解密链，封堵离线重签绕过）
> **目标**: 封堵 Round4 已证实的“关键位点 patch + D4 双哈希重算”稳定绕过链；将关键代码篡改从控制流域转移到数据域失败（解密认证失败）。
> **状态**: **Implemented (2026-05-05, validation in progress)**

- [x] **P0-1 关键代码域哈希并入密钥派生**（`hardening/src/loader.c`、`hardening/src/encrypt_core.py`、`shell/make.sh`）
  - 运行期 `self_text_hash16` 融入解密 key；构建期加密使用同口径 `loader_text_hash16`。

- [x] **P0-2 固化关键窗口口径并双端一致复算**（`hardening/src/encrypt_core.py`、`hardening/src/loader.c`）
  - 双端统一为首个 `PT_LOAD + PF_X` 可执行段哈希前 16B。

- [x] **P0-3 D4 角色调整为审计层，不再单点承载阻断职责**（`hardening/src/loader.c`）
  - 保留 `integrity_check_text()` / `integrity_check_critical_data()`；实际不可离线修复阻断由 key fusion 承担。

- [x] **P0-4 构建/发布门禁补齐关键补丁 smoke**（`shell/make.sh`、`release.sh`、`verify_mustpass_host.sh`）
  - 已新增 V0.55 static gate、`TC-016 round4 smoke` 与 release gate 必备项检查。

**V0.55 当前验收状态（2026-05-05）**
- [x] 构建门禁通过：`shell/make.sh --build cx` 成功，V0.55 gate 全绿。
- [x] 设备部署生效：`192.168.11.45:5004` 上 `arm96 -v` 为 `V0.55.0`，三件套 md5 与本地产物一致。
- [ ] mustpass 全绿：当前 `verify_mustpass_host.sh` 结果 `11 PASS / 1 FAIL`，剩余失败项为 `TC-008 PID-tracking Cascade`（非版本号问题，待单独收口）。

**V0.55 范围约束**
- 不引入 V0.50 challenge-response 全链路（避免扩大改动面）。
- 不改外部接口与部署形态。
- 保持 anti-debug / PID tracking / guardian 主链语义稳定。

#### V0.56（service UID Android release fallback）
> **目标**: 修复非 root init service UID 下平台 Android release 校验失败，确保 arm96 作为 binfmt 解释器承载 32 位 vendor service 时不因 `/system/build.prop` 权限导致静默退出。
> **状态**: **Deployed Candidate (2026-05-06, full mustpass pending)**

- [x] **P0-1 service UID fallback**（`hardening/src/loader.c`）
  - `platform_check_android_release()` 仍优先直接读取 `/system/build.prop`；当 service UID 无权读取该文件时，fallback fork/exec `/system/bin/getprop ro.build.version.release` 并按 `PLATFORM_ANDROID_RELEASE` 比对。

- [x] **P0-2 fallback 字符串混淆**（`shell/make.sh`、`hardening/src/loader.c`）
  - 新增 HKDF 索引 38/39：`STR_GETPROP_PATH` = `/system/bin/getprop`，`STR_BUILD_RELEASE_PROP` = `ro.build.version.release`；运行期解码后立即 `secure_zero()` 清理。

- [x] **P0-3 init service 兼容性收口**
  - 根因定位为 `/system/build.prop` 权限 `0600 root:root`，`mediacodec` 等非 root service UID 不能直接读取；fallback 解决 arm96 下 32 位 OMX/CAS service `status 0` 重启风暴。

**V0.56 当前状态（2026-05-06）**
- [x] 源码版本号已推进为 `CUSTOM_VERSION=V0.56`。
- [x] loader fallback 逻辑已提交：`a12d6e0 Fix Android release check for service UIDs`。
- [x] `./make_5min.sh --build vc` 构建通过，并已同步 `build/out/vc/bin/`、`prebuilts/` 与 `dist/arm96-V0.56.tgz`。
- [x] 已部署到 `192.168.11.45:5004`，快速复验通过：`arm96 -v` = `V0.56`，`arm96 -h` 输出 0 字节，`zygote_secondary` / `vendor.media.omx` / `vendor.cas-hal-1-1` 均为 running，binfmt flags = `POC`。
- [ ] 完整 `verify_mustpass_host.sh` 主链待单独复跑。

#### V0.57（构建期预翻译 opt-in 试点，历史调试分支）
> **目标**: 在 `main/1.0` 二进制二次加固/填制交付范围内，恢复 host/build-time 预翻译能力，先让 release/离线链路能对白名单 ARM32 系统/vendor 对象生成 `.arm96` 侧车，验证冷启动收益与 fallback 安全性。
> **状态**: **Paused (2026-05-09, debug branch only)**

> **说明**: 本节记录的是 `debug/20260509_arm96_new_dbg_v0.57` 上的历史性 host/build-time 试点。`main/1.0` 后续实际推进的交付主线已转为安装时预翻译（`arm96j`/`arm96d`/`arm96_sidecar_gen`），见下方 V0.58 小节。

**当前结论（2026-05-09）**
- 当前继续开发基线：`debug/20260506_arm96_new_dbg` @ `eec20882d53`。
- V0.57 冻结点：`debug/20260509_arm96_new_dbg_v0.57` @ `181a0cbb76007240ae25ab3764f0a5564e325593`（short: `181a0cbb760`）。
- 已在 `debug/20260509_arm96_new_dbg_v0.57` 分支完成 AOSP 树内接线：`vendor/arm96/host-tools/`、`vendor/arm96/tasks/native-postprocess.mk`、`build/make/core/*.mk` 与 `main.mk` 最小 include 已落地。
- `arm96-native-postprocess` 已能在镜像打包前统一补齐 `/system` 与 `/vendor` 下的 `.arm96`；当前测试设备 `192.168.11.45:5004` 统计为 `system=725`、`vendor=95`。
- 对比旧设备 `192.168.11.22:5004` 的 `.tango` 分布，旧链路 25 个对象已全部被当前 `.arm96` 覆盖，`only_old=0`。
- `com.delta.app.tt` 与 `com.dragon.read`（`armeabi-v7a`）均已在测试设备上成功拉起并稳定留存进程，说明侧车生成与运行链路成立。
- 但对 `com.dragon.read` 的实际启动体感未观察到明显优化收益，当前没有足够 A/B 数据支撑继续扩大实施面，因此本版本在 debug 分支暂停，不进入 release/交付链。

**规划依据**
- 已实测当前 arm96 `V0.56` runtime 在 binfmt `POC` 执行路径会打开同路径 `.arm96` 侧车。
- 已实测 host `tango_pretranslator.x86_64` 可生成 `MX64/v15` cache；通过显式 `--output-file-name=*.arm96` 可绕开 legacy `.tango` 后缀模板。
- 构建期预翻译覆盖的是系统镜像内置 ARM32 ELF/`.so`/部分 APK，不覆盖三方 app 私有 `/data/app/.../lib/arm/*.so`；本版本只验证系统/vendor 公共路径和少量热点 service/library。

**范围边界**
- 仅迁移 host/build-time 能力：复用 `prebuilts/tango_pretranslator.x86_64`，新增 arm96 命名包装与白名单脚本/配置。
- 不修改 `src/tango_pretranslator_reimpl/`、`src/tango_translator_reimpl/`，不做源码重实现。
- 不在生产镜像新增设备端 `tango_pretranslator.arm64` 或可见 `*pretranslator*` 工具。
- 不恢复 legacy `tango_pretranslate` 全局 make 链路，不接 PMS/installd，不接 ART boot oat/OAT 自动预翻译。
- 默认关闭；未显式开启时，构建产物、安装行为、mustpass 口径应与 V0.56 保持一致。

**实施拆分**
- [ ] **P0-1 工具封装**：新增离线 wrapper（建议命名 `arm96_build_pretranslate` 或同等隐蔽名称），内部调用 host `tango_pretranslator.x86_64`，固定使用 `--input-file-name` + `--output-file-name=<src>.arm96`，避免 `--output-dir` 自动生成 `.tango`。
- [ ] **P0-2 白名单配置**：新增构建期白名单，首批只覆盖 3 类样本：最小 ARM32 ELF/验证样本、`/system/lib` 或 `/vendor/lib` 少量热点库、一个 32 位 vendor service（例如 OMX/CAS 相关对象）。禁止递归扫全 image。
- [ ] **P0-3 release opt-in 接线**：在 `make_5min.sh` / `package.sh` 或独立 release helper 中增加开关（建议 `ARM96_PRETRANSLATE=1`），打开时生成并打包 `.arm96`；关闭时不生成、不安装、不改变包结构。
- [ ] **P0-4 安装/回滚语义**：安装脚本只在 opt-in 包存在 `.arm96` 时随原文件旁路部署；卸载或回滚时删除 `.arm96` 侧车即可恢复纯 JIT 路径。
- [ ] **P0-5 签名与 stale cache 策略**：优先使用 pretranslator 的 `--check-signature` / `--force` 语义做重复构建判断；产物清单记录源文件 hash、cache size、生成命令和 pretranslator build-id。
- [ ] **P1 性能 A/B**：对 32 位 zygote/app_process32、一个 32 位 vendor service、一个系统/vendor 32 位共享库做冷启动或首次加载 A/B；指标至少包含启动耗时、首次 CPU 峰值、log/tombstone、是否进入 fallback。

**验收判据（Acceptance Gate）**
- [ ] `ARM96_PRETRANSLATE=0` 或未设置时，V0.57 构建、安装、`arm96 -v/-h/-a`、binfmt flags、zygote/service 状态与 V0.56 行为一致。
- [x] `ARM96_PRETRANSLATE=1` 调试链路已打通：镜像内生成 `.arm96`，产物头为 `MX64`、version `15`，未产生 `.tango` 后缀产物。
- [x] 目标设备 `192.168.11.45:5004` 上，旧链路 25 个 `.tango` 对象已全部被 `.arm96` 覆盖；`com.delta.app.tt` 与 `com.dragon.read` 启动成功，未见功能性退化。
- [ ] 缺失、损坏、权限错误、版本不匹配的 `.arm96` 不导致 boot/zygote/vendor service 崩溃；必须能删除侧车回退 JIT。
- [ ] `verify_mustpass_host.sh` 主链不退化；尤其确认 `arm96 -v` 仍严格 1 行、`-h` 静默、binfmt flags 仍为 `POC`、32 位 zygote 更新后已重启生效。
- [ ] 生成物体积膨胀可控：首批白名单总 `.arm96` 增量必须有清单和上限，禁止把 `/data/app` 或全量 `/system/vendor` 纳入本版本。

**暂停原因**
- 当前验证只证明了“构建期 `.arm96` 生成 + 运行链路可用”，尚未证明“对目标应用有稳定、可感知的性能收益”。
- 在 `com.dragon.read` 上主观体感没有明显优化；在补齐严格的冷启动 A/B 指标、样本收敛和回退收益评估前，不继续推进 V0.57 的扩大覆盖、release 接线或交付化。
- 后续若要继续 V0.57，请优先从冻结点 `debug/20260509_arm96_new_dbg_v0.57` / `181a0cbb76007240ae25ab3764f0a5564e325593` 恢复，而不是从当前开发基线直接补丁前推。

**主要风险与控制**
- 磁盘膨胀：`.arm96` 通常可接近原始 ELF 的 2x 或更高；用白名单和 size gate 控制。
- stale cache：源文件更新后旧 `.arm96` 可能被误用；用源 hash/build-id 清单和 `--check-signature` 控制。
- 故障面扩大：vendor service 冷启动若命中异常 cache，可能造成 init restart；首版必须只选可快速回滚对象，并保留删除侧车回退路径。
- 逆向面增加：`.arm96` 是落盘 host code cache；首版只用于性能验证和受控交付，不把设备端生成工具暴露到生产镜像。

**本版本不做**
- 不做安装时预翻译，不处理三方 app 私有 `.so` 自动生成。
- 不接 PMS/installd，不恢复 PackageManager/Installer binder API。
- 不接 boot oat / odex / dexopt 自动链路。
- 不把 `tango_pretranslator.arm64` 放入 production `/system/bin`。

#### V0.58（arm96j/arm96d 安装时预翻译交付链）
> **目标**: 在 `main/1.0` 的既有二进制交付链上，补齐安装时预翻译能力，使 PMS 安装/卸载事件能够通过 `arm96j`/`arm96d` 驱动 `arm96_sidecar_gen` 为三方 32 位应用生成 `.arm96` 侧车，并把该能力并入默认 release 交付物。
> **状态**: **Current Candidate (2026-05-17 ~ 2026-05-20, full mustpass pending)**

**当前结论**
- `aa3a021` 已引入 `arm96j` / `arm96d` 基础实现；`system_server` 在启动时注册 `Arm96Service("arm96j")`，PMS 安装/卸载完成路径经 `android.os.arm96.Arm96j.onAction(...)` 发事件，`arm96j.jar` 作为 app_process64 插件进程转发给 `arm96d`。
- `0d06fcd` 已形成安装时预翻译主体链路：`arm96d` 只对 install/uninstall 事件做实质处理；安装时先做 eligibility 判定，再可选触发 `cmd package compile -m speed-profile -f`，最后对白名单目标生成 `.arm96` 侧车。
- 当前 `arm96d` 预翻译目标收集已覆盖 `code_paths`、`/data/app/<pkg>/lib/arm/*.so`、`/data/app/<pkg>/oat/arm/*.odex`；侧车后缀已统一为 `.arm96`。
- `83bfffa` 与 `824839d` 已把交付与构建产物流收口到六件套：`arm96`、`libarm96`、`arm96server`、`arm96_sidecar_gen`、`arm96j.jar`、`arm96d`。
- 当前 `V0.59.0` 交付口径下，`res/release/patchs` 已精简为 2 个 patch；安装时预翻译相关接线已并入第 2 个 patch，不再保留独立 install-pretranslate patch。
- `0257fad` 后，`arm96d --phase0` 已接管 binfmt 注册：启动时会清理旧 `arm96` / `tango_translator` 节点，再写回解释器为 `/system/bin/arm96` 的 `POC` 规则。
- `dd83ef2` 已对 `arm96_sidecar_gen` 加固；`0aa454e` 提供 `debug_pretranslate_apk.sh` 与 `arm96d --pretranslate-apk` 单文件调试入口，便于独立验证 APK 侧车生成。
- 对应地，`release.sh` 已固定按当前 patch 集和六件套打包，不再支持排除安装时预翻译的可选模式。

**与 V0.57 历史试点的关系**
- V0.57 的 host/build-time 预翻译试点是 debug 分支上的性能探索，不进入默认 release。
- V0.58 当前主线是安装时预翻译交付链，实施面位于 `frameworks/base` 事件接线、`arm96j.jar`、`arm96d`、release patch、打包脚本和镜像集成。
- 两条线都产出 `.arm96` 侧车，但触发时机、实施面和交付语义不同；文档必须区分，不得再把 V0.58 现态描述成“仅 host/build-time 试点”。

**提交映射（主线）**
- `aa3a021`: 添加 `arm96j` / `arm96d`。
- `0d06fcd`: 增加安装时预翻译流与 sidecar rename。
- `824839d`: 重构 arm96 build artifact flow。
- `83bfffa`: 打包脚本纳入完整 arm96 artifact set。
- `4cbcb62` / `f402108`: release/base patch 刷新重建。
- `9f8f797` / `b69d42b`: 安装时预翻译开关语义与默认值调整。
- `dd83ef2`: `arm96_sidecar_gen` 加固。
- `0aa454e`: APK 预翻译调试脚本。
- `0257fad`: `arm96d` 接管 binfmt 注册。
- `d7df43a`: 版本推进到 `V0.58.0`；当前工作区版本口径已继续前推到 `V0.59.0`。

**属性与默认语义现状（必须明确区分）**
- 交付/patch/调试口径已经引入短属性 `persist.vendor.p3a9c7d1e`，并在 release patch 的 `property_contexts` 中分配为 `arm96_prop`。
- 当前 `src/arm96d/service/Arm96dService.cpp` 源码仍保留旧属性 `persist.sys.ap3219402ff1`，且 `GetBoolProperty(..., false)` 默认关闭。
- 因此，当前仓库存在“源码口径”和“release/调试口径”未完全收口的问题；在 `main/1.0` 继续迭代时，必须避免把这两者混写成单一事实。

**当前验收结论**
- 交付形态已不再是仅 `arm96 + libarm96 + arm96server` 三件套，而是默认可带安装时预翻译能力的六件套交付。
- 当前 release patch 序列已精简为 2 个，且第 2 个 patch 已并入安装时预翻译实现；交付链不再维护“排除安装时预翻译”的独立 patch 变体。
- 设备侧已有安装后 `.arm96` 侧车生成的验证记录，但完整 `verify_mustpass_host.sh` 主链仍待对当前 V0.59.0 候选重新复跑。

**主要风险与未收口项**
- 文档漂移：旧版 V0.57 小节曾明确写“本版本不做安装时预翻译”，已不再适用于当前交付候选。
- 属性漂移：`persist.vendor.p3a9c7d1e` 与 `persist.sys.ap3219402ff1` 并存，容易导致源码验证、镜像行为和 release 说明出现互相矛盾的结论。
- 实现面暴露增加：相比 loader/core，`frameworks/base` 的 `Arm96j`、`Arm96Service`、AIDL、`arm96j.jar`、`arm96d` 构成了新的可见攻击面，后续需要单独评审其抗逆向与口径统一策略。

#### V0.59.0-vmp（vmp 接入 hardening pipeline 规划）
> **目标**: 将独立仓库 `vmp` 的 VMPacker 能力接入 `libarm96` 加固流水线，在不新增 `PT_LOAD`、不改变 loader memfd/fexecve 模型、不影响默认 release 稳定性的前提下，对少量低频安全逻辑做可选 VM 化保护。
> **状态**: **Planned (2026-06-10, opt-in prototype)**

**版本边界与当前状态**
- 当前 `main/1.0` 主线 HEAD 已推进到 V0.68.0 一带，并已具备 V0.67 Full Layout Repack 释放出的 RX island；用户口径中的 V0.59.0-vmp 更适合作为“VMP 接入规划/实验标签”，而不是回退覆盖既有 V0.59 安装时预翻译交付语义。
- `hardening/src/version.txt` 当前工作树存在脏状态，曾观察到 `CUSTOM_VERSION=V0.66.0` 与 HEAD 历史不一致；正式实施前必须先确认版本号/分支口径，避免把 vmp 接线落到错误配置基线。
- `vmp` 工具仓库已迁移到 workspace 内独立仓库 `vmp`，远端为 `git@github.com:menghaocheng/vmp.git`，分支 `main/1.0`，已实现并推送 `rx-cave` 注入模式。
- `android10/vendor/hello/arm96/res/bin/vmpacker` 已同步到支持 `-inject-mode rx-cave` 的新版 host 工具，但当前仍为 untracked；是否 vendoring 到 arm96 仓库需在实施前确认。

**核心结论**
- 不建议对 `libarm96` 使用 vmp upstream 默认 `PT_NOTE -> RX PT_LOAD` 模式；该模式会新增/改写可执行 program header，违背 V0.67 之后的 no-new-PT_LOAD 约束。
- 推荐路径是复用 V0.67 RX island，调用 `vmpacker -inject-mode rx-cave` 将 VM runtime payload 写入既有可执行 filler 区间。
- 第一版必须默认关闭，仅作为 opt-in 实验链路；不开启时构建产物、patch diff、E4903/E650x/E67xx 口径应与当前主线保持一致。

**建议 pipeline 顺序**
```text
1. V0.49 输入指纹与 patch 前镜像断言
2. 固定 offset patch / 授权与品牌擦除
3. V0.67 relayout，释放 RX island
4. V0.65/V0.66 core stub 注入
5. VMP rx-cave 注入（新增）
6. V0.68 scatter decoy 注入
7. A96D obfuscated padding
8. E4903/E59xx 结构化白名单核验
9. encrypt_core.py AES-GCM 加密
```

**配置规划（默认关闭）**
```text
VMP_CORE_ENABLE=0
VMP_CORE_MODE=rx-cave
VMP_CORE_BIN=res/bin/vmpacker
VMP_CORE_TARGETS=
VMP_CORE_STRICT_VERIFY=1
VMP_CORE_MIN_FREE=0x1000
VMP_CORE_TOOL_SHA256=76565dca6f3f4654db2791abc0c642f290e93d3b3629da728882cd013c93ddf5
```
- `VMP_CORE_TARGETS` 首版建议只支持显式地址范围，例如 `0xSTART-0xEND:name`；strip/prebuilt `libarm96` 不应依赖符号名选择函数。
- 后续可扩展多目标列表，但 V0.59.0-vmp 原型只允许 1 个显式 target range，先验证链路闭环；禁止自动扫描/批量 VM 化。
- 首版 target 优先选择 V0.66 Shadow Core Guard / stub 侧启动期安全判定块，即 `hardening/src/stub.s` 注入到 `libarm96` 后，shadow expire / fail-rate / witness 相关的短小 guard decision range。
- 若 shadow guard range 因手写汇编、特殊指令或 vmp 指令支持限制无法翻译，本版应只保留 pipeline/gate 接线，不得为了凑 target 转向原始 `libarm96` 翻译器热路径。

**实施拆分**
- [ ] **P0-1 配置与工具门禁**（`hardening/src/version.txt` / `main.py`）
  - 增加 `VMP_CORE_*` 配置，默认 `VMP_CORE_ENABLE=0`。
  - 开启时校验 `res/bin/vmpacker` 存在、可执行、SHA-256 匹配，且 `--help` 包含 `-inject-mode` / `-payload-off` / `-payload-va` / `-payload-max`。

- [ ] **P0-2 RX island 地址口径补齐**（`relayout_v067.py` / `main.py`）
  - `RelayoutResult` 增加 `island_va`，计算口径为 `rx.p_vaddr + (island_off - rx.p_offset)`。
  - 若不修改 dataclass，也可在 `main.py` 用现有 `_file_off_to_va()` 临时计算，但最终建议让 relayout helper 直接暴露。

- [ ] **P0-3 island 子分配器**（`hardening/src/main.py`）
  - 从 V0.67 island 中划分 `stub reserved range`、`vmp payload range`、`scatter decoy ranges`。
  - VMP 分配必须避开 stub、PHDR/SHDR、已登记 patch 点和后续 decoy 预留区；空间不足直接 fail-fast。

- [ ] **P0-4 vmp 注入 helper**（`hardening/src/main.py`）
  - 新增 `_apply_vmp_core()` / `_parse_vmp_targets()` / `_select_vmp_cave()` 等小 helper。
  - 通过临时文件调用：`vmpacker -inject-mode rx-cave -payload-off <off> -payload-va <va> -payload-max <size> -addr <targets> -o <out> <in>`。
  - 注入成功后把输出文件读回 `patched`，并记录 payload range 与被 trampoline 改写的目标函数范围。

- [ ] **P0-5 E4903/E59xx 门禁扩展**（`hardening/src/main.py` / host verify）
  - 将 VMP payload 写入范围、目标函数 trampoline 改写范围加入结构化 whitelist；禁止把整个 RX island 粗放白名单化。
  - 新增 E59xx gate：工具缺失/能力不符、island 不存在、payload 越界、range 重叠、PHDR 漂移、vmp 翻译失败、白名单不闭合均 fail-fast。

- [ ] **P1-1 目标选择与性能基线**
  - 首批只保护 1 个显式低频 target，首选 V0.66 Shadow Core Guard / stub 侧启动期安全判定块。
  - 实施时先通过 `stub.meta` / `objdump` 定位 stub 注入 VA 与 file offset，再从 shadow expire / fail-rate / witness 相关逻辑中选择短小连续 guard decision range。
  - 不 VM 化整个 stub entry trampoline；不覆盖保存/恢复寄存器序言、branch-back 尾段或复杂 syscall/时间读取片段。
  - 禁止首版覆盖 JIT/DBT 主循环、syscall fast path、signal/memory helper、code cache lookup 等热路径；若首选 target 不可翻译，允许本版仅交付默认关闭的 pipeline/gate 接线。
  - 未完成 VMP on/off A/B 冷启动对比前，不得把 `VMP_CORE_ENABLE` 改为默认开启。

**建议新增 gate 编号**
- `E5901`: vmpacker 缺失、不可执行、SHA-256 不匹配或 CLI 不支持 `rx-cave`。
- `E5902`: `VMP_CORE_ENABLE=1` 但 V0.67 RX island 不存在，或无法计算 `island_va`。
- `E5903`: VMP payload range 超出 island 可用空间，或与 stub/decoy/reserved range 重叠。
- `E5904`: VMP 注入后 `e_phnum`、`PT_LOAD` 数量、RX LOAD 数量或 RWX 权限状态发生漂移。
- `E5905`: VMP diff 未被 payload/trampoline 结构化白名单完全覆盖。
- `E5906`: vmpacker 翻译目标失败（unsupported instruction、外部 branch、range 不完整等），不得产出半保护文件。

**验收判据（Acceptance Gate）**
- [ ] `VMP_CORE_ENABLE=0`：构建输出与当前主线行为一致，不触发 E59xx，不改变 mustpass 口径。
- [ ] `VMP_CORE_ENABLE=1`：`./make.sh --build cx` 成功，E4903/E650x/E67xx/E59xx 均通过。
- [ ] 静态结构：`PT_LOAD` 数量不变，RX LOAD 数量不变，RW LOAD 不变 RWX，payload 完全位于 V0.67 RX island。
- [ ] VMP 可观测：payload range 不再保持全 NOP/filler，目标函数入口出现预期 trampoline，且 branch 目标合法。
- [ ] 设备主链：`arm96 -v` 仍严格 1 行，`arm96 -i` 仍严格 2 行，`arm96`/`-h`/`-a` 仍静默，`verify_mustpass.sh` 达到 `12 PASS / 0 FAIL / 0 SKIP`。
- [ ] 性能回归：必须保留 VMP off baseline 与 VMP on 实验组对比；至少覆盖 `app_process32` 启动、`zygote_secondary` 稳定性、一个典型 32 位 app 冷启动；不得出现 ANR、zygote restart loop 或明显冷启动劣化。
- [ ] 默认开关约束：未完成上述 A/B 性能验收前，`VMP_CORE_ENABLE` 必须保持默认关闭；若目标命中热路径或性能不可接受，必须回退 target 选择或仅保留 pipeline/gate。

**回滚策略**
- 单开关回滚：`VMP_CORE_ENABLE=0`。
- 回滚后不得要求删除 `res/bin/vmpacker` 或改变 relayout/stub/decoy 配置；vmp 接线必须成为纯 opt-in 分支。
- 若 `res/bin/vmpacker` 不纳入 arm96 仓库，则构建脚本需在开启时明确报错并提示从 `vmp/build/vmpacker` 同步，不能静默跳过保护。

**本版本不做**
- 不对 `libarm96` 使用 `PT_NOTE` 注入模式。
- 不新增第 9 个 `PT_LOAD`，不把 RW LOAD 改为 RWX。
- 不大面积 VM 化翻译器核心热路径。
- 不修改 loader 的 AES-GCM 容器格式、memfd/fexecve 执行模型或 binfmt `POC` 规则。
- 不把 vmp 应用于 `arm96` loader、`arm96d`、`arm96server`；这些组件如需 VM 化，应在单独版本中按各自 integrity/hash/OLLVM 约束重新评估。

#### V0.62.0（arm96server native 反 trace / 反 hook 增强）
> **目标**: 在不侵入预编译 `libarm96`、不改变 loader `fexecve` 启动模型的前提下，把 blackobfuscator 中适合 arm96 架构的 native 运行时反 hook / 反 trace 细节迁移到 `arm96server` 侧，补齐改名 Frida、可疑 RWX 匿名映射、server 运行中被 inline patch 等外部可观测风险。
> **状态**: **Done (2026-05-23)**

**设计边界**
- `arm96` loader 只是 `libarm96` 的启动封装，拉起 core 后自身不再作为长期检测主体存在；V0.62 不在 loader 中加入周期线程。
- `libarm96` 是预编译二进制且承载翻译器核心逻辑；V0.62 不向 `libarm96` 注入线程、信号处理、inline patch 或 hook 检测，避免引入稳定性风险。
- 长期反 trace 主体仍为 `arm96server`：继续沿用 V0.38 的 PID tracking + `TracerPid` 轮询 + guardian cascade 架构，不回退到 `PTRACE_SEIZE` app。
- blackobfuscator 仅作为 native 侧策略参考；不迁移 Java/DEX hook、反射保护、APK 签名检测等与 arm96 当前核心目标无关的策略。

**实施项（来自 blackobfuscator native 策略，按低侵入优先）**
- [x] **P0-1 Frida 默认端口探测**（`arm96server.c`）
  - 在 server 周期扫描中增加本机非阻塞连接探测：`127.0.0.1:27042`、`127.0.0.1:27043`。
  - 命中后优先写 V0.51 fault stamp，并可按配置选择 kill 可疑进程或仅进入延迟失效链。
  - 目标是补足 `cmdline` 改名后的 frida-server 监听默认端口场景。

- [x] **P0-2 匿名 RWX mapping 检测（谨慎白名单）**（`arm96server.c`）
  - 由 `arm96server` 扫描 tracked app 的 `/proc/<pid>/maps`，识别 `rwxp` 且无 pathname 的匿名映射。
  - 首版只作为强可疑信号写 fault stamp / 记录命中，不直接扩大 kill 面；待设备验证误报后再决定是否 kill app/tracer。
  - 必须增加 arm96/translator/JIT/memfd 合法映射白名单，避免误杀正常翻译缓存或合法 JIT 场景。

- [x] **P0-3 arm96server 自身关键函数入口巡检**（`arm96server.c`）
  - 借鉴 blackobfuscator 的 prologue check，只在 `arm96server` 自身检查关键函数入口是否被 direct branch / trampoline 覆写。
  - 首批候选：`scan_and_track()`、`scan_and_kill_debuggers()`、`read_proc_status()`、`spawn_guardian()`、`v051_emit_fault_stamp()`。
  - 检测对象限定为 server 自身，避免触碰 `libarm96`；命中后写 fault stamp、kill tracked apps、触发 guardian 重启或受控退出。

- [x] **P1-1 随机化 scan cadence + fake check dispatch**（`arm96server.c`）
  - 保持平均扫描间隔约 3s，但加入小范围抖动（例如 2~4s），降低固定节拍被观察和规避的收益。
  - 增加若干无害 fake check（假 maps / 假端口 / 假路径），并用轻量 dispatch 表混入真实检查路径，提高自动化 patch 定位成本。
  - 不改变 PID tracking 主语义，不引入会影响 app 信号处理的 ptrace 行为。

- [x] **P1-2 tamper 处置形态收口**（`arm96server.c` + loader fault stamp 消费链）
  - 对强异常（server 自身入口被 patch、Frida 端口 + tracked app 异常组合、可疑 RWX + TracerPid 组合）统一写 V0.51 fault stamp。
  - 避免简单 `exit/abort` 形成稳定定位点；优先复用现有延迟失效 / tracked app 清理链。
  - 命中前清理本地敏感临时数据、tracked pid 文件写入保持最小化，避免异常处置扩大可观测信号。

**实现结果（2026-05-23）**
- [x] 代码层实现已完成：`hardening/src/arm96server.c` 已接入端口探测、匿名 RWX 采样、prologue 基线巡检、scan 抖动与 fake dispatch、tamper 收口处置。
- [x] 本地构建通过：`shell/make.sh --build vc`（runtime artifacts）成功。
- [x] 设备侧 mustpass 主链通过：`verify_mustpass_host.sh` 在设备 `192.168.2.19:5004` 上达到 `12 PASS / 0 FAIL / 0 SKIP`（TC-000~TC-011、TC-013~TC-016）。
- [ ] host-orchestrated 过期重启附加项（脚本尾部 `TC-007 Expired Reboot`）需切换 5 分钟过期构建后复核；当前设备处于未过期态。

**实施前检查清单（Preflight）**
- [ ] 明确 `rwxp anonymous` 白名单：确认 `libarm96`、memfd、sidecar/JIT/cache 在真实设备上的 `/proc/<pid>/maps` 形态，先采样再启用阻断。
- [ ] 明确端口探测误报策略：默认端口命中是否直接 kill、仅 fault stamp，或结合进程名/TracerPid/maps 二次确认。
- [ ] 明确 server prologue 检测兼容性：静态链接、clang/OLLVM、不同优化级下首指令形态是否稳定；需要构建期生成或校验期自适应，避免硬编码单一指令模式。
- [ ] 明确 scan 抖动上限：不能扩大 TracerPid 检测窗口到可利用程度；平均开销与当前 3s 扫描保持同级。
- [ ] 明确与 guardian / hollow server / token hash 的触发顺序，避免自检命中后出现重启风暴或 zygote_secondary 循环抖动。

**验收判据（Acceptance Gate）**
- [x] 正常设备、正常 32 位 app：无新增误杀、无 zygote/service 重启风暴，`verify_mustpass_host.sh` 主链不退化（2026-05-23）。
- [ ] 改名 frida-server 监听 `27042/27043`：即使 `/proc/*/cmdline` 不含 `frida`，也必须触发 fault stamp 或受控处置。
- [ ] tracked app 出现可疑匿名 `rwxp` 映射：首版至少可观测命中并进入配置化处置路径；白名单样本不得误报。
- [ ] patch `arm96server` 关键函数入口为 branch/trampoline：server 自检应命中，不能安静继续提供“空壳保护”。
- [x] 扫描节拍随机化后，TracerPid attach 场景仍能在预期窗口内 kill tracer + app，guardian cascade 行为不退化（TC-008/TC-009 验证通过）。

**本版本不做**
- 不修改或注入 `libarm96`。
- 不恢复对 app 的 `PTRACE_SEIZE`。
- 不迁移 blackobfuscator 的 Java/DEX/反射保护策略。
- 不以 RWX 检测单独作为强杀条件发布；必须先经过设备采样与白名单验证。


#### V0.64.0（SIGSTOP 互检 + Action Token 去常量化 + 残留 IOC 收口）
> **目标**: 针对外部对抗 AI 在 V0.63.0 上 1 小时定向攻击后唯一摸到边缘的攻击路径（`kill -STOP arm96server` 让守护冻结但 init 不重启），以及静态可见的固定 action token、TracerPid/TGENC022 等残留 IOC，做一次定点收口；不展开 V0.50 持续 challenge-response（顺延到 V0.65 之后）。
> **状态**: **Partially Done (2026-05-24, 主链完成，剩余少量 IOC 尾项待收口)**
> **触发依据**: `.claude/chats/Adversarial_analysis/analysis-arm96-V0.63.0.md` §6 实测攻击 14 变体表，最佳 v5（stop shmkd + SIGSTOP arm96server）162s / 180s 未达标，剩余收敛点。

**设计边界**
- 代码改动范围**优先限定**在 `android10/vendor/hello/arm96/` 下；FW（`frameworks/base`）仅在 token 协议升级确实必须时才动，且需单独列出影响清单经主人确认。
- 不修改预编译 `libarm96`。
- 不引入新的常驻线程；新增检测全部走 `arm96d` / `arm96server` 现有主循环。
- 不动 V0.62 已稳定的端口探测 / RWX 扫描 / prologue 巡检主体，只在其 dispatch 链上挂新的检测项。
- 攻击者所谓 `vpkj.jar` 与 `shmkd` 耦合均与本项目无关（外部插件 + 假相关），不纳入本版本范围。

**外部观察到的真问题（已核实）**
1. `arm96server` 被 `kill -STOP` 后进入冻结态，init 仅响应 SIGCHLD，不会重启 `critical` 服务；该窗口内 PID tracking / TracerPid 巡检全部失效，是 v5 攻击成立的根因。
2. `arm96d` 中 6 个 action token（`461272e9` / `008a1be2` / `28efa08b` / `fedd7992` / `683fcfcf` / `e1afa2bd`）为固定 4 字节常量明文，且 `normalizeActionToken()` 同时接受 hex 与 plaintext 双形态（`on_pre_start_proccess` 等），可被静态 grep 直接提取并回放（参考 [Arm96dService.cpp#L700-L735](android10/vendor/hello/arm96/src/arm96d/service/Arm96dService.cpp#L700)）。
3. `arm96d` 自身缺少 V0.40 给 `arm96server` 做的那一套加固（OLLVM CFF/SUB、字符串加密、.text 自校验、token hash 融入解密链均未应用）。
4. release artifact 中仍有 grep-able IOC：`TracerPid`、`/proc/self/status` 明文（loader）、`TGENC022` 容器 magic 明文（`libarm96` 前 8 字节）、`verify_signature` 明文（`arm96_sidecar_gen`）。
5. ftrace 类内核观测手段不在本项目防御面内（属用户态反调试范畴之外），本版本不展开对抗，仅在文档上明确边界。

**实施项（按优先级）**

- [x] **P0-1 SIGSTOP 互检（arm96d ↔ arm96server）**（`arm96d` + `arm96server.c`）
  - 在 `arm96d` 主循环每个 tick 读 `/proc/<arm96server_pid>/status` 的 `State:` 字段，命中 `T (stopped)` / `t (tracing stop)` 立即触发 V0.51 fault stamp + tracked apps 清理 + 受控处置。
  - 在 `arm96server` 主循环同样轮询 `arm96d` 的 `State:`，命中后同等处置。
  - PID 发现：复用 V0.32 已有的双向互保进程绑定信息；如缺则按服务名 `pidof` 兜底。
  - 探测节拍随 V0.62 jitter 一起抖动，避免给攻击者新的固定时钟。
  - 阈值：连续 N=2 次命中再触发，避免极端调度抖动误报。

- [x] **P0-2' Action Token 去常量化（native 单边最小改动）**（`arm96d`）
  - 取消 `normalizeActionToken()` 对 hex 与 plaintext 同时接受的双形态，对外仅暴露一种形态。
  - 当前已落地 keyed-tag action 判定（`V051_TAG_ACTION_*`），旧 6 个固定 4B token 与 action 明文不再作为静态常量回放入口。
  - 保持 FW 协议兼容（未引入三元组 `{action_id, nonce, hmac}` 改造）；`arm96d` 侧已叠加调用者 UID/PID 门禁与动作频率窗口限制。

- [ ] **P0-2 Action Token 全量协议升级（含 FW 三元组）**（`frameworks/base` + `arm96j` + `arm96d`）
  - 该项属于跨 FW 改动，当前版本未做，留待 V0.65 前后按单独需求推进。

- [x] **P0-3 arm96d 静态加固关键链路落地（`arm96d` 构建脚本 + 源码）**
  - 接入 OLLVM CFF / SUB / BCF（与 `arm96server`、loader 同档位）。
  - 接入 V0.53 字符串加密池：6 个 token（如保留 plaintext 形态）、action 名、关键日志、`/proc/...`、调试旁路字符串。
  - 加入 `.text` 自校验：启动期与心跳期各做一次 SHA-256 比对，命中差异走 V0.51 fault stamp。
  - 把 `arm96d` 的 `.text` 哈希加入 V0.27 密钥融合链作为第 8 层 witness（与 `arm96server.text_sha256` 同列），未通过则 libarm96 解密失败。

- [x] **P1-1 残留 IOC 字符串加密（已完成主体，保留 1 条残留 IOC）**（loader + `main.py` 后处理）
  - loader 中 `TracerPid:` / `/proc/self/status` / `/proc/self/maps` 等明文走 V0.53 解码池运行时还原。
  - `arm96_sidecar_gen` 中 `verify_signature` 同处理。
  - `libarm96` 容器 `TGENC022` 8 字节 magic 改为 `HMAC(magic_seed, build_id)[0:8]`，loader 侧同步改读法；保持 16 字节 header 长度不变以避免 V0.39 obfuscation padding 偏移变化。
  - 当前状态：`TGENC022` 与 `verify_signature` 已收口，loader 逻辑路径中的相关明文已基本去除；`strings arm96` 仍有 1 条 `/proc/self/maps` 命中（静态 libc 残留文本，非业务逻辑路径），待后续专项收口。

- [x] **P1-2 fault stamp 触发面扩充**（V0.51 消费链复用）
  - SIGSTOP 互检命中、token HMAC 校验失败、`.text` 自校验失败、`arm96d` prologue 巡检失败均统一写入 V0.51 fault stamp，复用既有延迟失效路径，不引入新的处置形态。

**实施前检查清单（Preflight）**
- [ ] 确认 `arm96d` ↔ `arm96server` 当前是否已交换 PID（V0.32 双向互保结构），若未持久化则定 PID 发现兜底方案。
- [ ] 确认 `arm96d` 当前构建是否已接入 OLLVM 工具链（`build_arm96d_arm64.sh` 或等价脚本），未接入则先打通构建侧。
- [ ] 设备采样 `arm96d` / `arm96server` 的 `/proc/<pid>/status` 在正常负载下的 State 抖动情况，确认 N=2 阈值不会误报。
- [ ] 确认主人是否授权动 FW 那一段（决定走 P0-2 还是 P0-2'）。
- [ ] 确认 release 流程对 `PLATFORM_BINDING` / `OBFUSCATE_LOADER` 等关键 flag 是否有强制断言（防止评估版误外发，规避类似攻击者拿到非 release 构建的情形）。

**验收判据（Acceptance Gate）**
- [x] `verify_mustpass.sh` 12/12 不退化（TC-001~TC-010 + 011/013~016）。
- [x] 设备侧 `kill -STOP` 互检路径已在 `arm96d`/`arm96server` 双侧实现并接入 fault stamp 处置链。
- [ ] release artifact `strings -a arm96 | grep -E 'TracerPid|/proc/self/(status|maps)'` 命中 0 行。（当前仍有 1 条 `/proc/self/maps` 静态 libc 残留）
- [x] release artifact `head -c 16 libarm96 | xxd` 不再出现 ASCII `TGENC022`。
- [x] release artifact `strings -a arm96_sidecar_gen | grep verify_signature` 命中 0 行。
- [x] release artifact `strings -a arm96d | grep -E '461272e9|008a1be2|28efa08b|fedd7992|683fcfcf|e1afa2bd|on_pre_start_proccess|on_proccess_started|on_proccess_died'` 命中 0 行（按 P0-2' 路径实现）。
- [ ] 模拟 token 回放（旧 nonce/HMAC）全链路拒绝验证（依赖 P0-2 全量协议升级，当前未做）。
- [x] `arm96d` `.text` witness 已并入解密链；构建/运行双端口径一致。

**当前结论（2026-05-24）**
- V0.64 主目标已完成：SIGSTOP 互检、action 常量回放入口收口、`arm96d` `.text` witness 入链、5 分钟版本与常规版本设备侧 mustpass 均通过。
- 剩余尾项有 2 个：
  - `arm96` 中 1 条 `/proc/self/maps` 静态 libc 残留 IOC。
  - P0-2 全量 FW 协议升级（nonce/hmac 三元组）尚未实施。

**本版本不做**
- 不实现 V0.50 周期 challenge-response（顺延到 V0.65 之后）。
- 不动 `libarm96` 二进制（包括不在其中加 SIGSTOP 自检）。
- 不引入 ftrace / 内核观测对抗。
- 不修改 V0.62 已上线的 Frida 端口探测 / RWX 扫描 / prologue 主体逻辑。
- 不重构 `arm96j.jar` / `Arm96Service` 整体架构；如确实要动 FW，仅做 token 协议三元组化的最小补丁。

**对接后续版本的伏笔**
- P0-1 的 SIGSTOP 互检与 P0-3 的 `arm96d` `.text` witness，是后续持续 challenge-response 的两个天然挂载点：互检通道可升级为带应答的 challenge，witness 可升级为 challenge 输入的一部分。V0.64 的协议接口尽量保留 nonce 字段，便于在 V0.65.0 注入机制稳定后继续推进。

#### V0.65.0（libarm96 Stub 注入机制）
> **目标**: 把 `hardening/src/stub.s` 从“只编译不消费”的占位资源，升级为注入到明文 `libarm96` 原始 ELF 中的可执行逻辑岛；先交付可校验、可回滚的注入框架，不在本版本承载复杂概率防护业务。
> **状态**: **Done (2026-05-25)**

**定位调整**
- `stub.s` 不是给 `arm96` loader 链接使用；loader 继续保持解密 + memfd/fexecve 的启动封装职责。
- `stub.s` 面向 `libarm96`：由 `compile_stub.sh` 生成 `hardening/res/stub.bin`，再由 `hardening/src/main.py` 在加密前注入到 patched plaintext core。
- V0.65.0 只验证“能安全向原始二进制添加新逻辑”，概率性/多层次防护业务留到后续版本，避免注入机制风险与策略风险叠加。

**设计边界**
- 只改 `android10/vendor/hello/arm96/` hardening 构建链；原则上不动 `frameworks/base`、`arm96j`、`arm96d` 协议。
- 第一阶段优先使用已有可执行 `PT_LOAD` 内的 code cave；不新增 `PT_LOAD`，不扩大 ELF program header，避免引入对齐、装载权限和 boot 稳定性风险。
- 注入点首选 ELF entry hook 或经 IDA/objdump 确认的稳定早期 hook 点；入口被覆盖指令必须恢复语义后再 branch back。
- stub 首版必须 position-independent、无 libc 依赖、无 syscall 依赖、保存/恢复触碰的寄存器，行为保持 inert 或最小 witness，方便隔离验证。
- 所有注入改写必须进入 V0.49 白名单 diff 机制，任何额外 offset 改动都 fail-fast。

**实施项（V0.65.0 范围）**
- [x] **P0-1 配置与版本号**（`hardening/src/version.txt` / `make_common.sh`）
  - 推进 `CUSTOM_VERSION=V0.65.0`。
  - 新增 `CORE_STUB_INJECT_ENABLE`、`CORE_STUB_INJECT_MODE`、`CORE_STUB_CAVE_OFFSET`、`CORE_STUB_CAVE_SIZE`、`CORE_STUB_ENTRY_EXPECT`、`CORE_STUB_STRICT_VERIFY`。
  - 默认保留严格校验；注入开关允许 profile 级关闭，作为 V0.64 行为回滚路径。

- [x] **P0-2 stub 构建元数据**（`hardening/compile_stub.sh`）
  - 继续输出 `hardening/res/stub.bin`。
  - 新增 `hardening/res/stub.meta`，记录 stub size、SHA-256、branch-back placeholder offset、marker、预期架构和最大 branch 距离。
  - 编译器缺失时仍可 fail-soft，但若 `CORE_STUB_INJECT_ENABLE=1` 则必须 fail-fast。

- [x] **P0-3 最小 stub payload**（`hardening/src/stub.s`）
  - 从占位 `_start + b return_loc + 0x14000000` 改为真实 AArch64 注入 stub。
  - 保存/恢复必要寄存器；执行最小 witness/no-op marker；等价执行被覆盖入口指令；跳回 `entry + overwritten_len`。
  - 保留显式 branch-back placeholder，由 `main.py` 按实际注入位置重写。

- [x] **P0-4 ELF 注入工具层**（`hardening/src/main.py`）
  - 解析 ELF64 header / program header，校验 `e_machine=AARCH64`、entry VA、可执行 `PT_LOAD` 边界。
  - 实现 VA ↔ file offset 映射、AArch64 `B` 指令 encode/decode、branch reachability 检查。
  - 按配置或自动探测选择 PF_X code cave；校验 cave 内容为全 0 或已知 filler，大小满足 `stub.bin` + 对齐要求。

- [x] **P0-5 注入流程接入**（`hardening/src/main.py:patch_binary()`）
  - 推荐顺序：复制 input → V0.49 preimage 校验 → 现有 fixed patch → Tango 标识 wipe → V0.65 stub 注入 → append 30KB obfuscated padding → V0.49 whitelist diff。
  - 注入改动包括 entry branch、stub 写入、branch-back placeholder 重写，全部纳入 `offset_changes`。
  - 新增 gate code：`E6501` malformed ELF、`E6502` cave mismatch、`E6503` branch range failure、`E6504` stub metadata mismatch、`E6505` post-injection verify failure。

- [x] **P0-6 静态 post-verify**（`hardening/src/main.py` / 构建日志）
  - patch 后、encrypt 前复核 entry branch 指向 stub cave。
  - 复核 stub branch-back 目标等于原 entry 后续地址。
  - 复核 stub SHA / metadata / changed offsets 与白名单一致。
  - 构建日志输出简短摘要：entry VA、cave file offset、stub size、branch-back target、gate OK。

- [x] **P1-1 文档与回滚口径**（`hardening/README.md` / `.claude/context/tango_hardening.md`）
  - 明确 `stub.s` 当前用途已从 dormant placeholder 改为 `libarm96` 注入 payload。
  - 明确 `CORE_STUB_INJECT_ENABLE=0` 必须完整绕过注入，构建与运行行为回到 V0.64 口径。

**验收判据（Acceptance Gate）**
- [x] 单独运行 `compile_stub.sh` 能生成 `stub.bin` + `stub.meta`，metadata SHA 与文件一致。
- [x] 单独运行 `main.py --input <translator> --output <tmp-libarm96>` 时，V0.49 + V0.65 gate 均通过。
- [x] `objdump`/`llvm-objdump` 检查 patched plaintext `libarm96`：entry branch 指向 stub，stub branch-back 指向原始 continuation，指令合法。
- [x] `CORE_STUB_INJECT_ENABLE=0` 构建可回滚，不触发 E650x，不改变既有 mustpass 行为。
- [x] `CORE_STUB_INJECT_ENABLE=1` 全量构建通过：`cd android10/vendor/hello/arm96 && ./make.sh --build cx`。
- [x] 设备 smoke 不退化：`arm96 -v` 仍 1 行，`arm96 -i` 仍 2 行，`arm96 -h`/无参静默，binfmt flags 仍为 `POC`。
- [x] 注入产物会被 static post-verify 复核（entry branch / stub body / branch-back / 元数据）；异常进入 E650x fail-fast。

**V0.65 运行期补充结论（2026-05-25）**
- [x] `zygote_secondary` 重启风暴与 V0.65 注入机制非一一对应；直接探针显示 `/system/bin/app_process32` 在故障态返回 `127`，属于翻译/解密前置失败信号。
- [x] 设备 `/vendor/bin/arm96d` 与构建期 witness 漂移会触发上述失败；同步 `arm96d` 后，`app_process32` 恢复正常返回，`zygote_secondary` 恢复 `running`。
- [x] 当前剩余独立问题为 CAS 服务 `Could not setThreadPoolConfiguration: -25`（binder protocol mismatch），需在 V0.67+ 单独收口，不作为 V0.65 注入机制阻断项。

**本版本不做**
- 不实现周期 challenge-response；该项在 V0.65.0 注入机制稳定后重新排期。
- 不在 stub 中实现概率性奇怪故障、随机数据破坏或广义系统处置。
- 不新增 ELF executable segment，不改 program header 数量。
- 不把 stub 逻辑链接进 loader。
- 不修改 `libarm96` 外部部署路径、密文容器格式或 loader memfd 执行模型。

#### V0.66.0（Shadow Core Guard / 暗线授权兜底）
> **目标**: 基于 V0.65.0 已稳定的 `libarm96` stub 注入机制，把独立于明线 `EXPIRE_EPOCH` 的 shadow 授权期限落到 `hardening/src/stub.s` 中；用于在明线授权被绕过、或暗线先于明线到期的产权保护场景下，提供 core-local 的受控兜底失效链。
> **状态**: **Done / Device Verified (2026-05-27)**

**定位与边界**
- `stub.s` 继续作为注入到明文 `libarm96` 原始 ELF 的早期执行逻辑岛，不把该策略放回 loader 明线授权链。
- Shadow 期限必须独立于 `EXPIRE_EPOCH`；明线授权、AES 解密 key fusion、setitimer 周期自杀链保持既有语义。
- 外部表现允许低信息量、低可归因，但内部策略必须可配置、可复现、可验收；禁止做无界内存破坏、随机篡改翻译结果或不可诊断的广义系统处置。
- 失败动作首版限制为受控静默退出或短延迟后退出，作用域限制在 `libarm96` 翻译器启动链内。

**Shadow 时间与固定失败率策略**
- 新增独立暗线时间：`SHADOW_EXPIRE_EPOCH`（构建端拆分/混淆后喂给 stub，不以直白单常量落入二进制）。
- 新增固定失败率：`SHADOW_FAIL_RATE`，构建端钳位到 `0..100`，并生成 `STUB_SHADOW_FAIL_RATE`。
- 已移除旧方案中的 `BUILD_EPOCH`、`SHADOW_PREBUILD_FAIL_RATE`、`SHADOW_FAIL_CAP`、预构建/后构建分叉和按天增长曲线。

最终运行公式：
```text
if SHADOW_GUARD_ENABLE == 0:
    p_fail = 0
else if now <= SHADOW_EXPIRE_EPOCH:
    p_fail = 0
else:
    p_fail = SHADOW_FAIL_RATE
```

**确定性触发模型**
- 不使用真随机；触发值由 `day_index`、`pid`、stub marker、构建期 secret、可选入口 VA 等混合派生。
- 同一构建、同一设备/进程、同一时间桶内尽量保持可复现；跨日期/进程自然漂移，降低静态定位和一次性绕过收益。
- 判定口径固定为 `roll % 100 < p_fail`；`p_fail` 由暗线是否到期和 `SHADOW_FAIL_RATE` 唯一决定。

**实施项（V0.66.0）**
- [x] **P0-1 配置项与生成链**（`hardening/src/version.txt` / `make_common.sh` / `compile_stub.sh`）
  - 新增 `SHADOW_GUARD_ENABLE`、`SHADOW_EXPIRE_EPOCH`、`SHADOW_FAIL_RATE`。
  - `make_common.sh` 生成 `gen/version.py` / `gen/version.h` 中的钳位后 `SHADOW_FAIL_RATE`。
  - `compile_stub.sh` 生成 `hardening/res/stub_config.inc`，仅包含 `STUB_SHADOW_GUARD_ENABLE`、`STUB_SHADOW_FAIL_RATE`、`STUB_SHADOW_EPOCH_*`、`STUB_GUARD_SECRET`。

- [x] **P0-2 stub 侧时间与固定失败率逻辑**（`hardening/src/stub.s`）
  - 在现有 entry trampoline 中加入 shadow epoch 还原、当前时间读取、天数桶参与 deterministic roll、固定失败率触发判定。
  - 不再读取/比较 build epoch，不再计算 day-growth curve。
  - 保持 position-independent；保存/恢复触碰寄存器；原入口指令语义与 branch-back 不退化。

- [x] **P0-3 注入与门禁扩展**（`hardening/src/main.py`）
  - 更新 `CORE_STUB_CAVE_SIZE` 推荐值和 stub metadata 校验，覆盖更大 stub body。
  - V0.49 whitelist diff 必须纳入新增 stub body 改动；异常仍走 E650x fail-fast。
  - 增加 shadow guard 静态 post-verify：配置存在性、stub size、marker、branch-back、触发路径指令合法性。

- [x] **P0-4 验证脚本与口径收敛**
  - `verify_shadow_guard_host.sh` 按固定失败率模型验证公式与 deterministic roll。
  - `verify_shadow_guard_device_sample.sh` 输出 total / triggered 双口径；当前 CI/设备采样可用 `EVAL_MODE=triggered` 避免 app 启动噪声误伤。
  - `verify_mustpass_host.sh` 增加 TC-017 host smoke 与 TC-018 device sample 入口。

**V0.66 验收判据（Acceptance Gate）**
- [x] 未到 shadow epoch：`arm96 -v/-i`、binfmt 32 位 app 启动、mustpass 主链均不退化。
- [x] 固定失败率公式：host 侧 `./verify_shadow_guard_host.sh --samples 5000` 通过；`fail_rate(cfg/stub)=20/20`。
- [x] 构建通过：`cd android10/vendor/hello/arm96 && ./make.sh --build cx`。
- [x] 生成物收敛：`stub_config.inc` 不再包含 `STUB_BUILD_EPOCH_*`、`STUB_SHADOW_PREBUILD_RATE`、`STUB_SHADOW_FAIL_CAP`。
- [x] 旧增长曲线残留扫描通过：维护源码和生成配置中无 `BUILD_EPOCH` / `SHADOW_PREBUILD_FAIL_RATE` / `SHADOW_FAIL_CAP` / `STUB_BUILD_EPOCH` 相关引用。
- [x] 设备回归通过：测试设备 `192.168.2.19:5004` 三件套哈希与本地一致，重启 `zygote_secondary` 后设备 mustpass `12 PASS / 0 FAIL / 0 SKIP`。
- [x] V0.66 设备采样通过：未过暗线时 `p_fail=0`，`ATTEMPTS=30` 得到 `pass=30 fail=0 launch_fail=0 runtime_fail=0`，total/triggered 均 `observed=0%`。
- [x] V0.65 注入 gate 不退化：entry branch、stub body、branch-back、metadata、白名单 diff 均可静态复核。

**后续版本预留（V0.67+）**
- CAS `setThreadPoolConfiguration:-25` 路径单独收口，不和 V0.66 shadow guard 策略风险叠加。
- 周期 challenge-response 在 V0.66 shadow guard 验证后重新排期；可复用 V0.64 SIGSTOP 互检和 `arm96d` witness 作为天然挂载点。

#### V0.67.0（Full Layout Repack / RX Gap Expansion）
> **目标**: 在不新增 `PT_LOAD`、不把 RW 段改成 RWX 的前提下，通过文件布局重排释放原始 ELF 中 R-X 段后的虚拟地址空洞，为后续多策略 stub / multi-island 提供约 64KB 的可执行空间，突破当前单一 code cave 剩余空间限制。
> **状态**: **Done / Mainline Verified (2026-05-28)**

**规划依据（当前 ELF 布局）**
- 原始 `res/bin/tango_translator.cheersu` 与 patched plaintext `build/out/cx/bin/libarm96.preencrypt` 的 program headers 当前保持一致，说明现有 V0.65/V0.66 注入只改内容、不改映射布局。
- 当前 PHDR table 位于 `0x40`，8 个 program header 共 `0x1c0` 字节，结束于 `0x200`；`.note.gnu.build-id` 也从 `0x200` 开始，PHDR 后无空余空间，直接新增第 9 个 program header 不具备低风险空间。
- 当前 R-X LOAD：`offset=0x000000`、`vaddr=0x000000`、`filesz=memsz=0x104984`。
- 当前 RW LOAD：`offset=0x104b88`、`vaddr=0x114b88`、`filesz=0x00b220`、`memsz=0x10ebd0`，`DYNAMIC` / `TLS` / `GNU_RELRO` 均位于该 RW LOAD 内。
- R-X 文件末尾到 RW 文件起点只有 `0x204`（516B），但 R-X 末尾到 RW 虚拟地址起点有 `0x10204`（66052B）VA 空洞。
- 若将 RW LOAD 文件偏移整体后移 `0x10000`：`old_off=0x104b88 -> new_off=0x114b88`，同时保持 `vaddr=0x114b88` 不变，则 `new_off % 0x10000 == vaddr % 0x10000 == 0x4b88`，满足 ELF LOAD 对齐约束。

**核心设计**
- 不新增 `PT_LOAD`，避免搬移 PHDR table、复用 NOTE、破坏 build-id gate 或触发 linker 兼容风险。
- 不扩展 RW LOAD 为 RWX；尾部 append 即使被 RW LOAD 覆盖也不作为执行区，避免 RWX IOC 与权限面扩大。
- 首选“保持 VA 稳定，仅重排文件 offset”：把 RW LOAD 及其后续文件内容整体后移 `0x10000`，再把 R-X LOAD 的 `p_filesz/p_memsz` 扩展到 `RW vaddr - RX vaddr = 0x114b88`。
- 新增的文件区间 `0x104984 ~ 0x114b88` 被第一个 R-X LOAD 覆盖，可作为约 64.5KB 的 executable island，用于更大 stub、multi-island trampoline、decoy code 与后续策略承载。
- A96D padding 仍在重排与注入完成后 append；`encrypt_core.py` 继续以 A96D header 中的 `elf_original_size` 记录重排后真实 ELF 长度。

**实施项（V0.67.0）**
- [x] **P0-1 ELF relayout preflight gate**（`hardening/src/main.py`）
  - 校验目标 ELF 为 AArch64 ELF64，且存在首个 `PT_LOAD + PF_X` 与后续 RW LOAD。
  - 校验 RX LOAD 当前 `offset=0`、`vaddr=0`，RW LOAD 满足 `p_offset % p_align == p_vaddr % p_align`。
  - 校验 `e_phoff + e_phnum * e_phentsz == 0x200` 且 NOTE/build-id 当前占用 PHDR 后紧邻空间；明确记录“不走新增 PHDR”原因。

- [x] **P0-2 文件内容搬移与 header 修正**（`hardening/src/main.py`）
  - 在旧 RW LOAD offset 前插入 `0x10000` 字节 filler，使所有 `file offset >= old_rw_off` 的内容整体后移。
  - 更新所有 program header 中 `p_offset >= old_rw_off` 的条目：RW LOAD、`DYNAMIC`、`TLS`、`GNU_RELRO` 等同步 `+0x10000`。
  - 更新 section header table：`e_shoff += 0x10000`，所有 `sh_offset >= old_rw_off` 的 section 同步 `+0x10000`；`NOBITS` section 仅在其 file offset 作为布局锚点时审慎处理。
  - 保持所有 `p_vaddr` / `sh_addr` 不变，避免动态链接和运行期 VA 假设漂移。

- [x] **P0-3 R-X LOAD 扩展与 executable island 分配**
  - 将首个 R-X LOAD 的 `p_filesz/p_memsz` 扩展到 `0x114b88`，使 `0x104984 ~ 0x114b88` 成为 file-backed executable 区域。
  - island 默认填充 AArch64 NOP 或可审计 filler，避免全零页形成误判；后续 stub 分配器从该区间切片。
  - V0.67 首版可先把 V0.66 stub 从旧 cave 迁移到新 island，再保留旧 cave 作为 decoy 或回滚路径。

- [x] **P0-4 V0.49/E650x gate 适配**
  - 扩展 `E4903` post-patch whitelist 口径：布局重排造成的 bulk offset move 必须作为结构化 expected change 登记，不能用宽松全放行替代。
  - 新增 E67xx gate：relayout preflight 失败、PHDR/SHDR 修正不一致、LOAD 对齐不满足、section-to-segment 覆盖漂移、island 不在 executable PT_LOAD 内均 fail-fast。
  - 保持 `E4901` input fingerprint gate 仍只针对原始输入，重排发生在 patch pipeline 内，不改变输入基线。

- [x] **P0-5 静态与设备验收（主链）**
  - `readelf -lW/-SW` 对比确认：PHDR 数量不变，RW LOAD offset 后移，VA 不变，R-X LOAD 扩展，`DYNAMIC/TLS/GNU_RELRO` 仍落在 RW LOAD 内。
  - `objdump`/branch decode 确认 entry branch、island stub、branch-back 合法且目标均位于 executable PT_LOAD。
  - `./make.sh --build cx`、`verify_shadow_guard_host.sh` 通过；`verify_mustpass_host.sh` 在设备侧主链 `12 PASS / 0 FAIL / 0 SKIP` 通过。
  - host 编排中的过期态 reboot 用例（TC-007 host orchestrated）当前因设备未过 `EXPIRE_EPOCH` 未触发通过，待 5 分钟评估版或过期窗口复测收口。

**风险与控制**
- 新增 PHDR 风险高：PHDR 后无空位，复用 NOTE 会影响 build-id gate，搬移 PHDR 会扩大 ELF loader 兼容风险；V0.67 不走该路线。
- RWX 风险高：把 RW LOAD 改成可执行会产生明显 IOC，并扩大数据段执行面；V0.67 明确禁止。
- Section header 虽非运行期必需，但用于 `readelf`、调试与后续 gate；必须同步维护，避免工具链和人工复核失真。
- bulk move 会让原有 byte-level diff 白名单失效；必须以结构化 relayout model 重新描述变更，而不是关闭门禁。
- 首版只释放空间和迁移/验证现有 stub，不同时叠加 challenge-response 或 CAS 修复，避免布局风险与策略风险混合。

**本版本不做**
- 不新增第 9 个 `PT_LOAD`。
- 不复用 `PT_NOTE` 为 executable segment。
- 不把 RW LOAD 改为 RWX。
- 不改变 `libarm96` 密文容器格式、loader memfd/fexecve 模型或 binfmt `POC` 规则。
- 不在同一版本合入 CAS `setThreadPoolConfiguration:-25` 修复或周期 challenge-response。

#### V0.68.0（Scatter Decoy Attack Points / RX Island Honey Payload）
> **目标**: 基于 V0.67 已释放的 RX island，在不影响主执行链路与门禁完整性的前提下，分散注入非混淆诱饵攻击点（含授权截止时间/epoch/伪配置），提高攻击者静态定位与误修补成本。
> **状态**: **Planned (2026-05-29)**

**范围与边界**
- 仅在 V0.67 已验证 filler 窗口内插入 decoy payload；不改变现有 stub entry、branch-back 与真实授权关键分支。
- 诱饵点可使用明文（non-obfuscated）以增强“可见性诱导”，但必须与真实控制流解耦，禁止被真实鉴权路径读取。
- 不新增 `PT_LOAD`，不改 RWX，不改密文容器和 loader 执行模型。
- 不与 CAS/challenge-response 同版叠加，避免风险耦合。

**建议注入窗口（首批）**
- 小窗口：`[0xF5B5C,0xF5F4C)`，可布置短字符串点位（如 16~32B）。
- 大窗口：`[0x104B0C,0x114B88)`，可布置稀疏长点位（如 32~64B，步长 0x400）。
- 避开当前 stub 占用区（V0.67 现状约 `0x104984..0x104B0C`）。

**实施项（V0.68.0 规划）**
- [ ] **P0-1 Decoy 清单与布局模板**（`hardening/src/main.py`）
  - 增加结构化 decoy 描述：`name/type/payload/off/size/policy`。
  - 预置 3 类 payload：到期时间字符串、epoch 数值（LE32/LE64）、伪配置键值对。
  - 支持按窗口散点写入，禁止连续大块特征化落点。

- [ ] **P0-2 安全注入约束与冲突检测**（`hardening/src/main.py`）
  - 注入前检查：不得覆盖 stub、ELF 头、PHDR/SHDR、已登记 patch 点、A96D 头。
  - 若窗口不足或冲突，fail-fast（新增 E68xx gate），禁止静默降级到随机位置。

- [ ] **P0-3 E4903 结构化白名单扩展**（`hardening/src/main.py`）
  - 在现有“offset + bulk range”模型上追加 decoy ranges 白名单。
  - 保持默认最小放行，不允许“整段 RX 全放开”。

- [ ] **P0-4 Host 静态门禁补充**（`verify_mustpass_host.sh`）
  - 新增 TC-020：验证 decoy 点位都在声明窗口内，且不与 stub/关键段重叠。
  - 新增 TC-021：验证至少命中 N 个 decoy 点位（防止配置开关开启但未实际注入）。

- [ ] **P0-5 回归与设备主链验证**
  - `./make.sh --build cx` 必过，`verify_shadow_guard_host.sh` 必过。
  - `verify_mustpass_host.sh` 主链不退化；过期态用例按既有口径单独复测。

**V0.68 验收判据（Acceptance Gate）**
- [ ] readelf/objdump 结构不退化：`PT_LOAD` 数量不变，RW 非 RWX，stub entry/branch-back 一致。
- [ ] decoy 注入可复现：同构建输入下位置与内容确定，不引入随机漂移。
- [ ] decoy 注入可观测：静态扫描可见但不影响运行（mustpass 主链全绿）。
- [ ] E4903/E68xx 门禁有效：越界写入、冲突写入、缺失写入均 fail-fast。

**本版本不做**
- 不实现 decoy 与真实分支联动触发。
- 不引入自修改代码、运行时随机写入、不可诊断崩溃诱饵。
- 不合入 CAS `setThreadPoolConfiguration:-25` 与 challenge-response。


### 📋 迭代记录 (Iteration History)

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

## 4. 验证标准
1.  **运行正常**: 加固后的二进制在合法授权（或破解授权）下必须能正常运行业务。
2.  **日志干净**: 即使强制让其崩溃（如改系统时间），Logcat 中也不应出现任何提示 License 的文字。
3.  **IDA 分析困难**: 在 Strings 窗口找不到关键字，无法直接跳转到校验逻辑。

---

## 5. 版本状态与回归测试 (Version Status & Regression Policy)

> **⚠️ 关键准则 (CRITICAL POLICY)**
> 1.  **禁止回归 (No Regressions)**: 新版本的发布必须包含旧版本所有已验证的特性。
>     *   示例: r3.3 已经适配好了版本号字符串显示，后续版本 **绝不允许**出现显示错误。
> 2.  **原子性验证**: 在特性合并前，必须先在设备上完成独立特性验证。
> 3.  **失败回滚**: 若新版本导致 Boot Loop 或 Panic，必须立即回滚到上一个稳定版本并在此文档中记录失败原因。

### 版本状态矩阵 (Status Matrix)

| 版本 | Release Date | 关键特性 | 状态 | 已知问题 | 结论 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **0.49** | 2026-05-03 | 当前配置 `CUSTOM_VERSION=V0.49.0`；新增输入指纹门禁(E4901)、patch 前镜像断言(E4902)、patch 后白名单核验(E4903)、`orig_size` 一致性校验(E4905)；擦除策略 remap 为 `.tango→.arm96`、`tango_prop→arm96_prop`、`tango uid→arm96 uid`，保留 `/dev/tango*` | **Current** | 设备实装版本需重新部署后对外显示 `V0.49.0` | **Current Release** |
| **0.48** | 2026-05-03 | 关键数据域完整性与验收闭环；为 V0.49 门禁收口提供基线 | **Stable** | N/A | **Archived** |
| **0.45** | 2026-04-28 | 当前配置 `CUSTOM_VERSION=V0.45.0`；VC 交付版本号推进，继承 V0.44 PID tracking + guardian cascade 验收口径 | **Stable** | N/A | **Archived** |
| **0.44** | 2026-04-28 | 当前配置 `CUSTOM_VERSION=V0.44.0`；验收脚本与文档统一到 PID tracking + guardian cascade 口径 | **Stable** | N/A | **Archived** |
| **0.42** | 2026-04 | 正式版过期策略：`LICENSE_CHECK_INTERVAL=1800`，已过期首次 kill 派生到 1200-1800s 窗口，后续 1800s 周期 | **Stable** | 5 分钟评估版首次 kill 窗口为 120-180s | **Archived** |
| **0.41** | 2026-04-01 | Loader OLLVM 混淆覆盖扩展：新增解密链路与运行期防护函数混淆 | **Stable** | N/A | **Archived** |
| **0.40** | 2026-04 | arm96server 加固：OLLVM 混淆 + 字符串加密 + .text hash 融入 token (双端 XOR 消去) | **Stable** | BCF 暂不用于 arm96server，避免 BTI 兼容性风险 | **Archived** |
| **0.39** | 2026-03 | Core 混淆填充 30KB：心经 XOR 分片 + 伪 AES-CBC + 伪 DER + 伪 License TLV 追加块 | **Stable** | N/A | **Archived** |
| **0.38** | 2026-03-30 | arm96server PID 追踪替代 PTRACE_SEIZE：消除信号拦截导致的应用 ANR；扫描间隔 3s；guardian tracked-PID cascade | **Superseded** | N/A | **Archived** |
| **0.37** | 2026-03-29 | 迁移追溯文档 + prebuilt 对齐 + package.sh 版本格式 arm96-V0.Y.Z | **Superseded** | N/A | **Archived** |
| **0.36** | 2026-03-28 | Core 去 Tango 品牌指纹：`_wipe_tango_identifiers()` 擦除 117 处 tango/Tango/TANGO | **Superseded** | N/A | **Archived** |
| **0.35** | 2026-03-17 | 源码模块名去 Tango 化：公共文档/构建兼容层命名收敛 (main/1.0 范围) | **Superseded** | 目录级迁移留给 main/2.0 | **Archived** |
| **0.34** | 2026-03-17 | arm96-only 交付、镜像/安装包/验收入口收口、宿主残留 binfmt 清理后冷启动通过 | **Superseded** | 已确认设备侧 `11 PASS / 0 FAIL / 0 SKIP`，宿主侧编排用例全通过 | **Archived** |
| **0.33** | 2026-03-16 | 多客户 profile 构建 (`--build <target>`) + `-v`/`-i` CLI 合约 + 最小镜像集成仅保留 `tango_translator` | **Stable** | `myt` 当前以关闭 MAC 绑定适配 Docker 虚拟 MAC 环境 | |
| **0.32** | 2026-03-02 | Guardian 双向互保：arm96server 与 guardian 互相监控、任一被杀都可重建 | **Superseded** | N/A | **Archived** |
| **0.31** | 2026-03-02 | 空壳检测 + 延时下毒：假 arm96server 触发 15-45s 随机自杀 | **Superseded** | N/A | **Archived** |
| **0.30** | 2026-03-01 | arm96server Token 密钥融合：删除/替换 arm96server 直接导致 AES-GCM 解密失败 | **Superseded** | N/A | **Archived** |
| **0.29** | 2026-03-01 | Guardian 自愈 + init 注册：arm96server 被杀后可自动复活 | **Superseded** | N/A | **Archived** |
| **0.28** | 2026-02-28 | HMAC-SHA256 运行时种子推导 (17 处 seed 迁移, 无 `.rodata` seed 常量) + 哨兵 HMAC 派生 + 诱饵哨兵 (2 假 1 真) | **Superseded** | N/A | **Archived** |
| **0.27** | 2026-02-27 | 密钥派生硬化 (去全零逃逸 + sentinel 反定位 + per-string LCG keystream 抗 xortool) | **Superseded** | N/A | **Archived** |
| **0.26** | 2026-02-27 | Loader 完整性保护 (P0 .text SHA-256 自校验 + 两阶段构建 + OBFUS_CFF_BCF_SUB) | **Superseded** | N/A | **Archived** |
| **0.25** | 2026-02-27 | 静态分析对抗 (P0 辅助字符串 XOR 运行时解密 14 条 + P1 BCF 虚假控制流 + P2 调试器进程检测 kill) | **Superseded** | N/A | **Archived** |
| **0.24** | 2026-02-27 | pid_max 钳位 (loader is_zygote 路径自动写 65536) | **Superseded** | N/A | **Archived** |
| **0.23** | 2026-02-26 | 运行期反调试 (P0 watchdog 激活 + P1 maps 扫描 + P2 arm96server PTRACE_SEIZE+EXITKILL)；新增 TC-008/009/010 | **Superseded** | 容器 pid_max=4194304 时 32 位 app PID > 65535 → bionic abort；V0.24 自动钳位 | **Archived** |
| **0.22** | 2026-02-25 | EXPIRE 密钥融合 (Prong 3, quad_key) + secure_zero/explicit_bzero + AES-128-GCM (TGENC022, 纯 C GHASH) + .rodata 字符串 XOR 混淆 (-V 3 行不变) | **Superseded** | N/A | **Archived** |
| **0.21** | 2026-02-25 | 代码混淆 (OLLVM CFF+SUB, clang-14 pass plugin) + 反调试增强 (TRACERPID_TASKS + ACTION_SIGKILL + watchdog 配置) | **Superseded** | watchdog 实现在 `#if 0`，待 SELinux 验证 | **Archived** |
| **0.20** | 2026-02-23 | MAC 密钥融合 + 延迟暗桩 (MAC hash 融入 AES key + scatter-corrupt 投毒) | **Superseded** | N/A | **Archived** |
| **0.19** | 2026-02-23 | 平台绑定 (五层检查 + ISAR0 密钥融合 + MAC OUI 绑定, CIX P1 锁定) | **Superseded** | N/A | **Archived** |
| **0.18** | 2026-02-23 | 加壳+加密 (UPX+AES, memfd 内存执行, 密钥分片) | **Superseded** | N/A | **Archived** |
| **0.17** | 2026-02-23 | Core 改名 libarm96, 为加壳加密做准备 | **Superseded** | N/A | **Archived** |
| **0.16** | 2026-02-23 | setitimer 替换 watchdog, 孤儿清理 | **Superseded** | N/A | **Archived** |
| **0.12** | 2026-02-20 | POC binfmt 兼容, argv 重构 (--exe/--) | **Superseded** | N/A | **Archived** |
| **0.11** | 2026-02-17 | Split model, Watchdog, V12 Panic Fix, FD 合规 | **Superseded** | POC flags 不兼容 | **Archived** |
| **0.10** | 2026-02-12 | 0.9 Baseline only, Build Params | **Superseded** | 单体模型，无 panic 修复 | **Archived** |
| **0.9** | 2026-02-10 | Behavior Obfuscation (Fake Errors) | **Stable** | N/A | **Backup** |

## 6. 必过用例 (Must-Pass Test Cases)

新版本发布前，**必须** 按顺序通过以下所有测试用例。

当前验收入口分为两层：
*   设备侧: `verify_mustpass.sh`
*   宿主侧: `verify_mustpass_host.sh`

其中 `verify_mustpass_host.sh` 是最终验收入口：先推送并执行设备侧脚本，再补跑必须跨 reboot/adb 断连重连的用例，尤其是“授权过期后重启系统”。

### TC-001: 版本号显示 (Version Display)
*   **状态**: **Implemented (V0.33+ current contract)**
*   **命令**: `arm96 -v`
*   **预期**: 有且仅有 1 行输出（无多余内容）：
    *   第 1 行: 版本号，如 `V0.45.0`
    *   **禁止**: 出现 help 文本、版权信息、原始版本号 `2.1.0`。
    *   **禁止**: 程序崩溃或输出多行。

### TC-001b: 授权信息显示 (Info Display)
*   **状态**: **Implemented (V0.33+ current contract)**
*   **命令**: `arm96 -i`
*   **预期**: 有且仅有 2 行输出：
    *   第 1 行: 授权主体，如 `for moyunteng`
    *   第 2 行: 授权有效期，如 `to 20260401-19:00:00`
    *   **禁止**: 出现时区名 `Shanghai`、多余 banner 或 help 文本。

### TC-002: 帮助文档隐藏 (Help Suppressed)
*   **状态**: **Implemented (V0.11+)**
*   **命令**: `arm96 -h` 或 `arm96`（无参数）
*   **预期**: 不输出任何引导性信息（静默退出，输出 0 字节）。

### TC-003: 无效参数静默 (Invalid Args Silent)
*   **状态**: **Implemented (V0.12+)**
*   **命令**: `arm96 -a` 或任意非 `-v/-i` 的手动调用
*   **预期**: 不输出任何信息（静默退出，输出 0 字节）。

### TC-004: 授权期内重启系统 (Boot Stability During License)
*   **状态**: **Implemented (V0.11+)**
*   **操作**: 授权有效期内执行 `adb reboot` 或 `adb shell setprop ctl.restart zygote_secondary`，等待 30秒。
*   **预期**:
    *   系统能正常运行起来，ADB 保持在线。
    *   Logcat 中无 Zygote 持续崩溃日志。
    *   `zygote_secondary` socket 可达（32 位应用可启动）。

### TC-005: 授权期内 32 位应用持续运行 (App Persistence During License)
*   **状态**: **Implemented (V0.11+)**
*   **操作**: 授权有效期内启动 32 位应用（例：`com.dragon.read`）。
*   **预期**: 应用可持续运行，直到授权过期后自动执行自杀策略（watchdog SIGKILL）。

### TC-006: 授权过期后 32 位应用周期自杀 (Expiration Kill Loop)
*   **状态**: **Implemented (V0.11+, V0.16 改用 setitimer, V0.42 首次 kill 抖动)**
*   **操作**: 使用 `make_5min.sh` 构建 5 分钟评估版 (EXPIRE_IN_SEC=300, CHECK_INTERVAL=180)，部署后等待过期，启动 32 位应用。
*   **预期**: 每隔指定时长自杀一次。V0.42 后，若 zygote 启动时已过期，首次 kill 派生到 interval 后 1/3 窗口内；5 分钟评估版 interval=180s 时首次 kill 窗口为 120-180s，后续按 180s 周期执行。正式版 interval=1800s 时首次 kill 窗口为 1200-1800s，后续按 1800s 周期执行。
*   **V0.16 机制**: SIGALRM 杀 zygote → init 重启 → loader 扫 /proc 清理孤儿 APP → 重新布防 → 循环。

### TC-007: 授权过期后重启系统 (No Bootloop on Expiry)
*   **状态**: **Implemented (V0.11+, V0.16 verified)** — 跳转表补丁 (0xE55F0) 防止过期 panic；V0.16 setitimer 方案：已过期时 delay=interval（非 1s），避免紧密重启循环；system_server 由主 zygote (64-bit) fork，不在杀伤范围内。
*   **操作**: 授权过期后 `adb reboot`。
*   **预期**: 系统能正常运行起来。Logcat 中无 crash 日志。`zygote_secondary` socket 可达。
*   **自动化入口**: 由宿主侧 `verify_mustpass_host.sh` 编排执行；设备内 `verify_mustpass.sh` 无法独立覆盖该用例。

### TC-008: 杀 arm96server 后 tracked 32 位应用全部退出 (Guardian Cascade)
*   **状态**: **Implemented (V0.23 P2, V0.38 改为 PID tracking)**
*   **前置**: arm96server 正在运行，至少一个 32 位应用（如 `com.dragon.read`）正在运行，并已出现在 `/data/local/tmp/.arm96_tracked_pids`。
*   **操作**: `kill -9 <arm96server_pid>`
*   **预期**:
    *   guardian 发现 arm96server 死亡后读取 tracked PID 文件，并 SIGKILL 所有 tracked 32 位 app。
    *   `ps -A | grep <app_pid>` 应无结果（进程已不存在）。
    *   arm96server 应在 guardian 或 init 保护下恢复运行。
*   **机制**: V0.38 后为 PID tracking + guardian cascade，替代旧版内核 `PTRACE_O_EXITKILL` 级联。

### TC-009: Frida/strace 附加 32 位应用被拒绝或清理 (App Attach Blocked/Cleared)
*   **状态**: **Implemented (V0.23 P2, V0.38 改为 PID tracking)**
*   **前置**: arm96server 正在运行，32 位应用正在运行，并已出现在 `/data/local/tmp/.arm96_tracked_pids`。
*   **操作**:
    *   场景 A: `strace -p <app_pid>`（模拟 frida 的 ptrace ATTACH 行为）
    *   场景 B（可选真实 frida）: `frida -U -p <app_pid>`
*   **预期**:
    *   若内核/SELinux 直接拒绝 attach，输出 EPERM/Operation not permitted。
    *   若 attach 短暂成功，arm96server 在扫描窗口内发现 `TracerPid > 0`，杀 tracer + app。
*   **机制**: PID tracking + `TracerPid` 轮询检测；不再依赖 app 进程被 arm96server 长期 ptrace seize。

### TC-010: strace 跟踪 arm96server 被拒绝 (arm96server Self-Protection)
*   **状态**: **Implemented (V0.23 P2+)**
*   **前置**: arm96server 正在运行。
*   **操作**:
    *   `strace -p <arm96server_pid>` 尝试跟踪 arm96server 本身
*   **预期**:
    *   `EPERM`（Operation not permitted），无法跟踪。
    *   arm96server 继续正常运行。
*   **机制**: arm96server 的 `PTRACE_TRACEME` + `PR_SET_DUMPABLE=0` 自保护。

### V0.33 实测补充（2026-03-16）
*   使用 `make_5min.sh --build myt` 生成 5 分钟授权版本，设备侧显示到期时间 `to 20260316-21:59:10`，检查间隔为 `180s`。
*   过期后启动 `com.dragon.read`，第一次死亡发生在 `22:02:12 CST`，第二次死亡发生在 `22:05:12 CST`，符合“过期后按全局 180s 节拍持续执行自杀”的预期。
*   在已过期状态下执行整机 `adb reboot`，设备成功恢复，`sys.boot_completed=1`，`arm96 -v` 仍为 `V0.33`，并且 `com.dragon.read` 可再次拉起。
*   当前回归结果：`verify_mustpass.sh` 达到 `10 PASS / 0 FAIL / 0 SKIP`；`verify_mustpass_host.sh` 对“授权过期后 reboot”补测通过。

---

## 7. 扩展测试用例 (Extended Test Cases)

以下为辅助验证用例，非必过项但建议验证。

### TC-EXT-001: 可复现构建 (Reproducible Build)
*   **目的**: 确认在“固定过期时间 + 固定 BuildID seed”下，产物字节级一致（MD5 恒定）。
*   **操作**: 连续执行两次 `TANGO_LICENSE_EXPIRE_EPOCH=1760000000 TANGO_BUILD_ID_SEED=fixedseed ./make.sh`，对比 MD5。
*   **预期**: 两次 MD5 完全一致。

### TC-EXT-002: 编译参数覆盖 (Build Params Override)
*   **目的**: 验证固定过期时间与固定 BuildID 后产物可复现，且可按需覆盖。
*   **操作**:
    1.  固定过期时间并构建：`TANGO_LICENSE_EXPIRE_EPOCH=1760000000 ./make.sh`
    2.  固定 BuildID：`TANGO_BUILD_ID_SEED=fixedseed ./make.sh`
    3.  直接指定 BuildID：`TANGO_BUILD_ID_HEX=0123456789abcdef0123456789abcdef01234567 ./make.sh`
*   **预期**: 产物能正常生成；BuildID 与过期时间随覆盖参数变化。

### TC-EXT-003: 反调试有效性 (Anti-Debug)
*   **状态**: **Covered by TC-008/009/010 (V0.23+, V0.38 PID tracking)** — arm96server 当前通过 tracked PID 文件、TracerPid 轮询、guardian cascade 与自保护覆盖 Frida/strace 验证。独立的 ptrace 注入实验不再作为单独用例。

### TC-EXT-004: Zygote 孵化路径软熔断 (Zygote Specialize Soft Fuse)
*   **状态**: 待开发 / Pending（当前候选不包含软熔断/zygote specialize 相关实验）。

---

## 8. 开发环境与编译说明

### 编译环境
-   **编译机**: `ssh mhc@192.168.164.2`（Linux x86_64，含 aarch64-linux-gnu-gcc 交叉工具链）
-   **源码路径 (Linux)**: `/home/mhc/56T/cx10/android10/vendor/hello/arm96`
-   **源码路径 (Windows)**: `Y:\56T\cx10\android10\vendor\hello\arm96`
-   **一键构建**: `./make.sh`（从 Linux 侧执行）
-   **5分钟评估版**: `./make_5min.sh`（设 EXPIRE_IN_SEC=300, CHECK_INTERVAL=180）

### 测试设备
-   **目标容器**: `adb connect 192.168.11.35:5004`（容器化 Android 10）
-   **宿主**: `adb connect 192.168.11.35:5555`
-   **容器复位**: 在宿主运行 `/data/local/b4_build_cx10_con4.sh`

---

## 9. V0.15→V0.16 架构改进分析 — 去 Watchdog 化 (2026-02-22)

### 9.1 背景：外部 Watchdog 的问题

V0.11 引入的 double-fork watchdog daemon 存在三个严重问题：

1.  **进程暴露**: `ps -ef | grep tango` 可直接观测到 `tango_translator` 字样的额外进程，暴露翻译层存在。
2.  **单点故障**: watchdog 被强杀后不会自动重建，授权策略彻底失效——用户只需 `kill <watchdog_pid>` 即可永久绕过。
3.  **时钟不安全**: watchdog 使用 `time(NULL)` (CLOCK_REALTIME)，修改系统时间可直接绕过过期判定。

### 9.2 关键发现：原生 POSIX Timer 机制仍然存活

通过 IDA 逆向 `sub_444A0`（timer setup 路径，位于 core 内部），确认：

-   **杀伤信号**: `SIGFPE` (signal 8)，默认动作终止进程。
-   **时钟源**: `CLOCK_BOOTTIME` (7)，从系统启动时单调递增，**无法通过修改系统时间绕过**。
-   **Timer 类型**: `TIMER_ABSTIME` 绝对时间 + `it_interval = {1, 0}`（到期后每秒重复触发）。
-   **到期时间**: `current_boot_time + interval_constant`（interval 常量原值 14400 = 4小时）。
-   **杀伤链**: Timer 到期 → SIGFPE → 终止 zygote_secondary → init 重启 → 新 timer 设置 → 循环。

**对比两种方案**:

| 维度 | 外部 watchdog (loader fork) | 原生 timer (core 内置) |
|------|------|------|
| 额外进程 | ❌ ppid=1 daemon 可见 | ✅ 无额外进程 |
| 抗强杀 | ❌ 被杀则永久失效 | ✅ Timer 在进程内核态，无法单独杀 |
| 时钟安全 | ⚠️ CLOCK_REALTIME，可回拨 | ✅ CLOCK_BOOTTIME，不可篡改 |
| 杀伤方式 | SIGKILL zygote 子进程 | SIGFPE 杀 zygote 本身 → init 重启 |
| 隐蔽性 | ❌ watchdog.log + 进程 | ✅ 完全内生，零外部痕迹 |

### 9.3 interval 补丁 BUG 修复

**原有 BUG (V0.11)**:  补丁目标 `0xE1F02` (.rodata) 的 4 字节虽然值为 0x3840 (14400)，但这**不是 timer 代码实际读取的位置**。

**实际编码**: 14400 是 `sub_444A0` 内 0x44554 处 `MOVZ W9, #0x3840` 指令的**立即数**:
```
44554: 09 08 87 52    MOVZ W9, #0x3840     ; W9 = 14400
44558: 09 01 09 AB    ADDS X9, X8, X9      ; X9 = current_boot_time + 14400
```

**修正**: 直接 patch 指令编码：
```python
# ARM64 MOVZ Wd, #imm16 编码: 0x52800000 | (d) | (imm16 << 5)
# d=9 (W9), 所以基础 = 0x52800009
# interval=180: 0x52800009 | (180 << 5) = 0x52801689
f.seek(0x44554)
f.write(struct.pack("<I", 0x52800009 | ((interval & 0xFFFF) << 5)))
```

### 9.4 死代码区域（可用于未来注入）

jump table patch (entry 2/3/4 → 0x10) 使以下代码**完全不可达** (xrefs=0)：

| 地址范围 | 大小 | 原始功能 |
|----------|------|---------|
| 0x44614 – 0x4465B | 72 B | "License expired on ..." panic 构造 |
| 0x44660 – 0x44677 | 24 B | "Invalid CPU type" panic |
| 0x44678 – 0x446A7 | 48 B | "Invalid license" panic |
| **合计** | **~148 B** | 可用于未来 shellcode 注入 |

### 9.5 进程名隐藏分析

**问题**: binfmt_misc 翻译的 ARM32 进程在 `ps -ef` 中显示 `tango_translator` 前缀。

**方案 A（重命名二进制）不可行**: binfmt_misc 在当前内核版本（< 5.7）无 namespace 隔离。规则表全局唯一，同一宿主多容器（原生 + 加固）共享同一条 interpreter 路径。重命名会导致原生容器中 `/system/bin/tango_translator` 找不到 → ARM32 程序全部 ENOENT。

**可行方案**:
-   **argv[0] 伪装**: `new_argv[0] = orig_prog` 使 `ps -ef` CMD 列显示原始程序名
-   **prctl(PR_SET_NAME)**: 在 core 内部改 `/proc/<pid>/comm`
-   `/proc/<pid>/exe` 仍指向 `tango_translator`（需 memfd_create 匿名执行才能彻底隐藏，列为后续增强）
