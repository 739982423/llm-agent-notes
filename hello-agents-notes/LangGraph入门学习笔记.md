# 基于langgraph示例学习总结

原始代码和相关解析文档：
1、https://github.com/datawhalechina/hello-agents/blob/main/code/chapter6/Langgraph/Dialogue_System.py
2、https://datawhalechina.github.io/hello-agents/#/./chapter6/%E7%AC%AC%E5%85%AD%E7%AB%A0%20%E6%A1%86%E6%9E%B6%E5%BC%80%E5%8F%91%E5%AE%9E%E8%B7%B5?id=_65-%e6%a1%86%e6%9e%b6%e5%9b%9b%ef%bc%9alanggraph

## 一、核心模型：状态机 + 有向图

LangGraph 将 agent 的执行流程建模为一种**状态机（State Machine）**，并表示为**有向图（Directed Graph）**。图中：

- **节点（Nodes）**：代表一个具体的计算步骤（调用 LLM、执行工具、人工审核等），每个节点负责自身的逻辑；
- **边（Edges）**：定义节点之间的跳转逻辑，开发者通过控制边的连接关系控制流程走向。

与 AutoGen、CAMEL 这类"基于对话"的框架不同，LangGraph 把流程显式画成图，其最革命性的优势在于**天然支持循环（Cycles）**——通过条件边可以构建"反思-修正"循环、容错回退等复杂工作流。

## 二、全局状态（State）：节点间通信的唯一媒介

LangGraph 创建的 agent 执行流程**必须有一个全局状态结构**（通常用 `TypedDict` 定义），如示例中的 `SearchState`：

```python
class SearchState(TypedDict):
    messages: Annotated[list, add_messages]   # 对话历史（带归约器）
    user_query: str       # LLM理解后的用户需求总结
    search_query: str     # 优化后的搜索关键词
    search_results: str   # Tavily搜索结果
    final_answer: str     # 最终答案
    step: str             # 当前步骤标记
```

要点：

1. 该结构可包含对话历史、中间结果、当前步骤等任意需要追踪的信息；
2. **所有节点和边函数都能读写这个共享状态**；
3. 关键机制：**节点之间不直接传数据，全部通过读写共享 state 完成通信**。所以 state 不是"承载消息传递的容器"，而是"节点间通信的唯一媒介"——正因为如此，图才能只关心"谁先谁后"，不关心"数据怎么传"；
4. 字段级合并策略由 **归约器（Reducer）** 决定：未注解的字段默认"覆盖"，带 `Annotated[list, add_messages]` 的字段"追加合并"。`add_messages` 由框架在每个节点返回后自动调用，实现消息历史的累积与多轮对话。

## 三、节点（Node）：读完整状态，返回部分状态

节点是一个接收状态、返回状态更新的 Python 函数。精确的输入输出关系：

- **输入**：完整的全局状态（`state`）；
- **输出**：**要更新的字段子集**（部分状态 dict），不要求返回全部字段；
- 框架把返回的子集**合并**回全局状态，未返回的字段保持不变。

```python
def understand_query_node(state: SearchState) -> SearchState:
    # 读 state：取最新一条 HumanMessage
    ...
    return {
        "user_query": response.content,   # 只返回要更新的字段
        "search_query": search_query,
        "step": "understood",
        "messages": [AIMessage(...)],     # 经 add_messages 追加，而非覆盖
    }
```

注意：实际案例中的节点都是**原地返回部分字段**，而文档概念示例（`state["messages"].append(...); return state`）是原地修改整个 dict——两者都能跑，但前者的"部分更新 + 框架合并"才是 LangGraph 的设计意图。

## 四、边（Edge）：只定控制流，不搬数据

**最容易误解的地方：边并不把某个节点的输出"导入"另一个节点的输入。** 边只负责：

- **常规边**（`add_edge`）：固定执行顺序，A 执行完一定去 B；
- **条件边**（`add_conditional_edges`）：通过一个路由函数读取当前状态，动态决定下一跳。

```python
workflow.add_edge(START, "understand")
workflow.add_edge("understand", "search")
workflow.add_edge("search", "answer")
workflow.add_edge("answer", END)
```

数据流与控制流分离：

- **控制流**（谁先谁后、跳去哪里）由边决定；
- **数据流**（状态如何更新）由"节点返回值 + 框架合并机制"承担，与边无关。

**条件边是 LangGraph 最强大的功能**，是"实现循环和复杂逻辑分支的关键"。它由三部分构成：起始节点、路由函数（读 state、返回字符串键）、映射表（键 → 目标节点）。把映射表的键指回自己之前的节点，图就形成了循环。

补充说明：示例 `Dialogue_System.py` 是**纯线性图**（无条件边、无循环），搜索失败的降级是通过节点内部的 `step == "search_failed"` 判断实现的，属于节点内逻辑而非图层面的分支。真正的"搜索失败→重试/回退"循环可以用条件边改造。

## 五、其他关键要素

| 概念 | 说明 | 案例位置 |
|---|---|---|
| START / END | 特殊节点，图必须有明确入口和出口；1.0 用 `add_edge(START, ...)` | Dialogue_System.py:185 |
| 编译（compile） | 图定义好后必须 `compile()` 才能执行 | Dialogue_System.py:192 |
| Checkpointer | `InMemorySaver` + `thread_id` 实现会话级状态持久化 | Dialogue_System.py:191、224 |
| 流式执行 | `astream` 逐节点产出，便于实时展示中间过程 | Dialogue_System.py:240 |

## 六、优势与局限（设计权衡）

**优势：**

- **可控性与可预测性**：流程显式定义为"流程图"，适合高可靠性、可审计的生产级应用；
- **循环原生支持**：通过条件边轻松构建"反思-修正"循环，是构建自我优化、容错 agent 的关键；
- **模块化**：每个节点是独立函数，也便于插入"人工审核节点"（Human-in-the-loop）。

**局限：**

- **前期代码多（Boilerplate）**：定义状态、节点、边较繁琐，简单任务显得小题大做；
- **缺乏"涌现"式交互**：行为可预测但死板，强于执行确定的流程，弱于开放式协作；
- **调试挑战**：问题可能出在节点内部逻辑、节点间状态数据异变、边跳转条件三处之一，需要全局理解图的运行机制。

**设计哲学**：AutoGen / CAMEL 依赖"涌现式协作"（定义角色和目标，行为从简单规则中涌现），LangGraph 则要求显式定义每一步，牺牲部分动态性换取**可靠性、可控性、可观测性**——这是"显式控制 vs 涌现式协作"的核心权衡。
