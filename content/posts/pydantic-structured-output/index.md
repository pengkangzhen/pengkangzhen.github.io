---
title: "Pydantic 入门：为 LLM 结构化输出写好数据模型"
date: 2026-06-29
categories: ["技术"]
tags: ["Pydantic", "LLM", "结构化输出", "运筹优化", "Python"]
summary: "结构化输出的可靠性最终都落在那个 Pydantic 模型上。本文以一个'让 LLM 建立运筹优化模型'的真实场景，从基础定义讲到嵌套、Enum、before/after 验证器，再到 LangChain 接入。"
draft: false
math: false
---

## 为什么 LLM 结构化输出离不开 Pydantic

我在[上一篇](/posts/structured-output-techniques/)梳理了结构化输出的四层技术——从 Prompt 工程到约束解码、从 Function Calling 到 Structured Outputs。但你会发现，每一层最终都绕不开同一个东西：**一个描述输出"形状"的数据模型**，而且几乎都是用 Pydantic 写的。

> 旧文讲的是「在哪用 Pydantic」，这篇讲「怎么把 Pydantic 模型写好」。

为什么偏偏是 Pydantic？因为结构化输出的本质是给 LLM 一份**契约**：

```
Pydantic 模型  →  model_json_schema()  →  JSON Schema  →  发给 LLM
                    （这就是那座桥梁）         （模型据此生成）
```

- **Schema 即提示词**：字段名、类型、`description` 都会翻译进 JSON Schema，直接决定 LLM 往每个格子里填什么。
- **校验即兜底**：模型生成的 JSON 回来后，Pydantic 再验一遍。不通过就打回重试（旧文第三层的 Instructor / Guardrails 机制）。

这篇就围绕一个真实场景——**让 LLM 把一个业务问题翻译成数学优化模型**（决策变量、目标函数、约束）——来讲怎么把模型写好。这个场景来自我的多智能体优化项目，足够复杂，能把所有关键技巧都逼出来。

---

## 主要特点与用途

在深入"怎么为 LLM 写模型"之前，先快速了解 Pydantic 的主要能力——结构化输出用到的只是其中一部分：

1. **数据验证（Data Validation）**：数据传入模型时，Pydantic 自动检查字段的类型、长度、范围等是否符合定义；不通过就抛出一个清晰的、详细的错误，告诉你哪里出了问题。
2. **类型转换（Parsing & Serialization）**：即使接收到的是"字符串形式的数字"（如 `"123"`），只要字段定义为 `int`，就会自动转成 `123`；反过来也能轻松把模型实例转成字典或 JSON 字符串。
3. **设置管理（Settings Management）**：很适合管理应用配置（例如从环境变量或 `.env` 文件读取），能自动转换类型并提供默认值。
4. **与 IDE 完美配合**：因为基于标准类型注解，PyCharm / VSCode 等编辑器能提供出色的自动补全和类型检查支持。
5. **轻量且高性能**：核心逻辑用 Rust（`pydantic-core`）实现，速度非常快。
6. **生态系统的关键角色**：它是 FastAPI 等众多顶级工具的核心依赖——FastAPI 就是用 Pydantic 模型自动处理请求/响应的验证、序列化与文档生成；这也是 LangChain 等框架做结构化输出时默认选它的原因。

---

## 定义模型：从最简单的例子开始

Pydantic 的核心思路是：**你用类型注解定义数据的"形状"，验证、类型转换、序列化全部自动完成**。先从一个最简单的模型入门：

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    name: str
    age: int = Field(gt=0)       # 年龄必须 > 0
    hobbies: list[str] = []      # 默认空列表

# 直接传字典，Pydantic 自动解析 + 验证
user = User(**{"name": "Alice", "age": 30})
print(user.name)             # Alice
print(user.model_dump())     # {'name': 'Alice', 'age': 30, 'hobbies': []}
```

两个要点：

1. **类型转换（coercion）**：哪怕传进来的是字符串 `"30"`，只要字段声明为 `int`，Pydantic 也会尝试转成 `30`。对"不太守规矩"的 LLM 输出尤其有用。
2. **失败即清晰报错**：传 `age: -5` 会抛出 `ValidationError`，明确告诉你 `age` 必须大于 0——而不是某个莫名的 `AttributeError`。

掌握了这套基本套路，接下来进入本文的主线场景：**用 Pydantic 为"让 LLM 把业务问题翻译成优化模型"设计输出结构**。

---

## Field：约束与描述

`Field` 是类型注解之外、控制字段行为的主要工具。

**数值约束**（常用参数）：

| 参数 | 含义 | 示例 |
|------|------|------|
| `gt` / `ge` | 大于 / 大于等于 | `ge=0`（非负） |
| `lt` / `le` | 小于 / 小于等于 | `le=1`（不超过 1） |
| `multiple_of` | 倍数 | `multiple_of=5` |

**description——最容易被忽略、却对 LLM 最重要的参数**：

```python
class ObjectiveFunction(BaseModel):
    direction: str = Field(description="优化方向，只能是 'min' 或 'max'")
    expression: str = Field(description="目标函数的数学表达式，如 \\sum_{i} c_i x_i")
    description: str = ""
```

对普通程序，`description` 只是文档；但对 LLM 结构化输出，**它是提示词的一部分**。`model_json_schema()` 会把 description 原样塞进 schema 发给模型，模型就靠它理解"这个格子到底要填什么"。**写好 description，是提升输出质量投入产出比最高的一步。**

---

## 为 LLM 写模型的四个进阶技巧

真实的结构化输出几乎不会像 `DecisionVariable` 这样扁平。下面用优化模型这个场景，逐一演示四个关键技巧。

### 1. 嵌套模型：表达复杂结构

一个优化模型由"多个决策变量 + 一个目标函数 + 多条约束"组成，而且每个组件又是结构化的。先把决策变量、约束各拆成独立模型，再嵌套进上层：

```python
class DecisionVariable(BaseModel):
    symbol: str              # 符号，如 x
    indices: list[str] = []  # 下标，如 ["i", "j"]
    type: str                # "Continuous" | "Integer" | "Binary"
    description: str = ""

class Constraint(BaseModel):
    name: str                # 约束名（须唯一）
    expression: str
    description: str = ""

class ModelComponents(BaseModel):
    decision_variables: list[DecisionVariable]
    objective_function: ObjectiveFunction   # 上一节定义的 ObjectiveFunction
    constraints: list[Constraint]
```

`list[DecisionVariable]` 让你表达"任意多条、每条结构一致"的输出，这是扁平字段做不到的。嵌套可以继续往下套，复杂度不受限。

### 2. Enum：锁定枚举值

上面的 `type` 和 `direction` 是 `str` + 注释，靠"模型自觉"。但分类值最好是封闭集合，用 `Enum` 才严谨：

```python
from enum import Enum

class VarType(str, Enum):
    CONTINUOUS = "Continuous"
    INTEGER = "Integer"
    BINARY = "Binary"

class Direction(str, Enum):
    MIN = "min"
    MAX = "max"

class DecisionVariable(BaseModel):
    type: VarType            # 只能三选一

class ObjectiveFunction(BaseModel):
    direction: Direction     # 只能 min / max
```

继承 `(str, Enum)` 是为了自然序列化成 JSON 字符串。换上 Enum 后，配合约束解码，模型**不可能**吐出 `"linear"`、`"最大化"` 这些 schema 之外的值——它只能在枚举里选。

> 顺带一个收益：真实代码原本用 `str` + 注释，再用验证器手动判断 `direction not in {"min","max"}`。改用 Enum 后，这层手动校验就可以删掉了。

### 3. 验证器（after）：类型之外的语义校验

类型和约束只能查"格式"，但 LLM 输出常有"格式对、逻辑错"的问题——比如两条约束重名、变量下标和维度对不上。Pydantic v2 用 `@model_validator(mode="after")` 做跨字段语义校验：

```python
from pydantic import model_validator

class ModelComponents(BaseModel):
    decision_variables: list[DecisionVariable]
    objective_function: ObjectiveFunction
    constraints: list[Constraint]

    @model_validator(mode="after")
    def validate_consistency(self) -> "ModelComponents":
        # 约束名必须唯一——抓 LLM 的低级重复
        seen: set[str] = set()
        for c in self.constraints:
            if c.name in seen:
                raise ValueError(f"Duplicate constraint name: {c.name}")
            seen.add(c.name)
        return self
```

这种校验正是 Instructor "校验失败 → 把错误拼回 prompt → 重试"循环的**驱动力**：验证器写得越贴合业务，越能把不靠谱的 LLM 输出挡在门外。（单字段校验用 `@field_validator`，跨字段用 `@model_validator`。）

### 4. 验证器（before）：容忍 LLM 的"脏"输出

这是对 LLM 场景**最实用、却最少被提及**的技巧。即便用了结构化输出，模型仍会犯一些"格式性"错误：把本该是对象的字段吐成 JSON 字符串、把列表吐成嵌套列表 `[[...]]`、在表达式里写 LaTeX 反斜杠把 JSON 弄崩……与其一次次重试，不如在 Pydantic 解析**之前**先清洗一遍：

```python
import json

class ModelComponents(BaseModel):
    decision_variables: list[DecisionVariable]
    objective_function: ObjectiveFunction
    constraints: list[Constraint]

    @model_validator(mode="before")
    @classmethod
    def repair_llm_output(cls, data):
        if not isinstance(data, dict):
            return data
        # 字段本该是对象，LLM 却吐成了 JSON 字符串 → 解开它
        obj = data.get("objective_function")
        if isinstance(obj, str):
            data["objective_function"] = json.loads(obj)
        # 偶尔吐成嵌套列表：[["a","b"]] -> ["a","b"]
        cons = data.get("constraints")
        if isinstance(cons, list):
            data["constraints"] = [
                x for sub in cons for x in (sub if isinstance(sub, list) else [sub])
            ]
        return data
```

`mode="before"` 的验证器在 Pydantic 做类型校验**之前**拿到原始字典，所以能任意改写数据。它和 `mode="after"` 搭配，刚好覆盖两类问题：**before 修"脏格式"，after 查"错逻辑"**。

> 一个真实的坑：LLM 在表达式里写 `\sum`、`\delta`、`\text{}`，这些反斜杠在 JSON 字符串里是非法转义，会直接让 `json.loads` 崩掉。处理办法是先把不是合法 JSON 转义的反斜杠翻倍——这通常也是放在 `before` 验证器或前置清洗函数里做。

---

## 延伸：表达式该用什么表示法？

上面是"出了问题怎么修"，但其实换个表示法就能从源头绕开。表达式字段让 LLM 输出成什么样，本身就是一个值得权衡的设计决策。用同一个表达式——「对所有 i∈N 累加 c_i × x_i」——对比四种常见表示法：

**LaTeX**：`` `\sum_{i \in N} c_i x_i` ``
渲染后可读性最高，但前面那个反斜杠坑只是冰山一角——`\`、`{}`、`^`、`_` 在 JSON 字符串里**全都要双重转义**（`\sum` 得写成 `\\sum`）。这种"转义爆炸"带来两类故障： 漏一个反斜杠或括号，整段 JSON 就解析失败； 每个转义字符都额外占 token，推理成本上升。

**AMPL-style**：`` `sum {i in N} c[i] * x[i]` ``
对运筹学者最直观，但 `{}` 和 JSON 的定界符冲突，需要小心转义；更要命的是它只是一段扁平字符串，**无法自动校验符号一致性**——程序没法可靠地从中提取引用的变量、再去核对它们是否都声明在决策变量集合里。

**Structured JSON**：
```json
{"op": "sum", "over": "i", "set": "N",
 "body": {"op": "*",
          "left":  {"op": "ref", "name": "c", "idx": ["i"]},
          "right": {"op": "ref", "name": "x", "idx": ["i"]}}}
```
把表达式拆成算子-操作数树，**可校验性最强**——能遍历检查每个引用的变量是否合法。代价是 token 开销最高（重复的键名 + 深层嵌套），还显著抬高了结构化输出的 schema 复杂度，反而**增加 Pydantic 验证失败的概率**。

**S-expression**：`` `(sum i N (* c x))` ``
前缀括号式，同时避开了上面三者的短处。它只用 `()` 和符号，**不引入任何 JSON 转义字符**（`\ {} ^ _` 一个都没有），不会触发解析错误；又是结构化的，可遍历做符号一致性校验；token 开销也远低于 JSON 树。可读性虽不如 LaTeX，但作为"给机器消费"的中间表示，是各项权衡下的甜点。

| 表示法 | 可读性 | JSON 友好（无需转义） | 可结构化校验 | token 开销 |
|--------|:------:|:--------------------:|:------------:|:----------:|
| LaTeX | 高（渲染后） | ✗ `\ {} ^ _` 全要转义 | ✗ 扁平字符串 | 高 |
| AMPL-style | 高（OR 熟悉） | ✗ `{}` 与定界符冲突 | ✗ 扁平字符串 | 中 |
| Structured JSON | 低 | ✓ | ✓ 最强 | 最高 |
| **S-expression** | 中 | ✓ 只用 `()` 和符号 | ✓ | **低** |

一句话：**想要"JSON 安全 + 可校验 + 省 token"，s-expression 是更合适的默认选择**；LaTeX 适合最终给人渲染看的场景，但放进结构化输出里就得时刻准备着修复它的转义。本文示例为可读性沿用 LaTeX，实际工程中换成 s-expression 会省心很多。

---

## 完整示例：把模型接进 LLM

把上面的片段拼成一个完整的智能体输出模型，再用 LangChain 接入：

```python
from typing import Optional
from pydantic import BaseModel, Field, model_validator

class ModelExpertOutput(BaseModel):
    """ModelExpert 智能体的结构化输出：一个完整的优化模型定义。"""
    model_components: ModelComponents
    assumptions_made: list[str] = []        # LLM 自行补充的假设
    ambiguous_points: Optional[list[str]] = None  # 它标记的歧义点
```

接入只需关键一行——把 Pydantic 模型作为结构化输出的目标：

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="deepseek-chat", temperature=0)
prompt = ChatPromptTemplate.from_messages([
    ("system", "{role}"),
    ("human", "{task}"),
])

# 关键一行：with_structured_output 让 LLM 直接产出 ModelExpertOutput
chain = prompt | llm.with_structured_output(ModelExpertOutput, method="json_mode")

result: ModelExpertOutput = chain.invoke({
    "role": "你是一名运筹优化建模专家，负责把业务问题描述翻译成 MILP 模型。",
    "task": "请为以下运输问题建立模型：……（问题描述）",
})

# result 已经是验证过的 ModelExpertOutput 实例
print(result.model_components.objective_function.direction)   # Direction.MIN
print(len(result.model_components.decision_variables), "个决策变量")
```

这里有两个真实工程中很值钱的细节：

- **结构化输出让智能体之间可以"结构化交接"**：下一个智能体的 prompt 里可以直接塞上一个智能体的 `model_dump_json(indent=2)`，不再是难以解析的自然语言。
- **约束方法越弱，Pydantic 越要强**：这里用的是 `method="json_mode"`（旧文里可靠性最弱的一档，但跨 Provider 兼容性最好）。正因为约束弱、模型可能输出"脏 JSON"，前面那套 `before` 清洗 + `after` 校验就成了不可或缺的安全网。

---

## Pydantic v1 vs v2 小贴士

本文用的都是 **v2** 语法。但网上很多教程和存量代码还是 v1，几个最容易踩的差异记一下：

| 用途 | v1（旧） | v2（本文） |
|------|---------|-----------|
| 转字典 | `.dict()` | `.model_dump()` |
| 转 JSON | `.json()` | `.model_dump_json()` |
| 不可变更新 | `.copy(update=...)` | `.model_copy(update=...)` |
| 跨字段验证器 | `@root_validator` | `@model_validator` |
| 从 JSON 解析 | `.parse_raw()` | `.model_validate_json()` |

v2 核心用 Rust（`pydantic-core`）重写，速度比 v1 快一个量级，新项目直接用 v2 即可。

---

## 小结

结构化输出的四层技术再花哨，落点都是那个 Pydantic 模型。把它写好，输出就稳了一大半：

1. **`description` 是提示词**——模型靠它理解每个格子填什么，写清楚。
2. **嵌套 + `list`** 表达复杂结构，**Enum** 锁定分类输出。
3. **`@model_validator(mode="after")`** 做语义校验，抓 LLM 的逻辑错误并驱动重试。
4. **`@model_validator(mode="before")`** 清洗 LLM 的脏格式——before 修格式、after 查逻辑，两者搭配。
5. `model_json_schema()` 是连接 Pydantic 与 LLM 的那座桥。

> 想了解这些模型最终如何被四种技术"喂"给 LLM，回到[《LLM 结构化输出技术：从原理到实践》](/posts/structured-output-techniques/)。

## 参考资源

- [Pydantic v2 官方文档](https://docs.pydantic.dev/latest/)
- [LangChain `with_structured_output`](https://python.langchain.com/docs/how_to/structured_output/)
- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
