# obsidian-session v1.1 - 基于 Transcript 的会话文档生成

保存 Claude Code 会话到 Obsidian 知识库。**使用完整的 transcript 日志**，即使上下文被压缩也能完整记录会话历史。

## 🎉 v1.1 重大更新

**不再依赖上下文窗口！** 现在使用完整的 transcript 日志来生成文档。

| 特性 | v1.0.x (上下文) | v1.1 (Transcript) ✨ |
|-----|----------------|---------------------|
| **完整性** | ❌ 会被压缩丢失 | ✅ 完整保留 |
| **时间范围** | ⏱️ 30分钟左右 | ♾️ 整个会话 |
| **断点续传** | ❌ 困难 | ✅ 完美支持 |
| **实现方式** | Bash脚本解析 | AI智能分析 |

## 功能特性

- ✅ 基于 transcript 的完整会话解析
- ✅ AI 自动提取关键信息（需求、方案、成果）
- ✅ 智能标签生成（根据内容动态提取）
- ✅ 自动标题生成（基于完整内容分析）
- ✅ 智能文档合并（多次调用时自动更新）
- ✅ obsidian-cli 集成（带自动回退）
- ✅ 标准化文档结构

## 快速开始

### 使用方法

```bash
# 指定标题
/obsidian-session "优化数据库查询"

# 自动生成标题（AI 基于内容分析）
/obsidian-session
```

### 文档保存位置

```
Claude Code/
├── 2026-03-17/
│   └── xxx.md
├── 2026-03-18/
│   └── yyy.md
├── 2026-03-19/
│   └── 优化数据库查询.md
└── README.md
```

文档自动保存到 `Claude Code/YYYY-MM-DD/` 目录，按日期组织。

## 技能结构

```
obsidian-session/
├── SKILL.md       # 技能主文档（完整实现说明）
├── README.md      # 本文件（用户指南）
├── skill.json     # 技能配置
└── LICENSE        # MIT 许可证
```

## Transcript 文件位置

Claude Code 会自动保存完整的交流日志：

```
~/.claude/projects/<项目路径>/<session-id>.jsonl
```

**查找当前会话的 transcript**：

```bash
# 方法 1: 找到 session ID
cat ~/.claude/sessions/*.json | jq -r '.sessionId'

# 方法 2: 直接查找最新 transcript
ls -t ~/.claude/projects/-*/**/*.jsonl 2>/dev/null | head -1
```

## 环境变量

| 变量 | 说明 | 默认值 |
|-----|------|-------|
| `CLAUDE_SESSION_ID` | 会话 ID | 自动检测 |
| `OBSIDIAN_DEFAULT_VAULT` | 默认 vault 路径 | `~/Documents/work/obsidian` |

## 依赖要求

- **必需**：Bash、Read、Write
- **可选**：
  - obsidian-cli（推荐）- Obsidian CLI 工具
  - jq - JSON 解析器（用于 transcript 分析）

**安装 obsidian-cli**：
```bash
npm install -g @lynxtaa/obsidian-cli
```

## 与其他 Obsidian 技能的集成

本技能可与其他 Obsidian 技能配合使用：

### obsidian:obsidian-markdown
处理 Obsidian Flavored Markdown 语法（wikilinks、callouts、tags 等）

### obsidian:obsidian-bases
创建会话统计数据库，按日期、项目、标签筛选

### obsidian:json-canvas
可视化会话关系图和技能演进历程

详细集成方法请参考 [SKILL.md](SKILL.md)。

## 生成的文档结构

```markdown
---
title: 标题
date: 2026-03-19
tags: [claude-session, 已完成, database, optimization]
status: 已完成
type: session-summary
session_id: xxx
created_by: obsidian-session v1.1.0
---

# 标题

## 摘要
会话概要（2-3句话）

## 主要需求
- 需求 1
- 需求 2

## 解决方案
### 1. 方案标题
详细说明...

## 完成的成果
1. ✅ 成果 1
2. ✅ 成果 2

## 修改的文件
- [[path/to/file]] - 说明

## 统计信息
- 用户消息数量
- 会话时长
- 关键词统计
```

## 智能标签生成

标签根据内容动态提取：

**基础标签**：
- `claude-session` - 所有会话文档
- `已完成` / `进行中` - 状态标签

**内容标签**（自动提取）：
- **技术领域**：`database`、`frontend`、`backend`、`devops`
- **任务类型**：`bugfix`、`feature`、`optimization`、`refactor`
- **具体工具**：`mysql`、`redis`、`docker`、`git`
- **关键词**：`jwt`、`404-error`、`用户管理`、`api`

## 版本历史

- **1.1.0** (2026-03-19) - 使用完整 transcript 日志，AI 驱动分析
- **1.0.x** (2026-03-18) - 初始版本（基于上下文）

## 开发者

- 光
- 项目地址: https://github.com/yourusername/obsidian-session

## 许可证

MIT License
