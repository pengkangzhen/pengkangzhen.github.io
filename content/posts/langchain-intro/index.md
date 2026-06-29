---
title: "LangChain 入门：从核心组件到 LCEL 管道"
date: 2026-06-29
categories: ["技术"]
tags: ["LangChain", "LLM", "Python", "多智能体"]
summary: "用 mako 多智能体项目的真实代码，讲清 LangChain 的核心组件、LCEL 管道语法，以及网上旧教程里的废弃 API 该怎么替换。"
draft: false
math: false
---

> 这是「LLM 应用开发」系列的第三篇。前两篇[结构化输出](/posts/structured-output-techniques/)和 [Pydantic](/posts/pydantic-structured-output/) 解决的是「单个 agent 怎么说话」，这篇开始讲「agent 本身是怎么搭起来的」。

## 为什么需要 LangChain

先看**裸调 OpenAI SDK** 长什么样：

```python
from openai import OpenAI

client = OpenAI()
resp = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "把麻婆豆腐翻译成英文"}],
)
print(resp.choices[0].message.content)
```

能跑，但只要需求稍微复杂一点，痛点就全冒出来：

- **换模型要改代码**：想从 GPT 换成 DeepSeek、Claude，每个 SDK 接口都不一样，得重写。
- **Prompt 难管理**：`f"..."` 字符串拼来拼去，变量一多就乱，还没法复用。
- **输出是裸字符串**：想要结构化数据（JSON、对象），得自己写正则解析、自己处理失败重试。
- **没有组合能力**：想把「生成 → 校验 → 重写」串成流程，得自己拿 `if/for` 硬接。

**LangChain 就是来解决这些的**——它给 LLM 应用提供了一套**标准化的「积木」**：统一的模型接口、可复用的提示模板、声明式的流程组合（LCEL）。你不用再手搓这些轮子。

> 类比：调原生 SDK 像用 `socket` 手写 HTTP，用 LangChain 像用 FastAPI——框架帮你把重复劳动抽象掉，你只关心业务逻辑。

---

## 一、LangChain 是什么

一句话定义：

> **LangChain 是一个用于构建 LLM 应用的开发框架，核心价值是把「调用 LLM」这件原始的事，抽象成一组可组合、可替换、模型无关的标准组件。**

它解决三件事：

| 价值 | 说明 |
|------|------|
| **模型抽象** | 一套接口（`BaseChatModel`）适配所有主流 LLM，换模型只改一行 |
| **可组合（LCEL）** | 用 `\|` 把组件像 Unix 管道一样串成流程，声明式而非命令式 |
| **生态集成** | 结构化输出、工具调用、检索（RAG）、回调……都封装好了 |

本文只讲前两个——因为**我的 mako 项目就只用了这两个**。第三点（agent 工具、检索）mako 走了另一条路（LangGraph），会在下一篇讲。

---

## 二、主要组件（⚠️ 含版本陷阱）

### 先说一个非常重要的坑

LangChain 在 2024 年底发布了 **1.0 大版本**，重构了一批 API，**废弃了大量旧接口**。问题是：网上绝大多数中文教程（包括很多爆款文章）还在用 **0.x 时代的旧 API**，照着学全盘皆错。

我自己一开始总结的组件清单就是从旧教程抄的，后来对照项目代码才发现大半是废弃的。先把这张**「旧 → 新」对照表**摆出来，帮你避坑：

| 旧 API（已废弃 / 不推荐） | 新版写法 | mako 是否用到 |
|--------------------------|---------|:------------:|
| `langchain.chains.LLMChain` | LCEL 管道 `prompt \| llm` | ✅ |
| `langchain.memory.ConversationBufferMemory` | LangGraph 的 State | ❌ |
| `langchain.agents.initialize_agent` | `langgraph.create_react_agent` | ❌ |
| `langchain_community.llms.OpenAI`（补全模型） | `langchain_openai.ChatOpenAI`（聊天模型） | ✅ |
| `langchain.embeddings.OpenAIEmbeddings` | `langchain_openai.OpenAIEmbeddings` | ❌ |
| `langchain.document_loaders.TextLoader` | `langchain_community.document_loaders.*` | ❌ |
| `PromptTemplate`（纯文本） | `ChatPromptTemplate`（带角色） | ✅ |
| `PydanticOutputParser` | `.with_structured_output()` | ✅（用后者） |

记住一条判断标准：**看到 `LLMChain`、`initialize_agent`、`ConversationBufferMemory` 这三个，基本就是过时教程，直接关掉。**

### mako 实际用到的 4 个核心组件

LangChain 组件很多，但 mako 只用了下面 4 个。搞懂这 4 个，就理解了 mako 每个 agent 的内核。

#### 1. 模型抽象（Chat Models）

所有聊天模型实现同一个接口 `BaseChatModel`，换 provider 只改实例化代码：

```python
from langchain_openai import ChatOpenAI          # OpenAI / 兼容 OpenAI 的模型
from langchain_anthropic import ChatAnthropic    # Claude

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
# 想换 Claude？只改这一行：
# llm = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)
```

#### 2. 提示模板（Prompts）

`ChatPromptTemplate` 把 prompt 和变量分离，还能区分 system / human 角色：

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template(
    "把下面这道中国菜翻译成一句英文介绍：{dish}"
)
# 也可以带角色：
# prompt = ChatPromptTemplate.from_messages([
#     ("system", "你是翻译助手"),
#     ("user", "翻译：{dish}"),
# ])
```

#### 3. 输出解析 / 结构化输出

LangChain 有两种方式拿结构化数据：

- **老办法**：`StrOutputParser` / `PydanticOutputParser`——模型吐字符串，再用 parser 解析。
- **新办法（推荐）**：`.with_structured_output(Schema)`——直接让模型返回 Pydantic 对象，底层用 [结构化输出](/posts/structured-output-techniques/)那四层技术。

#### 4. LCEL 链（LangChain Expression Language）

**这是 LangChain 1.x 的灵魂。** 用 `|` 运算符把上面三个组件串成一条管道，替代了旧的 `LLMChain`。下一节专门讲。

### mako 没用的组件（简单了解即可）

为完整起见，列一下 LangChain 还有但 mako 没碰的几类——它们不是没用，而是 mako 选了别的方案：

| 组件 | 用途 | mako 的替代方案 |
|------|------|----------------|
| **Agents / Tools** | 让 LLM 自主调用工具 | 用 LangGraph 手搓多 agent 图 |
| **Memory** | 多轮对话记忆 | 用 LangGraph 的 `AgentState` 共享状态 |
| **Retrieval / RAG** | 向量检索外部知识 | 用结构化知识模块（IKD），不做向量检索 |

> 结论：**别试图学完整个 LangChain**。它太大，而且很多模块 mako 根本不用。盯住上面 4 个核心组件 + LCEL，就够看懂 mako 了。

---

## 三、LangChain 工作流：固定四步

不管多复杂的 LangChain 应用，核心流程永远是这四步：

```
① 创建提示模板  →  ② 初始化模型  →  ③ 用 | 组装成链  →  ④ 调用 invoke
```

### 一个通用例子：菜品翻译

需求：输入一道中国菜，输出一句英文介绍。

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# ① 提示模板
prompt = ChatPromptTemplate.from_template(
    "把下面这道中国菜翻译成一句英文介绍：{dish}"
)

# ② 初始化模型
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# ③ 组装链（LCEL：用 | 把组件串起来）
chain = prompt | llm | StrOutputParser()

# ④ 调用
print(chain.invoke({"dish": "麻婆豆腐"}))
# -> "Mapo tofu: a spicy Sichuan dish of tofu cooked with chili and minced pork."
```

### 关键：理解 `|` 管道（LCEL）的数据流

新手最容易卡在 `prompt | llm | StrOutputParser()` 这行——`|` 是什么？

它是**运算符重载**，行为类似 Unix 管道：**把左边组件的输出，喂给右边组件作为输入**。整条链的数据流是这样流动的：

```
invoke({"dish": "麻婆豆腐"})
        │
        ▼
┌─────────────────────┐
│  ChatPromptTemplate │  填充模板，输出 messages
└─────────────────────┘
        │  messages
        ▼
┌─────────────────────┐
│   ChatOpenAI (llm)  │  调用模型，输出 AIMessage
└─────────────────────┘
        │  AIMessage(content="Mapo tofu...")
        ▼
┌─────────────────────┐
│  StrOutputParser    │  从 Message 里提取字符串
└─────────────────────┘
        │
        ▼
   "Mapo tofu: ..."
```

每个组件都是一个「处理单元」，吃一种类型、吐一种类型，`|` 负责把它们接起来。**这就是声明式编程**——你描述「流程长什么样」，而不是写「先执行 A，再把结果传给 B」的命令式代码。

### 升级版：结构化输出

把第 ③ 步的 parser 换成 `.with_structured_output()`，就直接拿到结构化对象（衔接[前两篇](/posts/pydantic-structured-output/)）：

```python
from pydantic import BaseModel, Field

class DishIntro(BaseModel):
    en_name: str = Field(description="英文菜名")
    spicy: bool = Field(description="是否辣")

# 只改这一行：把 StrOutputParser() 换成结构化输出
chain = prompt | llm.with_structured_output(DishIntro)

result = chain.invoke({"dish": "麻婆豆腐"})
print(result.en_name, result.spicy)   # Mapo Tofu True
```

四步还是那四步，只是换了第 ③ 步的一个组件。**LCEL 的好处就在这：组件可插拔，流程不变。**

---

## 四、结合 mako 项目详解

通用例子看懂了，现在看 mako 真实代码。mako 有 6 个 agent，每个 agent 的内核都是**上面那套四步流程**。以第一个 agent `DataEngineer`（数据工程师）为例，它的职责是把原始数据整理成建模可用的数据集：

```python
# src/mako_langchain/agents/data_engineer.py （简化）

def create_data_engineer(provider, model):
    # ①② 模板和模型
    llm = get_llm(provider, model, temperature=0)        # 见下文：多 provider 抽象
    prompt = get_data_engineer_prompt()                  # 提示模板（从 prompts 模块加载）

    # ③ 组装链：LCEL 管道 + 结构化输出
    chain = prompt | llm.with_structured_output(
        DataEngineerOutput, method="json_mode"
    )
    return chain
```

发现了吗？**这和通用例子的「升级版」是同一个结构**：

```
通用例子:   prompt | llm.with_structured_output(DishIntro)
mako:       prompt | llm.with_structured_output(DataEngineerOutput)
```

唯一的区别是 `DataEngineerOutput` 这个 Schema 更复杂——它装的是「集合、参数、决策变量」这些优化建模元素（详见[ Pydantic 那篇](/posts/pydantic-structured-output/)）。**但骨架完全一样。**

第 ④ 步调用则被包进了一个 LangGraph 节点函数：

```python
def data_engineer_node(state: Dict) -> Dict:
    chain = create_data_engineer(state["provider"], state["model"])
    result = chain.invoke({
        "role": DATA_ENGINEER_ROLE,
        "task": task,            # 由模板里的 {task} 占位符填充
    })
    return {"data_engineer_output": result}   # 写回共享状态
```

### 亮点 1：多 provider 抽象

通用例子里 `llm = ChatOpenAI(...)` 是写死的，mako 则用 `get_llm(provider, model)` 动态创建：

```python
# src/mako_langchain/utils/llm_config.py （简化）
def get_llm(provider="DeepSeek", model=None, temperature=0):
    if provider in ANTHROPIC_PROVIDERS:        # MiniMax M2.x 走 Anthropic 端点
        return ChatAnthropic(model=model, ...)
    return ChatOpenAI(model=model, base_url=..., api_key=...)  # 其余走 OpenAI 兼容
```

这就是**模型抽象的价值**：mako 能在十几个模型（DeepSeek、Qwen、GLM、Kimi、Claude……）之间切换，业务代码（`prompt | llm.with_structured_output(...)`）一个字都不用改。换模型 = 换 `get_llm` 的参数。

### 亮点 2：为每个模型「打补丁」选结构化输出方法

这是真实项目才会遇到的工程细节。mako 在 `llm_config.py` 里给不同模型动态生成子类，强制指定结构化输出的 method：

```python
# 某些模型用 json_mode 会输出非法 JSON，强制改用 function_calling
class PatchedFCChatOpenAI(ChatOpenAI):
    def with_structured_output(self, schema, method=None, **kw):
        return super().with_structured_output(schema, method="function_calling", **kw)
return PatchedFCChatOpenAI(**kwargs)
```

> 这正好是[结构化输出那篇](/posts/structured-output-techniques/)的实战闭环：博客里讲了「四种 method 怎么选」，mako 在工程上就是**为每个模型踩坑后逐个定死 method**。理论 → 落地，对上了。

### 小结：mako agent 的本质

```
LangChain chain（内核）          LangGraph node（外壳）
prompt | llm.with_structured_   ──包装──>  def data_engineer_node(state):
  output(Schema)                            chain.invoke(...)
                                                │
                                             写回共享状态
```

- **LangChain 负责「一个 agent 怎么干活」**：prompt + 模型 + 结构化输出，组成一条 LCEL 链。
- **LangGraph 负责「多个 agent 怎么协作」**：把这些链包装成节点，连成一张有状态的图。

---

## 下篇预告

这篇讲清了单个 agent 的内核（LangChain chain）。但 mako 是**多智能体系统**——6 个 agent 要按顺序传数据、出错了要回溯诊断。这些「协作」逻辑，LangChain 本身管不了，是 **LangGraph** 的活。

下一篇就讲：怎么用 LangGraph 把这些 chain 节点连成一张会自我纠错的流水线图。敬请期待。
