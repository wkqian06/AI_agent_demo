# AI Agent Demo

这是我的 AI Agent 学习记录仓库，用来保存学习过程中的 demo、实验代码和笔记。

## 仓库定位

- 记录 AI Agent 学习过程中的 demo。
- 保存对 `easy-langent` 学习材料的复现、改写和扩展。
- 沉淀自己的笔记、踩坑记录、运行说明和变更记录。

## 当前内容

```text
AI_agent_demo/
  README.md                         # 仓库说明
  CHANGELOG.md                      # 变更追踪
  LICENSE                           # 许可证
  pyproject.toml                    # Python 项目与依赖配置
  uv.lock                           # uv 锁定文件
  notes/
    Day01.md                        # Day01 学习笔记
    Day02.md                        # Day02 学习笔记
    Day03.md                        # Day03 学习笔记
    Day04.md                        # Day04 学习笔记
  demos/
    api_test.ipynb                  # API 配置与调用测试
    chapter1_demo.ipynb             # Chapter 1 demo
    chapter2.ipynb                  # Chapter 2 demo
    chapter3.ipynb                  # Chapter 3 demo
    chapter4.ipynb                  # Chapter 4 demo
    learning_method_examples.json   # Few-shot 学习方法示例数据
    llm诗词.txt                     # 文件工具/LLM 生成示例
    test.txt                        # 文件工具生成的空文件示例
    knowledge_base/                 # RAG 文档加载示例文件
```

## 学习进度

- Day01：`langchain_openai` 基础调用、`langgraph.graph` 状态流转示例。
- Day02：多轮对话 role 结构、PromptTemplate、FewShotPromptTemplate、输出解析、BaseOutputParser、本地 Hugging Face/Qwen 模型调用尝试。
- Day03：对话记忆、`RunnableWithMessageHistory`、窗口记忆、摘要记忆、Tool 调用、Agent 执行循环和文件工具实践。
- Day04：LCEL/Runnable 串联、`RunnableBranch`、错误处理、fallback 和 RAG 文档加载器实践。

## 运行约定

本仓库使用 Python `>=3.11`，依赖信息以 `pyproject.toml` 和 `uv.lock` 为准。

常用命令：

```powershell
uv sync
uv run python path\to\demo.py
```

Notebook 建议通过 Jupyter 或 VS Code 运行：

```powershell
uv run jupyter lab
```

## 依赖说明

当前依赖包括：

- LangChain 相关：`langchain`、`langchain-openai`、`langchain-community`、`langchain-huggingface`、`langgraph`。
- 模型与本地推理：`openai`、`transformers`、`torch`、`torchvision`。
- 文档处理与辅助工具：`docx2txt`、`pdfplumber`、`pypdf`、`python-dotenv`、`python-markdown`。

`torch` 和 `torchvision` 当前配置为从 `pytorch-cu130` 索引安装，适用于 CUDA 13.0 相关环境。没有 GPU 或 CUDA 环境不匹配时，需要按本机环境调整 `pyproject.toml` 中的 PyTorch 源。

## 配置约定

API key 和模型服务地址请放在本地 `.env` 文件中，例如：

```text
API_KEY=your_api_key
BASE_URL=https://api.example.com
```

不要提交 API key、`.env`、虚拟环境目录或运行缓存。

## 与 easy-langent 的关系

`easy-langent` 是学习材料来源，本仓库是个人学习记录。

如果某个 demo 需要严格依赖 `easy-langent` 中的特定源码，可以在对应 demo 的说明里记录：

- 参考章节或文件路径；
- 对应的 commit hash；
- 本 demo 相比原材料做了哪些修改。

只有在 demo 必须随仓库一起固定学习材料源码时，再考虑引入 submodule。
