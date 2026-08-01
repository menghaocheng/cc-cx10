# chat41 — 2.0 tango32 边界接线清单（2026-08-01）

## 背景（为什么有这份清单）
- **1.0**：基于现成二进制的二次加固线（loader/libarm96/arm96server/arm96d/arm96_sidecar_gen/arm96j 六件套）。已改名去 tango 化、系统侧 FW 依赖已切到 arm96。
- **2.0**：从原理从头实现 ARM32→ARM64 动态翻译，冻结在 `main/2.0`（tip `94ee459`，2026-02，是 1.0 的祖先；翻译器源码两分支几乎逐字相同）。冻结态里 tangolic(license) 未工作、pretranslator 暂忽略，目标是先把**动态翻译核心**跑通。
- **决定**：2.0 近期**只开发功能、不上加固**（加固与可调试本质对立，作为 release 后置分层处理）；预翻译/sidecar 默认关、近期不投入；只继承系统侧命名/FW 契约（binfmt `POC`、arm96 交付名、`vendor/arm96`、zygote64_32_arm96.rc 等）。

## 翻译系统真实构成（三层，dev 不含加固层）
1. **binfmt_misc**：把 ARM32 ELF 路由到解释器（内核特性 + 用户态注册）。
2. **翻译器（用户态）**：`src/tango_translator_reimpl` 的 JIT，模块名 `arm96_translator_reimpl`（兼容 `tango_translator_reimpl`）。
3. **tango32 内核驱动**：misc 设备 `/dev/tango32`（major 10 minor 48），ABI **v2.0**。源码 `/home/mc/4T/cx10/tango32_host/`（= 内核树 `cix_cs8180/linux/drivers/tango32/`，两处逐字节相同 md5 `ba0cc3959c22`），**built-in 编进内核**。核心作用：操纵 `TIF_32BIT` 让 `in_compat_syscall()` 为真，逐 syscall 把线程切进 32 位 compat 态，借内核现成 compat 路径处理内存布局/struct 尺寸/ioctl/getdents64/lseek/robust_list。

## 关键发现（已实测坐实）
- 原版 `tango_translator`(v2.1.0) **能独立跑**：`./tango_translator [--no-pretrans] PROGRAM`，不需 binfmt/zygote/license，`/dev/tango32` 在位即可 → 可作**差分 oracle**。`prebuilts/tango_translator` 原件仍在（remove-tango 只删集成，没删本体）。
- 差分对照**不需要**系统同时集成 tango 和 arm96（它俩抢 binfmt/zygote 单占槽位，互斥）；直接调用原二进制即可。
- **2.0 边界是半成品**：`Tango32Device` wrapper 写全、ABI 对齐，但**只接了 getVersion（版本检查）**；`setMm`/`setMmapBase`/`compat*` **调用点为 0**。compat 敏感 syscall 全用原生 64 位 syscall 短路。
  - 这解释了症状：`hello_asm`(纯 write/exit) 能过；app_process32/真实程序踩这条没接的边界会崩。
  - 注意与指令级 `b00b399c`/Thumb 位 SIGSEGV(解码/发射层)**不是同一层**，那是另一条轨。

---

# 接线清单

**前提(dev 模式)**：带符号编译、不上加固、`/dev/tango32` 在位、SELinux permissive、`--no-pretrans` 纯 JIT。
**代码位**：`android10/vendor/hello/arm96/src/tango_translator_reimpl/`

## A. SET_MM —— 把 guest 内存布局告知内核（最高优先）
现状：从不调用 → `/proc/self/{stat,auxv}`、`brk()` 反映宿主布局，bionic/linker/ART 会错乱。

- [ ] **A1 填 `Tango32Mm`**：在 `translator_context.cpp` 建完 guest 栈/argv/env/auxv 之后（≈302–378 行 `AuxvBuilder` + `sp/env_start` 段），用 guest 值填：
  - `start_code/end_code/start_data/end_data` ← `elf_loader` 段边界
  - `start_brk/brk` ← `vm_manager` brk 区间
  - `start_stack` ← guest 栈顶；`arg_start/arg_end/env_start/env_end` ← 写栈时算出的地址
  - `auxv` ← guest auxv 块地址（host 可读），`auxv_size` ← 字节数
- [ ] **A2 调用**：栈布好、跳 guest entry 之前 `device_->setMm(&mm)`；检查返回（内核 `validate_mm_fields` 失败要能定位）。
- [ ] **A3 `setMmapBase()`**：设成 `vm_manager` guest mmap 区基址，保证无 hint 的 mmap 落 32 位空间。
- **验证**：跑读 `/proc/self/auxv`、`/proc/self/stat` 或调 `brk` 的程序与原版差分；或看 `/proc/<pid>/stat` 的 code/brk/stack 是否为 guest 地址。

## B. compat 敏感 syscall 改走驱动（替掉原生短路）
现状：`syscall_handler.cpp` 用原生 64 位 syscall 顶着，32 位 ABI 语义不符。

- [ ] **B1 `sys_ioctl`**(:187)：设备类 ioctl 改走 `device_->compatIoctl(fd,cmd,guest_arg)`；仅驱动明确不管的"通用 fd ioctl"(见 tango32.h 注释)保留原生。大概率可去掉用户态 `translateIoctlCmd()` hack。
- [ ] **B2 `sys_getdents64`**(:368)：改走 `device_->compatGetdents64(fd,guest_dirp,count,&result)`——ext4 目录 d_off 压 32 位，原生会给出不落 32 位的偏移，崩 readdir。
- [ ] **B3 `sys_lseek`**(:163)：目录 fd 走 `device_->compatLseek(...)`（ext4 目录偏移 32 位语义）；普通文件 lseek 可留原生。
- [ ] **B4 `sys_set_robust_list`**(:214) 改 `device_->compatSetRobustList(head,len)`；`get_robust_list` 接 `compatGetRobustList`。
- **验证**：分别用"ext4 目录 readdir"、"ioctl 密集路径"、"pthread(robust_list)"程序与原版差分。

## C. 指针/线程一致性（接线坑）
- [ ] 驱动作用于**调用线程的 mm/fd**——确保这些 ioctl 由**运行 guest 的同一进程/线程**发出（当前单进程满足；多线程 guest 要留意）。
- [ ] 传给驱动的 guest 指针（getdents 的 dirp、SET_MM 的 auxv）统一用 `vm_->guestToHost()` 转 host 可读地址，别把 guest 虚地址直接塞进去。

## D. 差分 oracle 验证台（贯穿 A/B）
- [ ] **D1** 搭差分：同一 ARM32 程序 → 原版 `tango_translator` vs 2.0，都加 `--no-pretrans`，比 stdout/exit/strace。
- [ ] **D2** 敲定原版**日志/trace 语法**（log 源 translate/elf/cfg + 级别 off/warn/…/trace），看它把哪些 syscall 路由到 `/dev/tango32`——作为 2.0 该补调用的权威参照。
- [ ] **D3** 测试阶梯：`hello_asm`(已过) → 静态 C hello(libc/brk/auxv) → 动态 hello(linker/mmap) → readdir(getdents/lseek) → pthread(robust_list) → **app_process32**(终关)。

## E. 顺序与边界外提醒
- [ ] 顺序：**A(SET_MM)先** → **B(compat syscall)** → 再攻 app_process32。
- ⚠️ 指令级 JIT 缺口(`b00b399c`/Thumb 位)是**另一条独立轨**（解码/发射层），不在本清单；app_process32 需两条轨都到位。
- 🚫 别做：别加加固、别改内核驱动(v2.0 完整现成)、别在用户态重写 compat 逻辑(交给驱动)。

---
**一句话**：接口和内核 ABI 都齐，缺的是 **A(SET_MM 接进启动)+ B(compat syscall 接进分发)**，拿原版 tango 当 oracle 逐项差分验证即可闭环。

## 现场遗留（本次会话）
- 容器 `192.168.2.19:5004` `/data/local/tmp/` 下留了 `tango_translator` + `hello_asm`，可直接接着试差分。
- 相关备忘：`tc008-guardian-cascade-unreliable`（1.0 的 TC-008 缺陷，与本清单无关）。
