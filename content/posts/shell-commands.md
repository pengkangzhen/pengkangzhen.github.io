---
title: "Shell 内置命令与常用外部命令速查"
date: 2026-04-29
categories: ["技术"]
tags: ["Shell", "Linux", "命令行"]
summary: "AI 辅助编码时经常遇到各种 Shell 命令，本文梳理内置命令与外部命令的区别，汇总常用命令的用法。"
draft: false
---

## 写在前面

用 AI 辅助编码时（比如 Claude Code、Cursor），AI 经常会给出各种 Shell 命令：`export` 设置环境变量、`source` 激活环境、`grep` 搜索代码……如果不清楚这些命令的含义，就只能盲目复制粘贴。这篇笔记把这些常见命令梳理一遍，搞清楚它们的类型和用法，让与 AI 的协作更高效。

## 内置命令 vs 外部命令

Shell 命令分为两类：

| 类型 | 例子 | 特点 |
|------|------|------|
| **Shell 内置命令** | `cd`, `export`, `source`, `history`, `echo`, `pwd` | 由 shell 自己实现，不依赖外部文件 |
| **外部命令** | `cat`, `grep`, `python`, `git` | 磁盘上的可执行文件，shell 负责启动它们 |

用 `type` 命令可以区分：

```bash
type cd      # cd is a shell builtin
type cat     # cat is /bin/cat
```

## 常用内置命令

### cd — 切换工作目录

```bash
cd ~/projects/vrp_solver   # 进入项目目录
cd ..                       # 返回上一级
cd -                        # 回到上一次所在的目录（高频操作）
cd ~                        # 回到主目录
pwd                         # 显示当前路径
```

运行脚本时，程序会从**当前目录**读取数据文件，用 `cd` 确保你在正确的位置。

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

### history — 查看历史命令

```bash
history                 # 显示最近命令
history | grep python   # 搜索含 "python" 的历史命令
```

快捷技巧：
- `↑/↓` 方向键浏览历史
- `!!` 重复上一条命令
- `!n` 重新执行第 n 条命令

## 常用外部命令

### cat — 查看文件内容

快速显示文件的全部内容：

```bash
cat config.yaml            # 显示文件内容
cat -n config.yaml         # 带行号显示
cat > newfile.txt          # 从键盘输入内容，Ctrl+D 结束并保存
cat a.txt b.txt > c.txt    # 合并多个文件
```

查看大文件时，`cat` 会一次性全部输出，建议改用 `less`（分页浏览）或 `head`/`tail`（只看头尾）：

```bash
less large.log             # 分页浏览，按 q 退出
head -20 large.log         # 只看前 20 行
tail -f app.log            # 实时追踪文件末尾（看日志常用）
```

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

## 总结

| 命令 | 类型 | 主要用途 |
|------|------|----------|
| `cd` | 内置 | 切换目录 |
| `export` | 内置 | 设置环境变量供子进程使用 |
| `source` | 内置 | 在当前 shell 中执行脚本 |
| `history` | 内置 | 查看命令历史 |
| `cat` | 外部 | 显示文件内容 |
| `grep` | 外部 | 文本搜索与过滤 |
