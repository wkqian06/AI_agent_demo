# Day05 Notes

图的核心组件:图状态、点、边

状态转移，是不是与有限状态机有关系

## 一些基础知识补充

```python
from typing import TypedDict, NotRequired

class name(TypedDict):
    key: type
    key_optional: NotRequired[type]
```

NotRequired， Optional 是个好东西

函数内部没有把 update 更新回 state, 只能理解为这是LangGraph 在节点函数返回之后自动完成

构建图重要的是了解怎么搭建节点函数和边，即StateGraph的方法

### 关于6.2的问题

为什么节点返回的字典只需包含要更新的字段，无需返回完整状态？

我不知道啊，这个和langgraph设计有关系。它设计了只用传入更新的键，就能更新图状态，那么就只包含更新字段，如果它设计的是比较庞大的整图状态更新，那么就只能在返回的时候返回整体的状态。

## 边

边的类型：

1. 固定边：add_node(node_name,node_function)
2. 条件边：add_conditional_edges(node_name,if_function,node_mapping_dict)
3. 循环边：add_conditional_edges(node_name,loop_function,node_mapping_dict)

2 3的区别在于mapping的时候，循环边是有一个映射到自己的node的，而条件边是其他的node

## 超步骤

激活和执行的分类我觉得有点overlap。

在并行演示中，edge里加入了START和END。之前的案例中并没有加入也可以运行，可能是因为链式过于简单，edge的定义中已经确定了开始和结束。但是如果是比较复杂的图，就需要确定开始和结束节点，虽然本案例也不复杂，可以省略，但是建议保留改习惯

## 综合实操

MemorySaver()怎么查看的？

如果需要检查图，那么node_mapping_dict很重要。否则，即使实际流程正确，但是画出来的图会少映射
