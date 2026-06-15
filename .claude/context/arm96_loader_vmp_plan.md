# arm96(loader) VMP 保护规划（V0.71 / V0.72 / V0.73）

> 目标：把 arm96 loader 承载的核心策略（导入 libarm96、参数处理、关联组件
> md5/hash 校验、硬件平台绑定）用 VMPacker 做 VM 化保护，消除逆向路标。
> 本文为规划文档，供后续独立会话实施。**编写时（2026-06-15）尚未改任何代码。**

---

## 0. 背景与现状

- 当前发布版本 **V0.70.0**：libarm96 内部 shadow_guard 段已 VMP 化
  （`VMP_CORE_TARGETS=0x1049C0-0x104A90:shadow_guard`），verify_mustpass 8/11
  与基线持平。本规划是把 VMP **从 libarm96 扩展到 arm96(loader) 自身**。
- vmp 独立仓库：`/home/mc/2T/cx10/vmp`（GitHub `menghaocheng/vmp`，分支 `main/1.0`）。
- arm96 加固仓库：`android10/vendor/hello/arm96`（分支 `main/1.0`）。

---

## 1. 为什么保护 loader 比保护 libarm96 难一个量级

arm96 的 `.text` 同时被两条链强依赖，VMP 改写 `.text` 会同时动到它们：

1. **运行时自校验** `integrity_check_text()`（`hardening/src/loader.c:534`）：
   开机读 `/proc/self/exe`，对**第一个 `PT_LOAD+PF_X`** 段做 SHA-256，与 build 时
   patch 进 sentinel（`_text_hash_sentinel`，loader.c:490）的 hash 比对，不符即静默退出。
2. **Prong5 密钥融合**：build 时 `LOADER_TEXT_HASH16 = SHA-256(arm96 .text)[:16]`
   （`build_arm96_runtime_artifacts.sh:418`，函数 `_get_elf_text_hash16`）喂给
   `encrypt_core.py` 加密 libarm96；并且 `reconstruct_key()`（loader.c:3084，
   "V0.55 loader self .text hash fusion"）在**运行时**再 hash 一次自身 `.text`
   复原 Prong5 —— **自指环**（VM 化后的 reconstruct_key 自身字节也在被 hash 范围内）。

结论：只要 VMP 阶段插在正确位置，这两条链会自然自洽；插错位置 = 100% 解密失败或开机即退。

---

## 2. 唯一正确的 build 插入点

现有顺序（`build_arm96_runtime_artifacts.sh`）：

```
编译/链接 loader.c (clang+OLLVM 或 gcc，~line 274/289)
  → [UPX]                                   (~line 295)
  → V0.28 sentinel patch (.text 自校验 hash)  (~line 301)   ← 必须在 VMP 之后
  → V0.48 critical-data hash                 (~line 352)
  → V0.55 算 LOADER_TEXT_HASH16 (Prong5)      (~line 418)   ← 必须在 VMP 之后
  → encrypt_core 用 Prong5 加密 libarm96       (~line 431)
```

**VMP 阶段必须插在"链接完成"与"sentinel patch"之间**（若开 UPX，则在 UPX 之前）。
这样 sentinel hash、Prong5、运行时自校验三者都覆盖 VM 化之后的最终 `.text`，链路一致。

**硬约束**：VMP 注入物（解释器 + 字节码）必须落在**同一个、第一个 `PF_X` PT_LOAD 段**内
（rx-cave 风格），否则 build 时 hash 的"第一个 PF_X 段"与运行时不一致即崩。
→ loader 不像 libarm96 有现成 RX island，**V0.71 首先要量 loader 第一个 PF_X 段是否有
   足够 cave 容纳解释器+字节码；不足则需新增 cave / 调整链接布局**（R3）。

---

## 3. vmpacker 现状能力（已实测/读码确认）

| 能力 | 状态 | 出处 |
|---|---|---|
| svc / adr / adrp / ldr-literal / sp / x16–x28 | ✅ 已支持 | 上轮 shadow_guard 52/52 翻译 |
| BL / BLR / BR / RET 翻译骨架 | ✅ `OpCallNative`/`OpCallReg`/`OpBrReg`/`OpRet` | `pkg/arch/arm64/tr_branch.go:72-96` |
| BR 函数内跳转（CFF computed-goto 回跳） | ✅ addr_map 二分查找 | `stub/linux/arm64/vm_handlers/h_system.h:42` |
| FindFunctionByAddr 接受 .text 外 RX 地址 | ✅ V0.70 已放宽到 PT_LOAD(PF_X) | `pkg/binary/elf/packer.go`（commit 6a114ed） |
| **native call ABI** | ⚠️ **只传 x0–x7、只回收 x0** | `h_system.h:19-35` |

### 3.1 native-call ABI 限制（关键，决定目标选择）

`h_call_nat` / `h_call_reg`：

```c
vm->R[0] = fn(vm->R[0], vm->R[1], ... vm->R[7]);   // 只传 x0-x7，只回收 x0
```

- **不传 x8**（不支持以 x8 传 sret 指针的"返回结构体"函数）。
- **不处理栈传参**（>8 整型参数的函数）。
- **不处理 v0–v7 浮点/SIMD 参数**。

→ **目标选择硬规则**：被 VM 化的函数，其对外调用必须满足"≤8 整型参数、返回值在 x0"。
  loader.c 里 `open/read/sha256/memcpy/_exit/malloc/getprop` 等绝大多数满足。
  **不可** VM 化调用 `snprintf/printf` 等变参函数、或返回结构体的函数所在代码段。

- 栈缓冲区传参（`sub sp,#N; mov x0,sp; bl callee`）**OK**：vm_stk 是真实 mmap 内存，
  native callee 写进去、VM 读回来 —— shadow_guard 的 `clock_gettime` 已验证此模式。

---

## 4. 性能红线（不能 VM 化的部分）

arm96 每次 32 位进程启动都跑，但函数级只跑一次，VM 化几百条指令是微秒级，相对
memfd 解密+exec 可忽略。**但绝对不能 VM 化 `sha256()` 与 `aes128` 的内层循环**
（几千次迭代的紧循环，VM 税爆炸）。

原则：**只 VM 化"编排/决策"逻辑，crypto 原语保持原生。**

---

## 5. OLLVM 叠加决策

候选函数现带 `OBFUS_CFF_BCF_SUB` 等注解。CFF 把函数变成巨型 switch-dispatch，
VM 化后字节码体积爆炸 + 依赖 computed-branch。

**第一版策略**：对被选为 VMP target 的函数**关掉 OLLVM**（把该函数的 OBFUS 宏改为
`noinline`，宏定义见 loader.c:159-172），让 VMP 成为唯一保护层。字节码干净、出错面小。
后续再评估是否叠加 OLLVM+VMP。

---

## 6. 分期版本规划

### V0.71 —— 脚手架 + 1 个叶子函数（de-risk，地基）

1. build 流水线新增 `vmpacker` 对 arm96(OUT_BIN) 的阶段，插在**链接后、sentinel patch 前**
   （`build_arm96_runtime_artifacts.sh`，UPX 之前）。
2. version.txt 新增配置块：`VMP_LOADER_ENABLE` / `VMP_LOADER_MODE` / `VMP_LOADER_TARGETS`
   （复用 `VMP_CORE_BIN` / `VMP_CORE_TOOL_SHA256` 校验机制）。
3. **首个 target 选最自包含的叶子函数**：`platform_check_isa_features()`（loader.c:1461，
   读 MRS、比对、返回 int，几乎无 bl）或 `platform_check_cpu_part()`（loader.c:1430）。
4. **先量 loader 第一个 PF_X 段 cave 容量**（R3）；不足则解决注入空间问题。
5. 验收：`arm96 -v` 正常、**libarm96 能解密**（证明 Prong5 一致）、`verify_mustpass`
   与基线持平、reboot（TC-004/007）正常、dmesg 无 SIGILL/segfault。

> V0.71 把"loader 上做 VMP"的 build 时序 + 自校验/Prong5 自洽 + 运行时正确性全部跑通，
> 是后续所有版本的地基。**务必先单独跑通再扩 target。**

> **2026-06-15 状态更新**: 脚手架代码（`vmp_loader.py`/build 阶段/version.txt 配置块）
> 已实现并验证 Prong5 自洽（libarm96 可解密）、热重启 verify_mustpass 正常。但 V0.71.0
> 冷启动（cold boot）TC-004 FAIL（`zygote_secondary`/`vendor.cas-hal-1-1` 反复
> `signal 11` crash-loop）。排查发现该崩溃**与 V0.71 loader-VMP 无关**——V0.71.1
> （`VMP_LOADER_ENABLE=0`，仅关闭本计划的脚手架）冷启动仍崩溃；继续排查定位到根因是
> **V0.70 引入的 `VMP_CORE_TARGETS=shadow_guard`**（libarm96 侧 VMP，与本计划无关）。
> V0.71.2（`VMP_LOADER_ENABLE=0` + `VMP_CORE_ENABLE=0`）冷启动 11/11 mustpass 通过，
> 详见 `.claude/context/tango_hardening.md` 第 5 节版本状态矩阵的 0.69/0.70/0.71.x 行。
> 结论：**本计划的 V0.71 脚手架本身工作正常**，`VMP_LOADER_ENABLE=0` 当前只是为了
> 与同期发现的 V0.70 回归隔离测试变量；待 V0.72 评估时可考虑重新开启
> `VMP_LOADER_ENABLE=1`（同时保持 `VMP_CORE_ENABLE=0`）单独验证冷启动是否仍通过。

### V0.72 —— 验证/补强 native-call，纳入平台绑定主体

> **Step 0（已完成，2026-06-16）**：在 V0.71.2 基础上将 `VMP_LOADER_ENABLE` 改为 `1`
> （`VMP_CORE_ENABLE` 保持 `0`），`CUSTOM_VERSION=V0.72.0`，重新构建六件套并走
> `docker rm -f con4` 重建容器 → docker cp 新六件套 → `docker restart con4` 冷启动验证：
> - `sys.boot_completed=1` 全程保持，`zygote_secondary` 全程 `running`，dmesg 清空后
>   无新增 `signal 11`（仅既有的 `vendor.cas-hal-1-1` 1 次，与 V0.71.2 一致）。
> - `verify_mustpass` 复测 **11 PASS / 0 FAIL / 0 SKIP**（首次跑 TC-008 因
>   arm96server 刚重启的时序问题 FAIL，复测通过，判定为偶发非回归）。
> - **结论：V0.71 的 loader-VMP 脚手架（含 `platform_check_isa_features` 叶子函数
>   VM 化）在冷启动下是安全的，`VMP_LOADER_ENABLE=1` 已转正为 V0.72.0 默认配置。**
>   这意味着 V0.70/V0.71.0 的冷启动 crash-loop 根因确认为 `VMP_CORE_TARGETS=shadow_guard`
>   （独立问题，见下方 VMP_CORE/shadow_guard 一节），与 loader-VMP 无关。
> - 详见 `.claude/context/tango_hardening.md` 版本状态矩阵 0.72.0 行。

> **第二个 target（已完成，2026-06-16，V0.72.1）**：在 V0.72.0 基础上把
> `platform_check_cpu_part()`（loader.c:1430，MRS MIDR_EL1 + `PLATFORM_CPU_PARTS`
> 字符串解析循环）由 `OBFUS_CFF` 改为 `noinline,used`（关闭 OLLVM，同 V0.71 做法），
> 加入 `VMP_LOADER_TARGETS=platform_check_isa_features,platform_check_cpu_part`，
> `CUSTOM_VERSION=V0.72.1`。同样走 `docker rm -f con4` 冷启动验证：
> - seg0 grow 0x814fc→0x8c268，rx-cave payload=12819B（max=0x9d68，余量约 8KB）。
> - `sys.boot_completed=1` 全程、`zygote_secondary` 全程 running、dmesg 全程
>   **无任何 signal 11**（含 cas-hal-1-1 本次也未触发，比 0.72.0 更干净）。
> - `verify_mustpass` 复测 **11 PASS / 0 FAIL / 0 SKIP**（首次同样 TC-008 偶发 FAIL）。
> - **意义**：`platform_check_cpu_part` 含分支/循环/字符串解析（非纯直线代码），
>   验证了 vmpacker 对这类目标在冷启动下也正确。详见 tango_hardening.md 0.72.1 行。
>
> 下一步：继续 native-call ABI 扩展（target 扩到 `platform_check()`）。

> **Step 1（已完成，2026-06-16）**：写了一个贴近 `platform_check()` 调用形态的
> 测试 ELF（`check_orchestration(mode)`，源码暂存于 `tmp/demo_platform_check_shape.c`，
> 未提交进 vmp 仓库）：
> - 5 次 `bl` 到单参 helper（`helper_l1/l2/l3a/l3b/fail_gate`），覆盖 platform_check
>   的"多次连续 bl + 结果做 &&/早退分支 + errno 单参 bl"形态。
> - 4 条不同分支路径（mode=0~3），每条路径 `x0` 返回值不同。
> - `vmpacker -func check_orchestration`（note 注入模式）翻译无 WARN/ERROR，
>   保护后在 aarch64 宿主上跑 4 种 mode，输出与未保护基线**逐字节一致**
>   （`GATE:L1` / `OK:1` / `GATE:errno=42` / `OK_WITH_ERRNO`，exit code 3）。
> - **结论**：多 `bl`、条件早退分支、单参 native call、`x0` 返回值——
>   `platform_check()` 实际会用到的这些形态 vmpacker 均正确，**无需先补
>   栈传参/x8 ABI**（platform_check 内部调用均 ≤8 整型参且非 sret/变参，
>   §3.1 R2 风险在该函数范围内不触发）。

> **Step 1.5（已完成，2026-06-16）**：在同一测试 ELF 上追加**嵌套 VM 重入**验证——
> `vmpacker -func check_orchestration,helper_l1`（一次性对调用者和被调用者两个函数
> 同时做 VM 保护），即 `check_orchestration` 的字节码在解释执行中 `h_call_nat` 到
> `helper_l1`，而 `helper_l1` 的入口此时也已被替换成 trampoline→`vm_entry_token`
> （第二个 `func_id=1, token=0xA5000001`）。
> - vmpacker 翻译/注入均无 WARN/ERROR（`Translated: 31/31` + `3/3`，token 表
>   `entries: 2`）。
> - 运行 4 种 mode，输出与未保护基线**逐字节一致**（`GATE:L1/OK:1/GATE:errno=42/
>   OK_WITH_ERRNO`，exit=3）。
> - **结论**：解释器对 `vm_entry_token` 的重入是安全的（VM 上下文按调用帧隔离，
>   不会被内层调用覆盖外层状态）。**此前风险评估中"嵌套 VM 重入未验证"这一条已解除**——
>   `platform_check()` 整体 VM 化后内部 `bl platform_check_cpu_part/isa_features`
>   落到已是 trampoline 的入口，可以正常工作。

> **Step 1.6（已完成，2026-06-16）**：新增测试 ELF（`tmp/demo_sigaction_shape.c`，
> 未提交进 vmp 仓库）覆盖风险点2——`struct sigaction sa; sa.sa_handler = fn;` 这种
> "ADRP+ADD 取函数地址 → 写入栈上结构体字段 → 经字段 BLR 间接调用" 形态：
> - `check_with_handler(mode)`：栈上 `struct simple_sa{ handler; flags; }`，按 mode
>   选择 `handler_a`/`handler_b` 两个函数地址存入字段，再经字段做 BLR 调用，
>   handler 写全局变量留下可观察副作用。
> - 反汇编含 `ADRP/ADD`(×2，取两个函数地址) + `STP/LDP`(栈帧) + `BLR`(间接调用) +
>   `ADRP/LDR`(全局变量访问)，13/13 翻译无 WARN/ERROR。
> - 保护后两种 mode 输出与基线**逐字节一致**（`HANDLER_A:1000`/`HANDLER_B:2001`，
>   exit=209=2001&0xFF）。
> - **结论**：风险点2（`sa.sa_handler = platform_sigill_exit` 这类"结构体字段存
>   函数指针 + ADRP/ADD 取址 + BLR 间接调用"模式）已验证可行，**已解除**。

> **Step 2 = V0.72.2（已完成并冷启动验收通过，2026-06-16）**：把 `platform_check`
> 编排主体纳入 loader-VMP。过程中撞到一个**真实的 vmpacker 能力边界**并解决：
>
> - **首次尝试失败（直接把 `platform_check` 设为 target）**：vmpacker 报
>   `translation aborted: 6 unsupported instruction(s) in platform_check`（`Translated: 42/48`），
>   **拒绝产出**。6 条不支持指令全是 **NEON/SIMD**（`MOVI v0.16b,#0` / `STP q0,q0` /
>   `STR q`），来源是 `struct sigaction sa`（约 150 B）的 `memset` 零初始化 + 字段写入
>   被编译器**向量化**。VMP 的 VM 是纯 GPR（x0–x30+sp，无 V 寄存器），无法翻译。
>   **注意这是进步**：对比 V0.69 的静默产崩溃码，vmpacker 现在遇不支持指令会 abort，
>   不再"构建成功、运行期崩"。
> - **`target("general-regs-only")` 此路不通**：与 fortified `memset`（`always_inline` +
>   默认 NEON target）冲突，编译报 `inlining failed ... target specific option mismatch`。
> - **采用的解法 = 拆函数**（loader.c）：新增 `platform_check_core(struct sigaction *old_sa)`
>   承载**全部编排逻辑**（含在原位置 `sigaction(SIGILL, old_sa, NULL)` 恢复 handler，
>   语义逐字节保持）；`platform_check()` 退回 `OBFUS_CFF`、只保留 NEON 结构体初始化 +
>   安装 handler + `platform_check_core(&old_sa)` 一句委托。`old_sa` 经 x0 传入
>   （native-call-in ABI）。VMP target 由 `platform_check` 改为 `platform_check_core`。
>   core 无大结构体 → 纯 GPR → **完整翻译，0 不支持指令**。它内部 `bl` 到同样 VM 化的
>   `cpu_part`/`isa_features`（嵌套 VM 重入，Step 1.5 已验证）。
> - **构建**：`VMP_LOADER_TARGETS=platform_check_isa_features,platform_check_cpu_part,platform_check_core`，
>   seg0 grow 0x8168c→0x8c268，rx-cave payload=13167B（max=0x9bd8，余量约 26KB），三 target 全 VM 化。
> - **冷启动验收**（`docker rm -f con4`→mb4 重建基线[镜像内烤的是 V0.69.0，确实
>   crash-loop 作对照]→docker cp V0.72.2 六件套→`docker restart con4`）：
>   `sys.boot_completed=1`、zygote/zygote_secondary 40s 全程 running 无 flapping、
>   dmesg **零 signal 11**（比 0.72.0 更干净）。`verify_mustpass` 复测 **11 PASS /
>   0 FAIL / 0 SKIP**（首次 TC-008 偶发 FAIL，既有时序问题）。
> - **结论**：`platform_check` 编排主体（含嵌套 VM 调用 cpu_part/isa_features）已纳入
>   loader-VMP 并冷启动安全。**风险点3（fail_exit_gate inline + 函数复杂度）实测不触发**——
>   真正的边界是 NEON，靠拆函数把 NEON 留在 native 侧规避，而非函数大小。

剩余可选方向（非阻塞）：
1. ~~先在 **vmp 仓库自带测试 ELF** 上验证多 bl、栈传参、x0 返回的完整链路~~ **已完成**。
2. ~~target 扩到 `platform_check()` 编排主体~~ **已完成（V0.72.2，见上）**。
3. 参数派发逻辑（`-v/-i/-h/-a` 分支）下沉为 VMP target——尚未做；同样需先排查是否含
   NEON（字符串/buffer 处理易被向量化），策略同上（把 NEON 段留 native、只 VM 编排）。
4. **教训固化**：选 loader-VMP target 前，先 `objdump -d` 看目标函数有无 `q`/`v` 向量
   寄存器或 `MOVI`/`STP q`；有则要么换更小的纯编排子函数，要么拆分。`memset`/`memcpy`
   大结构体、`-O` 自动向量化都是 NEON 来源。

> **VMP_CORE/shadow_guard 冷启动回归（V0.70，独立问题）**：本次发现的另一个问题——
> `VMP_CORE_TARGETS=shadow_guard` 在冷启动下让 32 位翻译进程 SIGSEGV——与本计划的
> loader-VMP 无关，修复涉及 vmpacker 对 sp/寄存器上下文在冷启动场景下的正确性，
> 工作量可能不小。建议作为**独立任务**排期（可以是 V0.72 的一部分，也可以推迟到
> V0.73 之后），不要和上面的 loader-VMP Step 0 / native-call ABI 扩展混在一个改动里，
> 避免变量耦合（这正是这次排查耗时的原因）。详见 `[[v069-vmp-regression]]` 记忆
> 与 `tango_hardening.md` 第 5 节 0.70 行。

### V0.73 —— 导入/密钥编排（最后做，最敏感）

1. VM 化 `reconstruct_key`（loader.c:2835）的**编排部分**（读组件、比 hash、XOR 串联），
   **保留 sha256/aes 原语原生**；以及 memfd 导入决策逻辑、关联组件 md5 校验 glue。
2. 因触碰 Prong5 自指环，需额外做 reboot / 授权过期（TC-004/006/007）回归。

---

## 7. 风险登记

| ID | 风险 | 缓解 |
|---|---|---|
| R1 | 自指环：VMP 注入须严格落在第一个 PF_X 段内，否则 build/runtime hash 口径不一致 → 解密失败 | V0.71 即验证 libarm96 能否解密 |
| R2 | native-call ABI：目标函数对外调用须 ≤8 整型参 / x0 返回；变参、sret 函数所在代码不可入选 | 目标选择规则 + V0.72 单测 ELF 验证 |
| R3 | 体积：loader 无现成 RX island，需确认第一个 PF_X 段 cave 容量 | V0.71 先量，不足则新增 cave/调链接布局 |
| R4 | 性能：crypto 原语紧循环不可 VM 化 | 只 VM 化编排逻辑 |

---

## 8. 关键代码定位速查

| 项 | 位置 |
|---|---|
| 运行时 .text 自校验 | `hardening/src/loader.c:534` `integrity_check_text()` |
| 自校验 sentinel（.data） | `loader.c:490` `_text_hash_sentinel[40]` |
| Prong5 运行时复原（自指环） | `loader.c:3084` `reconstruct_key()` 内 V0.55 段 |
| 平台绑定编排（5 层） | `loader.c:2214` `platform_check()` |
| 平台-CPU part（叶子候选） | `loader.c:1430` `platform_check_cpu_part()` |
| 平台-ISA features（叶子候选） | `loader.c:1461` `platform_check_isa_features()` |
| OBFUS 宏定义 | `loader.c:159-172` |
| build：编译 loader | `build_arm96_runtime_artifacts.sh:274/289` |
| build：sentinel patch | `build_arm96_runtime_artifacts.sh:301`（VMP 须在此前） |
| build：Prong5 计算 | `build_arm96_runtime_artifacts.sh:418` |
| build：encrypt_core 调用 | `build_arm96_runtime_artifacts.sh:431` |
| Prong5 融合实现 | `hardening/src/encrypt_core.py:421` |
| vmpacker native-call handler | `vmp/stub/linux/arm64/vm_handlers/h_system.h:19` |
| vmpacker BL/BLR/BR 翻译 | `vmp/pkg/arch/arm64/tr_branch.go:72` |
| 现有 libarm96 VMP 接线（参考样板） | `hardening/src/main.py` `_inject_core_stub` / `_apply_vmp_core` |
| 现有 VMP 配置样板 | `hardening/src/version.txt` `VMP_CORE_*` 块 |

---

## 9. 实施约定提醒

- 环境2（192.168.2.x）：编译机 `ssh -o BatchMode=yes mc@192.168.2.18`，仓库
  `/home/mc/2T/cx10`，测试设备 `adb connect 192.168.2.19:5004`。
- 六件套硬耦合（arm96/arm96server/arm96d/libarm96），必须同批构建整套替换。
- 构建：`cd android10/vendor/hello/arm96 && ./make.sh --build cx`；读
  `build/last_build.env` 取实际 OUT_BIN/VERSION。
- 每改一项 vmpacker 能力，先用 vmp 自带测试 ELF 单独验证，再上六件套整体验证。
