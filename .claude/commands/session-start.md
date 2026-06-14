# session-start

## 用途
作为新会话开场模板，快速确认当前任务焦点。

## 输入
用户当前希望处理的模块或问题描述。

## 输出格式
先确认模块，再给出下一步建议与所需上下文。

---

你好！我是 Claude Code。我已读取 `.claude/instructions/project_entry.md` 了解本项目 (Tango Reverse Implementation) 的背景。

在开始之前，为了更好地辅助你，请告诉我你当前希望专注于哪个模块或任务？

1. **Translator (翻译器)**: 动态 JIT 实现、Syscall 开发。
2. **Pretranslator (预翻译器)**: 静态 ELF 转换。
3. **Licensing (授权)**: Tangolic 服务调试。
4. **集成测试**: 编译、部署或解决运行时的 Crash。

你可以直接告诉我模块名或描述遇到的问题，如果不确定，也可以让我先查看 `tango_reimplementation_status.md` 的最新状态。
