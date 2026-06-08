---
title: "Claude Code 并行模式：从 Worktree 到 Agent Teams"
date: 2026-06-08
categories: ["技术"]
tags: ["Claude Code", "AI 编程", "Git Worktree", "Agent Teams", "并行开发"]
summary: "Claude Code 提供了多层级的并行能力，从基础的 Git Worktree 隔离到子代理自动调度，再到 Agent Teams 多代理协作。本文系统梳理这些并行模式，帮你告别排队等任务。"
draft: false
---

## 背景

用过 Claude Code 的开发者大概都遇到过这样的场景：一个终端在跑测试，你只能干等着；一个模块在重构，另一个模块的 bug 修复只能排队。串行工作流浪费了大量等待时间。

Claude Code 提供了多层级的并行能力来解决这个问题。本文从基础到进阶，依次介绍 **Git Worktree 隔离并行**、**子代理（Subagent）并行** 和 **Agent Teams 多代理协作**，帮你根据场景选择最合适的并行模式。

## 第一层：Git Worktree 隔离并行

### 最朴素的方式：多终端多实例

最直观的做法是开多个终端窗口，每个终端跑一个 `claude` 会话。但问题很快出现：多个会话操作同一个工作目录，文件互相覆盖，Git 分支混乱，冲突不断。

核心矛盾在于**没有隔离**。

### Git Worktree：隔离的并行

Git Worktree 让同一个仓库拥有多个独立的工作目录，每个目录对应一个独立分支。Claude Code 原生支持这个能力，只需一个参数：

```bash
# 终端 1：创建 feature-auth 分支的 worktree 并启动 Claude
claude --worktree feature-auth

# 终端 2：创建 bugfix-123 分支的 worktree 并启动 Claude
claude --worktree bugfix-123
```

也可以简写为 `claude -w`：

```bash
claude -w feature-auth
```

这样做的好处：

- **自动创建分支**：Worktree 会基于默认分支（`origin/HEAD`）自动创建一个新分支
- **文件完全隔离**：两个 Claude 会话可以同时修改同名文件，互不影响
- **合并时再解决冲突**：每个 Worktree 是独立的工作目录 + 独立的分支，最终通过 `git merge` 合并，只在合并时解决冲突

默认情况下，Worktree 会创建在仓库根目录下的 `.claude/worktrees/<name>/` 中。

### 实际工作流

假设你有两个独立任务：重构认证模块和修复支付 bug。

```bash
# 终端 1
claude -w refactor-auth
# → 在新 worktree 中启动 Claude，开始重构

# 终端 2
claude -w fix-payment-bug
# → 在另一个 worktree 中启动 Claude，修复 bug

# 两个 Claude 同时工作，互不干扰
# 完成后各自提交，然后合并回主分支
```

### 配置技巧

**携带环境文件**：Worktree 是全新的检出，未跟踪的文件（如 `.env`）不会自动复制。在项目根目录创建 `.worktreeinclude` 文件来指定需要复制的内容（语法与 `.gitignore` 相同）：

```
.env
.env.local
config/secrets.json
```

**选择基础分支**：默认从远程默认分支创建。如果想从当前 `HEAD`（包含未推送的提交）创建，在设置中配置：

```json
{
  "worktree": {
    "baseRef": "head"
  }
}
```

**从 PR 创建**：可以直接基于某个 Pull Request 创建 Worktree 来 review 或修改：

```bash
claude --worktree "#1234"
```

## 第二层：子代理（Subagent）并行

Worktree 解决了文件隔离的问题，但你仍然需要手动开多个终端、管理多个会话。**子代理**（Subagent）更进一步——不需要打开新终端，所有并行任务都在当前会话内完成。

子代理在独立的上下文中执行辅助任务，完成后将结果摘要返回主会话。你不需要关心它的调度过程。

### 子代理的几种模式

Claude Code 会根据任务类型自动选择合适的子代理模式，你也可以显式指定：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **Explore** | 文件/代码探索，搜索项目结构 | 查找 API 端点、理解代码架构 |
| **Plan** | 制定实施计划 | 复杂功能的拆分与规划 |
| **General** | 通用多步骤任务执行 | 编写代码、运行命令、修改文件 |

例如，你可以在会话中说「搜索项目中所有的 API 端点」，Claude 会自动启动一个 Explore 类型的子代理来执行。

### 前台 vs 后台运行

子代理有两种运行方式：

**前台运行（默认）**：
- 阻塞主会话，子代理完成后才能继续
- 所有权限操作需要你手动审批
- 适合需要你监督的任务

**后台运行**：
- 子代理在后台执行，完成后通知你结果
- 不阻塞主会话，你可以继续做其他事
- 启动时预审批权限，之后自动执行
- 安全建议：后台任务一般只给**只读权限**

后台运行尤其适合这些场景：
- 搜索和汇总大量代码
- 分析项目依赖关系
- 读取日志文件

### 子代理 + Worktree 组合

当子代理需要编辑文件时，可以让它在独立的 Worktree 中运行，避免与主会话冲突：

- **临时方式**：要求 Claude「在 worktree 中运行子代理」
- **永久方式**：在自定义子代理的 frontmatter 中添加 `isolation: worktree`

每个子代理获得一个临时 Worktree，子代理完成且没有更改时会自动删除。

## 三种子代理编排模式

掌握了基本的子代理用法后，以下是三种常见的编排模式：

### 1. 并行探索 + 汇总

让多个子代理同时从不同角度探索代码库，最后汇总结果。

```
你：帮我分析这个项目的架构，分别从路由、数据层、测试覆盖三个角度探索
```

Claude 会启动多个 Explore 子代理并行工作，各自探索不同维度，最终在主会话中汇总成一份架构分析报告。

### 2. 链式闭环：发现 → 修复 → 验证

子代理之间形成自动化链路：一个负责发现问题，另一个负责修复，第三个负责验证。

```
你：跑一遍测试套件，失败的测试自动修复，然后重新验证
```

这适合处理重复性的修复工作，Claude 会自动闭环处理。

### 3. 隔离测试运行

让子代理跑测试套件，大量输出不会污染主会话的上下文，只报告失败和摘要结果。

```
你：在后台跑完整测试套件，只报告失败用例和总结
```

这种方式特别适合测试输出量大的项目，避免上下文浪费。

## 第三层：Agent Teams 多代理协作

子代理适合「主代理派活、干完汇报」的场景。但如果任务更复杂——成员之间需要讨论、互相挑战、协调接口——就需要 Agent Teams。

**Agent Teams 的核心区别**：子代理是星型拓扑（所有结果汇总到主代理），而 Agent Teams 是网状拓扑——团队成员之间可以直接通信、共享任务列表、自主认领任务。

### 架构：四种核心组件

一个 Agent Team 由以下部分构成：

| 组件 | 角色 |
|------|------|
| **Team Lead（团队负责人）** | 主会话，负责任务拆解、成员管理、结果综合 |
| **Teammates（团队成员）** | 独立的 Claude Code 实例，各自拥有独立的上下文窗口 |
| **Task List（共享任务板）** | 类似 Jira/Trello，支持依赖关系和自动解锁 |
| **Messaging（消息系统）** | 成员之间直接通信，不一定要经过 Lead 中转 |

一个类比：子代理像「经理给几个助理派活，助理只向经理汇报」；Agent Teams 像「项目经理带一支真正的开发团队，成员之间可以互相讨论和协作」。

### 启用 Agent Teams

Agent Teams 目前是**实验性功能**，默认关闭。启用方式：

在 `~/.claude/settings.json` 中添加：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

修改后需要重启 Claude Code。

### 创建第一个团队

启用后，直接用自然语言描述任务和团队结构即可：

```
我要重构这个项目的认证模块。创建一个团队：
- 一个成员负责分析现有代码的安全问题
- 一个成员负责设计新的认证方案
- 一个成员负责编写测试用例
让它们先各自调研，然后讨论确定最终方案。
```

Claude 会自动创建团队、分配任务、协调工作。

### 两种显示模式

| 模式 | 说明 | 要求 |
|------|------|------|
| **In-process** | 所有成员在主终端内运行，用 `Shift+Down` 切换 | 无额外要求 |
| **Split panes** | 每个成员一个独立面板，同时可见 | 需要 tmux 或 iTerm2 |

默认是 `auto`：如果在 tmux 会话中则自动分屏，否则 in-process。可以在 settings.json 中配置：

```json
{
  "teammateMode": "tmux"
}
```

或通过命令行指定：

```bash
claude --teammate-mode in-process
```

分屏效果示意：

```
┌─────────────────┬─────────────────┬─────────────────┐
│  Teammate 1     │  Teammate 2     │  Teammate 3     │
│  分析安全问题... │  设计认证方案... │  编写测试用例... │
│                 │                 │                 │
│  "@Teammate 2,  │                 │                 │
│   JWT 的过期    │                 │                 │
│   策略你定了吗？"│                 │                 │
└─────────────────┴─────────────────┴─────────────────┘
```

### 指定成员数量和模型

Claude 会根据任务自动决定成员数量，你也可以显式指定：

```
创建一个 4 人团队来并行重构这些模块。每个成员使用 Sonnet 模型。
```

推荐配置：**Team Lead 用 Opus**（需要强大推理来协调），**Teammates 用 Sonnet**（性价比高）。

### 任务管理与成员交互

**共享任务板**：Lead 创建任务，成员可以认领（Claim）或被分配。任务支持依赖关系——只有前置任务完成后，后续任务才会解锁。

**直接交互**：你可以和任何成员直接对话，不需要经过 Lead：
- In-process 模式：`Shift+Down` 切换成员，直接输入消息
- 分屏模式：点击对应面板直接操作

**质量门控**：可以要求成员在实施前先提交计划，经 Lead 审批后再动手：

```
创建一个架构师成员来重构认证模块。
要求它在修改代码前先提交计划，审批通过后才能开始实施。
```

### 子代理 vs Agent Teams：如何选择

| 对比维度 | 子代理（Subagent） | Agent Teams |
|----------|-------------------|-------------|
| **通信方式** | 只向主代理汇报 | 成员之间可直接通信 |
| **协调机制** | 主代理统一管理 | 共享任务板 + 自主认领 |
| **上下文** | 独立上下文，结果返回主会话 | 每个成员完全独立的上下文 |
| **Token 成本** | 较低 | 较高（每个成员独立消耗） |
| **适合场景** | 快速、明确的单一任务 | 需要讨论、协作的复杂任务 |

简单记：
- **子代理**：任务分发工具，「干完活汇报」
- **Agent Teams**：真正的协作团队，「成员之间可以讨论、辩论、协同」

### Agent Teams 最佳实践

**1. 合同优先**：在成员并行工作前，先定义好接口契约（函数签名、数据格式、API 规范），避免各做各的对不上。

**2. 控制团队规模**：建议 3-5 人起步。每个成员负责 5-6 个中等粒度的任务（15-30 分钟可独立完成的单元）。

**3. 避免文件冲突**：让不同成员负责不同的文件/目录。如果必须改同一文件，设计串行修改阶段。

**4. 提供充分上下文**：成员启动时不知道之前的对话历史，在 prompt 中交代清楚项目背景、技术栈、文件结构。

**5. 先研究再实现**：先让成员做调研和方案设计，讨论确定后再动手写代码，避免写到一半发现方案行不通。

### 实战场景：竞争性调试

这是 Agent Teams 最亮眼的使用模式之一——当 bug 的根因不明确时，让多个成员从不同假设出发并行调查：

```
用户报告应用在发送一条消息后就退出了，而不是保持连接。
创建 5 个成员来调查不同的假设。让它们互相质疑对方的结论，
像科学辩论一样。把最终的共识更新到调查文档中。
```

这种「竞争性假设」模式的优势：单个 AI 容易锚定在第一个看起来合理的解释上，而多个独立调查者互相挑战，能更快收敛到真正的根因。

## 第四层：Headless 模式与脚本编排

Headless 模式（`claude -p`）本身**不是并行模式**，而是**非交互执行模式**。但它是批量并行的基础设施——你可以用 Shell 脚本编排多个 Headless 会话实现自定义的批量并行。

### 什么是 Headless 模式

```bash
# 非交互式执行，结果直接输出到 stdout
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"

# 管道输入输出
cat build-error.txt | claude -p 'explain this build error' > output.txt

# 获取结构化 JSON 输出
claude -p "Summarize this project" --output-format json
```

核心特征：
- **无交互界面**——一条命令出结果，不需要终端对话
- **可脚本化**——适合集成到 CI/CD、Shell 脚本、Python/TypeScript 程序中
- **单次执行**——每次 `claude -p` 跑一个任务

### 用脚本编排批量并行

视频里说的"Headless 批量并行"，指的是用 Shell 脚本 + `claude -p` + Worktree 组合实现：

```bash
# 在 Shell 脚本中并行启动多个 Headless 会话
claude -w task-1 -p "实现用户登录功能" --allowedTools "Read,Edit,Bash" &
claude -w task-2 -p "实现支付接口" --allowedTools "Read,Edit,Bash" &
claude -w task-3 -p "编写单元测试" --allowedTools "Read,Edit,Bash" &

# 等待所有任务完成
wait

echo "All tasks completed"
```

每个 `claude -p` 在自己的 Worktree 中独立运行，Shell 的 `&` 让它们并行执行。这种方式的灵活性最高——你可以完全控制任务分配、错误处理和结果收集。

### 典型应用场景

**CI/CD 集成**：在 GitHub Actions 中自动审查 PR、生成提交信息。

```yaml
# GitHub Actions 示例
- name: AI Code Review
  run: |
    git diff main | claude -p \
      "You are a code reviewer. Report any bugs or issues." \
      --output-format json > review.json
```

**仓库级批量操作**：对每个子目录执行相同任务。

```bash
# 对每个微服务目录并行生成 API 文档
for dir in services/*/; do
  claude -w "docs-$(basename $dir)" \
    -p "Generate API documentation for this service" \
    --allowedTools "Read" &
done
wait
```

**Bare 模式加速启动**：加上 `--bare` 跳过 hooks、skills、MCP 等自动发现，加快 CI 环境的启动速度：

```bash
claude --bare -p "Summarize this file" --allowedTools "Read"
```

### Headless 的定位

Headless 是**底层能力**而非**并行协调器**。它不管理任务依赖、不协调通信——这些需要你在脚本中自行处理。当你需要比内置并行模式更精细的控制时（如自定义调度逻辑、集成到现有流水线），Headless 就是你的工具。

## 第五层：`/batch` Skill

`/batch` 是一个内置 Skill，专门处理**仓库级别的机械重构**。它的工作方式是：把一个大型变更拆成 5-30 个 Worktree 隔离的子代理，每个自动开一个 Pull Request。

### 使用方式

```bash
/batch 把所有 JS 文件迁移到 TypeScript
```

```bash
/batch 将项目中所有 class 组件重写为函数组件
```

### 它做了什么

1. **分析变更范围**：Claude 评估任务涉及的文件数量和依赖关系
2. **拆分为子任务**：将大任务拆成 5-30 个独立的子任务
3. **并行执行**：每个子任务在独立的 Worktree 中由一个子代理执行
4. **自动开 PR**：每个子任务完成后自动创建 Pull Request

### 适用场景

- 仓库级别的机械重构（批量重命名、统一代码风格）
- API 迁移（旧接口 → 新接口的批量替换）
- 依赖升级（批量修改 import 路径、更新配置文件）
- 代码规范化（统一错误处理模式、统一日志格式）

### 与其他方式的关系

`/batch` 本质上是子代理 + Worktree 的**打包使用**，不是一个独立的协调风格。如果你的任务不是"同一件事重复很多遍"，就不适合用 `/batch`。

## 第六层：Dynamic Workflows 动态工作流

当前面的方式都不够用——任务大到子代理的逐轮调度无法协调，或者需要多轮交叉验证——Dynamic Workflows 登场。

Dynamic Workflows 是 Claude Code 最新的大规模并行方案（Research Preview）。它的核心思想是：**让计划住在脚本里，而不是 Claude 的上下文里**。

### 与其他方式的本质区别

|  | 子代理 | Agent Teams | Dynamic Workflows |
|---|--------|-------------|-------------------|
| **计划在哪里** | Claude 的上下文窗口 | Lead 的上下文窗口 | JavaScript 脚本文件 |
| **谁决定下一步** | Claude，逐轮判断 | Lead，逐轮分配 | 脚本，按代码逻辑执行 |
| **中间结果存在哪** | Claude 的上下文窗口 | 共享任务板 | 脚本变量 |
| **可重复性** | 低（每次重新规划） | 中（团队定义可复用） | 高（脚本是可审查、可重跑的代码） |
| **规模** | 几个子代理 | 3-10 个成员 | 几十到上百个代理 |

### 工作原理

1. **你描述任务**：在 prompt 中包含 `ultracode` 关键词，或直接说"用 workflow 来做"
2. **Claude 写脚本**：自动生成一个 JavaScript 编排脚本
3. **后台运行**：运行时在后台执行脚本，你的会话保持可用
4. **结果汇总**：所有代理完成后，只把最终结果返回给你

```
ultracode: audit every API endpoint under src/routes/ for missing auth checks
```

Claude 会高亮 `ultracode` 关键词确认，然后编写编排脚本并运行。

### 关键能力：对抗性验证（Adversarial Review）

Dynamic Workflows 不只是"开更多代理"。它可以编排一种质量模式：让独立的代理互相审查对方的发现，过滤掉不可靠的结论。

典型流程：
1. 多个代理并行调研同一问题
2. 代理之间交叉验证（Agent A 审查 Agent B 的结论）
3. 投票表决每个结论的可信度
4. 只输出通过验证的结果

内置的 `/deep-research` 就是一个 Dynamic Workflow：

```bash
/deep-research Claude Code 的 Dynamic Workflows 和 Agent Teams 有什么区别？
```

它会：多角度搜索 → 获取源内容 → 交叉验证 → 投票过滤 → 输出带引用的报告。

### 管理运行

```bash
/workflows          # 查看运行中和已完成的 workflow
```

运行中可以暂停（`p`）、停止单个代理（`x`）、查看代理详情（`Enter`）。已完成的 workflow 可以保存为命令（`s`），以后通过 `/<name>` 直接调用。

### 约束

| 约束 | 值 | 原因 |
|------|----|----|
| 最大并发代理数 | 16 | 限制本地资源占用 |
| 单次运行最大代理数 | 1,000 | 防止无限循环 |
| 运行中用户输入 | 不支持 | 如需人工审批，分阶段运行 |

### 适用场景

- 500 个文件的仓库级迁移
- 需要多角度交叉验证的代码审计
- 从多个独立角度调研后权衡的架构决策
- 任何"单次子代理调度搞不定"的大规模任务

## 全景对比总结

| 方法 | 是否需要新终端 | 规模 | 计划在哪里 | 适合场景 |
|------|---------------|------|-----------|----------|
| 多终端多实例 | ✅ 需要 | 2-3 | 你自己 | 临时应急 |
| Git Worktree | ✅ 需要 | 2-5 | 你自己 | 独立大任务并行 |
| 子代理 | ❌ 不需要 | 几个 | Claude 上下文 | 辅助探索、计划制定 |
| Agent Teams | ❌ 不需要 | 3-10 | Lead 上下文 | 复杂协作、竞争性调试 |
| Headless + 脚本 | ❌ 不需要 | 自定义 | 你的脚本 | CI/CD、自定义调度 |
| `/batch` | ❌ 不需要 | 5-30 | Claude 自动拆分 | 仓库级机械重构 |
| Dynamic Workflows | ❌ 不需要 | 几十~上百 | JS 脚本 | 大规模审计、交叉验证 |

辅助工具（不是独立的并行方式）：

- **Worktree**：文件隔离，被上述多种方式组合使用
- **Agent View** `claude agents`：后台会话管理仪表盘，适合"派出去稍后看结果"

## 小结

Claude Code 的并行能力从基础到进阶，可以归纳为以下层级：

1. **Worktree**——文件级别的隔离，适合你手动管理的并行任务
2. **子代理**——任务级别的自动并行，"干完活汇报"
3. **Agent Teams**——团队级别的协作并行，成员间可以讨论和协调
4. **Headless + 脚本**——完全自定义的批量并行，适合 CI/CD 和高级编排
5. **`/batch`**——一键拆分大型机械重构，自动开 PR
6. **Dynamic Workflows**——脚本驱动的大规模并行，支持交叉验证

选择建议：
- **简单辅助任务** → 子代理
- **需要深度参与** → Worktree 开独立会话
- **复杂多模块协作** → Agent Teams
- **CI/CD 自动化** → Headless 模式
- **仓库级批量重构** → `/batch`
- **大规模审计/调研** → Dynamic Workflows

## 参考资料

- [并行运行代理 - Claude Code 官方文档](https://code.claude.com/docs/zh-CN/agents)
- [使用 worktrees 运行并行会话 - Claude Code 官方文档](https://code.claude.com/docs/zh-CN/worktrees)
- [Orchestrate teams of Claude Code sessions - Claude Code 官方文档](https://code.claude.com/docs/en/agent-teams)
- [Manage multiple agents with agent view - Claude Code 官方文档](https://code.claude.com/docs/en/agent-view)
- [Run Claude Code programmatically - Claude Code 官方文档](https://code.claude.com/docs/en/headless)
- [Orchestrate subagents at scale with dynamic workflows - Claude Code 官方文档](https://code.claude.com/docs/en/workflows)
