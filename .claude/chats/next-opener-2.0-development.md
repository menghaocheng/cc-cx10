# next-opener — 2.0 开发接手文档（2026-08-01）

> 开新会话继续 2.0 时先读这份 + [[chat41-2.0-tango32-boundary-wiring]]。
> 本文记录本次会话里"边界清单之外"的决策级分析（六件套盘点 / 1.0→2.0 切换策略 / 可调试vs加固 / 逆向策略）。
> 配套：memory `tc008-guardian-cascade-unreliable`；chat41 = tango32 边界接线清单。

## 项目双线定位
- **1.0**（`main/1.0`）：基于现成二进制的二次加固线，快速见效、最终不上生产。已改名去 tango 化、系统侧 FW 依赖切到 arm96。当前发布 V0.73.x，授权到期 2026-10-01。
- **2.0**（`main/2.0`）：从原理从头实现 ARM32→ARM64 动态翻译，**最终上生产的目标**。冻结在 tip `94ee459`（2026-02）。

## 核心决定（本次会话拍板）
1. **2.0 近期只开发功能、不上加固**（加固作 release 后置分层，dev 默认全关）。
2. **预翻译 / sidecar_gen 砍掉**（实测收益不明显、默认已关）；**独立 license 服务不要**（1.0 也早已把授权收进 loader）。
3. 目标：先把**动态翻译核心**跑通到可调试、能过真实程序，再谈其它。

---

## 1) 1.0 六件套业务盘点
根本区别：加固复杂度主要来自**两个原始黑盒二进制**；其余四件是自研外壳。

| 件 | 来源 | 业务承载 | 加固耦合 |
|---|---|---|---|
| **arm96(loader)** | 自研 C 静态 | binfmt 入口(重构 argv 成 `--exe ... -- ...`)+ CLI 门面 + AES-GCM 解密 libarm96→memfd 执行 + 7-prong 密钥派生 + 反调试 + 平台绑定 + 授权执法(setitimer/杀孤儿) + 启 arm96server | 自身 .text hash=Prong5,硬耦合 |
| **libarm96(core)** | **原始 tango_translator 黑盒** | ARM32→ARM64 动态翻译引擎 | 被 7-prong 融合密钥加密,绑定对象 |
| **arm96server** | 自研 C | 反调试看门狗(扫 32 位进程 TracerPid>0 即杀)+ guardian 互保/级联 + PTRACE_TRACEME 自保护 | .text 派生 token=Prong4,硬耦合 |
| **arm96d** | 自研 C++ binder | ①binfmt 注册(--phase0)②安装期预编译守护 | .text hash=Prong6,硬耦合 |
| **arm96_sidecar_gen** | **原始 tango_pretranslator 黑盒** | 安装期预翻译(.arm96) | 独立 sidecar 密钥,松耦合 |
| **arm96j.jar** | 自研 Java | 转发 PMS 安装事件给 arm96d | 无绑定,松耦合 |

**投影到 2.0**："因黑盒而生"的一整层(输入指纹门禁/字符串擦除/sig-bypass patch/stub 注入/relayout/VMP 硬啃 stripped PIE/memfd 解密)在 2.0 **直接消失**——有源码后防护编译期内建。近期 2.0 交付集合 ≈ **翻译核心 + 极简 binfmt 注册**(arm96server/加密壳/预翻译/license 全不上)。

## 2) 1.0 → 2.0 切换策略（关键，别走错）
- **结构事实**：`main/2.0` 是 `main/1.0` 的**祖先**(2.0 领先 1.0=0,1.0 领先 2.0=188)。翻译器源码两分支**几乎逐字相同**(188 提交里只有 1 个碰过 `src/tango_translator_reimpl`,仅改 main.cpp 4 行)。1.0 那 188 提交几乎全是外围(hardening/改名/构建/验收),没动翻译引擎。
- **FW 依赖在外层 AOSP 树**(`/home/mc/4T/cx10/android10` 自身 git 仓,分支 `debug/20260522_arm96_dbg2`),不在 arm96 加固仓。见提交 `6289e51 remove legacy tango integration` / `97bf3e919ac arm96 add base image integration`——tango 与 arm96 抢同一批**单占槽位**(init.zygote64_32_*.rc / fd_utils.cpp / debuggerd / PMS / sepolicy / binfmt),**系统级互斥,不能同时集成**。
- **⚠️ 坑**：直接切冻结的 `main/2.0` 去构建,产出的是旧 tango 交付(tango.mk/init/sepolicy/模块名 tango_translator_reimpl),**接不上当前 arm96 系统**。
- **✅ 正确切法**：**从 `main/1.0` tip 拉一条翻译器开发分支**(如 `main/2.1` 或 `dev/translator`)——1.0 已 =「2.0 翻译器源码 + arm96 命名/构建/FW 对齐」,一步到位。仅在新分支放开"1.0 不动翻译器源码"约束,`main/1.0` 本身仍只做加固。
- **要捞回的**：1.0 删掉的翻译器文档 `git checkout main/2.0 -- design/ doc/`(design/tango_translator_design.md、implementation_todo.md、doc/instruction-support.md、TROUBLESHOOTING_LOG.md、QA_TEST_CASES.md、ARM 手册)。
- **必继承**:系统侧命名/FW 契约(binfmt `POC`、交付名 arm96/libarm96/arm96server、`vendor/arm96`、zygote64_32_arm96.rc)。
- **注意**:翻译器改动会流进硬耦合核,dev 阶段用干净可调二进制即可,loader/core 是否二分体等要加固时再定,**现在别过早决定结构**。

## 3) 可调试 vs 加固（架构原则）
- 加固与可调试**本质对立**。1.0 的每样加固都在"让人调不了":反调试(gdb/frida attach 被弹/自杀)、guardian(杀被 trace 的进程)、.text 自校验(下断点即失败)、加密+memfd(无符号)、OLLVM/VMP(控制流天书)、平台绑定/密钥融合(锁死硬件)、strip+全静默。
- **原则**:核心可调试 + 加固作**可开关、后置、release 才施加**的一层,dev 默认全关。2.0 有源码 → 加固能做成**编译期条件开关**(默认 off)+ 后置 release 变换;攻击者只拿 release 产物,编译期开关对威胁模型足够,不损安全。
- **别做**:把 anti-debug/自校验/平台检查织进翻译器源码默认路径(=不可调,最典型死法)。
- 反向风险:加固路径只在 release 走会烂掉(TC-008 就是活例);release 加固要保留独立验收(1.0 的 verify_mustpass)。

## 4) 逆向策略（后续开发怎么用原二进制）
- 分两种模式:**(a)静态逆向**(读汇编)价值随进度**下降**、易抄错、IP 风险高;**(b)动态 oracle**(跑原二进制差分)价值**一直高**。
- **判断**:后续静态逆向降到接近零,不再做主驱动。指令覆盖/JIT/VFP-NEON **照 ARM 官方手册**(DDI0406 for ARMv7/Thumb、ARMv8 ARM for AArch64)。原二进制**当动态 oracle**:攻 app_process32/syscall/信号做差分基准。静态逆向仅救急。
- cleanroom/IP:手册实现 + 原件当黑盒 oracle,技术和法律都更干净。

---

## 现场遗留 & 状态（交接时）
- **arm96 仓** `main/1.0`:干净、与 origin 同步;3 个 symlink 的 skip-worktree 已复原。
- **顶层 cc-cx10** `main`:干净、已推送(chat40/41、本文)。
- **AOSP 树** `android10`:分支 `debug/20260522_arm96_dbg2`(arm96 集成态)。
- **设备** `192.168.2.19:5004`:长效版(到期 2026-10-01)稳定;`/data/local/tmp/` 留了差分素材 `tango_translator`(v2.1.0) + `hello_asm`(ARM32 最小样本,已验证原版能独立翻译执行)。
- **内核驱动** tango32 ABI v2.0:`/home/mc/4T/cx10/tango32_host/`(= 内核树 cix_cs8180,逐字节相同),built-in 编进内核,完整现成。

## 下一步（新会话入口）
1. 读本文 + chat41 接线清单。
2. 决定并建 2.0 开发分支(建议从 `main/1.0` 拉,见 §2),捞回 `design/`+`doc/`。
3. 按 chat41 的 A(SET_MM)→B(compat syscall) 接线,用 `/data/local/tmp` 差分素材 + 原版 oracle 逐项验证。
4. 指令级 JIT 缺口(`b00b399c`/Thumb 位)是另一条并行轨。
