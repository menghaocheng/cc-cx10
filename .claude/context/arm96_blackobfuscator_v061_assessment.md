# arm96 V0.61.0 与 blackobfuscator 加固评估笔记

> 生成时间：2026-05-22  
> 范围：只读评估；未修改 arm96/blackobfuscator 代码。

## 1. arm96 当前工作模型

当前主线实际路径为 `android10/vendor/hello/arm96/`，不是旧文档中的 `vendor/vclusters/.../tango_reimpl`。V0.61.0 release 包位于 `android10/vendor/hello/arm96/release/arm96-V0.61.0-cx-release/`。

核心运行链：

1. `arm96d --phase0` 注册 `binfmt_misc`，使用 `POC` flags，使 ARM32 ELF 进入 `/system/bin/arm96`。
2. `/system/bin/arm96` 是静态 loader，负责 argv 重组、FD 清理、平台/反调试/完整性检查、过期 timer、启动 `arm96server`、解密 `libarm96`。
3. `libarm96` 以 AES-GCM 加密容器形式存在，loader 运行时解密到 `memfd` 后 `fexecve`，release 禁止明文 core fallback。
4. `arm96server` 是 PID tracking 反调试 daemon，追踪 zygote_secondary 派生的 app UID 进程，轮询 `TracerPid`，同时 guardian 双向互保。
5. 安装时预翻译链由 AOSP patch 中 `Arm96Service`/`Arm96BinderService`、`arm96j`、`arm96d` 组成，负责接收 PMS/ProcessList 事件并生成 `.arm96` sidecar。

## 2. arm96 已有加固策略要点

### 数据域绑定优先于单点分支

`loader.c::reconstruct_key()` 的核心价值在于把多层见证材料融入解密密钥，而不是只靠 `if(valid)`：

- base key/iv 分片 XOR 重组。
- `ID_AA64ISAR0_EL1` 融合：非目标硬件直接解密失败。
- MAC OUI salted hash 融合：源码已有该策略；是否启用取决于 build profile。当前 `version.txt` 和 `profiles/vc.txt` 配置为启用，但当前生成态 `gen/version.py` 为 `PLATFORM_MAC_BINDING=0`、`PLATFORM_MAC_OUIS=""`，且 `gen/enc_key.h` 未生成 `MAC_OUI_HASH_*`，说明当前这份已生成/最近构建配置没有打开 MAC 融合。
- `EXPIRE_EPOCH` 融合：直接改过期常量会改变 key。
- D1 witness 融合：运行时 witness 被篡改会扰动 key。
- `arm96server` token 融合：尾部 token 与 `arm96server .text` hash 共同恢复 base token；删除或 patch `arm96server` 会使 `libarm96` 解密失败。

这说明 V0.61 不是“找到过期判断分支 NOP 掉即可永久绕过”的简单模型。

### 反调试与延迟失效

- loader 启动时做 `.text` 自校验和关键数据域校验。
- loader 扫 `/proc/self/maps`、`TracerPid`、线程级 `TracerPid`，V0.51 keyed-tag 开关已启用。
- `arm96server` 自身 `PTRACE_TRACEME` + `PR_SET_DUMPABLE=0`，阻止普通 attach。
- `arm96server` 不再 `PTRACE_SEIZE` app，避免信号干扰；改为 PID tracking + `TracerPid` 轮询。
- `verify_arm96server_alive()` 会探测空壳 server：如果 `PTRACE_ATTACH` 成功，说明没有 TRACEME，进入 15-45s 左右延迟故障/投毒窗口。
- V0.51 fault stamp 会把 arm96server 侧 debugger 命中传递给 loader，loader 在解密后做受控扰动或缩短 timer。

### 隐蔽与构建门禁

- loader/arm96server 字符串运行时解码，部分判定迁移到 keyed-tag。
- OLLVM CFF/SUB/BCF 覆盖 loader/arm96server 多个关键函数。
- `libarm96` 使用 AES-GCM，tag 错误即失败。
- patch/构建链已有输入指纹、patch 前后断言、`orig_size` 一致性、release 禁明文 fallback 等门禁。

## 3. blackobfuscator 策略概览

blackobfuscator 路径：`/home/mc/2T/blackobfuscator`。它是 APK/SO 加固套件，分三层：DEX IR 混淆、DEX/SO 加壳、native-hook 运行时保护。

### DEX IR 混淆链

`IRObfuscator` 固定顺序：`StringEncryption -> ReflectionObfuscator -> SubObfuscator -> IfObfuscator -> FlowObfuscator -> PineProtection -> HookDetection`。

可确认点：

- `FlowObfuscator`：按基本块拆分，把直线控制流改造成字符串 key/hash 驱动的 while/if 分发；跳过敏感构造器/try-catch 场景以降低兼容风险。
- `StringEncryption`：字符串常量替换为 int array + `_d()` 运行时解码；当前是 per-string 随机单字节 key + index 混合。
- `ReflectionObfuscator`：反射参数加密，降低 `Class.forName/getDeclared*` 静态路标。
- `IfObfuscator`、`SubObfuscator`：控制流与算术表达式扰动。
- `PineProtection`/`HookDetection`：针对 Java hook 框架的调用包装与入口检测注入。

### DEX/SO 加壳

- `DexPacker` 将 shell dex 放到 `classes.dex`，原始 multi-dex 加密成 `assets/original_dex.bin`。
- 非 bootstrap 的 `.so` 被 AES-CBC 加密后放到 `assets/encrypted_libs/<abi>/*.so.enc`，运行时由 native 解密加载。
- `CryptoUtil` 使用 S-Box + KEY_FRAGMENT 派生 AES-128 key；每次构建可生成不同 fragment。
- `IntegrityBuilder` 构建 HMAC-SHA256 级联完整性链，覆盖 `original_dex.bin`、shell dex、`libnative_hook.so` 等关键文件。

### native-hook 运行时保护

`native-hook/jni/hack.cpp` 已实现：

- 子进程 `PTRACE_TRACEME` 检测，避免 zygote 主线程被 trace-stop 冻结。
- `/proc/self/maps` 扫描 Frida/Xposed/Substrate/Gum 等。
- 匿名 RWX mapping 检测。
- libc `open/read` inline hook 检测，ARM64 查直接 branch，ARM32 查 Thumb BL/B.W/LDR PC 类特征。
- 加密字符串、诱饵检测函数、混淆 dispatch 表。
- DEX checksum、完整性链、SO 解密加载、Java hook/Xposed 侧检测、监控线程等（README 与代码搜索一致）。

## 4. 对 V0.61.0 外部报告的逐条核验

### A. SELinux 全局 `execmem`：基本成立，且是系统面真实风险

`sepolicy/arm96.te` 仍通过 `tango_translated(appdomain)` 等宏授予 `self:process execmem` 和 `tmpfs:file rw_file_perms`。这是兼容 DBT/JIT 的设计妥协，但报告指出“扩大所有 32 位 app RCE 后利用面”成立。

建议优先级：高，但工程代价也高。理想方向是把 32 位翻译进程收敛到更窄域，或把可执行内存授权最小化到确实需要的进程/路径。

### B. `arm96server` 使用 `u:r:su:s0`：成立，属于高价值收口点

`init/arm96server.rc` 中 `seclabel u:r:su:s0` 确认存在。虽然 arm96server 逻辑简单、字符串加密、阻止 attach，但任何 native 内存安全问题都会在 `su` 域放大影响。

建议优先级：高。应设计独立 `arm96server` SELinux domain，仅保留 `/proc` 扫描、kill app/tracer、读自身/写 tracked pid/fault stamp、重建 rc 所需的最小权限。若 `.rc 自修复` 依赖 su 权限，建议重新评估这项能力是否值得保留在常态 release。

### C. Java/Binder plugin 缺少鉴权：成立

V0.61 release patch 中 `Arm96BinderService.registerPlugin()` 仍有 `Binder.getCallingUid()` + `TODO: Add permission check`，随后直接 `mService.registerPlugin(plugin)`。`unregisterPlugin()`/`dispatchEvent()` 也未见强鉴权。

这条风险主要影响安装时预翻译事件链：第三方若能找到服务并调用，可能注册恶意 plugin、接收或扰乱事件、造成 DoS 或侧信道。它不直接等于破解 `libarm96` 解密链，但属于系统服务暴露面。

建议优先级：高。最小修复是只允许 `system/root/arm96j` UID 或 platform 签名持有者；更稳的是 Binder 层 + SELinux service_manager 双重限制。

### D. 时间 hook/回滚：对“过期自杀”部分成立，但不等同于核心永久解密绕过

loader 确实依赖 `time(NULL)`/`setitimer` 判定是否过期和何时杀 zygote，因此 root/hook/系统时间回滚可影响 kill timer 行为。

但报告漏掉了 `EXPIRE_EPOCH` 已融入 AES-GCM key：静态修改过期常量会导致解密失败。也就是说：

- “改二进制过期时间常量”不是稳定 bypass。
- “运行时伪造时间源”仍可能延后/绕过 kill timer，因为 key 中融合的是构建期 `EXPIRE_EPOCH`，不是当前时间。

建议优先级：中高。补强方向是多时间源/单调时间见证/周期 challenge，把“当前时间合理性”也变成数据域 witness 或 fault trigger，而不是只影响 timer 分支。

### E. 属性伪造：报告混淆了授权与运维开关

当前 `arm96d` 中 `persist.sys.ap2901e99e08`、`persist.sys.ap3219402ff1` 等主要是日志/安装时预翻译开关，不是 loader 解密授权根。`persist.vendor.p3a9c7d1e` release 文档口径存在，但 V0.61 核心解密链不依赖“读取一个属性得到授权成功”。

结论：作为“破解 arm96 授权”的论据偏弱；作为“运维属性被 root 改写可能影响功能/可观测性”的风险成立。

### F. 静态 patch 单点分支：报告对旧式授权模型成立，对 V0.61 核心链明显低估

V0.61 有 `.text` hash、critical data hash、AES-GCM tag、EXPIRE/MAC/HW/D1/server-token 多层 key fusion，单点 NOP 成功率远低于报告描述。尤其是改 `arm96server`、删 server、改过期常量，都可能转化为错误 key 或延迟 fault。

仍需注意：如果存在某些检查只进入 `_exit(0)` 而不扰动数据域，或 V0.51 fallback/明文关键词仍有残留，仍会成为脚本化 patch 目标。V0.52 已规划/部分实现的“强制 keyed-tag + 去明文 fallback + release strings gate”方向是对的。

### G. 拆除/空壳 `arm96server`：报告只说对了一半

直接删 rc 或让 server 不启动：loader 的 `reconstruct_key()` 会读取 `arm96server` 文件尾 token 与 `.text` hash；文件缺失或 `.text` 改动会导致 `libarm96` 解密失败。

启动一个空壳同名进程：loader 有 `verify_arm96server_alive()`，通过 `PTRACE_ATTACH` 探测是否真的 `PTRACE_TRACEME`，空壳会进入延迟投毒/短 timer。

剩余真实风险：拥有 root/定制内核的攻击者仍可改 `/proc`/ptrace 语义、同时伪造文件 token/hash 和进程行为；这属于系统已被高权限控制后的对抗，不是当前用户态机制能完全解决。

### H. 内存 dump 解密后 core：部分成立，但实际门槛高于报告描述

明文 `libarm96` 必须在内存中存在并执行，这是所有壳/加密执行模型的共同边界。arm96 通过 memfd、反调试、`PR_SET_DUMPABLE`、arm96server、maps/debugger 扫描提高 dump 成本，但不能从原理上消灭“运行后明文可被高权限内核/物理内存观察”的问题。

建议优先级：中。后续可以考虑分段解密、执行后擦除冷段、关键表按需解密、代码页校验、JIT 热路径 witness 混入等，提高完整 dump 的可用性门槛。

## 5. 明显需要加强的点（建议优先级）

P0：Binder 访问控制

- `Arm96BinderService` 对 `registerPlugin/unregisterPlugin/dispatchEvent` 做 UID/签名/权限校验。
- AIDL service 在 SELinux `service_contexts`/`service_manager` 上收窄 find/call 范围。
- `arm96d` 的 `onEvent()` 也应校验 caller UID，避免绕过 arm96j 直接喂事件。

P0：`arm96server` 去 `su:s0`

- 独立 domain，最小化 `/proc`、`kill`、`ptrace/self`、文件读写权限。
- 重新评估 `.rc 自修复`，避免为了自修复长期保留 su 域。

P0/P1：`execmem` 授权收口

- 短期：精确列出必须支持的 32 位 domain，剔除不需要的 server 域。
- 中期：探索翻译进程专用 domain 或 memfd/ashmem 执行路径的更细粒度策略。
- 长期：尽量把 W^X 例外从“所有 appdomain”变成“由 arm96 管控的翻译执行面”。

P1：时间源与过期策略强化

- 多源时间：wall clock + monotonic + 文件系统可信时间戳 + boot count/last-seen 状态。
- 回滚检测进入 fault mode 或 key witness，而不仅是调整 timer。
- 将周期 challenge-response 补上：loader/server 周期 nonce + keyed response，失败进入延迟失效链。

P1：运行后 dump 成本提升

- 对 core 分段/按需解密；非热区用后擦除。
- 核心 dispatch/table 做运行时 keyed decode，避免一次 dump 得到全部可用材料。
- 在 translated/JIT 热路径加入轻量 witness 校验，失败时产生错误结果而非立即 abort。

P1：V0.51/0.52 release gate 复核

- 确认 release 产物无 `frida/gadget/xposed/substrate/lsposed` 明文。
- 确认 `V051_ENABLE=1`、`V051_KEYED_TAG_ENABLE=1`、`V051_ERROR_MODE=1` 在 release gate 中 fail-fast。

## 6. native-only：blackobfuscator 比 arm96 强或 arm96 未覆盖的策略

本节只比较 native 可执行文件保护，不纳入 Java/DEX hook、反射、插件链策略。

边界修正：`arm96` loader 只是 `libarm96` 的启动封装，`fexecve` 后自身不再作为独立检测主体存在；`libarm96` 是预编译二进制，无源码，向其中强塞周期检测会引入较高稳定性风险。因此后续 native 运行时反 trace/反 hook 加固应优先落在 `arm96server`、loader 启动窗口、构建/发布完整性链，而不是改造 `libarm96` 运行时。

| blackobfuscator native 策略 | 当前 arm96 状态 | 判断 |
|------|------|------|
| libc 函数入口 inline hook/prologue 检测 | 未见等价实现 | blackobfuscator 检查 libc `open()`/`read()` 首指令：ARM64 检 direct branch，ARM32 检 Thumb BL/B.W/LDR PC 类 trampoline。arm96 当前主要依赖 maps 关键词、TracerPid、ptrace/prctl、`.text` 自校验，没有对 libc wrapper/PLT/GOT/prologue 做细粒度检测。 |
| 关键保护函数首指令自完整性巡检 | 部分已有，但形态不同 | arm96 有 loader `.text` SHA-256、critical data hash、arm96server `.text` hash/token 融合，强于“只看首指令”的静态完整性；但 blackobfuscator 会对 `tamper_exit`、`anti_debug_checks`、`check_ptrace`、`derive_key`、decoy 函数等关键函数入口做运行时/周期性 prologue 扫描，能抓后置 inline patch。arm96 目前未见同类周期性入口巡检。 |
| 匿名 RWX mapping 检测 | 未见独立规则 | blackobfuscator 扫 `/proc/self/maps`，命中 `rwxp` 且无 pathname 的匿名映射即判异常。arm96 已扫 frida/gadget/gum/xposed/substrate/lsposed 等关键词，但未见 `rwxp anonymous` 规则。引入 arm96 时需为 DBT/JIT/memfd 映射做白名单，避免误杀。 |
| Frida 默认端口主动探测 | 未见等价实现 | blackobfuscator 非阻塞连接 `127.0.0.1:27042/27043`。arm96server 会扫进程名并杀 `frida*`，loader 扫 maps 关键词；但未见本机端口探测。该策略对改名但仍监听默认端口的 frida-server 有补充价值。 |
| 启动后周期性 TracerPid 线程 | 不建议照搬到 loader/libarm96 | blackobfuscator 在本进程里记录 baseline `TracerPid`，之后 3-7s 随机轮询，发现变化就 crash。arm96 的长期反 trace 主体是 `arm96server`：它每 3s 扫描 32 位 app UID 进程，对 tracked PID 轮询 `TracerPid`，命中后同时 kill tracer 和 app。考虑到 loader 会退出、`libarm96` 无源码，不建议把该策略下沉进 core；更合理的是继续增强 `arm96server` 的覆盖面和抗篡改能力。 |
| 随机化检测顺序 + fake 检测函数 dispatch | 部分已有但不系统 | arm96 有 OLLVM CFF/SUB、keyed-tag、sentinel decoy；blackobfuscator 额外有 per-build `ANTI_DEBUG_ORDER`、fake check table、假 maps/port/path 检测函数，干扰静态定位“真正检查链”。arm96 可借鉴这种运行时检查编排层，而不是只依赖编译器混淆。 |
| tamper_exit 清理敏感明文 + 随机延迟 + 硬件异常崩溃 | 部分已有相邻能力 | arm96 有 V0.51 延迟 fault、解密失败/投毒、secure_zero；blackobfuscator 在命中后主动清零解密 DEX、随机 100ms-2s 延迟，再空指针写触发 SIGSEGV，避免 `exit/abort` 被 hook。arm96 可借鉴“命中后清理明文 + 不走 libc exit 路径”的退出形态。 |
| XOM-safe code read | 未见单独实现 | blackobfuscator 在读代码页首指令前临时 `mprotect(PROT_READ|PROT_EXEC)`，失败则跳过，兼容 execute-only memory。arm96 当前做整段 `.text` hash 更偏文件/内存段哈希，未见这类针对入口指令读取的 XOM 兼容 helper。 |
| 加密 SO asset 运行时解密加载 | arm96 主目标不同，已有更强 core 加密 | blackobfuscator 把非 bootstrap `.so` 加密进 assets，native 解密加载；arm96 已对核心 `libarm96` 用 AES-GCM + memfd + key fusion，强于 blackobfuscator 的 AES-CBC/S-box fragment。这里不算 blackobfuscator 更强，只是对“多 native 子模块”可借鉴容器化管理。 |
| HMAC/manifest 级多产物链 | arm96 部分已有，不是同形态 | blackobfuscator 有 APK 场景下的完整性链；arm96 有 patch 门禁、loader/server/core 多点 hash/token，但未见把 release 六件套、rc、sepolicy、profile 摘要做统一 manifest chain。若保护交付包完整性，可借鉴；若只保护运行中 native core，arm96 当前 key fusion 更关键。 |
| MAC OUI 绑定 | arm96 源码已有，当前生成配置可能关闭 | 这不是 blackobfuscator 强项。arm96 loader/encrypt_core 支持 MAC OUI gate-check + key fusion；vc/cs profile 可开，cx/myt profile 关闭。当前 `gen/version.py` 显示关闭。 |

native-only 结论：blackobfuscator 值得 arm96 借鉴的不是加密强度，也不是把检测线程塞入 `libarm96`，而是可迁移到 `arm96server` 或 loader 启动窗口的“外部可观测反 hook/反 trace 细节”：匿名 RWX mapping、Frida 端口、server 自身/关键函数入口防 inline hook、fake check dispatch、tamper_exit 退出形态。arm96 在核心授权绑定、AES-GCM core 加密、server token/text hash、平台 witness、外部 PID tracking 方面更强。

### arm96server 当前反 trace 策略是否足够

当前 `arm96server` 的设计总体是合理的，并且比 blackobfuscator 的本进程 `TracerPid` 线程更适合 arm96 的架构：

- 不 ptrace app：V0.38 从 `PTRACE_SEIZE` 改为 PID tracking，避免破坏 translator 和 app 自身信号/反调试逻辑。
- 对 app 生效：扫描 UID 10000-19999、父进程为 `zygote_secondary` 或 orphan 到 init 的进程；新进程若 `TracerPid > 0` 直接 kill。
- 对已跟踪进程持续生效：每轮重读 tracked PID 的 `TracerPid`，命中后 kill tracer 和 app。
- 对工具进程抢先处理：扫描 `/proc/*/cmdline`，按 `gdbserver/lldb-server/strace/frida*` 命中后 kill，并写 V0.51 fault stamp。
- 自保护：main 和 guardian 都 `PR_SET_DUMPABLE=0`、block signals、`PTRACE_TRACEME`，利用单 tracer 规则让外部 attach 返回 EPERM。
- 反拆除：guardian 伪装、重启 main、kill tracked apps；loader 启动窗口用 `PTRACE_ATTACH` 探测 hollow server，失败则延迟投毒。

主要剩余缺口不是“没有周期检测”，而是：

- 3s 扫描窗口内仍有短时间 attach/dump 的机会；对高权限攻击者无法根除。
- 只看 `cmdline` 的工具名会被改名绕过；`TracerPid` 能抓 attach 后状态，但抓不到未 attach 的隐蔽 server。
- 未见端口探测、匿名 RWX maps 这类跨进程环境检测；可补在 arm96server，避免改 libarm96。
- server 自身完整性主要依赖 loader 的 token/text hash 和自保护；server 运行中未见 blackobfuscator 那种关键函数入口周期 prologue 检测。
- `arm96server` 目前仍运行在 `u:r:su:s0`，安全边界过大；这是比补小检测更优先的系统面风险。

因此建议：保留当前 `arm96server` PID tracking 架构，不把 blackobfuscator 的本进程线程照搬到 `libarm96`。优先增强 server 侧的“低侵入外部检测”：Frida 端口、本机可疑 RWX/Frida maps、server 自身入口 prologue/完整性巡检、随机化 scan cadence/fake check；同时推进 SELinux 去 `su:s0` 和 Binder/权限面收口。

## 7. 总体判断

外部报告对系统级暴露面（SELinux `execmem`、`su:s0`、Binder 无鉴权）判断基本有效，这些是 V0.61 后续最明显的 P0/P1 加固点。

但报告明显低估了 arm96 当前核心授权/解密链：它不是传统属性授权 + 单分支过期判断，而是以 AES-GCM 解密为中心的多 witness 数据域绑定。对“改时间常量、删 arm96server、patch 单个分支”的攻击结论需要降级：其中不少会触发错误 key、完整性失败或延迟故障。

下一轮最建议的落地方向：先收 Binder 与 SELinux 权限面，再补周期 challenge-response 和运行时 dump 成本提升。这样能同时解决真实系统面风险和报告中仍有部分成立的动态攻击面。


可以借鉴，但应放在 arm96server 或 loader 启动窗口：

Frida 端口探测
在 arm96server 中周期性检查 127.0.0.1:27042/27043，作为 cmdline 改名的补充。

匿名 RWX mapping 检测
由 arm96server 扫目标 app 的 /proc/<pid>/maps，检测可疑 rwxp anonymous mapping。这个要谨慎做白名单，因为 arm96/translator/JIT/memfd 场景可能有合法 executable mapping。

server 自身关键函数入口巡检
可在 arm96server 自己内部检查 scan_and_track、scan_and_kill_debuggers、read_proc_status 等关键函数首指令是否变成 branch trampoline。这个比改 libarm96 稳定得多。

随机化 scan cadence / fake check dispatch
当前固定 3s 扫描，易被观察。可以保留平均 3s，但加入小抖动，比如 2-4s；同时增加 fake check，提升自动 patch 成本。

tamper 处置形态
server 检测到强异常时，不一定只 kill；可以写 fault stamp，让 loader 后续走 V0.51 延迟投毒。这个与现有模型兼容。

我已经把这个补充同步进评估文档：arm96_blackobfuscator_v061_assessment.md。