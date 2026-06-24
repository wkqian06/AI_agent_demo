# Day04 Notes

## 本章主题

Day04 主要围绕 LangChain 应用级系统设计展开，重点包括：

- LCEL 中 `|` 管道串联的基本用法。
- 多输入、多输出链路中的 `RunnableLambda` 与 `RunnablePassthrough`。
- `RunnableBranch` 的条件分支。
- 重试机制、异常捕获和 fallback。
- RAG 中常见文档加载器的选择。

## 4.1 LCEL 与 Runnable 串联

### 4.1.1 `|` 管道串联：简单线性链

问题：`extract_sell_points = RunnableLambda(...)` 的功能是什么？

理解：

- `RunnableLambda` 用于把普通 Python 函数包装成 LangChain 可串联的 Runnable 组件。
- 当一个自定义函数需要接入 LCEL 管道时，通常需要用 `RunnableLambda` 包一层。
- 它的核心作用不是改变函数逻辑，而是让函数具备 Runnable 接口，从而可以参与 `|` 串联。

示意：

```python
extract_sell_points = RunnableLambda(lambda x: x.content)

chain = prompt | llm | extract_sell_points
```

### 4.1.2 多输入、多输出

链路中的：

```python
lambda x: x.content
```

和简单线性链中的：

```python
extract_sell_points = RunnableLambda(...)
```

在功能上是等价的，都是把上一个组件的输出取出并转换为下一个组件需要的输入结构。

### 4.1.3 `RunnablePassthrough` 使用问题

当前 `overall_chain` 中的 `RunnablePassthrough()` 使用有误。

如果按教程中的方式使用，可能会把整个输入对象：

```text
{
  "product_intro": "这款无线耳机采用蓝牙5.3芯片，连接稳定无延迟...",
  "target_audience": "大学生群体（喜欢运动、预算有限、注重性价比）"
}
```

整体作为 `target_audience` 传入。

问题原因：

- `RunnablePassthrough` 会原样接收上一个组件的完整输出。
- 当它和 map 结构共用时，如果没有显式取字段，就会把完整 input 直接传给目标字段。
- 详见 `chapter4.ipynb` 中相关 cell。

更稳妥的写法应该显式取字段：

```python
"target_audience": lambda x: x["target_audience"]
```

## 4.2 `RunnableBranch`

`RunnableBranch` 用于根据条件把输入分发到不同链路。

### 4.2.1 `StrOutputParser`

在案例中，`StrOutputParser` 主要用于把模型输出解析为字符串。

可以简单理解为：

```text
LLM output object -> str
```

## 4.3 错误处理、重试与 Fallback

### 4.3.1 `Runnable` 类型标注

关于 `Runnable` 类的 import：

```python
base_chain: Runnable = summary_prompt | llm | StrOutputParser()
```

和下面写法在运行逻辑上等价：

```python
base_chain = summary_prompt | llm | StrOutputParser()
```

理解：

- `Runnable` 这里只是类型标注。
- 在 `|` 管道组合过程中，LangChain 已经完成了 Runnable 结构的构建。
- 前面的 chain 示例也没有强制加这个类型标注，因此这里不一定需要特别强调。

### 4.3.2 异常捕获

异常捕获案例本身和 LangChain 架构关系不大，更接近 Python 基础。

这里主要用于区分：

- 重试机制：失败后再尝试同一条链。
- 异常退出机制：捕获错误并终止或返回错误信息。
- fallback 机制：核心链失败后切换到备用链。

### 4.3.3 `with_fallbacks`

示例：

```python
chain_with_fallback: RunnableWithFallbacks = core_chain.with_fallbacks(
    fallbacks=[fallback_chain],
    # exceptions_to_handle=(ConnectionError, TimeoutError),
    # 可以捕获连接异常和超时异常。
    # 这里为了示范捕获所有异常；错误 API key 返回的通常是认证异常。
)
```

理解：

- `RunnableWithFallbacks` 也是类型标注，不一定需要显式强调。
- `fallbacks=[fallback_chain]` 表示核心链失败时转向备用链。
- 需要注意异常类型，认证错误通常不适合自动 fallback，因为它往往是配置问题。

## 4.4 RAG

### 4.4.1 文档加载器记录

RAG 中，文档加载器负责把不同格式的文件转成 LangChain 的 `Document` 对象。

| 文档格式 | 官方推荐加载器 | 需要安装的依赖包 |
| --- | --- | --- |
| TXT | `TextLoader` | 无需额外安装，`langchain-community` 内置 |
| PDF | `PyPDFLoader` / `PDFPlumberLoader` | 基础款：`pypdf`；复杂款：`pdfplumber` |
| Word `.docx` | `Docx2txtLoader` | `docx2txt` |
| Markdown `.md` | `MarkdownLoader` / `UnstructuredFileLoader` | 轻量款：`python-markdown`；通用款：`unstructured` |

### 4.4.2 待跟进问题

- `Docx2txtLoader` 无法提取文档中的图片。如果需要图片信息，可能需要 OCR。
- 还需要进一步确认：`Docx2txtLoader` 对表格的提取效果如何？
- Markdown 的通用加载器依赖较重，需要比较轻量方案和通用方案的实际效果。

## 4.5 当前理解

Day04 的重点不只是 API 用法，而是理解应用级链路中不同组件的职责：

```text
输入
  -> prompt
  -> llm
  -> parser
  -> branch / retry / fallback
  -> retrieval / document loader
  -> final output
```

后续需要重点关注：

- `RunnablePassthrough` 在复杂输入结构中的字段传递方式。
- fallback 和 retry 的适用边界。
- RAG 文档加载、切分、向量化和检索之间的完整数据流。
