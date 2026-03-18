---
name: obsidian-session
description: 保存当前 Claude Code 会话摘要到 Obsidian 仓库，使用组织化的文件夹结构。支持中文标题和标签，可按任务拆分为多个文档。集成 Templater、Tasks、Dataview 插件功能。
allowed_tools: ["Bash", "AskUserQuestion"]
---

# /obsidian-session - 保存会话到 Obsidian

将当前 Claude Code 会话保存为格式化的笔记，存储在 Obsidian 仓库中，包含完整的上下文、决策和成果。如果会话包含多个独立任务，应拆分为多个文档。

## 使用方法

```bash
/obsidian-session                                          # 自动分析并生成文档
/obsidian-session 自定义标题                              # 使用自定义标题创建新文档
/obsidian-session --continue                              # 追加到当前 session 的已有文档
/obsidian-session --file "现有文档路径"                   # 追加到指定文档
```

## Session 持续更新机制

### 同一文档的持续追加

在同一个 Claude Code session 中多次使用 `/obsidian-session` 时：

| 调用方式 | 行为 |
|---------|------|
| `/obsidian-session` | 首次调用创建新文档，后续调用追加到同一文档 |
| `/obsidian-session 新标题` | 始终创建新文档（重置当前文档引用） |
| `/obsidian-session --continue` | 明确追加到当前文档 |
| `/obsidian-session --file "路径"` | 追加到指定文档 |

**状态追踪**：

使用临时文件追踪当前 session 的文档路径：

\`\`\`bash
# Session 文档状态文件
SESSION_STATE_FILE="/tmp/claude-session-current-$$.txt"

# 读取当前文档路径
CURRENT_DOC=$(cat "$SESSION_STATE_FILE" 2>/dev/null || echo "")

# 保存当前文档路径
echo "Claude Code/2026-03-18/文档.md" > "$SESSION_STATE_FILE"
\`\`\`

**追加内容示例**：

\`\`\`bash
if [ -n "$CURRENT_DOC" ]; then
    # 追加到现有文档
    obsidian-cli create "$CURRENT_DOC" --content "$NEW_CONTENT" --append
else
    # 创建新文档
    obsidian-cli create "Claude Code/$DATE/$TITLE.md" --content "$CONTENT"
    echo "Claude Code/$DATE/$TITLE.md" > "$SESSION_STATE_FILE"
fi
\`\`\`

### 上下文压缩前的保存提示

**问题**：当对话接近上下文限制时，Claude Code 会自动压缩早期消息，导致内容缺失，影响后续分析。

**解决方案**：在压缩前主动保存会话快照。

**触发条件**（满足任一即提示）：

| 条件 | 阈值 |
|-----|------|
| Token 使用率 | > 80% |
| 对话轮次 | > 50 轮 |
| 距离上次保存 | > 30 分钟 |
| 重要里程碑 | Git commit、重大功能完成 |

**提示消息示例**：

\`\`\`
⚠️ 检测到会话内容较多（已使用 85% token），建议先保存当前进度到 Obsidian。

选项：
1. 保存当前进度 → /obsidian-session
2. 继续对话 → 稍后手动保存
3. 忽略提示 → 本次会话不再提示

已保存的内容将被追加到当前文档，后续可继续更新。
\`\`\`

### 断点续传支持

当上下文被压缩后，再次使用 `/obsidian-session` 时：

1. **读取已保存的文档**：从 Obsidian 读取当前文档内容
2. **补充缺失信息**：基于现有文档补充新内容
3. **更新时间戳**：更新 frontmatter 中的时间字段
4. **标记续传点**：添加"续传"标记，说明这是从压缩点继续的

**续传标记示例**：

\`\`\`markdown
## 续传记录

- 📅 初始创建: 2026-03-18 10:00
- 📝 第一次更新: 2026-03-18 11:30（上下文压缩前保存）
- 📝 第二次更新: 2026-03-18 14:00（补充后续内容）
\`\`\`

## 功能说明

1. **分析会话内容** - 提取关键问题、解决方案、决策和成果
2. **识别独立任务** - 判断是否应拆分为多个文档
3. **用户确认拆分** - 通过多选界面让用户确认或修改任务拆分
4. **Session 持续更新** - 同一 session 中多次调用时追加到已有文档
5. **压缩前保存提示** - Token 使用率 > 80% 时提示用户保存进度
6. **断点续传支持** - 上下文压缩后基于已保存文档继续更新
7. **生成结构化笔记** - 使用标准化的模板组织内容
8. **保存到 Obsidian** - 按日期组织文件夹：`Claude Code/YYYY-MM-DD/`
9. **添加元数据** - 包含标签、日期、状态和相关文件
10. **Wikilinks 支持** - 自动将文件引用转换为 wikilinks 格式
11. **任务管理** - 集成 Tasks 插件语法
12. **数据查询** - 支持 Dataview 查询和索引

## 技术实现

### Obsidian CLI 使用

**设置默认 vault**（首次使用）：

```bash
# 使用 vault 名称设置默认
obsidian-cli set-default obsidian
# 验证配置
obsidian-cli print-default
# 输出：Default vault name: obsidian
#      Default vault path: /Users/hortor/Documents/work/obsidian
```

### 创建笔记的方法

#### 方法 1: 使用 obsidian-cli create（推荐）

```bash
# 创建笔记（使用相对路径）
obsidian-cli create "Claude Code/2026-03-18/标题.md" --content "# 标题\n\n内容"

# 创建并打开笔记
obsidian-cli create "Claude Code/2026-03-18/标题.md" --content "内容" --open

# 追加内容到现有笔记
obsidian-cli create "Claude Code/现有文档.md" --content "追加的内容" --append
```

#### 方法 2: 使用 Templater 模板

如果配置了 Templater 插件，可以使用模板创建：

```bash
# 先创建模板文件（如果不存在）
obsidian-cli create "Templates/会话笔记模板.md" --content "模板内容..."

# 使用模板创建笔记（需在 Obsidian 中操作）
# 1. 打开 Obsidian
# 2. 按 Ctrl/Cmd + P 打开命令面板
# 3. 选择 "Templater: Create new note from template"
# 4. 选择会话笔记模板
```

#### 方法 3: 使用 Bash 直接创建（适用于复杂内容）

```bash
# 创建文件夹
mkdir -p "/Users/hortor/Documents/work/obsidian/Claude Code/2026-03-18"

# 创建文件
cat > "/Users/hortor/Documents/work/obsidian/Claude Code/2026-03-18/标题.md" << 'EOF'
---
title: 标题
date: 2026-03-18
tags:
  - claude-session
status: 已完成
type: session-summary
---

# 标题

## 摘要
内容...
EOF
```

### Obsidian CLI 常用命令

| 命令 | 用途 | 示例 |
|-----|------|------|
| `create` | 创建笔记 | `obsidian-cli create "路径/文件.md" --content "内容"` |
| `list` | 列出文件 | `obsidian-cli list "Claude Code"` |
| `open` | 打开文件 | `obsidian-cli open "Claude Code/2026-03-18/标题.md"` |
| `search` | 搜索文件 | `obsidian-cli search "标题"` |
| `search-content` | 搜索内容 | `obsidian-cli search-content "关键词"` |
| `frontmatter` | 管理属性 | `obsidian-cli frontmatter "标题" --print` |
| `daily` | 创建/打开日记 | `obsidian-cli daily` |
| `set-default` | 设置默认 vault | `obsidian-cli set-default obsidian` |
| `print-default` | 查看默认配置 | `obsidian-cli print-default` |

## Hook 集成（自动保存提示）

### 上下文压缩前提示 Hook

使用 Claude Code 的压缩前 hook 在系统即将压缩上下文时自动提示用户保存会话。

**查找可用的压缩相关 Hook**：

首先检查 Claude Code 提供的压缩相关 hooks：

\`\`\`bash
# 查看可用的 hook 类型
claude --help hooks 2>&1 | grep -i compress

# 或者检查文档中的 hooks 列表
# 常见的压缩相关 hooks 可能包括：
# - PreCompact: 上下文压缩前触发
# - BeforeCompact: 压缩前
# - OnCompact: 压缩时
# 等等（具体以 Claude Code 文档为准）
\`\`\`

**创建压缩前提示 Hook**（假设 hook 名为 PreCompact）：

\`\`\`bash
# 创建 hook 目录
mkdir -p ~/.claude/hooks

# 创建压缩前提示 hook
cat > ~/.claude/hooks/pre-compact-save-reminder.sh << 'EOF'
#!/bin/bash
# Hook: 在上下文压缩前提示用户保存会话
# 触发时机: Claude Code 即将压缩上下文时

echo ""
echo "═══════════════════════════════════════════════════════════"
echo "⚠️  上下文即将压缩"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "Claude Code 即将压缩对话历史以释放上下文空间。"
echo "建议先保存当前会话到 Obsidian，避免内容缺失。"
echo ""
echo "👉 保存会话: /obsidian-session"
echo "👉 忽略提示: 继续当前操作"
echo ""
echo "═══════════════════════════════════════════════════════════"
echo ""

# 可选：使用 AskUserQuestion 询问用户
# 这需要通过返回值告诉 Claude Code 调用该工具
EOF

chmod +x ~/.claude/hooks/pre-compact-save-reminder.sh
\`\`\`

**在 settings.json 中启用 hook**：

\`\`\`json
{
  "hooks": {
    "PreCompact": [
      {
        "name": "pre-compact-save-reminder",
        "command": "bash",
        "args": ["/Users/hortor/.claude/hooks/pre-compact-save-reminder.sh"]
      }
    ]
  }
}
\`\`\`

**注意**：请根据实际的 Claude Code 版本和文档调整 hook 名称。具体可用的 hooks 请参考：
- Claude Code 官方文档
- `~/.claude/settings.json` 中的 hooks 配置示例
- `claude --help` 命令输出

### Session 清理 Hook

创建 stop hook 在 session 结束时清理临时文件。

\`\`\`bash
cat > ~/.claude/hooks/cleanup-session.sh << 'EOF'
#!/bin/bash
# Hook: session 结束时清理临时文件

rm -f /tmp/claude-session-current-*.txt
rm -f /tmp/claude-save-reminder-*

echo "🧹 Session 临时文件已清理"
EOF

chmod +x ~/.claude/hooks/cleanup-session.sh
\`\`\`

**在 settings.json 中添加**：

\`\`\`json
{
  "hooks": {
    "Stop": [
      {
        "name": "cleanup-session",
        "command": "bash",
        "args": ["/Users/hortor/.claude/hooks/cleanup-session.sh"]
      }
    ]
  }
}
\`\`\`

## 插件集成

### Tasks 插件

在会话笔记中添加任务列表，使用 Tasks 插件语法：

```markdown
## 任务列表

- [ ] 分析当前问题
- [/] 实现解决方案（进行中）
- [x] 测试功能（已完成）
- [- ] 已取消的任务
```

**任务状态符号**：
- `[ ]` - Todo（待办）
- `[/]` - In Progress（进行中）
- `[x]` - Done（已完成）
- `[- ]` - Cancelled（已取消）

**任务元数据**（可选）：
```markdown
- [ ] 重要任务 ⏫ 📅 2026-03-20
- [ ] 低优先级任务 🔽
- [ ] 有截止日期的任务 📅 2026-03-25
- [ ] 重复任务 🔁 every day
```

### Dataview 插件

使用 Dataview 创建索引和查询：

#### 创建索引页面

```markdown
---
title: Claude Code 会话索引
type: index
---

# Claude Code 会话索引

## 按日期查询

\`\`\`dataview
TABLE date, status, tags
FROM "Claude Code"
WHERE type = "session-summary"
SORT date DESC
\`\`\`

## 按标签分组

\`\`\`dataview
LIST
FROM "Claude Code"
WHERE type = "session-summary"
GROUP BY tags
SORT tags ASC
\`\`\`

## 统计信息

\`\`\`dataview
TABLE length(rows) as "文档数"
FROM "Claude Code"
WHERE type = "session-summary"
GROUP BY date_format(date, "yyyy-MM")
SORT date DESC
\`\`\`
```

#### 常用查询示例

\`\`\`dataview
# 查询所有未完成的任务
TASK
FROM "Claude Code"
WHERE !completed
GROUP BY file.link

# 查询特定标签的文档
LIST
FROM "Claude Code"
WHERE contains(tags, "#bug修复")
SORT date DESC

# 查询最近7天的文档
TABLE date, status
FROM "Claude Code"
WHERE date >= date(today) - dur(7 days)
SORT date DESC
\`\`\`

### Templater 插件

创建动态模板，自动生成内容。

#### 模板文件位置

技能模板目录：`~/.claude/skills/obsidian-session/templates/`

包含文件：
- `会话笔记模板.md` - 标准会话笔记模板

#### 使用模板

**步骤 1：复制模板到目标 vault**

\`\`\`bash
# 复制模板到当前 vault 的 Templates 目录
cp ~/.claude/skills/obsidian-session/templates/会话笔记模板.md \
   "$(pwd)/Templates/会话笔记模板.md"
\`\`\`

**步骤 2：在 Obsidian 中使用模板**

1. 打开 Obsidian
2. 按 `Ctrl/Cmd + P` 打开命令面板
3. 选择 "Templater: Create new note from template"
4. 选择 "会话笔记模板"

#### 模板内容

模板包含以下字段：
- 自动生成的日期和时间戳
- 光标位置标记（创建时光标自动定位）
- Tasks 插件语法的任务列表
- 标准化的 frontmatter 格式
- 预定义的内容结构

#### Templater 变量

| 变量 | 说明 | 示例输出 |
|-----|------|---------|
| `<% tp.date.now() %>` | 当前日期时间 | 2026-03-18 10:30:00 |
| `<% tp.date.now('YYYY-MM-DD') %>` | 格式化日期 | 2026-03-18 |
| `<% tp.file.title %>` | 文件标题 | 标题 |
| `<% tp.file.cursor() %>` | 光标位置 | （光标在此处） |
| `<% tp.system.clipboard() %>` | 剪贴板内容 | （剪贴板文本） |

## 多任务拆分规则

### 何时拆分为多个文档？

**自动拆分条件**（满足任一即应拆分）：

| 条件 | 说明 | 示例 |
|-----|------|------|
| **多个 Git 提交** | 不同的 commit 表示不同的任务 | commit 1: 修复bug<br>commit 2: 新功能 |
| **不同功能模块** | 涉及不同的代码模块或系统 | 模块A: 用户系统<br>模块B: 支付系统 |
| **明确的任务切换** | 用户明确切换到新任务 | "现在做X"<br>"接下来做Y" |
| **时间间隔明显** | 两个任务之间有较长时间间隔 | 上午：修复缓存<br>下午：实现 buff |
| **独立的 bug 修复** | 每个 bug 修复都是独立的任务 | bug1: 登录失败<br>bug2: 支付异常 |

**合并为单个文档**（同时满足）：

- 任务之间有强依赖关系
- 属于同一个功能的连续迭代
- 文档内容较少（< 50行）

### 文件组织结构

**单个任务时**：

\`\`\`
Claude Code/2026-03-18/
└── 优化数据库查询性能.md
\`\`\`

**多个任务时**：

\`\`\`
Claude Code/2026-03-18/
├── 修复跨服务缓存一致性.md
├── 实现 Auxiliary 组队 buff.md
└── 优化 obsidian-session 技能文档.md
\`\`\`

### 任务间关联

使用 frontmatter 中的 `related_tasks` 字段链接相关任务：

\`\`\`yaml
---
title: 实现 Auxiliary 组队 buff
date: 2026-03-18
related_tasks:
  - 修复跨服务缓存一致性
---
\`\`\`

在文档末尾添加相关任务链接：

\`\`\`markdown
## 相关
- 相关任务: [[修复跨服务缓存一致性]]
- 前置任务: [[优化角色属性计算]]
- 后续任务: [[添加更多 Auxiliary 类型]]
\`\`\`

## 会话分析流程

### 步骤 0: 检查 Session 状态

在开始分析前，检查当前 session 是否已有文档：

**检查状态文件**：

\`\`\`bash
SESSION_STATE_FILE="/tmp/claude-session-current-$$.txt"
CURRENT_DOC=$(cat "$SESSION_STATE_FILE" 2>/dev/null || echo "")
SESSION_ID=$$  # 使用进程 ID 作为 session 标识
\`\`\`

**决策逻辑**：

| 情况 | 行为 |
|-----|------|
| 首次调用（无状态文件） | 创建新文档 |
| 后续调用（有状态文件）+ 无新标题 | 追加到当前文档 |
| 提供新标题 | 创建新文档，更新状态文件 |
| 使用 `--continue` | 强制追加到当前文档 |
| 使用 `--file` | 追加到指定文档，更新状态文件 |

**状态文件格式**：

\`\`\`
Claude Code/2026-03-18/添加用户确认任务拆分步骤.md
\`\`\`

**读取已保存文档**（用于断点续传）：

\`\`\`bash
if [ -n "$CURRENT_DOC" ]; then
    # 读取现有文档内容
    EXISTING_CONTENT=$(obsidian-cli create "$CURRENT_DOC" --read)
    echo "📋 已读取现有文档: $CURRENT_DOC"
    echo "📝 将追加新内容到文档末尾"
fi
\`\`\`

### 步骤 1: 识别关键信息

从对话历史中提取：

| 信息类型 | 提取方法 |
|---------|---------|
| **主要问题** | 识别用户提出的初始问题或任务 |
| **解决方案** | 分析实施的步骤和代码更改 |
| **关键决策** | 记录重要的技术决策和原因 |
| **修改文件** | 收集所有更改的文件路径 |
| **成果** | 总结具体的结果和影响 |
| **提交记录** | 收集所有 Git commit |

### 步骤 2: 判断是否拆分

分析任务边界的信号：

\`\`\`
任务切换信号：
├── Git 提交 → "已推送"、"commit 成功"
├── 用户明确指示 → "现在做X"、"接下来"
├── 主题变化 → 从模块A切换到模块B
└── 时间间隔 → 长时间无活动
\`\`\`

### 步骤 3: 生成标题

**标题原则**：
- 使用简练的中文（2-8个字）
- 突出核心问题或成果
- 使用动词开头（如"修复"、"实现"、"优化"）
- 每个任务独立标题

**示例标题**：
\`\`\`
✅ 修复身份验证中间件
✅ 实现用户仪表板
✅ 优化数据库查询
✅ 配置 Obsidian 集成
✅ 修复跨服务缓存一致性
✅ 实现 Auxiliary 组队 buff
✅ 优化技能文档格式
\`\`\`

### 步骤 3.5: 用户确认拆分（交互式）

在生成标题后，使用 `AskUserQuestion` 工具向用户展示拆分结果，允许用户确认、修改或合并任务。

**使用 AskUserQuestion 工具**：

```json
{
  "questions": [
    {
      "question": "检测到会话包含多个独立任务，请确认拆分结果是否正确？您可以修改任务名称（将作为文档标题），或选择合并为单个文档。",
      "header": "确认任务拆分",
      "multiSelect": true,
      "options": [
        {
          "label": "任务1: 修复跨服务缓存一致性",
          "description": "涉及 Redis 缓存同步和消息队列"
        },
        {
          "label": "任务2: 实现 Auxiliary 组队 buff",
          "description": "新增游戏机制和属性计算"
        }
      ]
    }
  ]
}
```

**用户响应处理**：

| 用户选择 | 行为 |
|---------|------|
| **选中所有任务** | 按原计划创建多个文档 |
| **只选部分任务** | 仅创建选中的任务文档 |
| **选择"其他"并输入** | 使用用户输入的标题替代原建议 |
| **未选择任何选项** | 询问用户是否要合并为单个文档 |

**用户修改标题示例**：

```
用户选择"其他"并输入：
- "修复 Redis 缓存一致性问题"（替代"修复跨服务缓存一致性"）
- "添加组队 buff 功能"（替代"实现 Auxiliary 组队 buff"）
```

**实现代码示例**：

```python
# 伪代码示例
tasks = analyze_session(conversation)
titles = [task.title for task in tasks]

response = await AskUserQuestion(
    questions=[{
        "question": "会话分析完成，检测到以下独立任务。请确认要创建哪些文档？您可以选择多个，或在'其他'中修改标题。",
        "header": "确认文档创建",
        "multiSelect": True,
        "options": [
            {"label": title, "description": task.brief}
            for task in tasks
        ] + [
            {"label": "合并为单个文档", "description": "将所有任务合并到一个文档中"}
        ]
    }]
)

confirmed_tasks = process_user_response(response, tasks)
```

**特殊场景处理**：

1. **用户选择"合并为单个文档"**：
   - 综合所有任务内容生成单一文档
   - 标题使用最突出的任务或通用描述
   - 在 frontmatter 中使用 `related_tasks` 列出所有任务

2. **用户修改标题**：
   - 使用用户输入的新标题
   - 保持原有的任务内容和结构
   - 检查标题是否符合规范（2-8个字）

3. **用户取消操作**：
   - 不创建任何文档
   - 询问用户是否需要重新分析
   - 或允许用户提供自定义标题

### 步骤 4: 创建笔记结构

使用以下模板：

\`\`\`markdown
---
title: {生成的标题}
date: {YYYY-MM-DD}
tags:
  - claude-session
  - {分类标签}
status: {已完成|进行中}
type: session-summary
related_tasks: [{相关任务标题}]
---

# {生成的标题}

## 摘要
{2-3句话概述完成的工作}

## 问题 / 任务
{最初的请求或问题是什么？}

## 解决方案
{采取的关键步骤、做出的决策、编写的代码}

## 关键成果
- {成果 1}
- {成果 2}
- {成果 3}

## 任务列表
- [ ] 待完成任务
- [/] 进行中任务
- [x] 已完成任务

## 修改的文件
{列出更改的文件: [[file1.md]], [[file2.py]] 等}

## 相关
- Git commit: {commit-id}
- 相关任务: [[{相关任务文档}]]
\`\`\`

### 步骤 5: 保存到 Obsidian

**推荐使用 obsidian-cli create**：

\`\`\`bash
#!/bin/bash
# 设置变量
TITLE="标题"
DATE=$(date +%Y-%m-%d)
SESSION_STATE_FILE="/tmp/claude-session-current-$$.txt"

# 检查是否有当前文档
CURRENT_DOC=$(cat "$SESSION_STATE_FILE" 2>/dev/null || echo "")

# 准备内容
CONTENT="---
title: $TITLE
date: $DATE
tags:
  - claude-session
status: 已完成
type: session-summary
---

# $TITLE

## 摘要
内容...
"

if [ -n "$CURRENT_DOC" ] && [ "$#" -eq 0 ]; then
    # 追加到现有文档
    echo "📝 追加内容到: $CURRENT_DOC"

    # 添加续传分隔符
    APPEND_CONTENT="

---

## 续传更新

**更新时间**: $(date '+%Y-%m-%d %H:%M:%S')

$CONTENT
"

    obsidian-cli create "$CURRENT_DOC" --content "$APPEND_CONTENT" --append
    echo "✅ 内容已追加到: $CURRENT_DOC"
else
    # 创建新文档
    DOC_PATH="Claude Code/$DATE/$TITLE.md"
    obsidian-cli create "$DOC_PATH" --content "$CONTENT"

    # 保存到状态文件
    echo "$DOC_PATH" > "$SESSION_STATE_FILE"

    echo "✅ 笔记已创建: $DOC_PATH"
    echo "💾 状态已保存: $SESSION_STATE_FILE"
fi
\`\`\`

**清理状态文件**（session 结束时）：

\`\`\`bash
# 在 .claude/hooks/stop 中清理
rm -f /tmp/claude-session-current-*.txt
\`\`\`

**使用 --file 参数追加到指定文档**：

\`\`\`bash
# 追加到指定文档
TARGET_DOC="Claude Code/2026-03-17/之前的任务.md"
obsidian-cli create "$TARGET_DOC" --content "$NEW_CONTENT" --append

# 更新状态文件
echo "$TARGET_DOC" > "$SESSION_STATE_FILE"
\`\`\`

## 标签分类

根据工作类型添加标签：

| 标签 | 使用场景 |
|-----|---------|
| `#bug修复` | 修复缺陷或问题 |
| `#新功能` | 实现新特性 |
| `#重构` | 代码重构或优化 |
| `#性能优化` | 性能改进 |
| `#文档` | 文档更新 |
| `#测试` | 测试相关 |
| `#配置` | 环境或配置更改 |

## 示例对比

### 示例 1: 单个任务（不应拆分）

**会话内容**：优化数据库查询性能

\`\`\`markdown
---
title: 优化数据库查询性能
date: 2026-03-18
tags:
  - claude-session
  - 性能优化
status: 已完成
type: session-summary
---

# 优化数据库查询性能

## 摘要
优化了用户列表查询，添加索引并重构查询逻辑，查询时间从 2s 降低到 200ms。

## 问题 / 任务
用户列表加载缓慢，影响用户体验。

## 解决方案
1. 添加联合索引
2. 重构查询逻辑
3. 添加查询缓存

## 关键成果
- ✅ 查询时间从 2s 降低到 200m
- ✅ 数据库负载降低 60%

## 任务列表
- [x] 分析查询性能瓶颈
- [x] 添加数据库索引
- [x] 重构查询逻辑
- [x] 性能测试验证

## 修改的文件
- [[servers/api/user.go]]
- [[models/user_query.go]]

## 相关
- Git commit: abc123
\`\`\`

### 示例 2: 多个任务（应拆分）

**会话内容**：修复缓存bug + 实现新功能

**应创建两个文档**：

1. `修复跨服务缓存一致性.md`
2. `实现 Auxiliary 组队 buff.md`

每个文档包含各自的内容，并通过 `related_tasks` 字段关联。

## 最佳实践

✅ **推荐做法**：
- 使用具体的、描述性的中文标题
- **每个独立任务创建单独的文档**
- 使用 `related_tasks` 字段关联相关任务
- 保持摘要简洁（2-3句话）
- 所有文件引用使用 wikilinks：`[[文件路径]]`
- 添加相关标签便于后续搜索
- 使用 Tasks 插件语法管理任务列表
- 利用 Dataview 创建索引和查询

❌ **避免**：
- 使用通用标题如"会话笔记"
- **将多个独立任务混在一个文档**
- 摘要过长或过短
- 混合中英文内容
- 忘记添加标签
- 省略修改的文件列表

## 相关命令

- `/commit` - 创建符合规范的 Git 提交
- `/obsidian-session` - 保存会话到 Obsidian（当前命令）
- `obsidian-cli` - Obsidian CLI 工具
- Templater 插件 - 模板和动态内容生成
- Tasks 插件 - 任务管理和跟踪
- Dataview 插件 - 数据查询和索引

## 插件参考

- **Templater**: 使用 `<% %>` 语法动态生成内容
- **Tasks**: 使用 `- [ ]` 语法创建任务，支持状态和元数据
- **Dataview**: 使用 `\`\`\`dataview` 代码块查询和展示数据

***

*保存会话笔记，便于知识管理和经验积累。支持多任务拆分，每个任务独立记录。集成 Templater、Tasks、Dataview 插件功能。*
