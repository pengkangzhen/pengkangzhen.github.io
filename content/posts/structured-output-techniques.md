---
title: "LLM 结构化输出技术：从原理到实践"
date: 2026-06-06
categories: ["技术"]
tags: ["LLM", "结构化输出", "OpenAI", "Prompt 工程"]
summary: "结合'餐厅点菜'比喻，系统梳理 4 层结构化输出技术：从 Prompt 工程到约束解码，从 API 层到后处理中间件。"
draft: false
---

## 核心比喻

你开了一家餐厅，厨师是 LLM，需要厨师**按固定格式出菜单**。
不同技术 = 不同约束厨师的方式，从"靠自觉"到"逐字监控"。

---

## 第一层：API 层面（厨师用不同的下单工具）

### 1.1 Prompt 工程（口头吩咐）

```
你对厨师说："请用 JSON 格式写下订单，包含菜名和份量"
```

- 没有任何工具，全靠厨师自觉
- 厨师可能写成 `{"菜名": "土豆丝"}`（字段名不对）
- 甚至可能写成 `"土豆丝，中份"`（根本不是 JSON）
- **最通用，但最不可靠**

### 1.2 JSON Mode（要求用标准格式纸）

```python
client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[...],
    response_format={"type": "json_object"},  # 只要求输出合法 JSON
)
```

- 给厨师一张标准格式纸，要求"必须写 JSON"
- **只保证输出是合法 JSON，不保证字段名/结构正确**
- 厨师可能写成 `{"dish_name": "土豆丝"}`（你期望的是 `dish`）
- 对应 LangChain：`llm.with_structured_output(Schema, method="json_mode")`

### 1.3 Function Calling（用点菜机）

```python
client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[...],
    tools=[{
        "type": "function",
        "function": {
            "name": "place_order",
            "parameters": Order.model_json_schema(),  # schema 作为点菜机的选项
        }
    }],
    tool_choice={"type": "function", "function": {"name": "place_order"}},
)
```

- 给厨师一台**点菜机**，机器里有固定选项
- 厨师按按钮选择，机器自动输出标准格式
- **API 层面强制 schema 约束**，字段名和类型一定正确
- 对应 LangChain：`llm.with_structured_output(Schema, method="function_calling")`

### 1.4 Structured Outputs（智能点菜机 + 实时校验）

```python
client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[...],
    response_format=Order,  # 直接传 Pydantic 模型
)
```

- 最新最强的点菜机，厨师每选一个选项，机器**实时校验**是否符合规则
- 底层使用**约束解码（Constrained Decoding）**——每生成一个 token 都保证合法
- **最严格**：不可能输出不符合 schema 的内容
- 对应 LangChain：`llm.with_structured_output(Schema, method="json_schema")`

> **注意**：不是所有模型/Provider 都支持全部方式。例如 Qwen 思考模式只支持 Function Calling，不支持 JSON Mode 与思考模式的组合。

---

## 第二层：底层引擎（点菜机是怎么工作的）

### 约束解码（Constrained Decoding）

核心思想：模型预测下一个 token 时，把不合法的 token 概率直接设为 0。

```
模型输出到此时：{"dish": "酸辣土豆丝", "size": "
                                       ↓
模型预测下一个 token 的概率分布：
  "大份"    → 0.3   ✅ 合法
  "中份"    → 0.5   ✅ 合法
  "小份"    → 0.2   ✅ 合法
  "超级辣"  → 0.1   ❌ schema 中 size 是枚举，不包含此值

约束解码：直接把 "超级辣" 概率设为 0
                                       ↓
最终只会从 ["大份", "中份", "小份"] 中选择
```

这个技术的开源实现：

| 工具 | 约束方式 | 适用场景 |
|------|---------|---------|
| **llama.cpp** | Grammar（BNF 文法） | 本地部署开源模型 |
| **Outlines** | JSON Schema / 正则 → 有限状态机 | 学术研究、灵活定制 |
| **vLLM** | Guided Decoding | 生产级推理，支持 JSON/Regex/Grammar |
| **LMQL** | SQL 风格声明式语法 | 实验性探索 |

---

## 第三层：后处理 / 中间件（厨师写完后的质检）

模型输出后做校验，不通过就打回去重写。

```
模型输出 → 解析 → Pydantic 校验 →
  ├─ 通过 → 返回结构化对象 ✅
  └─ 失败 → 把错误信息拼回 prompt → 让模型重试 🔄
```

### 主要工具

| 工具 | 特点 |
|------|------|
| **Instructor** | Pydantic 校验 + 自动重试，装饰器风格 |
| **Guardrails AI** | 定义 schema，校验不通过自动重生成 |
| **Marvin** | 轻量级，函数装饰器风格 |
| **LangChain `with_structured_output`** | 统一接口，封装上述所有 method |

---

## 第四层：Prompt 层面（不依赖任何 API 特性）

最原始但最通用的方法——纯靠提示词描述输出格式：

```
请严格按照以下 JSON 格式输出，不要输出其他内容：
{
  "dish": "菜名（字符串）",
  "size": "大份/中份/小份（三选一）"
}
```

兼容性最好（任何 LLM 都能用），但最不可靠（模型可能不听话）。

---

## 可靠性对比总览

```
可靠性：弱 ──────────────────────────────────────────────→ 强

Prompt 工程     JSON Mode      Function Calling     Structured Outputs
(口头吩咐)      (标准格式纸)    (点菜机)              (智能点菜机+实时校验)
                                                              ↑
                                                     Constrained Decoding
                                                     (llama.cpp/Outlines/vLLM)

                                     后处理兜底
                                     (Instructor/Guardrails)
```

## 实践建议

1. **优先用约束最强的方法**：`json_schema` > `function_calling` > `json_mode` > prompt 工程
2. **不同 Provider 兼容性不同**：不是所有模型都支持所有方式，需要逐个测试
3. **后处理作为最后防线**：即使用了强约束，也建议加 Pydantic 校验 + 修复
4. **思考模式与结构化输出有冲突风险**：思考 token 可能挤占输出预算，且不同 Provider 的兼容性差异大

## 参考资源

- [OpenAI Structured Outputs 官方文档](https://platform.openai.com/docs/guides/structured-outputs)
- [OpenAI Function Calling 官方文档](https://platform.openai.com/docs/guides/function-calling)
- [Qwen Function Calling 文档](https://help.aliyun.com/zh/model-studio/qwen-function-calling)
- [Outlines 论文与 GitHub](https://github.com/dottxt-ai/outlines)
