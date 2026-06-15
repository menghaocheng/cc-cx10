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

1. 先在 **vmp 仓库自带测试 ELF** 上验证多 bl、栈传参、x0 返回的完整链路
   （遵循"每项 vmpacker 能力先单测 ELF 再上六件套"的规矩）。如发现 ABI 缺口
   （x8 / 栈传参），在 vmpacker 补。
2. target 扩到 `platform_check()`（loader.c:2214，5 层编排）+ 参数派发逻辑
   （`-v/-i/-h/-a` 分支）。

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
