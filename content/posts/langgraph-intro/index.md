---
title: "LangGraph 入门：把多个 agent 连成一张会循环的图"
date: 2026-06-29
categories: ["技术"]
tags: ["LangGraph", "LangChain", "LLM", "多智能体"]
summary: "LangChain 的链跑完就结束。真实的多 agent 系统需要分支、循环、共享状态——这正是 LangGraph 干的事。用 mako 项目的真实工作流讲清 State、Node、Edge 三件套。"
draft: false
math: false
---

> 这是「LLM 应用开发」系列的第四篇。[上一篇](/posts/langchain-intro/)讲了单个 agent 的内核（LangChain 的 LCEL 链），这篇讲怎么把多个 agent 连成一张能协作、能纠错的图。

## 上一篇的链，有个硬伤

上一篇我们搭了这样一条 LangChain 链：

```python
chain = prompt | llm | StrOutputParser()
chain.invoke({"dish": "麻婆豆腐"})
```

它跑一次就结束了。但只要需求稍微真实一点，立刻不够用：

- **要分支**：翻译完想检查一下，通过了才输出，不通过就重翻——`|` 管道没法「看结果决定下一步」。
- **要循环**：翻得不好得重来，甚至重试好几次——`|` 管道是直的，拐不回来。
- **要共享中间结果**：多个步骤要读同一份数据、往里写新字段——管道里数据只能往前流，没地方「存」。

**LangGraph 就是来补这三块的。** 它给 LangChain 的链加上了「有状态的图」：能分支、能循环、能让多个步骤共享一块「黑板」。

> 关系一句话：**LangChain 提供 agent 的「内核」（上一篇的 chain），LangGraph 提供「图纸」——把这些 agent 连成一张图。** LangGraph 也是 LangChain 公司出的，依赖 `langchain-core`，不是竞品，是同一生态的进阶层。

---

## 一、三个核心概念

LangGraph 万变不离其宗，就三样东西：

### 1. State（状态）—— 所有节点共享的「黑板」

一个 `TypedDict`，整张图共享。每个节点能读它，也能往里写新字段。

```python
from typing import TypedDict

class MyState(TypedDict):
    dish: str           # 输入
    translation: str    # 某个节点产出
    approved: bool      # 另一个节点产出
```

**关键机制**：节点不返回完整 state，而是返回一个**部分字典**（只含要更新的字段），LangGraph 自动 merge 回去。所以节点之间不用互相传参，大家都读写这块黑板。

### 2. Node（节点）—— 干活的函数

一个普通函数，签名固定 `def node(state) -> dict`：读 state、干活、返回部分更新。

```python
def translate(state: MyState) -> dict:
    text = chain.invoke({"dish": state["dish"]})   # 用上一篇的 chain
    return {"translation": text}                    # 只返回要更新的字段
```

**节点内部就是上一篇讲的 LangChain 链**——LangGraph 不改变 agent 的内核，只是把它包成一个节点。

### 3. Edge（边）—— 节点之间的连线

两种边：

- **直连** `add_edge(A, B)`：A 跑完，**一定**去 B。
- **条件路由** `add_conditional_edges(A, router, mapping)`：A 跑完，调用 `router(state)` 函数，**根据返回值决定去哪**。这是实现「分支」和「循环」的关键。

```python
def decide(state) -> str:
    return "done" if state["approved"] else "retry"   # 根据状态决定

graph.add_conditional_edges(
    "review", decide,
    {"retry": "translate", "done": END},   # 返回值 → 目标节点
)
```

把 `"retry"` 指回 `"translate"`，就形成了**循环**——这是 LCEL 做不到的。

---

## 二、四步搭一张图

不管多复杂的 LangGraph 应用，搭图永远是这四步：

```
① 定义 State  →  ② 定义 Node 函数  →  ③ 连边（直连 + 条件路由）→  ④ compile & invoke
```

### 通用例子：翻译 + 审校（带循环）

把上一篇的「菜品翻译」升级：翻译完让一个「审校」节点检查，不通过就回去重翻。

```python
from typing import TypedDict, Literal
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langgraph.graph import StateGraph, END

# 复用上一篇的翻译链
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
prompt = ChatPromptTemplate.from_template("把这道菜翻译成英文：{dish}")
translate_chain = prompt | llm | StrOutputParser()

# ① 定义状态
class State(TypedDict):
    dish: str
    translation: str
    approved: bool

# ② 定义节点
def translate(state: State) -> dict:
    text = translate_chain.invoke({"dish": state["dish"]})
    return {"translation": text}

def review(state: State) -> dict:
    # demo 用简单规则；真实场景这里换成 LLM 审校（mako 就是这么做的）
    ok = len(state["translation"]) > 5
    return {"approved": ok}

def decide(state: State) -> Literal["retry", "done"]:
    return "done" if state["approved"] else "retry"

# ③ 连边
g = StateGraph(State)
g.add_node("translate", translate)
g.add_node("review", review)
g.set_entry_point("translate")                # 起点
g.add_edge("translate", "review")             # 直连
g.add_conditional_edges("review", decide, {   # 条件路由
    "retry": "translate", "done": END,
})

# ④ 编译 & 调用
app = g.compile()
result = app.invoke({"dish": "麻婆豆腐"})
print(result["translation"])
```

对应的数据流长这样——注意 `review` 到 `translate` 那条**回头的箭头**，那就是循环：

```
            ┌───────────┐
      ┌────>│ translate  │  （复用上一篇的 chain）
      │     └─────┬─────┘
      │           ▼ (直连)
   retry     ┌───────────┐
      │     │  review    │  审校
      └──────┤            │
            └─────┬─────┘
            decide│ approved?
         ┌────────┴────────┐
      done               retry → 回到 translate（循环！）
         ▼
        END
```

跑起来你会发现：`review` 不通过时，图会自动回到 `translate` 重翻，直到通过才结束。**这就是 LangGraph 相比 LangChain 链的杀手锏——会循环。**

---

## 三、结合 mako 项目详解

通用例子看懂了，现在看 mako 的真实工作流。mako 有 6 个 agent，整张图（`src/mako_langchain/graph/workflow.py`）就是上面这套模式的工业级版本。

### 1. State：mako 的 `AgentState`

mako 的共享状态（`graph/state.py`）字段更多，但本质就是那张「黑板」：

```python
class AgentState(TypedDict, total=False):
    # 输入
    problem_description: str
    sample: Dict[str, Any]
    # 各 agent 的产出（写回黑板）
    data_engineer_output: Optional[DataEngineerOutput]
    model_expert_output: Optional[ModelExpertOutput]
    python_code: Optional[str]
    execution_result: Optional[Dict[str, Any]]
    # 错误处理 & 回溯
    error_info: Optional[str]
    retry_count: int
    # ...
```

每个 agent 往里写自己的产出（如 `data_engineer_output`），下游 agent 直接从 state 里读——**节点之间从不直接传参**。

### 2. Node：6 个 agent + 3 个回溯节点

```python
# workflow.py
workflow.add_node("data_engineer",   data_engineer_node)      # 数据工程师
workflow.add_node("model_expert",    model_expert_node)       # 建模专家
workflow.add_node("knowledge_loader", knowledge_loader_node)  # 知识加载
workflow.add_node("python_developer", python_developer_node)  # 写 Gurobi 代码
workflow.add_node("solver_executor",  solver_executor_node)   # 执行求解
workflow.add_node("diagnosis_agent",  diagnosis_agent_node)   # 错误诊断
```

每个 `xxx_node` 内部就是上一篇讲的 `prompt | llm.with_structured_output(...)` 那条链。**节点 = 把 LangChain 链包了一层。**

### 3. Edge：主线流水线 + 条件路由

mako 的主线是一条流水线，中间夹着两个条件路由：

```python
# 主线：直连
workflow.add_edge("data_engineer", "model_expert")
workflow.add_edge("python_developer", "solver_executor")

# 条件路由①：建模后，要不要先加载知识？
workflow.add_conditional_edges("model_expert", route_after_model_expert, {
    "knowledge_loader": "knowledge_loader",   # 模型请求了新知识 → 先加载
    "python_developer": "python_developer",   # 没请求 → 直接写代码
})

# 条件路由②：求解后，成功还是诊断？
workflow.add_conditional_edges("solver_executor", should_diagnose, {
    "success":  END,                # 求解成功 → 结束
    "diagnose": "diagnosis_agent",  # 失败 → 去诊断（进入循环！）
})
```

`should_diagnose` 的逻辑（`workflow.py:45`）就是通用例子里 `decide` 的翻版——看 `execution_result` 里的 `diagnosis_required` 标志，决定是收工还是进诊断。

### 4. 整张图长这样

```
data_engineer → model_expert ──┬(无知识请求)──────────────┐
                               │(请求知识)                  │
                               ▼                            │
                        knowledge_loader ──(加载后回到 model_expert)
                               │                            │
                               └────────────────────────────┤
                                                            ▼
                                                  python_developer
                                                            │
                                                            ▼
                                                   solver_executor
                                                            │ decide
                                            ┌───────────────┴───────────────┐
                                         成功                              失败
                                            │                                │
                                           END                          diagnosis_agent
                                                                           │ 指控某 agent
                                                                           ▼
                                                                   {agent}_backward
                                                                           │ 修正了？
                                                                ┌──────────┴──────────┐
                                                              是                    否
                                                                │                    │
                                                         重跑下游           回 diagnosis 换人指控（循环）
```

**对比通用例子**：mako 就是「翻译 + 审校」的复杂版——`solver_executor` 是「审校」，求解失败就进诊断、回到出错的 agent 修正（`{agent}_backward`），修正了重跑、没修正就换个人指控。**整个右侧就是一个会自我纠错的循环。**

### 5. 关键洞察

回头看，mako 这套架构其实只用到了 LangGraph 最核心的能力：

| LangGraph 能力 | mako 怎么用 |
|---------------|------------|
| **共享 State** | `AgentState` 黑板，agent 间不传参 |
| **条件路由** | 建模后要不要加载知识、求解后成功还是诊断 |
| **循环** | 求解失败 → 诊断 → 回溯 → 重跑，直到成功或耗尽重试 |

没有黑魔法。把这三样用熟，你也能搭出自己的多 agent 系统。

---

## 下篇预告

这篇讲了图怎么搭、怎么循环。但 mako 最有意思的地方，是**循环里到底在干嘛**——

求解失败后，`diagnosis_agent` 会像「检察官」一样**指控**最可能出错的那个 agent，被指控的 agent 要自我审查、承认或反驳。这不是普通的「重试」，而是一套**对抗式纠错**机制。

下一篇讲：怎么让多个 agent 互相「甩锅」来定位错误。敬请期待。
