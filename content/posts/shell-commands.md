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

### wc — 统计行数/词数/字节数

```bash
wc -l data.csv             # 统计行数（常用：快速知道数据集有多少行）
wc -w readme.md            # 统计词数
wc -l *.py                 # 统计每个 Python 文件的行数
```

### sort — 排序

```bash
sort results.txt                   # 按字母排序
sort -n results.txt                # 按数值排序（默认是字典序，数字 9 会排在 10 后面）
sort -rn results.txt               # 按数值倒序
sort -t, -k2 data.csv              # 以逗号分隔，按第 2 列排序
```

### uniq — 去重

通常跟 `sort` 搭配使用（`uniq` 只去除相邻的重复行）：

```bash
sort names.txt | uniq              # 排序并去重
sort names.txt | uniq -c           # 去重并统计每个值出现的次数
sort names.txt | uniq -d           # 只显示重复的行
```

### cut — 提取列/字段

```bash
cut -d, -f1 data.csv               # 以逗号分隔，提取第 1 列
cut -d: -f1,3 /etc/passwd          # 以冒号分隔，提取第 1 和 3 列
cut -c1-10 file.txt                # 提取每行的前 10 个字符
```

### tr — 字符替换

```bash
echo "hello world" | tr 'a-z' 'A-Z'    # 转大写
cat file.txt | tr '\t' ','              # Tab 替换为逗号
cat file.txt | tr -d '\r'               # 删除 Windows 换行符 (\r)
```

### sed — 流编辑器，批量替换

AI 辅助编码中最常见的用法是批量替换文件中的文本：

```bash
# 替换文件中的文本（不会修改原文件，只输出结果）
sed 's/old/new/' file.txt

# 直接修改文件（-i）
sed -i '' 's/old/new/g' file.txt      # macOS 写法
sed -i 's/old/new/g' file.txt         # Linux 写法

# 删除空行
sed '/^$/d' file.txt

# 删除第 5 行
sed '5d' file.txt
```

### awk — 模式扫描与处理

功能强大，这里只列最常用的场景：

```bash
# 按列提取（以空格/tab 分隔）
awk '{print $1, $3}' data.txt         # 打印第 1 和第 3 列

# 指定分隔符
awk -F, '{print $2}' data.csv         # 以逗号分隔，打印第 2 列

# 带条件的处理
awk '$3 > 100 {print $1}' data.txt    # 第 3 列大于 100 时，打印第 1 列
```

### 组合使用

这些命令配合管道可以完成复杂的数据处理：

```bash
# 从日志中提取状态码，排序并统计频次
grep "HTTP" access.log | awk '{print $9}' | sort | uniq -c | sort -rn

# 统计 CSV 文件中有多少个不同的城市
cut -d, -f3 data.csv | sort | uniq | wc -l
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
| 文本搜索与处理 | `grep`, `wc`, `sort`, `uniq`, `cut`, `tr`, `sed`, `awk` | 搜索、统计、排序、替换、提取 |
| 环境与进程 | `export`, `source` | 设置变量、加载脚本 |
| 信息查询 | `history`, `type` | 查看历史、查看命令类型 |
