# AI Agent Demo

这是我的 AI Agent 学习记录仓库，用来保存学习过程中的 demo、实验代码和笔记。

## 仓库定位

- 记录 AI Agent 学习过程中的demo。
- 保存对 `easy-langent` 学习材料的复现、改写和扩展。
- 沉淀自己的笔记、踩坑记录和运行说明。

## 建议目录

```text
AI_agent_demo/
  README.md      # 仓库说明
  notes/         # 学习笔记
  demos/         # 可运行 demo
```

后续可以按章节或主题继续细分，例如：

```text
demos/
  01_langchain_basic/
  02_tool_calling/
  03_langgraph_agent/
```

## 与 easy-langent 的关系

`easy-langent` 是学习材料来源，本仓库是个人学习记录。

如果某个 demo 需要严格依赖 `easy-langent` 中的特定源码，可以在对应 demo 的说明里记录：

- 参考章节或文件路径；
- 对应的 commit hash；
- 本 demo 相比原材料做了哪些修改。

只有在 demo 必须随仓库一起固定学习材料源码时，再考虑引入 submodule。

## 运行约定

本仓库使用 Python `>=3.11`，依赖信息以 `pyproject.toml` 和 `uv.lock` 为准。

常用命令：

```powershell
uv sync
uv run python path\to\demo.py
```

不要提交 API key、`.env`、虚拟环境目录或运行缓存。
