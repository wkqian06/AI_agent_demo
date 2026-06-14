# Notes
API 需要有额度！！！（充钱！

# langchain_openai 简单案例
核心
```python
from langchain_openai import ChatOpenAI # 问题： 还有其他的api接口么？有什么区别

llm = ChatOpenAI(
    api_key=API_KEY,
    base_url=BASE_URL,
    model="deepseek-chat",
    temperature=0.3
) #lang封装下的模型调用，看上去已经很全面了，针对不同模型的适应性接口。 问题：初始化模型以后，是新开的会话么？即如果我在response = llm.invoke(prompt)后再加一个response = llm.invoke(prompt_new)这是相当于上一个response.content+prompt_new的输入还是新开的会话？

prompt = 'str'

response = llm.invoke(prompt)

response.content # 结果，需要print。问题：response类还有其他需要注意的方法么？
```

# langgraph.graph 简单案例
核心

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict
```
`StateGraph` 用来定义一个基于状态流转的工作流。它不是直接调用一次模型，而是把任务拆成多个节点，每个节点读取当前 `state`，再返回一部分新的 `state`。

## 1. 定义 State

```python
class WorkflowState(TypedDict, total=False):
    user_role: str
    original_advice: str
    simplified_advice: str
```

`WorkflowState` 用来声明整个工作流中允许出现的状态字段，也就是每一步可以读取和写入的 key。

- `user_role`：初始输入，表示用户身份，例如“高校学生”。
- `original_advice`：第一个节点生成的原始建议。
- `simplified_advice`：第二个节点生成的精简建议。

问题：如何设定 key？

key 应该来自这个工作流真正需要在节点之间传递的信息。一般可以按“输入 -> 中间结果 -> 最终输出”来设计。这里的流程是：

```text
user_role -> original_advice -> simplified_advice
```

`total=False` 表示这些字段不要求一开始全部都有。比如执行时只传入：

```python
{"user_role": "高校学生"}
```

后面的 `original_advice` 和 `simplified_advice` 会由节点逐步补上。

## 2. 定义节点函数

```python
def generate_advice(state: WorkflowState):
    prompt = f"给{state['user_role']}写一段50字左右的 AI 学习建议。"
    result = llm.invoke(prompt)
    return {"original_advice": result.content}

def simplify_advice(state: WorkflowState):
    prompt = f"把下面的学习建议精简到30字以内：{state['original_advice']}"
    result = llm.invoke(prompt)
    return {"simplified_advice": result.content}
```

节点函数的特点：

- 参数通常是当前 `state`。
- 函数内部可以读取 `state` 中已有的 key。
- 返回值是一个字典，用来更新 `state`。

`generate_advice` 固定了主要 prompt，只把 `state["user_role"]` 当作变量输入；`simplify_advice` 则依赖上一个节点写入的 `state["original_advice"]`。

## 3. 构建工作流

```python
workflow = StateGraph(WorkflowState)

workflow.add_node("generate", generate_advice)
workflow.add_node("simplify", simplify_advice)

workflow.add_edge(START, "generate")
workflow.add_edge("generate", "simplify")
workflow.add_edge("simplify", END)
```

这里定义的是一个线性流程：

```text
START -> generate -> simplify -> END
```

`add_node("generate", generate_advice)` 的含义是给节点起名，并绑定实际执行的函数。

`add_edge` 用来定义执行顺序：

- 从 `START` 进入 `generate`。
- `generate` 执行完后进入 `simplify`。
- `simplify` 执行完后到 `END`。

## 4. 编译和执行

```python
app = workflow.compile()

result = app.invoke({"user_role": "高校学生"})
```

`compile()` 会把前面定义的节点和边编译成可执行的工作流。之后调用 `app.invoke(...)`，传入初始状态。

最终 `result` 会包含流程中积累下来的状态：

```python
result["original_advice"]
result["simplified_advice"]
```

注意：

- `invoke` 传入的 key 应该和 `WorkflowState` 中定义的字段一致。
- 节点函数中访问的 key 必须在执行到该节点前已经存在。
- 节点返回的字典会合并回整体 `state`，供后续节点继续使用。
