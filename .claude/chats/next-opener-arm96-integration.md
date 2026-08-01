# 新会话开场白 —— arm96 集成（vmp 专项之后的 step2/3）

> 生成于 2026-06-19。vmp 专项已完成 step0(MRS 静默读 0 修复)+ step1(128-bit q 指令覆盖)，
> 提交在 vmp 仓库 main/1.0，tag `vmp-mrs-fix-qinsn-2026-06-19`。
> 下面这段直接复制粘贴到新会话即可接续。

---

接续 arm96 加固开发。vmp 专项刚收尾——step0(MRS 静默读 0 修复)+ step1（128-bit q 指令覆盖）
已完成并提交在 vmp 仓库 main/1.0，tag `vmp-mrs-fix-qinsn-2026-06-19`。现在做 arm96 集成。

请先读记忆 `vmp-completeness-plan`（重点看「2026-06-19 第0步/第1步 完成」两段）、
`v069-vmp-regression`、`ollvm-silently-off`，再开工。

环境：本机 192.168.2.x = 环境2，编译机 `ssh -o BatchMode=yes mc@192.168.2.18`，仓库
`/home/mc/2T/cx10`。重建 vmpacker：`PATH=/home/mc/sdk/go1.21.13/bin:$PATH; cd ~/2T/cx10/vmp;
touch stub/linux/arm64/vm_interp.c; make stub && make packer`（改 .h 头必须先 touch vm_interp.c，
否则 Makefile 不重建 blob）。

任务顺序：
1) step2 集成回归：把新 `vmp/build/vmpacker` 拷到 `android10/vendor/hello/arm96/res/bin/vmpacker`，
   把 `hardening/src/version.txt` 的 `VMP_CORE_TOOL_SHA256` 改成该二进制的**实际 sha256**；
   构建六件套，对现有 26 个 VMP_LOADER target 做**冷启动** verify_mustpass 回归。
   （MRS 修复使 platform_check_isa_features/cpu_part 在 VM 内从「读 0」变「读真值」，要确认不破坏冷启动。）
2) step3 crown jewel：MRS 已修、reconstruct_key 在 optnone 下无 NEON，原本卡死它的唯一原因
   （VM 读 ISAR0 得 0→错密钥）已消除——尝试把 `reconstruct_key`（以及 decrypt_core_to_memfd）
   重新加进 VMP_LOADER_TARGETS，冷启动 verify_mustpass 验证 crown jewel 现在能正确 VM 化。

必守 / 别重踩：
- 冷启动才是门槛，不能只测热重启（redroid `/proc/uptime` 是宿主内核 uptime，判 Android 冷启动
  要用 init pid1 etime 或 sys.boot_completed）。
- **IPRA 坑**：启用 VM target 后若出现「VM 算对了却让调用方崩、且对二进制布局敏感」的怪象，
  先查 `vmp-completeness-plan` 的 IPRA 段（loader 用 clang 一般不受影响，但要知道）。
- 便宜测试回路：部署后先不冷启动，手动跑 `app_process32` 触发 loader→reconstruct_key→解密，
  错只死该进程、容器不倒、adb 重推即恢复；最后再做冷启动 verify_mustpass。
- 不改 arm96 业务逻辑（加 noinline、调 VMP_LOADER_TARGETS、换 vmpacker、改版本号/sha 属构建配置，不算业务逻辑）。

请用全程中文沟通。
