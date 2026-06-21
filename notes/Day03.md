# Day03 Notes

## 本章主题

Day03 主要围绕 LangChain 的进阶组件展开，包括：

- 状态管理与对话记忆。
- 外部工具调用与 Agent 执行循环。
- 从数据流和工作流角度理解前三天的学习内容。

## 3.1 状态管理

对话记忆的实现方式与 Day02 中关于多轮对话上下文叠加的思考相似。本章重点学习三种简单记忆策略：

| 记忆方式 | 核心思路 | 适用场景 |
| --- | --- | --- |
| 全量记忆 | 保存全部历史消息 | 短对话、需要完整上下文的任务 |
| 窗口记忆 | 只保留最近 `N` 轮对话 | 控制 token 成本、近期上下文更重要的任务 |
| 摘要记忆 | 将长历史压缩为摘要后继续对话 | 长对话、需要继承前文但不能无限堆叠 token 的任务 |

### 3.1.1 全量记忆

适用场景：短对话。因为该方法会把全部对话历史都作为记忆内容。

#### Prompt 结构

```python
full_memory_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是友好的对话助手，需基于完整的历史对话回答用户问题。"),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{user_input}")
])
```

这个列表结构与 Day02 中直接构造 `messages` 的逻辑相似，差异在于：

1. 本例结构为 `List[tuple]`，其中元组第一个元素相当于原来字典里的 role，第二个元素相当于 `content`。
2. 原来一轮对话可以存在一个 list 元素里；这里变成按 role 顺序堆叠消息。
3. `MessagesPlaceholder(variable_name="chat_history")` 是历史消息占位符，可以理解为运行时注入的上下文。

#### 完整用法

```python
base_chain = full_memory_prompt | llm

def get_full_memory_history(session_id: str) -> BaseChatMessageHistory:
    """根据 session_id 获取会话历史，不存在则创建新的历史记录。"""
    if session_id not in full_memory_store:
        full_memory_store[session_id] = InMemoryChatMessageHistory()
    return full_memory_store[session_id]

full_memory_chain = RunnableWithMessageHistory(
    runnable=base_chain,
    get_session_history=get_full_memory_history,
    input_messages_key="user_input",
    history_messages_key="chat_history"
)

config = {"configurable": {"session_id": "user_001"}}

response1 = full_memory_chain.invoke(
    {"user_input": "我叫小明，喜欢编程"},
    config=config
)
print("助手回复1：", response1.content)

for msg in get_full_memory_history("user_001").messages:
    print(f"{msg.type}: {msg.content}")
```

#### 工作流理解

```text
pipeline
  -> RunnableWithMessageHistory(
       runnable=pipeline,
       get_session_history=memory_store_function
     )
  -> memory_store[session_id]
  -> InMemoryChatMessageHistory
```

关键理解：

- `memory_store` 是字典，value 是 `InMemoryChatMessageHistory` 实例。
- `RunnableWithMessageHistory` 封装了 pipeline 和历史消息管理逻辑。
- 这个封装继承了 pipeline 的主要调用方式，因此可以直接使用 `.invoke()`。
- 它在 pipeline 之外增加了记忆存储和历史注入逻辑。
- `memory_store_function` 使用全局变量，个人不太习惯，但对固定工作流比较方便。
- 如果需要额外参数，`memory_store_function` 的传参结构可能会变复杂。
- `RunnableWithMessageHistory.invoke()` 返回的是 `AIMessage` 类。

### 3.1.2 窗口记忆

适用场景：只保留最近 `N` 轮对话，`N` 通常由 `k` 参数控制。

窗口记忆的整体逻辑和 API 用法与全量记忆一致，区别主要在 `get_session_history` 的定义中增加了窗口截断逻辑。

#### 幻觉现象记录

以下结果来自对 `inputs` 的第三次改动和运行，也就是同一段 step5 逻辑根据不同输入运行了三次。

1. 第一次：使用原始 input，得到类似示例的回答。
2. 第二次：在询问城市后继续询问名字，二者在同一轮中，测试是否真的遗忘。结果是没有报出名字，回复也较简练。
3. 第三次：在询问城市后继续询问名字，二者在不同轮中。如下所示，出现了幻觉，回答也更复杂。

> 问题：这三次理论上应该都是全新对话。比如跑第二次时，并没有注入和覆盖第一轮信息。它们的差异是否和 `window_memory_store["user_002"]` 没有重置有关？虽然模型概率预测是主要因素，但如果 `window_memory_store` 没有重置，其 footprint 和 ID 信息可能是三次共享的。

示例结果：

```text
第1轮 - 助手回复： 小红你好呀！这个名字真可爱，像课本里会偷偷给同桌塞薄荷糖的女生～
下次想画校园故事的话，我可以帮你设计一个扎着蝴蝶结、总在窗边看云的主角哦！
（需要帮你把名字藏进漫画对话框里吗？比如“小红发现数学书里夹着一片银杏叶”这样的分镜～）

第2轮 - 助手回复： 哇，小红是位小画家呀！🎨 画笔就是你的魔法棒对吧？
下次我们可以一起设计一个“会跳舞的调色盘”角色——红色是转圈圈的草莓裙，蓝色是踮脚尖的浪花，黄色是打哈欠的向日葵…
（需要帮你把铅笔稿变成会动的故事吗？比如“小红画的小猫突然从纸上跳出来，打翻了彩虹色的墨水瓶”这样的脑洞～）

第3轮 - 助手回复： 哇，上海来的小红！✨ 那你的画笔下一定藏着外滩的钟声、弄堂里晾衣绳上的水珠，还有南翔小笼包冒出的热气吧～
下次画城市主题时，我们可以让东方明珠塔变成会眨眼的机器人，或者让武康路的梧桐叶飘进你的速写本里当书签！
（需要帮你把石库门的砖缝画成藏着彩虹的密码吗？或者让黄浦江的波浪卷起你调色盘里的星光～）

第4轮 - 助手回复： 原来小红是背着画板穿梭在教室和走廊的小艺术家呀！📚🎨
你的铅笔盒里是不是藏着会变形的橡皮擦——擦掉数学公式时，偷偷在草稿纸上留下半朵樱花？
下次画作业本上的格子时，可以让它们突然长出藤蔓，把“解：”变成秋千，让“答：”变成滑梯～
（需要帮你把课桌涂鸦变成会讲冷笑话的云朵吗？或者让校服袖口的墨水渍开出小小的星空？）

第5轮 - 助手回复： 你刚才提到自己来自**上海**呀～✨
（需要我帮你把东方明珠塔画成会转圈的彩虹棒棒糖，还是让外滩的钟声变成跳跳糖蹦进你的速写本里？）

第6轮 - 助手回复： 你刚才提到自己叫**林小艺**呀～✨
（需要我把你的名字编成藏在铅笔盒里的萤火虫密码，还是让作业本上的签名突然长出小翅膀？）

窗口记忆的最终对话历史（最近2轮）：
human: 我刚才说我来自哪里？
ai: 你刚才提到自己来自**上海**呀～✨
（需要我帮你把东方明珠塔画成会转圈的彩虹棒棒糖，还是让外滩的钟声变成跳跳糖蹦进你的速写本里？）
human: 我刚才说我叫什么？
ai: 你刚才提到自己叫**林小艺**呀～✨
（需要我把你的名字编成藏在铅笔盒里的萤火虫密码，还是让作业本上的签名突然长出小翅膀？）
```

### 3.1.3 摘要记忆

适用场景：当前对话过长，但新对话仍需要继承前文内容。

#### 关键代码

```python
summary_base_chain = (
    RunnablePassthrough.assign(
        chat_summary=lambda x: summary_chain.invoke(
            {
                "chat_history_text": "\n".join(
                    [f"{msg.type}: {msg.content}" for msg in x["chat_history"]]
                )
            }
        ).content
    )
    | summary_memory_prompt
    | llm
)
```

`chat_summary` 使用 `lambda` 将传入的 `x["chat_history"]` 中所有历史消息用 `"\n"` 聚合成一个字符串，作为总历史记录输入摘要链。

#### 待继续理解

- `RunnablePassthrough.assign` 的功能还需要进一步学习。
- 需要理解为什么这里必须将它放在 chain 中。

#### 工作流理解

```text
pipeline(summary_base_chain)
  -> RunnableWithMessageHistory(
       runnable=pipeline,
       get_session_history=memory_store_function
     )
  -> summary_memory_store[session_id]
  -> summary text
  -> summary_memory_prompt
  -> llm
```

与全量记忆和窗口记忆相比，摘要记忆的结构差异不大，核心区别是中间多了一步历史压缩。

> Note：手动实现流程更符合我的代码习惯，果然还是更适应面向过程的方式。

## 3.2 外部行动层：Tool

Tool 的目标是让模型调用外部工具完成动作。

### 3.2.1 Agent 与 `create_agent`

`langchain.agents` 下的 `create_agent` 函数用于创建一个 agent graph。它会循环调用工具，直到满足停止条件。

原文说明：

```text
Creates an agent graph that calls tools in a loop until a stopping condition is met.
```

理解：

- `langchain.agents` 组件实际是在 graph 逻辑下运行的。
- `create_agent(debug=True)` 可以打印 log。
- 如果要理清 agent 内部的循环、记忆拼接和消息更新方式，还需要阅读源代码。
- 当前粗略理解是：agent 会合并 `HumanMessage`、`ToolMessage` 和 `AIMessage` 形成全量过程上下文。

### 3.2.2 案例：温度转换

`TemperatureConvertInput(BaseModel)` 继承 `BaseModel` 后，会成为一个可校验、可序列化、可生成 schema 的数据模型类。

基本格式：

```python
class 类名(BaseModel):
    字段名: 字段类型 = Field(...)
```

理解：

- 字段类型用于校验输入类型。
- `Field` 给字段添加说明。
- `description` 可以为 LLM 提供描述性校验信息。
- `Field` 还有其他约束校验方法。

#### 工具定义

```python
@tool(args_schema=TemperatureConvertInput)
def temperature_converter(temperature: float, from_unit: str) -> str:
    """温度单位转换工具"""
    if from_unit not in ["celsius", "fahrenheit"]:
        return f"错误：单位'{from_unit}'不合法，仅支持'celsius'或'fahrenheit'"

    if from_unit == "celsius":
        fahrenheit = temperature * 9 / 5 + 32
        return f"{temperature}摄氏度 = {fahrenheit:.2f}华氏度"

    celsius = (temperature - 32) * 5 / 9
    return f"{temperature}华氏度 = {celsius:.2f}摄氏度"

tools = [temperature_converter]
```

这里的单位判断放在工具函数中。但更理想的结构是：单位判断属于输入校验，应该尽量放在 `BaseModel` 中。这样工具只负责转换，不负责校验。

#### 重要观察：工具边界

当 query 为：

```text
将237开尔文转换为华氏度
```

以下逻辑没有按预期直接生效：

```python
if from_unit not in ["celsius", "fahrenheit"]:
    return f"错误：单位'{from_unit}'不合法，仅支持'celsius'或'fahrenheit'"
```

根据 log，模型在调用工具前先进行了单位判断和自我转换。或者说，它先过了一轮 loop，发现校验失败后，在下一轮 loop 中把输入改成了可以通过校验的 `celsius`。

这不符合当前工具设计逻辑。后来将 docstring 从：

```text
温度单位转换工具
```

改成：

```text
只支持摄氏度和华氏度之间的互相转换，不支持开尔文。
```

模型才停止调用工具，但仍然根据自身推理给出了结果。

问题在于：描述性约束不能保证硬约束。更理想的行为是：模型在调用工具校验失败时直接输出报错，而不是给出所谓的弥补措施。因此需要设计校验失败后的工具截断机制。

这个现象说明：Agent 的任务边界需要被认真设计。

- 从结果论看，流程完成了 query 的请求。
- 从过程论看，这个结果是危险的，因为答案可能是在超越工具边界的基础上得到的。
- 如果希望结果可审计，直接给出结果时至少应包括：
  1. 初始 query 校验是否通过。
  2. 调用 tool 前是否改写了 query。
  3. query 改写的摘要和流程。

### 3.2.3 `@tool`

关键参数：

- `description`：Agent 判断何时调用该工具的核心依据。
- `infer_schema`：控制是否自动从函数类型注解推导参数 schema。

### 3.2.4 Agent 代码运行

`langchain_experimental` 部分建议后续更新到 `langsmith` 和 `deepagents` 的相关调用。

### 3.2.5 文件查询与 CLI 模拟

文件查询 demo 中，输入以 `while True` 对话框形式开启，输入 `quit` 退出，模仿 Claude Code 或 Codex CLI。

另外看到一种新的 pipeline 方式：

```python
agent = prompt | llm.bind_tools(tools)
```

原版 GitHub 文件中使用的是 `create_agent`。

## 3.3 工作流总结

我更习惯把这三天的学习总结为数据流和工作流，尝试找一种宏观架构来理解和串联不同的名词与过程。

当前可以粗略总结为：

```text
用户输入
  -> prompt / messages
  -> model / chat_model
  -> parser / memory / tool
  -> chain / agent / graph
  -> 输出或外部动作
```

后续学习重点：

- 不同记忆策略的成本、可靠性和边界。
- tool schema 与硬校验的关系。
- agent 内部消息循环和工具调用轨迹。
- 用工作流视角重新组织 LangChain、LangGraph 和 Agent 的概念。
