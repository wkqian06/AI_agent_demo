# Change Log

本文件用于追踪 `AI_agent_demo` 仓库的重要变更。

## 2026-06-14

### Added

- 新增 `CHANGELOG.md`，用于持续记录仓库的学习笔记、demo、依赖和配置变更。
- 初始化 AI Agent 学习记录结构，包括 `demos/`、`notes/`、`pyproject.toml` 和 `uv.lock`。
- 新增 LangChain/OpenAI API 测试 notebook 与 Chapter 1 demo notebook。

### Changed

- 优化 `notes/Day01.md` 中 `langchain_openai` 示例的 Markdown 代码块格式。
- 扩展 `README.md`，说明仓库定位、目录约定、运行方式和密钥提交注意事项。

### Security

- notebook 示例从 `.env` 读取 API 配置，避免在仓库中提交真实密钥。
