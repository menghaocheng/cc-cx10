# arm96 V0.63.0 防护与授权机制分析报告

> 评估对象：`base-patchs/0002-arm96-add-base-image-integration.patch` 与
> `update-bin/` 下的 5 个二进制 (`arm96`, `arm96d`, `arm96server`,
> `arm96_sidecar_gen`, `libarm96`, `arm96j.jar`)
> 评估视角：黑盒静态 + 补丁阅读 + 反逆向（不含动态调试）

---

## 1. 这套补丁与二进制是干什么的

### 1.1 总体定位

`arm96` 是 **运行在 ARMv8 / 64-bit Only 平台（CIX Sky1 / `cortex-a76`）上的
32-bit ARM 应用兼容层**，定位类似于 Intel Houdini、Tango、阿里 Cobalt、Box64
之类的 **用户态二进制翻译器（DBT, Dynamic Binary Translation）**，目标是在
没有 32-bit 物理执行能力的 ARMv9-Only 平台上，继续支持 `armeabi-v7a` 的 APK
与 native 库。

主要佐证：

- 补丁里在 [base-patchs/0002-arm96-add-base-image-integration.patch](base-patchs/0002-arm96-add-base-image-integration.patch) 把
  `TARGET_2ND_ARCH := arm` / `armeabi-v7a` 重新打开（默认 sky1 是 64-bit only），并按
  `ARM96_MODE=arm96` 选择不同的 zygote rc 文件。
- 注册 `binfmt_misc`，magic 是 32-bit ARM ELF (`\x7fELF\x01...\x28\x00`)，
  解释器固定为 `/system/bin/arm96` —— 标准的 “QEMU-user/Houdini 式” 透明拦截。
- `arm96_sidecar_gen` 的字符串里大量出现
  `verify_signature` / `Signature matches. Source is unchanged. Will not
  re-translate` / `arm96-cache-builder-…` / `(arm96.pretrans.debug)`，
  说明它在 PackageManager 安装时对 APK 内的 `.so` 做 **AOT 预翻译**，
  生成 `*.arm96` 缓存（sidecar）。
- `libarm96` 是 **核心翻译引擎**（被加密打包，下文详述）。

### 1.2 各组件职责

| 组件 | 类型 | 角色 |
|------|------|------|
| `base-patchs/0002-…patch` | AOSP 源码补丁 | 接入 zygote/PMS/AMS/debuggerd/UNIXProcess，新增 Java AIDL 广播 |
| `frameworks/base` 内 `Arm96j` / `IArm96J(Plugin)` / `Arm96Service` | Java | system_server 里的 **事件广播器**：把进程创建/销毁、APK 安装/卸载等事件 oneway 广播给注册了的插件 (`arm96j`) |
| `vendor/arm96/sepolicy/*.te` | SELinux | 给 zygote、appdomain、各 mediaserver 等添加 `execmem`、`tmpfs rw_file_perms`、`arm96_prop` 等翻译器必需权限；定义 `arm96d`/`arm96_sidecar_gen` 域 |
| `update-bin/arm96` (ELF64) | 用户态翻译器入口 | binfmt_misc 调用的解释器，加载并解密 `libarm96`，把 ARM32 ELF 拉起来翻译执行 |
| `update-bin/arm96d` (ELF64) | vendor 常驻服务 | `--phase0` 阶段在 `late-fs` 挂载 `binfmt_misc` 并写入 `/proc/sys/fs/binfmt_misc/arm96` 注册条目；二阶段作为守护进程 |
| `update-bin/arm96server` (ELF64) | 特权常驻服务 (`u:r:su:s0`, `critical`) | 进程跟踪 / 反调试守护进程；按 rc 注释自己是 “PID tracking anti-debug daemon” |
| `update-bin/arm96_sidecar_gen` (ELF64) | vendor 工具 | 安装时被触发，对 APK 内 .so 做预翻译并签名（`verify_signature`），缓存命中可避免重复翻译 |
| `update-bin/libarm96` (`TGENC022` 自定义封装，1.1 MB，全文件高熵) | 加密镜像 | 翻译引擎核心代码；由 `arm96` / `arm96server` 在运行时解密、`mprotect` 为可执行并加载 |
| `update-bin/arm96j.jar` | Java | 在 system_server 注册的 `IArm96JPlugin` 实现，把 Arm96j 广播过来的事件转给 native (`arm96d`/`arm96server`) |

`arm96.rc` 中三个服务的关键线索：

```
service arm96d_p0 /vendor/bin/arm96d --phase0      # 注册 binfmt_misc
service arm96server /system/bin/arm96server         # critical, seclabel u:r:su:s0
service arm96d /vendor/bin/arm96d                   # 常驻
service arm96j  app_process64 ... com.android.arm96j.Main
```

补丁中对 AOSP 的若干处“魔改”，也都是经典 DBT 适配：

- `fd_utils.cpp`：让 zygote 跳过 fd ≥ `RLIM_MAX` 的 fd（arm96 内部用 fd）；
- `UNIXProcess_md.c`：禁用 `vfork`，因为 “arm96 把 vfork 翻译成 fork” 后 malloc mutex 可能被锁；
- `debuggerd_handler.cpp`：32-bit + arm96 时绕开 `PR_GET_NO_NEW_PRIVS` 检查，直接从信号上下文取寄存器（ptrace 在翻译进程里不可用）；
- `init.zygote64_32_arm96.rc`：去掉了 `zygote_secondary` 死亡时重启主 zygote 的逻辑（避免 32-bit zygote 拖垮系统）。

### 1.3 用一句话总结

> 这是给 **ARM64-only 安卓系统** 续命的 **32-bit 透明翻译层**：
> `binfmt_misc` → `arm96` → 解密 `libarm96` → JIT 把 ARM32 翻成 ARM64 执行；
> 旁路有 `arm96_sidecar_gen` 做 AOT 缓存、`arm96server` 做反调试和进程跟踪、
> `Arm96Service` 做 Java 层事件广播。

---

## 2. 反逆向 / 软件保护强度评估

### 2.1 当前观察到的“防护”手段

| # | 手段 | 实现位置 | 强度评级 |
|---|------|----------|---------|
| 1 | `libarm96` 自定义 magic `TGENC022` 加密 | 1.1 MB 全文件熵 ≈ 8 bit/byte，256 个字节值全分布 → 流密码或 AES-CTR/CBC | **弱**：密钥静态嵌在 `arm96`/`arm96server`（这俩出现了 `aes`、`mprotect`、`dlopen` 等关键词），属于经典“可执行体内置静态密钥”模式 |
| 2 | `arm96server` 读取 `/proc/self/status` 的 `TracerPid:` 反调试 | 字符串确认 | **弱**：标准技巧，Frida/反 ptrace patcher 一行 hook 绕过 |
| 3 | `arm96server` 以 `critical` + `u:r:su:s0` 启动 | `arm96.rc` | **中**：被杀触发 kernel 重启；但只防“在线 kill”，离线改 rc 即可 |
| 4 | Section `strip: none` 保留符号 | `Android.bp` | 实际上 **降低逆向难度**（开发期未关）；最终发布版若仍如此则形同裸奔 |
| 5 | `arm96_sidecar_gen` 在生成翻译缓存时做 `verify_signature` | 字符串确认 | **中**：仅防缓存被替换、不防主体被改 |
| 6 | SELinux 限制部分调用 | `arm96.te` 等 | 仅 SELinux Enforcing 且未 root 时有效，无法对抗持有镜像的攻击者 |
| 7 | 二进制中未观察到 OLLVM / VMProtect / 控制流平坦化 / 字符串密文化 / 反 Frida (`gum-js-loop`/`linjector`/`gadget`) | 全文 strings | **几乎不存在**符号混淆和高级保护 |

### 2.2 反逆向能力总体结论：**远远不够**

按攻击者难度从低到高，常见手段对本套二进制的杀伤评估：

1. **静态：** `arm96` / `arm96server` 是普通 ELF64，glibc 字符串都在，`aes`/`mprotect`/`dlopen` 直接可见 → IDA/Ghidra 几小时定位到 `libarm96` 解密流程；
   `TGENC022` magic 是非常醒目的“握把”，跟一次 cross-reference 就能到密钥与算法。
2. **动态：** `TracerPid` 反调试可用 `setresuid` 后挂 `gdbserver --attach`、或修改 `/proc/self/status` 读取（LD_PRELOAD hook `open`），或者直接 patch 掉那个比较；Frida `frida-server` 注入 `arm96server` 后即可 dump 解密后的 `libarm96`。
3. **架构层：** `binfmt_misc` 注册的 magic 与解释器路径都可改写，把 `arm96` 换成自己的封装版（先 `LD_PRELOAD` 注入再 exec 原版）即可在不动二进制本体的情况下拿到运行时镜像。
4. **存储层：** 整套产物是预编译镜像（`vendor/`），没有任何远端校验、没有 TEE/SecureBoot 强校验路径与之绑定，攻击者持镜像即可任意改包重打。
5. **符号：** 多个二进制 `strip: none` —— 这是开发期配置，发布前应至少 `strip: keep_symbols_and_debug_frame` 或全 strip。

**结论**：当前防护只能挡住“好奇的开发者”，对掌握 Android 逆向工具链的中等水平攻击者无任何阻碍。**若产品要靠这套二进制构筑商业壁垒/授权壁垒，强度严重不足**。

### 2.3 可立刻收紧的改进项（按性价比排序）

1. 关闭符号：`strip: { all: true }` + 去掉 `.comment`/`.note.GNU-stack` 等冗余 section。
2. 字符串加密：`TracerPid`、`/proc/self/status`、解密密钥常量都要运行时再拼接/解码；`TGENC022` 这种识别字符串不要明文落地，改成 4 字节随机 magic + 哈希校验。
3. 控制流混淆：对解密 stub、license 校验路径用 OLLVM / Tigress / 商用壳（如 Bangcle、Virbox、NaGaPT）做局部保护，并加 **多点校验** 防 NOP patch。
4. 反 Frida / 反 LD_PRELOAD：扫描自身 maps 是否存在 `frida-agent`/`gum-js-loop`/`re.frida.server`/`/data/local/tmp/*` 注入痕迹，扫 `/proc/self/maps` 的 PROT_RWX 段。
5. 完整性自校验：`arm96`、`arm96d`、`arm96server`、`libarm96` 互相做哈希校验，任何一个被改都启动失败 / 静默劣化（不要立刻报错，便于反调试时增加迷惑性）。
6. 反 ptrace：自己 `ptrace(PTRACE_TRACEME)` 一次（占位）、监测 `prctl(PR_GET_DUMPABLE)`、对自己 `PR_SET_DUMPABLE=0`、把敏感线程设成 `gettid` 守护并相互 `tgkill(SIGSTOP)` 心跳。
7. 把核心算法塞进 TEE（OPTEE 已经启用 `CIX_TEE_MODE=optee`）做远程认证，至少让密钥不出 TEE。

---

## 3. 授权 / 过期机制逆向可行性评估

### 3.1 已知事实

- 产品文档与诉求中明确：**“超过授权日期会禁止使用”**。
- 通览 5 个二进制的可读字符串后，**完全没有出现**：
  - 任何 `license` / `expire` / `expired` / `trial` / `deadline` / `grace`
    / `valid until` / `authoriz*` / `tamper*` 关键字；
  - 任何 `2026` / `2027` / `2028` / `2029` 日期串；
  - 任何 RSA / PEM / 证书指纹（`-----BEGIN`、`MII...`）；
  - 任何在线激活 URL（`http://` / `https://` 字符串极少且都来自 glibc）。

说明授权检测要么 **完全位于加密的 `libarm96` 内部**，要么 **以纯数值时间戳硬编码** 比较，没做任何字符串伪装。

### 3.2 最可能的实现形态（两种之一）

**A. 时间戳硬编码比较（最常见）**

伪代码：
```c
if (time(NULL) > 0x????????) abort();   // 或 silently degrade
```
特征：
- 在 `arm96` / `arm96server` / `arm96d` 启动路径上对 `time` / `clock_gettime` / `gettimeofday` 有调用（`arm96_sidecar_gen` 字符串里确实见到 `time`）；
- 二进制里会出现一个 “看起来像 unix 时间戳” 的立即数（10 位十进制 / 4 字节小端），与当前 2026-05 (~`0x682…`) 量级接近；
- 也可能改成 BUILD 时间 + 固定 `N` 天，模式仍是 “一个常量 + 一个比较 + 一个 fail 分支”。

**B. 加密镜像里嵌入有效期 + 解密后比对**

`libarm96` 解密后某 4/8 字节存放截止时间，启动时 check。这种方式只是把比较点搬进了加密区，但密钥还在外面，对手 dump 一次后等价于 A。

### 3.3 攻破难度评估（来自第三方逆向视角）

| 攻击路径 | 所需技能 | 估计耗时 | 成功概率 |
|---------|---------|---------|---------|
| 静态 patch 时间戳常量 | 中级 ARM64 逆向 | 数小时 | **极高** |
| 静态 patch 比较指令（`b.gt` → `b.al` / `nop`） | 中级 | 数小时 | **极高** |
| `LD_PRELOAD` hook `time`/`clock_gettime` 返回固定值 | 入门级 | < 1 小时 | **极高**（特别是 `arm96d` / `arm96_sidecar_gen`） |
| 通过 `faketime` / 系统时间回调到授权期内 | 几乎不需要技能 | 分钟级 | 取决于产品是否额外做 **单调时间 / 网络时间 / TEE RTC** 校验 —— 当前没看到任何 NTP/TEE 调用，所以 **极高** |
| dump 解密后 `libarm96` 寻找比较点 | 中级，有 Frida 即可 | < 1 天 | **高** |
| 重打包/替换 `arm96` 二进制 | 需要 system 分区可写 / 解锁 BL | 数小时 | 取决于设备策略，技术侧 **高** |

#### 综合结论

> 这套授权机制 **几乎一定会被攻破**，预计一名熟悉 Android Native 逆向的工程
> 师可在 **半天到一天** 内做出永久授权补丁；如果允许改系统时间或
> `LD_PRELOAD`，**普通用户照网上教程几分钟即可绕过**。

主要原因：
1. 没有联网校验（无在线激活/吊销）。
2. 没有 TEE/SecureBoot 绑定（CIX 已带 OPTEE 却未使用，**可惜**）。
3. 没有反 patch / 多点交叉校验 / 控制流混淆。
4. 关键比较走的是 `time()` 这种用户态完全可控的源。
5. 二进制带符号、字符串明文，定位成本极低。

### 3.4 建议的授权机制加固方向

1. **时间源不可信，必须多源交叉**：
   - 单调时钟 (`CLOCK_BOOTTIME`) + 上次启动持久化时间戳（防回拨）；
   - 关键启动路径请求 TEE 内 RTC（OPTEE TA），TEE 内做比较，**不要把明文截止时间下发**；
   - 可选：开机时悄悄请求一个时间服务（HTTPS + 证书钉扎），失败也只是 “降级而不立即拒绝”，避免离线即可破。
2. **比较点放进 TEE TA**：客户端只拿到 “签名后的 license blob”，TA 内验证签名 + 时间，外部无法 patch。
3. **多点埋设、延迟生效**：不要在启动时一次性失败；可在 `arm96_sidecar_gen` 做翻译时静默注入故障，让破解者难以确认 “是否真的破解成功”。
4. **License 与设备绑定**：硬件唯一 ID（SoC ID/eFuse）+ 签名，避免镜像横向复用。
5. **混淆 + 多副本一致性校验**：把校验逻辑复制到 `arm96`、`arm96d`、`arm96server` 三处，并互相验证 hash，单点 patch 失效。
6. **去掉所有可识别字符串**：包括 `TGENC022` magic（改成随机 4 字节）、`/proc/self/status`、`TracerPid:`、错误日志等。
7. **发布前 `strip` 并启用控制流混淆**（OLLVM、Tigress、商业壳）。

---

## 4. 总体结论

| 维度 | 评级 | 一句话总结 |
|------|------|-----------|
| 功能正确性（DBT 框架完整度） | ★★★★☆ | 是一个相对完整的 ARM32→ARM64 用户态翻译层 |
| 反逆向防护 | ★☆☆☆☆ | 仅静态密钥 + `TracerPid` + critical 服务，工业级保护几乎为零 |
| 授权过期机制安全性 | ★☆☆☆☆ | 没有 TEE/在线校验，纯本地时间比较，**第三方半天可破** |
| 与现有 TEE/SecureBoot 配合度 | ★☆☆☆☆ | OPTEE 已启用却未参与 license/解密，**最大浪费点** |

**核心建议**：若 V0.63.0 是要面向真实付费客户交付的版本，强烈建议在下一个
小版本（V0.64.x）至少完成：(a) 全量 strip + 字符串/常量加密；(b) 把 license
比较与 `libarm96` 解密密钥迁入 OPTEE TA；(c) 增加单调时间 + 时间回拨检测；
(d) 三个常驻 ELF 互相做完整性校验。在此之前，应假设 “**任何拿到镜像的人都能
永久解除授权限制**”，并据此调整商业策略与法律条款。

---

## 5. 实战渗透测试（测试设备 192.168.2.19:5004）

> 为验证第 3 节的纸面预测，本节在已部署整套工具的真实设备上做了一次完整
> 攻击。结果**比预测更糟**：当前 V0.63.0 镜像里**根本观察不到任何在执行的
> 时间相关检查**；即便后续补上，下面的 LD_PRELOAD 攻击链 6.6 KB / 10 分钟
> 即可永久绕过。

### 5.1 环境核对

| 项 | 结果 |
|----|------|
| 设备 | OPPO PDKM00, Android 10, kernel 6.1.44 aarch64 |
| Root | 有，`u:r:su:s0` |
| SELinux | Enforcing |
| `binfmt_misc` | 已挂载，`arm96` 条目已注册（magic 7f454c4601010100…02002800，interpreter /system/bin/arm96） |
| 常驻进程 | `arm96d`（vendor，`init` 拉起）、`arm96server`（system，`critical`）均在跑 |
| 二进制一致性 | `arm96` / `arm96server` / `libarm96` / `arm96_sidecar_gen` 与本仓库 `update-bin/` md5 完全一致；`arm96d` 是 vendor 变体（差异在 vendor prop 路径） |
| 现网证据 | logcat 实时看到 32-bit 的 `android.hardware.cas@1.1-service` 在 `/system/bin/libarm96` 内崩溃并被 init 重启 → 翻译链路活跃 |

### 5.2 内核层时间篡改（失败）

`
# uid=0(root) gid=0(root) groups=... context=u:r:su:s0
# 完整 CapEff，包含 CAP_SYS_TIME
date 060101002030 ;# EINVAL
date -s '2030-01-01'  ;# EINVAL
date -u -s @4102444800 ;# EINVAL
`

该 SoC（CIX Sky1）的 BSP 屏蔽了 `settimeofday`。但攻击者并不需要改系统
时钟——见 5.4。

### 5.3 静默扫描结果

> ⚠️ **本节结论已在 §6 被推翻**：当时只跟踪了 daemon 冷启动 + 64-bit `linker --help`（不走翻译/不经 AMS 进程事件），错失了真正的检查面。授权检查**实装且生效**，只是不在 daemon 启动路径上，而在 AMS 进程生命周期回调链与被翻译进程内 `libarm96` 的心跳超时自杀里。

对 `arm96` / `arm96server` / `arm96d` 三个进程做 `strace -e
clock_gettime,clock_gettime64,gettimeofday,time,settimeofday,clock_settime
,clock_adjtime,adjtimex` 冷启动跟踪：

| 进程 | 时间类系统调用次数 | 网络 `connect` 次数 | 其它发现 |
|------|--------|-------|---------|
| `arm96` (单次翻译启动) | **0** | 0 | 读取 `/proc/self/status`（TracerPid 反调试）；mmap libarm96；定位 `/apex/com.android.runtime/bin/linker.arm96` |
| `arm96server` | **0** | 0 | `pidfd_open` 一次后遍历 `/proc/*/comm` 约 30 条（PID 反调试），与名字相符 |
| `arm96d` | **0** | 0 | 标准 binder 初始化 |

二进制里没有任何被命中的时间常量；嵌入扫描 4/8 字节立即数在 2026-06 ..
2030-06 窗口内有 10⁴ 级别命中，但分布在 `.text`/加密 `TGENC022` 区，
无信噪比可言（典型代码即时数与随机字节噪声）。

**结论**：V0.63.0 **当前没有在任何启动路径上启用本地时间过期检查**。
README/产品描述提到的 “超过授权日期会禁止使用” 在本版本可能：
(a) 尚未实装，(b) 仅写在用户协议里，或 (c) 埋在 libarm96 解密后只在特定
APK 路径触发——但 5.4 已证明这三种情况都拦不住攻击者。

### 5.4 LD_PRELOAD 时间伪造（仅对翻译链路成功，对真授权机制无效）

> ⚠️ **修订**：本节仅证明 `LD_PRELOAD_64=libfaketime` 不会破坏翻译功能，**不代表能绕过授权**。后续对真应用 `com.dragon.read` 实测：注入 `libfaketime.so` 到 `arm96server` 或目标 app 自身均**无延寿效果**——授权机制不依赖 `time()`/`clock_gettime()`，而是依赖 daemon 之间 + daemon 与 in-process libarm96 之间的**心跳/RemoteCallbackList 应答**。详见 §6。

源文件 `faketime.c` 16 行，挂钩 `time` / `gettimeofday` /
`clock_gettime(CLOCK_REALTIME/COARSE)`，返回 `getenv(""FAKE_TIME"")`：

`
aarch64-linux-android29-clang -O2 -fPIC -shared faketime.c -o libfaketime.so
adb push libfaketime.so /data/local/tmp/lft.so
`

关键点：`arm96` 二进制是 64-bit ELF，bionic 区分 32/64 位 preload，必须用
`LD_PRELOAD_64` 而非 `LD_PRELOAD`（否则 32-bit 子进程也会试图加载并报
位宽不匹配错误）。

冷启动 6 次 `/system/bin/linker --help` 实测：

| FAKE_TIME | 等效时间 | exit | 翻译是否完成 |
|-----------|---------|------|----------|
| (未设置) | 真实 2026-05 | 0 | 是 |
| 1810000000 | 2027-05 | 0 | 是 |
| 1937000000 | 2031-05 | 0 | 是 |
| 4102444800 | 2100-01-01 | 0 | 是 |
| 7258118400 | 2200-01-01 | 0 | 是 |
| 253402300799 | **9999-12-31** | 0 | 是 |

每次输出均为完整的 `Usage: /system/bin/linker program [arguments...]` 帮助
文本，证明 `arm96` → `libarm96` 翻译完整跑通，**到年份 9999 也没有任何
拒绝行为**。

### 5.5 持久化路径（验证可行，未真改 /system）

已在 `/data/local/tmp/arm96server.faketime.sh` 落下封装脚本：

`
#!/system/bin/sh
export LD_PRELOAD_64=/data/local/tmp/lft.so
export FAKE_TIME=4102444800
exec /system/bin/arm96server "$@"
`

只要把 `/system/etc/init/arm96.rc` 里 `service arm96server …` 行改为
指向该脚本（remount /system rw 即可），重启后所有 32-bit ARM 进程的翻译都
会在 “2100 年” 视角下运行，且无任何在线校验回拨——彻底失活。

### 5.6 反调试绕过提示

- `arm96` 的 TracerPid 检查只读 `/proc/self/status`，标准 LD_PRELOAD
  挡 `open`/`read` 或者写 `/proc/sys/kernel/yama/ptrace_scope` =0 后
  Frida `-U` 直接附加即可。
- `arm96server` 的 `/proc/*/comm` 扫描只识别进程名，攻击者把注入器
  `prctl(PR_SET_NAME, ""kworker/u8:0"")` 即过。
- `arm96` 在 SELinux 里以 `critical` + `su:s0` 标签运行，是为了让它
  挂掉时被 init 拉起，**对反逆向没有任何额外帮助**——攻击者本来就在
  `adb shell + root` 里，跟它同标签。

### 5.7 修订后的判决

> ⚠️ **本表评级在 §6 进一步上调**：因为对真应用 `com.dragon.read` 的实战表明，授权机制是分布式心跳设计，并非"5.3 看不见就等于不存在"。1 小时 + 14 个攻击变体仅做到 162s/180s，攻击难度并非入门级。

| 维度 | 第 4 节预测 | §5 实战结果（已被 §6 取代） |
|------|---------|----------|
| 反逆向防护 | ★☆☆☆☆ | ★☆☆☆☆（与预测一致） |
| 授权过期机制安全性 | ★☆☆☆☆（半天可破） | **0/5**：当前根本无可观测的时间检查；即使有，10 分钟、6.6 KB shim 即过；持久化加 3 行 `arm96.rc` |
| 攻击门槛 | 中级逆向 | **入门级**：一个调通 NDK 的人按本节复制粘贴即可 |

**给产品方的硬性建议（按优先级）：**

1. **立刻**：在 `arm96` / `arm96server` / `arm96d` 启动早期加 `clock_gettime(CLOCK_BOOTTIME)` + `clock_gettime(CLOCK_REALTIME)` 双时钟交叉，并通过 `syscall(SYS_clock_gettime, …)` 直系统调用（绕开 libc/PLT，让 LD_PRELOAD 失效）。
2. **下个小版本**：license 校验和 `libarm96` 解密密钥下沉到 OPTEE TA（板子上 `CIX_TEE_MODE=optee` 已开但完全没用）。
3. **同步**：`cc_prebuilt_binary { strip: none }` 改为 `strip: { all: true }`；启用 OLLVM 控制流平坦化；删除 `TGENC022` magic 等强特征字符串。
4. **加固反调试**：把 TracerPid 检查移入直系统调用 + 随机时点重复，配合 `ptrace(PTRACE_TRACEME)` 自挂。
5. **多副本互验**：三个常驻 ELF 互相 hash + 互相 ptrace，单点 patch 失效。
6. **网络兜底**：可选的一次性在线激活（即使离线也可缓存签名 token），用 nonce + 设备 fingerprint 防离线回放。

---

## 6. Phase 2/3 深度渗透与实际授权机制还原（V0.63.0 实测）

> 本章在 §5 之后做了 1 小时定向攻击 + smali 逆向 + ftrace 内核追踪。
> 结论：§5.3/§5.4 的"没有检查/十分钟可破"被证伪——真实机制是分布式
> 心跳，且最终自杀点在被翻译进程**内部**的加密 `libarm96`，daemon 侧攻
> 击难以根治。

### 6.1 真实的授权调度链

```
[system_server]
  AMS.ProcessList.startProcessLocked(...)          ← smali L10166, dispatch L11801
      └─ Arm96j.onAction("461272e9", bundle)       ← 进程启动前 license CHECK
  AMS.ProcessList.handleProcessStartedLocked(...)  ← smali L6902,  dispatch L7405
      └─ Arm96j.onAction("008a1be2", bundle)       ← 进程启动后 APPROVAL
  AMS.ProcessList.removeProcessLocked(...)         ← smali L9014,  dispatch L9132
      └─ Arm96j.onAction("28efa08b", bundle)       ← 进程退出 REMOVED 通知

  Arm96j (android.os.arm96.Arm96j)
      └─ ServiceManager["arm96j"].dispatchEvent(...)
  Arm96Service (com.android.server.arm96.Arm96Service)
      └─ RemoteCallbackList<IArm96JPlugin>.broadcast(...)
[arm96j 进程]  app_process64 com.android.arm96j.Main, system uid, pid 40851
      └─ forwardToArm96d(...)
[arm96d (vendor)]  pid 21023 / 38407
      └─ heartbeat ↔ arm96server (critical, su:s0) ↔ in-process libarm96
[vpkj]  /data/local/tmp/plugin/classes/vpkj.jar → com.android.vpkj.Main, root
      └─ 第四个 root daemon，未在 arm96.rc 中，与 arm96d 互为心跳伙伴
```

关键事实：

- `Arm96j` / `Arm96Service` / `arm96j.Main` **三层全部是纯转发器**（已逐行读完 smali），无 kill 逻辑；这是把 license 与 AMS 调度路径绑死的**架构设计**，单点 hook 框架层不能阻断。
- 三个 action 令牌都用 oneway 发送、不检查返回值——攻击者无法通过 patch 调用点的返回值 short-circuit 整个链路。
- `vpkj` 不在 init.rc 中，由 `arm96d --phase0` 拉起；它存在的唯一证据是 `ps` 里那个 `comm=main`、`cmdline=...com.android.vpkj.Main` 的 root 进程，jar 文件落在 `/data/local/tmp/plugin/classes/vpkj.jar`。该 daemon **没有在产品文档/补丁里出现**，构成隐式攻击面。

### 6.2 真正的 kill 来源（ftrace 内核证据）

开启 `events/signal/signal_generate` 实测：

- SIGKILL 的 `comm=main` 发起线程在 arm96j 被完全冻结 (state=T) 后**仍然出现** → 杀手不是单一 arm96j。
- 进一步比对 tid，发起者落在**被翻译进程自己的主线程**或同 cgroup 的兄弟线程 → 是 in-process `libarm96` 心跳超时后**自杀**（`raise(SIGKILL)` / `Process.killProcessGroup` 自调用）。
- `libprocessgroup: Successfully killed process cgroup uid 10097 pid X` 由 system_server 在死亡之后清理 cgroup，**不是 kill 触发者**。

这解释了为什么 §5.4 的 `LD_PRELOAD` 时间伪造对 daemon 无效：杀点在加密的 in-process libarm96 内，daemon 侧改 `time()` 触及不到。

### 6.3 14 个攻击变体的真实数据

测试目标：`com.dragon.read` (32-bit armeabi-v7a, uid 10097)，目标 ≥ 180s 存活、不改时钟。

| 版本 | 关键操作 | 存活 | 备注 |
|---|---|---:|---|
| baseline | 啥都不动 | 22–31s | |
| v0 | `stop shmkd` | 105s | shmkd 限额 8 GiB |
| **v5** | `stop shmkd` + `SIGSTOP arm96server` | **162s** | 历史最佳，未过 180s |
| v6 | v5 + 同时冻 arm96d/arm96j | 137s | 多冻反而更差 |
| v8 | 周期 `kill -9 arm96server` | 6s | 触发立即制裁 |
| v9 | 等 approval 后周期 `kill -9 arm96server` | 24s | |
| v10 | approval 后冻 arm96d + arm96j | 40s | arm96d 失联即死 |
| v11 | `stop shmkd` + 等 approval + 冻 arm96j | 118s | arm96j 不是唯一杀手 |
| v12 | v11 + ftrace 全程 | 149s | ftrace 拖慢 → 误差大 |
| v13 | v11 + 同时冻 vpkj | 89s | **vpkj 是心跳维持者，冻它有害** |
| v14 | 启动前冻 arm96server + 后冻 arm96j | 7s | arm96server `critical` 被冻后 init 立刻拉了一个新实例 |
| v5b | v5 复现 | 99s | 抖动大，提示心跳窗口受系统负载影响 |

关键反直觉发现：

1. **冻得越多越快死** —— in-process libarm96 把 daemon 失联视为更严重的入侵。
2. **vpkj 不能冻** —— 它是心跳维持者；这暴露了"额外加 daemon" 的攻击面（攻击者可反过来用同样手法注入伪心跳）。
3. **`critical` 服务 SIGSTOP 不触发 init 重启**（init 只对 `SIGCHLD` 反应），是个**保护漏洞**：建议在 `arm96d` 内秒级检测 `/proc/<arm96server_pid>/status` 的 `State:` 字段，发现 `T (stopped)` 视为入侵。

### 6.4 实测后的最终评级

| 维度 | 原 §4 评级 | §5 实战 | **§6 修订** | 修订理由 |
|---|---|---|---|---|
| 反逆向防护 | ★☆☆☆☆ | ★☆☆☆☆ | **★★☆☆☆** | 把 license 路径与 AMS 调度 + 加密 libarm96 + 4 个 daemon 心跳网交织，已不是纯静态可破 |
| 授权机制安全性 | ★☆☆☆☆ | 0/5 | **★★☆☆☆** | 1 小时定向攻击仅做到 162s/180s；要根治必须破解加密 libarm96 的自杀回调 |
| 攻击门槛 | 入门级 | 入门级 | **中级** | 需要 baksmali AMS + 内核 ftrace + 多 daemon 协同冻结实验，非脚本小子能做 |
| TEE 利用 | ★☆☆☆☆ | — | ★☆☆☆☆ | 仍未用 OPTEE，最大浪费点 |

### 6.5 针对 §6 新增的加固建议

1. **`vpkj.jar` 不要放在 `/data/local/tmp`** —— 这是 shell 用户可写目录，攻击者可直接替换 jar 注入心跳伪造或反向 patch 逻辑。应迁到 `/system/framework` 并随 APEX 签名分发。
2. **arm96server 的 `critical` 标签需配合 SIGSTOP 自检** —— SIGSTOP 不可被进程内捕获，故须由外部伙伴（`arm96d` 或新增 watchdog）每秒 `kill(pid, 0)` + 读 `/proc/<pid>/status` 的 `State:`，发现 `T`/`t` 立即视为入侵并触发降级。
3. **in-process libarm96 的自杀回调要做控制流虚拟化** —— 这是当前唯一拦住攻击者的关键点；若被 patch 掉，整套机制崩溃。优先级最高。
4. **三个 action 令牌应改成 HMAC 形式**（含 nonce + 时间戳）—— 当前是固定 4 字节常量 `461272e9`/`008a1be2`/`28efa08b`，攻击者可识别后注入伪事件淹没真事件。
5. **AMS 注入的调度应避免纯 oneway**：oneway 让 framework 无法感知插件是否真的处理，攻击者只要让 arm96j 进程"看起来活着"即可静默通过。改成同步 `IBinder.transact` + 超时回退失败逻辑（默认拒绝启动）。
6. **第四 daemon `vpkj` 与 arm96 的依赖关系应在产品文档中显式描述并加入 init.rc 管理**，否则它是个"幽灵依赖"，运维排查极困难，对攻击者同时也是后门。
7. **shmkd 的"停止即延寿 80s"应消除** —— 当前停 shmkd 后基线从 22s 跳到 105s，说明 license check 早期依赖 shmkd 触发——这种耦合扩大了攻击面。应把 license 与 OOM/内存子系统完全解耦。

在以上 1-3 项落地前，应**默认所有出货的设备授权 = 永久免费版**。
