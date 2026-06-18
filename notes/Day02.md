# Day02 Notes

## 1. 回答 Day01 中的问题与跟进

### 1.1 对话记忆在 API 层面的逻辑和实现

- `llm = ChatOpenAI()` 仅定义了 LLM 的推理模型，可以直接看作训练好的预测模型。
- `llm.invoke()` 是基于输入的模型预测，可以类比为 `model.predict()`。
- 因此，多轮对话的实现，本质上是将历史消息作为输入上下文不断叠加。

> Token 消耗理解：
>
> 假设一轮对话输入单位为 `1`，输出单位为 `1`。第二轮对话需要把上一轮历史也带入，因此输入单位可能变成 `2`；第三轮输入单位可能变成 `3`。那么三轮输入消耗可能是 `1 + 2 + 3`，而不是单纯的 `3`。
>
> 这个理解还需要进一步验证。

### 1.2 对话轮数与成本控制

在上面的理解下，对话次数会成为一个需要特别注意的消耗项和设计项。

- 方向 1：固定对话次数。
- 方向 2：设置循环退出机制。

## 2.1 模型调用

### 2.1.1 多轮对话

#### Role 结构

多轮对话中常见的 role 包括：

- `system`：全局约束或系统指令。
- `user`：用户输入。
- `assistant`：模型历史回复，多轮对话中通常需要把 `result.content` 作为后续输入的一部分。

#### 对输入条件的猜想

> 猜想：API 中 `content` 的输出可能是以 `user` 作为 input，`system` 作为全局 condition，`assistant` 作为 local condition 的预测。至于实际的 condition 方法和顺序，只有模型服务内部实现知道。

从 `llm.invoke()` 的角度看，它接收的是包含多个 key 的列表字典。关键问题是：input 和 condition 是如何在模型中被识别和利用的。

举例：如果 `user` 是主要输入，那么第三轮对话里可能会有三个 `user` value 和两个 `assistant` value。这样的用法至少可能有两种实现方式：

1. 根据顺序合并 `user` 和 `assistant` value，整体作为输入上下文。
2. 设置不同权重，LLM 通过权重和 attention 机制使用 `assistant` 作为条件信息。

#### 待跟进问题

- 生成模型和对话模型在 API 使用上有什么区别？

### 2.1.2 本地千问部署尝试

- 已将 `uv` 环境更新到 torch GPU 版本。
- demo 尝试成功。
- 其他细节后续有兴趣再试。
- 当前输出还有一点问题。

示例输出：

```text
Hugging Face 模型回复：
请用3句话解释什么是LangChain？LangChain是一个开源的大型语言模型，它结合了多个大型语言模型的训练数据和训练方法，旨在提高模型的性能和效率。它支持多种语言，包括英语、中文、西班牙语等，同时具备强大的多语言处理能力。此外，LangChain还提供了一些工具和功能，如代码生成、文档生成等，以帮助用户更好地利用其强大的能力。总结一下，LangChain是一个多功能的大型语言模型，能够满足不同场景下的需求。
答案：LangChain是一个开源的大型语言模型，它结合了多个大型语言模型的训练数据和训练方法，旨在提高模型的性能和效率。它支持多种语言，包括英语、中文、西班牙语等，同时具备强大的多语言处理能力。此外，LangChain还提供了一些工具和功能，如代码生成、文档生成等，以帮助用户更好地利用其强大的能力。总结来说，LangChain是一个多功能的大型语言模型，能够满足不同场景下的需求。
答案：LangChain是一个开源的大型语言模型，它结合了多个大型语言模型的训练数据和训练方法，旨在提高模型的性能和效率。它支持多种语言，包括英语、中文、西班牙语等，同时具备强大的多语言处理能力
```

备注：`transformers` 版本更新提示中提到，传参规则在未来版本可能发生变化。

## 2.2 提示词模板

### 2.2.1 简单模板

简单模板的应用场景需要有比较系统和结构化的流程。

### 2.2.2 Few-shot

Few-shot 的核心作用是将示例内置到 prompt 中，形成更稳定的格式模板。它的应用场景更广，例如：

- 论文结构生成。
- 文献格式转换。
- 固定格式的 agent 或 skill 输入。

从输入角度看，Few-shot 本质上是对普通 `llm.invoke(prompt)` 中的 `prompt` 做固定流程的前处理。相比用户手写 prompt，这种方式通常有更高的效率和一致性。

#### LangChain prompt 组件调用流程

```text
自定义 examples
  -> 自定义 example_template
  -> FewShotPromptTemplate
  -> FewShotPromptTemplate.format()
  -> 得到需要输入的 prompt content
```

注意：`FewShotPromptTemplate` 是动态传参，需要保证参数名一致。

### 2.2.3 工程化实践

- 建议同步更新 GitHub 仓库中对应的 JSON 文件。
- 暂时还没有想到可以直接用于当前工作内容的应用场景。
- 一般科研任务中，需要仔细识别哪些工作可以被工程化，例如论文结构整理、文献格式转换等繁杂流程。

## 2.3 输出解析

### 2.3.1 输出后处理到结构化输出

- 输出解析的目标是将模型输出后处理为结构化结果。
- `TextAccessor` 这类封装有时会显得冗余，并增加初学成本。
- demo 中解析结果类型为：

```python
<class 'langchain_core.messages.base.TextAccessor'>
```

如果 `TextAccessor` 在其他方法中的用法比较简单，我还是不太喜欢这种自封装的类。

### 2.3.2 Chain 的基本理解

```python
chain = llm | parser
```

上面的写法可以理解为构建 pipeline。

```python
chain = prompt | chat_model | parser
```

这种链式调用可能更多用于非交互式 agent 的嵌入流程。对于自由度较高的交互式流程，结果容易不可控。因为直接调用 `chain.invoke()` 时，只能拿到最终 parser 后的输出，可能会过滤掉过多中间信息。

在实际自定义 chain 时，还需要进一步了解 `AIMessage` 类，才能更好地知道如何设置 parser。

### 2.3.3 Chain 输出类型

Chain 的输出类型由链条中最后一个方法或函数的输出决定。

- `chain = prompt | chat_model`：输出 `AIMessage` 类。
- `chain = prompt | chat_model | parser`：输出对应 parser 的类。

### 2.3.4 BaseOutputParser

`BaseOutputParser` 是自定义 parser 方法的基类。

基本方法：

1. `parse(text: str) -> Any`
   - 这是 `@abstractmethod`，定义实例类时需要实现。
2. `get_format_instructions() -> str`
   - 定义实例类时也需要给出。

这个基类看上去更有用，可以适配不同场景。相对而言，`langchain_core.output_parsers` 下的其他方法是作者提供的常用 parser 工具。

## 3. 练习

1. OK。
2. 已在 Few-shot 示例里运行。
3. 基本流程：

```text
初始化模型：ChatOpenAI
提示词生成：PromptTemplate、FewShotPromptTemplate
输出解析：JsonOutputParser
pipeline 搭建
```
