---
name: obsidian-session
description: 保存当前 Claude Code 会话摘要到 Obsidian 仓库。支持智能合并、多任务拆分、session 持续更新。集成 Templater、Tasks、Dataview 插件。
version: 1.0.1
allowed_tools: ["Bash", "AskUserQuestion"]
---

# /obsidian-session - 保存会话到 Obsidian

将 Claude Code 会话保存为结构化笔记，存储在 Obsidian 仓库中。自动分析会话内容，识别独立任务，支持智能文档合并和断点续传。

## 快速开始

### 首次使用

bash
# 1. 设置默认 vault（仅需一次）
obsidian-cli set-default obsidian

# 2. 保存当前会话
/obsidian-session

# 3. 使用自定义标题
/obsidian-session 优化数据库查询性能


### Session 中多次使用

bash
# 首次调用 - 创建新文档
/obsidian-session

# 后续调用 - 自动追加到同一文档
/obsidian-session

# 或明确指定继续
/obsidian-session --continue

# 追加到指定文档
/obsidian-session --file "Claude Code/2026-03-17/之前的任务.md"


## 核心功能

### 1. 自动会话分析

提取关键信息：
- 主要问题和解决方案
- 关键决策和成果
- 修改的文件列表
- Git commit 记录

### 2. 智能任务拆分

自动检测独立任务并询问是否拆分：

| 拆分信号 | 示例 |
|---------|------|
| 多个 Git 提交 | commit 1: 修复bug + commit 2: 新功能 |
| 不同功能模块 | 模块A: 用户系统 + 模块B: 支付系统 |
| 明确的任务切换 | "现在做X" → "接下来做Y" |
| 时间间隔明显 | 上午: 修复缓存 + 下午: 实现 buff |

### 3. 智能文档合并

当文档已存在时，智能合并而非简单追加：
- 更新摘要（整合新旧内容）
- 扩展解决方案（追加新步骤并重新编号）
- 合并成果列表
- 同步任务状态
- 检测并删除重复内容
- 重组文档结构

### 4. Session 持续更新

使用状态文件追踪当前文档：

bash
# 状态文件位置
/tmp/claude-session-current-$$.txt

# 检查当前文档
CURRENT_DOC=$(cat "/tmp/claude-session-current-$$.txt" 2>/dev/null || echo "")


### 5. 断点续传

当上下文被压缩后：
- 读取已保存文档
- 补充缺失信息
- 添加续传标记
- 继续更新文档

## 文档结构

生成的文档使用统一模板：

markdown
---
title: 标题
date: 2026-03-18
tags:
  - claude-session
  - 新功能
status: 已完成
type: session-summary
related_tasks: [相关任务]
---

# 标题

## 摘要
2-3句话概述完成的工作

## 问题 / 任务
最初的请求或问题

## 解决方案
关键步骤、决策和代码

## 关键成果
- 成果 1
- 成果 2

## 任务列表
- [x] 已完成任务
- [/] 进行中任务
- [ ] 待完成任务

## 修改的文件
- [[file1.md]]
- [[file2.py]]

## 相关
- Git commit: {commit-id}
- 相关任务: [[相关文档]]


## 最佳实践

### 推荐做法

- 使用具体的、描述性的中文标题（2-8个字）
- 动词开头（如"修复"、"实现"、"优化"）
- 每个独立任务创建单独文档
- 使用 related_tasks 字段关联相关任务
- 保持摘要简洁（2-3句话）
- 所有文件引用使用 wikilinks：[[文件路径]]
- 添加相关标签便于搜索

### 避免

- 使用通用标题如"会话笔记"
- 将多个独立任务混在一个文档
- 摘要过长或过短
- 混合中英文内容
- 忘记添加标签
- 省略修改的文件列表

## 版本历史

- **1.0.1** (2026-03-18) - 添加智能文档合并机制，优化文档结构，从 1197 行精简到 169 行
- **1.0.0** (2026-03-18) - 初始版本，包含会话分析、任务拆分、session 持续更新功能

---

*保存会话笔记，便于知识管理和经验积累。支持多任务拆分、智能合并、session 持续更新。集成 Templater、Tasks、Dataview 插件功能。*
