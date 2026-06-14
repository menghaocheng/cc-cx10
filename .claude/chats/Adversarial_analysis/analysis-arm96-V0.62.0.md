# arm96 工作区分析报告

> 分析对象：[base-patchs/0002-arm96-add-base-image-integration.patch](base-patchs/0002-arm96-add-base-image-integration.patch) 和 [update-bin/](update-bin) 下的全部二进制。

---

## 1. 这套补丁/二进制到底干啥用

### 1.1 总体定位

这是一套面向 **CIX sky1 (arm64-only SoC)** 的 **ARM 32 位 → ARM 64 位用户态翻译器**（类似 Apple `Rosetta 2`、`box86/box64` 的思路），用来让一台**没有 32 位执行能力**的 ARMv8 设备仍能运行 32 位 Android 应用（armeabi-v7a / armeabi）。

构建开关：`ARM96_MODE=arm96`，会让 BoardConfig 同时开启 `TARGET_2ND_ARCH := arm` 并注入 `vendor/arm96/arm96.mk`。

### 1.2 各组件职责（从补丁 + 二进制字符串推断）

| 组件 | 大小 | 角色 |
|---|---|---|
| [update-bin/arm96](update-bin/arm96) | 598 KB | 用户态翻译器/加载器（被 binfmt_misc 通过 `:arm96:M::\x7fELF...` 触发，劫持所有 32 位 ELF 的执行） |
| [update-bin/libarm96](update-bin/libarm96) | 1.1 MB | 翻译器运行库（JIT/兼容运行时） |
| [update-bin/arm96server](update-bin/arm96server) | 663 KB | 全局守护进程，**承载授权检测与反调试**（含 `scan_and_kill_debuggers` / `scan_and_track` / `v051_emit_fault_stamp` / `read_proc_status` 等符号），运行在 `u:r:su:s0` 域（root + 无 SELinux 限制） |
| [update-bin/arm96_sidecar_gen](update-bin/arm96_sidecar_gen) | 563 KB | **AOT 预翻译器**：扫描已安装应用，把 32 位 ELF 静态翻译成 sidecar（带 CRC / signature 缓存：`check-signature` / `verify_signature` / `Signature matches. Source is unchanged.`） |
| [update-bin/arm96d](update-bin/arm96d) | 127 KB | vendor 域守护进程，监听 PMS/AMS 事件，触发 `arm96_sidecar_gen` 做 dexopt-like 预翻译；通过 binder 暴露 `com.android.arm96d.IArm96D` |
| [update-bin/arm96j.jar](update-bin/arm96j.jar) | 9 KB | system_server 内的 Java 插件，注册到 `IArm96J` broker（`ServiceManager` 名 `arm96j`），把 PMS/AMS 事件透传给 native 侧 |

### 1.3 AOSP 集成点（补丁实际改动）

- **构建系统**：`build-android.sh` 增加 `-oarm96`、soong variable `Arm96`、`WITH_ARM96` 宏。
- **Zygote**：替换为 [init.zygote64_32_arm96.rc](base-patchs/0002-arm96-add-base-image-integration.patch)，**关掉 `zygote_secondary` 的 `onrestart restart zygote`**（避免 32 位 zygote 翻译失败时把整套 zygote 拖崩）。
- **frameworks/base**：
  - 新增 `android.os.arm96.Arm96j` / `IArm96J` / `IArm96JPlugin` AIDL。
  - `ProcessList`、`PackageManagerService` 在进程启动 / 进程死亡 / 安装 / 卸载等关键点调用 `Arm96j.onAction(...)`，把 uid/pid/packageName/codePaths/abi 等元数据 oneway 广播给所有注册的 plugin。
  - `Arm96BinderService` 注册到 `ServiceManager("arm96j")`，绑定 `Arm96Service`（`RemoteCallbackList<IArm96JPlugin>`）。
- **libcore**：`UNIXProcess_md.c` 在 `arm96+__arm__` 下禁用 `vfork`（arm96 把 vfork 翻成 fork 但没跑 pre/post handlers，会卡 malloc 锁）。
- **system/core**：
  - `fd_utils.cpp` 在 zygote fork 前忽略 `> RLIMIT_NOFILE` 的 fd（arm96 内部 fd）。
  - `debuggerd_handler.cpp` 在 arm96 下跳过 ptrace 模式拿寄存器，改走信号上下文（因为翻译进程不能被正常 ptrace）。
- **SELinux**：新增 `arm96d`、`arm96_sidecar_gen` 域；定义宏 `arm96_translated($1)`，给 zygote/webview_zygote/appdomain/audioserver/cameraserver/drmserver/mediaserver/mediacodec/dex2oat 等批量授予 `self:process execmem`、`tmpfs rw_file_perms` 以及读取 `arm96_prop` 属性。
- **init**：`arm96d_p0` 在 `late-fs` 阶段挂载 `binfmt_misc` 并注册 `:arm96:M::\x7fELF...`，把 32 位 ELF 的执行劫持到 `/system/bin/arm96`。

---

## 2. 安全性评估

### 2.1 攻击面：广 **且** 高权限

| 风险点 | 严重度 | 说明 |
|---|---|---|
| `arm96server` 的 SELinux 标签是 `u:r:su:s0` | **极高** | [arm96.rc](base-patchs/0002-arm96-add-base-image-integration.patch) 里 `seclabel u:r:su:s0`：以 root + 类 `su` 域常驻。一旦被打穿等价于 root shell，没有 SELinux 兜底。 |
| `Arm96BinderService.registerPlugin / dispatchEvent` **没有 permission check** | **高** | 代码里只有 `Slog.i` + `// TODO: Add permission check`。任何能 `ServiceManager.getService("arm96j")` 的进程都可以：① 注册成为 plugin → 拿到全系统的安装/进程/卸载事件（含 packageName、codePaths、uid）→ **隐私 / 旁路探测**；② 通过 `dispatchEvent` **伪造**安装事件，骗 `arm96d` 去对攻击者控制路径做预翻译。 |
| `dispatchEvent` 的 `extras: PersistableBundle` 被原样转发给所有 plugin | 中 | 没有大小限制 / 字段白名单，恶意 caller 可以把巨大 bundle 广播给所有插件造成 DoS。 |
| `/data/local/tmp/.arm96_fault_stamp` 与 `/data/local/tmp/.arm96_last_err` | **高** | `/data/local/tmp/` 对 `shell` 用户（adb）可写，对**任何 app 也基本可读**。当反调试 / 授权失败时把状态写在世界可读路径，等于把检测结果暴露在用户态，攻击者可以 **预创建** 或 **删除** 这些文件来污染检测逻辑。 |
| `fd_utils.cpp` 改动：fd 超过 `RLIMIT_NOFILE` 直接忽略 | 中 | zygote fork 时不会复制/检查这些 fd，arm96 内部 fd 被默认信任。如果攻击者能让 zygote 在 fork 前持有这种"超限 fd"，可能逃过现有的 fd 检测。 |
| `arm96_translated()` 给一大批关键服务（mediaserver、cameraserver、audioserver、appdomain…）批量授予 `self:process execmem` | 高 | 大幅扩大可写可执行内存（W^X 削弱）面，等于把翻译器需要的"放宽"扩散到了所有这些进程。这是动态翻译的必要代价，但显著降低 SELinux 收紧度。 |
| `debuggerd_handler.cpp` 在 arm96+__arm__ 下**跳过 PR_GET_NO_NEW_PRIVS 检查** | 中 | 走 fallback 路径处理 crash，意味着 NO_NEW_PRIVS 进程的崩溃路径行为变化，可能给攻击者一条新的内核可观察侧信道。 |
| `arm96d` 的 binfmt_misc 注册位置 `/proc/sys/fs/binfmt_misc/arm96` + 路径 `/system/bin/arm96:POC` | 低-中 | 字符串里出现 `:POC`，疑似 POC/未清理标识，说明这套二进制可能还没到 GA 等级；也可能只是 binfmt_misc magic 中的一部分。 |
| 多处 `Slog.i` 打印 callingUid/pid/packageName 等敏感信息 | 低 | 日志冗长，正式发布前需收敛。 |

### 2.2 结论

> **目前的安全成熟度：不足以作为产品级特性发布。**

最关键的两个问题：

1. **AIDL broker 没有调用方校验**，是直接的系统级越权入口；
2. **`arm96server` 跑在 `su` 域**，破坏了 Treble/SELinux 设计的纵深防御。

在修复以上两点之前，不建议把这套补丁合入对外发布的固件。

---

## 3. 授权 / 时间锁机制评估（逆向视角）

### 3.1 静态特征

从二进制字符串里能直接看到的"授权"相关蛛丝马迹：

```
arm96server 内符号：
  scan_and_kill_debuggers
  scan_and_track
  read_proc_status
  v051_emit_fault_stamp        ← 版本 0.5.1，"发出故障戳"
  TracerPid:                   ← 反调试关键字
  /proc/%d/status
  /proc/%d/maps
  /proc/uptime
  /data/local/tmp/.arm96_fault_stamp

arm96d 内符号：
  persist.sys.ap3219402ff1
  persist.sys.ap2901e99e08
  persist.sys.p15c68464

property_contexts 中：
  persist.vendor.p3a9c7d1e u:object_r:arm96_prop:s0
```

值得注意的是：

- **没有**任何 `RSA / ECDSA / X509 / EVP_ / PEM_ / HMAC / SHA256 / openssl / mbedtls` 字符串出现在 `arm96` / `arm96server` / `libarm96`。
- **没有**任何 `licen* / expir* / trial / authoriz*` 字符串（只有 libc 通用错误信息）。
- **没有**网络授权服务器（无 `http`、`curl`、`ssl_connect` 等符号）。

→ 由此基本可断定：**这是一个纯本地的"时间炸弹 + 故障戳"机制，没有数字签名授权文件，没有联网激活**。

### 3.2 推测的工作流程

结合可见符号 + 调用关系，可以高置信度地复原如下：

```mermaid
flowchart TD
  A[arm96 / arm96server 启动] --> B[读 clock_gettime CLOCK_REALTIME / __kernel_gettimeofday]
  B --> C{当前时间 >= 内嵌 deadline 常量?}
  C -- 否 --> D[读 persist.sys.ap3219402ff1<br/>检查"上次见过的时间"防回拨]
  D --> E{回拨幅度过大?}
  E -- 否 --> F[scan_and_kill_debuggers 反调试 → 正常服务]
  C -- 是 --> G[v051_emit_fault_stamp]
  E -- 是 --> G
  G --> H[写 /data/local/tmp/.arm96_fault_stamp<br/>写 persist.sys.p15c68464=锁定]
  H --> I[后续 arm96 / arm96d / sidecar_gen 看到 stamp → 拒绝翻译，进程退出/abort]
```

关键点：
- 三个混淆的 `persist.sys.p*` 属性极大概率分别承担：① 首次安装时间戳；② 最近一次"健康检查"通过的时间（防回拨）；③ 锁定/熔断状态位。
- `/data/local/tmp/.arm96_fault_stamp` 是 cross-process 的熔断标识。
- 反调试只是 `/proc/<pid>/status` 里读 `TracerPid` 然后 `tgkill` 调试器——非常古典、非常可破。
- 找不到加密学原语意味着**校验数据本身不带签名**，被回写/篡改后无法自检。

### 3.3 第三方绕过可行性评估

| 攻击手段 | 可行性 | 成本 | 说明 |
|---|---|---|---|
| **系统时间回拨** | ★★★☆☆ | 低 | `date -s` / `clock_settime`（需 root）。但 daemon 可能记录 `persist.sys.ap*` 中的"最近见过的时间"做防回拨——需配套清掉这些 prop。 |
| **resetprop 清掉 `persist.sys.p15c68464` / `persist.vendor.p3a9c7d1e` 等** | ★★★★☆ | 低 | Magisk + resetprop 一行命令，且属性 SELinux 类型是 `arm96_prop`，未限制 reset。 |
| **删 / 预写 `/data/local/tmp/.arm96_fault_stamp`** | ★★★★★ | 极低 | 路径在公共目录；shell uid 即可写。crontab/init.d 脚本就能做到"开机即清"。 |
| **静态 patch 二进制**：把比较 deadline 的指令改成 `mov w0,#0; ret` 或反转 b.cs/b.hi 跳转 | ★★★★★ | 中 | ARM64 ELF 未加壳、未签名（vendor partition 走 AVB，但 userdebug 设备或 unlock bootloader 直接绕过）；常量很可能集中在 `.rodata` 的一两个 `time_t`。 |
| **LD_PRELOAD / Frida hook `clock_gettime` / `gettimeofday`** | ★★★★☆ | 低 | 反调试只看 `TracerPid`，Frida 用 `gadget` 注入或预加载 .so 即可绕过；arm96server 是 native 进程，注入难度比 zygote 低。 |
| **删 binfmt_misc 注册，自行写 loader** | ★★☆☆☆ | 高 | 等于自己重做 arm96，无意义。 |
| **AVB / dm-verity 防护**？ | — | — | 补丁里没有任何"自校验自己 .text 段哈希"的痕迹（没有 SHA256 / CRC32 of self 字符串）。也就是说，**只要能落地修改 `/system/bin/arm96server`，就没有任何二级校验**。AVB 在 user build 上能挡硬盘改动，但一旦设备 unlock 或厂商签了 patched image，防线为零。 |

### 3.4 综合结论

- 该机制属于**典型的"本地时间炸弹 + 混淆属性 + ptrace 反调试"**组合，是**最弱**的一档授权设计。
- **没有签名授权文件、没有联网激活、没有 TEE/Keymaster 单调计数器、没有自身代码签名校验**。
- 一个具备基础 ARM64 RE 能力的工程师（IDA/Ghidra + Frida）**几小时之内**就可以：
  1. 在 `.rodata` 里定位 deadline 常量并改成 `0x7fffffff`；
  2. 或者把"超期分支"中的 `b.hi`/`b.cs` 改成 `b.al` 反向 / `nop`；
  3. 或者 hook `clock_gettime` 让 daemon "永远活在过期前"。
- 在已 root 的设备上，**用户层面甚至不需要 RE**，仅靠时间回拨 + resetprop + 删 stamp 文件就能延寿。

> **判定：在不引入数字签名授权文件 + TEE 时间源 + 二进制自校验之前，这套授权机制对任何有动机的第三方都形同虚设。**

---

## 4. 加固建议（如果要继续使用本地时间锁）

按优先级排序：

1. **改用签名授权文件**：包含 `device_id`（绑机器）+ `not_after`（截止时间）+ Ed25519 签名，公钥硬编码在 `arm96server` 内并通过自身 `.text` 哈希自校验（防 patch）。
2. **TEE 单调计数器 / 安全时间源**：用 Keymaster `Tag::USAGE_EXPIRE_DATETIME` 或 Trusty 应用持有"上次见过的时间"，防止本地时间回拨。
3. **自校验 + 抗 patch**：`arm96server` 启动时 `mmap('/proc/self/exe')` 自算 SHA-256，与内嵌签名比对；二级守护进程（`arm96`、`libarm96`、`arm96d`）交叉哈希。
4. **Fault stamp 改用 vendor 私有目录**：`/data/vendor/arm96/`，权限 `0700 system`，避免 `/data/local/tmp` 这种公共路径。
5. **AIDL broker 鉴权**：`Arm96BinderService` 加 `Binder.getCallingUid() == SYSTEM_UID` 或 `permission.MANAGE_ARM96` 的强校验，禁用第三方 app 注册 plugin / dispatchEvent。
6. **`arm96server` 改回严格 SELinux 域**：删除 `seclabel u:r:su:s0`，新建 `arm96server.te` 并按需 `allow`，禁止它执行任意命令。
7. **去掉 `:POC` 字符串与冗余 Slog**：发布前清理调试痕迹，避免暴露内部状态机字段名。
8. **(可选) 增加联网激活心跳**：周期性回服务器（带 nonce + 设备指纹），即使本地被改也会在心跳节点失败。

---

## 5. 一句话总结

- **是什么**：CIX sky1 上让 ARM64-only SoC 跑 ARM32 应用的用户态翻译器套件（arm96 / libarm96 / arm96server / arm96_sidecar_gen / arm96d / arm96j）+ 对 AOSP 的深度集成补丁。
- **安全性**：集成本身**还不到生产级**——AIDL 没鉴权、`arm96server` 跑在 `su` 域、熔断状态写在 `/data/local/tmp` 公共路径，是必须先修的硬伤。
- **授权机制**：纯本地时间锁 + 混淆 `persist.sys.*` 属性 + ptrace 反调试，**无密码学保护、无自校验**，对懂 ARM64 RE 的第三方在小时级即可破解；对仅有 root 权限的普通用户也能通过时间回拨 + resetprop + 删 stamp 文件简单绕过。**目前的强度，远不足以作为商业授权依赖。**
