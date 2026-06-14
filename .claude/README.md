# Claude Code Workspace Docs

本目录承载 Claude Code 会话所需的项目指令、命令模板与背景上下文。

## 目录说明

- `instructions/`：会话入口与工作流程规则。
- `commands/`：可直接复用的提示词模板。
- `context/`：专题背景文档与深度分析资料。

## 推荐读取顺序

1. `instructions/project_entry.md`
2. `instructions/build_and_debug.md`
3. 按需读取 `commands/*.md`
4. 按需读取 `context/*.md`

## 迁移说明

本目录内容由 `.github` 下历史说明迁移而来；旧路径暂时保留用于兼容，后续以 `.claude` 为主入口。