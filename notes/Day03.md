# Day03 Notes

## 3.1 状态管理
对话记忆的方法与Day02中的部分思考相似，具体的简单实现可以在本章中学习
- 全量记忆
- 窗口记忆 (截断的全量记忆)
- 摘要记忆

### 全量记忆
适用场景：推荐短对话，因为是记忆的全部内容


样例：
```python
full_memory_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是友好的对话助手，需基于完整的历史对话回答用户问题。"),
    MessagesPlaceholder(variable_name="chat_history"),  # 历史消息占位符
    ("human", "{user_input}")  # 用户当前输入
])
```
输入的列表结构与day02中的messages构造逻辑，差异在于
1. 本例结构为List[tuple] 其中，元组1为原来dict里的key，2为原来的content
2. 从原来的一轮存在一个list元素里变成按原来的key堆叠。
3. MessagesPlaceholder占位符名字可以理解为assistant

完整用法进源代码查看

```python
base_chain = full_memory_prompt | llm

def get_full_memory_history(session_id: str) -> BaseChatMessageHistory:
    """根据session_id获取会话历史，不存在则创建新的历史记录"""
    if session_id not in full_memory_store:
        full_memory_store[session_id] = InMemoryChatMessageHistory()
    return full_memory_store[session_id]

full_memory_chain = RunnableWithMessageHistory(
    runnable=base_chain,
    get_session_history=get_full_memory_history,
    input_messages_key="user_input",  # 输入中用户问题的键名
    history_messages_key="chat_history"  # 传入提示词的历史消息键名
)

# 测试多轮对话（指定session_id=user_001，隔离不同用户）
config = {"configurable": {"session_id": "user_001"}}

# 第一轮对话
response1 = full_memory_chain.invoke({"user_input": "我叫小明，喜欢编程"}, config=config)
print("助手回复1：", response1.content)

for msg in get_full_memory_history("user_001").messages:
    print(f"{msg.type}: {msg.content}")
```
分析代码工作流：
pipline               ->
memory_store (Global) ->
                           RunnableWithMessageHistory(runnable=pipline, get_session_history=memory_store_function) ->  自定义函数调取memory_store历史

memory_store为字典，value为InMemoryChatMessageHistory类

RunnableWithMessageHistory类 封装 pipline和InMemoryChatMessageHistory 继承了pipline的主要功能，所以可以直接用invoke

RunnableWithMessageHistory类封装 功能是在pipline基础上多了一个记忆存贮，到memory_store

memory_store_function定义时用的是全局变量，个人不太习惯，但是可能对于固定的工作流来讲比较方便。另一个原因可能是memory_store_function需要传参到封装中，如果有其他的需要额外输入的话，函数传参结构有点麻烦。

RunnableWithMessageHistory类 invoke时返回的的AIMessage类。

### 窗口记忆
适用场景：保留最近的N轮对话（N用k参数控制），说不好

所有逻辑和api用法与全量记忆一致，仅仅只是在定义get_session_history上加上了根据K (Global)覆盖的限制

####
有关幻觉：
以下结果是我在inputs中第三次改动和运行后的结果。也就是说step5根据input运行了三次。
第一次：原始input。得到类似示例的回答
第二次：在问城市后继续加问名字（二者在同一轮），测试是否是真的遗忘。也得到了 没有报名字 类似的回复，且回复较为简练
第三次：在问城市后继续加问名字（二者在不同轮）。如下所示，出现了幻觉，且回答也十分复杂

问题：这三次应该都是全新对话，如跑第二次的时候，并没有注入和覆盖第一轮任何信息.它们的差异和window_memory_store['user_002']没有重置有关系么,虽然本身模型的概率预测是主要因素？毕竟window_memory_store没有重置的话，其内的footprint和ID信息应该是三次共享的。

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

### 摘要记忆
适用场景：前对话过长，需要新对话继承前文内容

关键代码：
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
chat_summary用lambda把传入的x中的所有历史都以"\n"聚合成了一个str作为总历史记录

RunnablePassthrough.assign的功能还需细致学习，比如为什么需要这个在chain里

代码工作流：

pipline (summary_base_chain)  ->
summary_memory_store (Global) ->
                           RunnableWithMessageHistory(runnable=pipline, get_session_history=memory_store_function) ->  自定义函数调取memory_store历史

与前两个流程的区别不大

Note：手动实现流程更符合我的代码习惯，果然还是只能适应面向过程的方式

## 3.2 外部行动层（Tool）
调用外部工具
langchain.agents下的create_agent函数
Creates an agent graph that calls tools in a loop until a stopping condition is met.

langchain.agents 组件实际是在graph逻辑下的

create_agent的debug=True的情况下，虽然能打印log，但是要理清楚agent内部的循环记忆拼接和更新方式还需要看一下源代码。大致理解是根据全量记忆合并Human，Tool和AI message

### 案例3.2温度转换

TemperatureConvertInput(BaseModel)是为了继承BaseModel变成一个可校验、可序列化、可生成 schema 的“数据模型”类

基本格式

class 类名(BaseModel):
    字段名: 字段类型 = Field(...)

字段类型作为被校验的类型,Field给这个字段添加说明，description为llm提供描述性性校验。Field有其他约束校验的方法

```python
@tool(args_schema=TemperatureConvertInput)
def temperature_converter(temperature: float, from_unit: str) -> str:
    """温度单位转换工具"""
    if from_unit not in ["celsius", "fahrenheit"]:
        return f"错误：单位'{from_unit}'不合法，仅支持'celsius'或'fahrenheit'"

    if from_unit == "celsius":
        fahrenheit = temperature * 9/5 + 32
        return f"{temperature}摄氏度 = {fahrenheit:.2f}华氏度"
    else:
        celsius = (temperature - 32) * 5/9
        return f"{temperature}华氏度 = {celsius:.2f}摄氏度"

tools = [temperature_converter]
```
单位判断这里是在工具定义里了，但是实际上，单位判断也应该是校验的一部分，如果能放在class里设定的话应该会让结构更加清晰。也就是说，工具只做转换，不涉及校验，只让BaseModel做校验。


！！！重要
query = "将237开尔文转换为华氏度" 时
```python
    if from_unit not in ["celsius", "fahrenheit"]:
        return f"错误：单位'{from_unit}'不合法，仅支持'celsius'或'fahrenheit'"
```
并没有生效，根据log，模型在调用到工具前首先进行了单位判断和自我转换，或者说是过了一遍loop，在发现校验失败以后，在下一轮loop里先把输入改成了可以通过校验的celsius。这是不符合现有的工具设计逻辑的。我在把```text """温度单位转换工具""" ```换成了 ```text """只支持摄氏度和华氏度之间的互相转换，不支持开尔文。"""```后，模型才会停止调用工具，但是还是根据自身推理给出了结果。但是有一个问题在于，描述性支持不能保证硬约束。第二点，我们其实是希望如果模型在调用工具时直接输出报错的，而不是输出一个所谓的弥补措施，因此，需要的是校验失败后的工具截断。description 很重要

这个问题说明了一个需要考虑的问题，怎么设定agent的任务边界。从结果来看，似乎流程完成了query的请求，但实际上，从流程看这是危险的。因为给出的答案是在超越工具边界的基础上进行的，但是我们很难判断这个结果是在边界内还是外得到的。除非我们加了一些看上去比较硬性的描述性约束。

结果论而言，我们设定agent是为了query跑通。
过程论而言，query跑通重要，但是我们更想知道query跑通的主要轨迹来确保非幻觉，至少在直接给出结果时，还需要包括：1.初始query校验是否通过；2.调用tool前对query更改的摘要和流程

### @tool
description: Agent 判断 “何时调用该工具” 的核心依据
infer_schema: 控制是否自动从函数的类型注解推导参数 schema




### 我的主要工具需求总结


## 工作流总结

我更习惯把这三天的学习总结为数据流和工作流，尝试找一种宏观的总体架构来理解和串联不同的名词和过程。
