---
title: "Shell 常用命令速查"
date: 2026-04-29
categories: ["技术"]
tags: ["Shell", "Linux", "命令行"]
summary: "AI 辅助编码时经常遇到各种 Shell 命令，本文按功能分类汇总常用命令的用法。"
draft: false
---

## 写在前面

用 AI 辅助编码时（比如 Claude Code、Cursor），AI 经常会给出各种 Shell 命令：`export` 设置环境变量、`source` 激活环境、`grep` 搜索代码……如果不清楚这些命令的含义，就只能盲目复制粘贴。这篇笔记按功能分类梳理这些常见命令，让与 AI 的协作更高效。

## 文件与目录操作

### cd — 切换工作目录

```bash
cd ~/projects/vrp_solver   # 进入项目目录
cd ..                       # 返回上一级
cd -                        # 回到上一次所在的目录（高频操作）
cd ~                        # 回到主目录
pwd                         # 显示当前路径
```

运行脚本时，程序会从**当前目录**读取数据文件，用 `cd` 确保你在正确的位置。

### cat — 查看文件内容

快速显示文件的全部内容：

```bash
cat config.yaml            # 显示文件内容
cat -n config.yaml         # 带行号显示
cat a.txt b.txt > c.txt    # 合并多个文件
```

查看大文件时，`cat` 会一次性全部输出，建议改用 `less`（分页浏览）或 `head`/`tail`（只看头尾）：

```bash
less large.log             # 分页浏览，按 q 退出
head -20 large.log         # 只看前 20 行
tail -f app.log            # 实时追踪文件末尾（看日志常用）
```

## 文本搜索与处理

### grep — 文本搜索

`grep`（global regular expression print）在文件或文本中搜索匹配特定模式的行，相当于命令行里的 Ctrl+F。

基本语法：

```bash
grep "关键词" 文件名
```

#### 常用选项

| 选项 | 作用 | 示例 |
|------|------|------|
| `-i` | 忽略大小写 | `grep -i "error" log.txt` |
| `-n` | 显示行号 | `grep -n "timeout" config.py` |
| `-r` | 递归搜索子目录 | `grep -r "GUROBI" ./projects/` |
| `-v` | 反向匹配（排除） | `grep -v "#" config.txt` |
| `-l` | 只输出文件名 | `grep -l "main" *.py` |
| `-c` | 统计匹配行数 | `grep -c "Optimal" results.txt` |
| `-E` | 支持扩展正则 | `grep -E "A\|B" file` |

#### 正则表达式

`grep` 支持正则匹配，加 `-E` 可以使用扩展正则：

```bash
grep "^NODE" instance.vrp          # ^ 匹配行首，找以 NODE 开头的行
grep -E "Optimal|Infeasible" log   # | 表示 OR，匹配多个关键词
grep "[0-9]\{4\}" data.txt         # 匹配四位数字
```

#### 实际场景

```bash
# 检查求解器是否找到最优解
grep "Optimal solution found" gurobi.log

# 批量检查实验结果
grep "infeasible" result_*.txt

# 配合管道使用
history | grep claude
env | grep TASK
```

## 环境与进程管理

### export — 设置环境变量

让后续启动的子进程能读取到该变量：

```bash
# 设置环境变量
export GRB_LICENSE_FILE=/path/to/gurobi.lic

# 查看已设置的变量
env | grep GRB
```

注意区分：
- `VAR=value` — 只在当前 shell 有效，子进程看不到
- `export VAR=value` — 子进程也能继承

**持久化**：`export` 设置的变量在关闭终端后就消失了。要永久生效，需要写入 `~/.zshrc`（zsh）或 `~/.bashrc`（bash），然后 `source` 使其生效。

很多求解器（Gurobi/CPLEX）都通过环境变量读取许可证或配置。

### source — 在当前 shell 中执行脚本

脚本中的变量和设置会直接影响当前终端环境：

```bash
# 重新加载 shell 配置（修改 ~/.zshrc 后执行）
source ~/.zshrc

# 激活 Python 虚拟环境
source venv/bin/activate
```

`source` 也可以写成 `.`，两者等价：

```bash
. ~/.zshrc              # 等同于 source ~/.zshrc
```

对比：直接运行 `./script.sh` 会在新进程中执行，退出后变量失效；`source` 则保留在当前 shell 中。

## 信息查询

### history — 查看历史命令

```bash
history                 # 显示最近命令
history | grep python   # 搜索含 "python" 的历史命令
```

快捷技巧：
- `↑/↓` 方向键浏览历史
- `!!` 重复上一条命令
- `!n` 重新执行第 n 条命令

### type — 查看命令类型

```bash
type cd      # cd is a shell builtin
type cat     # cat is /bin/cat
type ll      # ll is an alias for ls -l
```

这引出了一个冷知识：Shell 命令其实分为**内置命令**（由 shell 自己实现，如 `cd`、`export`、`source`）和**外部命令**（磁盘上的可执行文件，如 `cat`、`grep`、`git`）。日常使用中不需要区分，但遇到"为什么 `cd` 找不到可执行文件"这类问题时，`type` 能帮你快速定位。

## 总结

| 分类 | 命令 | 用途 |
|------|------|------|
| 文件与目录 | `cd`, `cat`, `less`, `head`, `tail` | 切换目录、查看文件内容 |
| 文本搜索 | `grep` | 搜索、过滤文本 |
| 环境与进程 | `export`, `source` | 设置变量、加载脚本 |
| 信息查询 | `history`, `type` | 查看历史、查看命令类型 |
