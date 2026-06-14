# Tango (ARM32 转 ARM64 翻译器) 项目文档索引

> **最后更新**: 2026-03-17
> **项目类型**: 逆向工程与二进制翻译系统
> **核心目标**: 重构 Tango 翻译器以支持 ARM32 应用在 ARM64-only 的 Android 10 环境中运行。

---

## 🎯 会话启动流程

### 第一步：确认活动组件
在开始之前，请确认当前任务涉及哪个组件：

1. **Pretranslator** (静态预翻译) - `src/tango_pretranslator_reimpl`
2. **Translator** (动态/JIT 翻译) - `src/tango_translator_reimpl`（源码目录；当前主模块名：`arm96_translator_reimpl`，兼容目标：`tango_translator_reimpl`）
3. **Licensing** (授权服务) - `src/tangolic_reimpl`
4. **Panic 补丁** (授权绕过) - `android10/vendor/vclusters/opensource/thirdpart/tango/patchs`
5. **Binary Hardening** (加固防护) - `.claude/context/tango_hardening.md`
6. **构建系统** - `Android.mk`, `build.sh`, `make` 集成
7. **集成与兼容性** - 部署, 系统调用 (Syscalls), 兼容性测试

### 第二步：加载组件文档
阅读涉及组件的详细说明文档：

- 📄 **核心翻译器与 JIT**: [.claude/context/translator.md](../context/translator.md) (Syscalls, JIT, 解码)
- 📄 **构建与环境**: [.claude/instructions/build_and_debug.md](./build_and_debug.md) (编译, 部署)
- 📄 **项目状态**: [.github/tango_reimplementation_status.md](../../.github/tango_reimplementation_status.md) (里程碑, 进度)
- 📄 **Panic 补丁指南**: [android10/vendor/vclusters/opensource/thirdpart/tango/patchs/panic_patch_guide.md](android10/vendor/vclusters/opensource/thirdpart/tango/patchs/panic_patch_guide.md)

### 第三步：确认分支边界
- `android10/vendor/hello/arm96/src/` 是 C++ 主线源码区。
- `main/2.0` 负责 `src/` 下主线源码持续迭代，尤其是 `src/tango_translator_reimpl/`。
- `main/1.0` 当前只做交付链、hardening、验收链和公共文档维护；为了避免分支漂移，**暂不修改** `src/tango_translator_reimpl/` 目录本体。
- 当前命名迁移只在公共文档和构建兼容层记录：源码目录仍为 `src/tango_translator_reimpl/`，源码模块主名优先记为 `arm96_translator_reimpl`。

---

## ⚠️ 关键开发准则

1. **Syscall 一致性**: 添加 Syscall 时，确保同时更新 `syscalls.h` (结构体定义) 和 `syscall_handler.cpp` (逻辑实现)。
2. **ARM 兼容性**: 始终对照 ARMv7-A/ARMv8-A 手册验证指令编码。
3. **构建安全**: 使用统一的 `make <模块名>` 方式。避免手动使用 `clang++` 命令，除非是为了测试隔离的代码片段。
4. **授权依赖**: Tangolic 服务必须正在运行，翻译器才能正常工作。
5. **补丁可复现**: 二进制补丁记录 offset 和 MD5，确保可回滚。
6. **会话管理约定**:
   - 每解决完一个具体问题，必须评估是否需要重启会话。
   - 如果需要重启，请直接给出推荐的 **新会话开场白 (Prompt)**。
   - 始终使用中文沟通。
   - 任何新解决的问题，必须在测试设备上完成验证 (`adb push` + 运行)。
7. **分支纪律**: 若当前工作在 `main/1.0`，不要对 `src/tango_translator_reimpl/` 做物理目录改名、源码搬迁或大规模源码内引用改写；这类动作留给 `main/2.0`。

---

## 📁 项目结构速览

```text
android10/vendor/hello/arm96/
├── src/
│   ├── tango_pretranslator_reimpl/  # 静态二进制预翻译器 (ELF -> ELF)
│   ├── tango_translator_reimpl/     # 动态二进制翻译器源码目录（当前主模块名：arm96_translator_reimpl）
│   │   ├── syscall_handler.cpp      # 系统调用模拟逻辑
│   │   ├── translator.cpp           # 主翻译循环
│   │   └── ...
│   └── tangolic_reimpl/             # 许可证验证服务
├── hardening/                       # loader/core 加固与构建
│   ├── src/                         # loader.c, arm96server.c, main.py, encrypt_core.py
│   └── ...
├── build/                           # 构建产物
├── init/                            # init service 脚本
├── sepolicy/                        # SELinux 策略
└── prebuilts/                       # 原始二进制文件及构建产物
```

---

## 🚀 快速开始
1. 参考 [构建与环境](./build_and_debug.md) 设置开发环境。
2. 查看 [项目状态](.github/tango_reimplementation_status.md) 了解当前进度。
3. 如需补丁，参考 [Panic 补丁指南](android10/vendor/vclusters/opensource/thirdpart/tango/patchs/panic_patch_guide.md)。

## 贡献
请更新相应文档并在工作日志中记录进展。