---
title: "Poetry 开发依赖管理与 mypy 类型检查实践"
date: 2026-04-29
categories: ["技术"]
tags: ["Python", "Poetry", "mypy", "工程实践"]
summary: "介绍如何用 Poetry 的 dependency groups 管理开发依赖，并通过 mypy 为项目添加静态类型检查。"
draft: false
---

在 Python 项目中，生产依赖和开发依赖往往混杂在一起，导致部署环境臃肿。同时，缺乏类型注解的代码在大型项目中维护成本很高。本文分享如何用 Poetry 分离管理这两类依赖，并用 mypy 建立类型安全机制。

## 用 Dependency Groups 分离开发依赖

Poetry 提供了 **dependency groups** 机制，可以将开发阶段才需要的工具与运行时依赖隔离开：

```toml
# pyproject.toml

[tool.poetry.dependencies]
python = "^3.10"
# 生产环境依赖...

[tool.poetry.group.dev.dependencies]
mypy = "^1.10"
sphinx = "^7.0"
sphinx-rtd-theme = "^2.0"
autodocsumm = "^0.5"
```

这样分类的好处：

- `poetry install` 只安装生产依赖（默认跳过 dev group）
- `poetry install --with dev` 才会安装开发工具
- 部署时不会带入 mypy、sphinx 等不必要的包

其中这几个开发工具的用途：

| 工具 | 用途 |
|------|------|
| mypy | 静态类型检查，在运行前发现类型错误 |
| sphinx | 从代码和注释生成项目文档 |
| sphinx-rtd-theme | ReadTheDocs 风格的文档主题 |
| autodocsumm | 自动生成 API 文档摘要 |

## 用 mypy 做静态类型检查

### 为什么需要类型注解

Python 是动态类型语言，但这不代表类型不重要。在一个多文件协作的项目中，缺少类型信息会导致：

- 调用方不知道该传什么参数
- 重构时无法确认改动的完整影响范围
- IDE 无法提供准确的自动补全

### 加上类型注解

改造前，函数签名没有类型信息：

```python
def forward_step(self, request):
    ...
```

改造后，参数和返回值都有明确的类型声明：

```python
from typing import Dict, Any

def forward_step(self, request: Dict[str, Any]) -> Dict[str, Any]:
    ...
```

运行 `mypy` 验证：

```bash
poetry run mypy src/
```

如果输出 `0 errors`，说明所有类型注解与实际使用一致。

### 带来的收益

- **更早发现问题**：类型错误在编写阶段就被捕获，而不是运行时崩溃
- **更好的 IDE 支持**：自动补全、跳转定义、重构提示都更准确
- **代码即文档**：函数签名本身就说明了参数和返回值的结构，不需要额外注释

## 小结

- 用 Poetry 的 `group.dev` 分离开发依赖，保持生产环境干净
- 给核心模块加上类型注解，用 mypy 守住类型安全
- 这两个实践投入不大，但对项目的长期可维护性有明显提升
