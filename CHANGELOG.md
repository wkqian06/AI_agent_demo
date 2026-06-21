# Change Log

本文件用于追踪 `AI_agent_demo` 仓库的重要变更。

## 2026-06-20

### Added

- 新增并同步 Chapter 3 相关内容，包括 `demos/chapter3.ipynb` 和 `notes/Day03.md`。
- 新增文件工具/LLM 调用产生的示例文件：`demos/llm诗词.txt` 和 `demos/test.txt`。

### Changed

- 优化 `notes/Day03.md` 的 Markdown 层级与正文结构，按状态管理、Tool/Agent 和工作流总结重新组织内容。
- 更新 `README.md`，同步 Day03 学习进度、Chapter 3 demo 和新增示例文件。
- 保留并同步 `demos/chapter3.ipynb` 的最新运行输出。

## 2026-06-18

### Added

- 新增 `notes/Day02.md`，整理 Day02 关于多轮对话、PromptTemplate、Few-shot、输出解析和 `BaseOutputParser` 的学习笔记。
- 新增 `demos/chapter2.ipynb`，覆盖 Chapter 2 中模型调用、Prompt 模板、Few-shot 示例、输出解析和本地 Hugging Face/Qwen 调用尝试。
- 新增 `demos/learning_method_examples.json`，用于工程化 Few-shot 示例选择。

### Changed

- 更新 `pyproject.toml` 和 `uv.lock`，加入 `langchain-huggingface`、`transformers`、`torch`、`torchvision` 等本地模型调用相关依赖。
- 为 `torch` 和 `torchvision` 配置 `pytorch-cu130` 安装源。
- 更新 `README.md`，同步当前文件结构、学习进度、运行方式、依赖说明和配置约定。

### Fixed

- 恢复 `pyproject.toml` 的 `[project]` 表头，保证项目配置可以被标准工具正确解析。

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
