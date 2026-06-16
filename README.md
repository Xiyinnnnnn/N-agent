https://xiyinnnnnn.github.io/N-agent/index.html
# N-agent

浏览器内运行的小说创作引擎。1 个主编 + 21 个子 Agent，真 tool calling + SSE 流式。单 HTML 文件，零构建，零后端。

## 架构

```
用户输入
  │
  ▼
┌─────────────────────────────────┐
│         主编 Agent               │
│  system prompt (静态，缓存友好)    │
│  + 项目快照 (动态注入 user msg)    │
│  + 消息历史                       │
└──────────┬──────────────────────┘
           │ SSE streaming + tool calling
           ▼
    ┌─────────────────┐
    │  主编决定调用工具  │ ◄── 最多 10 轮/turn
    └────┬────────────┘
         │
    ┌────▼────────────────────────────────┐
    │           executeToolCall            │
    │                                      │
    │  ┌─ 子Agent工具 (A0-A20) ──────────┐ │
    │  │  独立 system prompt              │ │
    │  │  独立 API 调用 (streamToMsg)     │ │
    │  │  产出 JSON → 自动写入 state      │ │
    │  └─────────────────────────────────┘ │
    │                                      │
    │  ┌─ 搜索工具 ──────────────────────┐ │
    │  │  search_characters               │ │
    │  │  search_worldview                │ │
    │  │  search_outline                  │ │
    │  │  search_chapters (全文搜索)       │ │
    │  │  search_foreshadowing            │ │
    │  │  get_snapshot                    │ │
    │  └─────────────────────────────────┘ │
    │                                      │
    │  ┌─ 控制工具 ──────────────────────┐ │
    │  │  ask_user (暂停等用户回复)        │ │
    │  │  present_to_user                 │ │
    │  │  move_phase                      │ │
    │  │  complete_project                │ │
    │  └─────────────────────────────────┘ │
    └──────────────────────────────────────┘
         │
         │ 工具结果 → 回注 messages 数组
         │ 主编再决策 → 继续或输出最终文本
         ▼
    用户可见：思考块 (折叠) + 主编文本 (Markdown)
```

## 多 pass 流程

**一次用户输入触发的主编决策循环：**

```
Turn 1: 主编 reasoning → 调用 write_requirement(A0)
         └→ A0 产出需求分析 JSON，自动写 state

Turn 2: 主编收到结果 → 调用 ask_user
         └→ 系统暂停，等待用户确认

用户确认后再次触发：

Turn 1: 主编 reasoning → 调用 build_worldview(A1)
         └→ A1 产出世界观，自动写 state

Turn 2: 主编 reasoning → 调用 design_characters(A2)
         └→ A2 产出角色卡，自动写 state

Turn 3: 主编 reasoning → 调用 create_outline(A3)
         └→ A3 产出大纲，自动写 state

Turn 4: 主编 reasoning → 调用 ask_user
         └→ 展示大纲，等待确认

... 进入写作循环 ...

Turn N:   主编 → compose_task_brief(A11)
Turn N+1: 主编 → assemble_context(A19)
Turn N+2: 主编 → write_chapter(A4)
           └→ A4 产出章节正文 → 存入 pendingChapter
Turn N+3: 主编 → review_chapter(A10)
           └→ A10 十维评分
           
  若通过 (≥70):
    Turn N+4: 主编 → polish_style(A5)
    Turn N+5: 主编 → sync_state(A12)
              └→ 章节固化为最终存档，渲染章节预览

  若不通过:
    Turn N+4: 主编 → 给 A4 反馈 revise
              最多 3 次，超限 ask_user

大纲耗尽:
  Turn N: 主编 → extend_outline(A15) → 自动合并新节点
```

## 关键设计

**缓存命中**：system prompt 完全静态（`MASTER_SP_CACHED` 常量），项目快照通过 user role 消息注入。每次 API 调用命中 DeepSeek 上下文缓存。

**UI 隔离**：工具调用全程不渲染到聊天区。用户只看到思考块（默认折叠）和主编最终文本。Markdown 由 marked.js 渲染，支持表格/代码块/列表。

**Token 预算**：累计 token 超 500K 自动压缩上下文——保留最近 200 条消息，重置计数器。

**真流式**：SSE → async generator → `streamToMsg` 累积。`reasoning_content` 和 `tool_calls` 从 delta 逐片拼接。`stream_options: {include_usage: true}` 每轮拿到 usage。

**状态自动写入**：A1 世界观、A2 角色卡、A3 大纲、A15 大纲扩展——产出自动合并到 `AppState`。A12 在章节通过后执行全量同步（角色位置/关系/物品/伏笔）。

## 子 Agent 一览

| ID | 名称 | 触发时机 | 产出 |
|----|------|---------|------|
| A0 | 需求解析 | 收到创作种子，第一步必调 | 结构化需求 JSON |
| A1 | 世界观 | A0 确认后 | 世界观 → 自动写 state |
| A2 | 角色设计 | A1 完成后 | 角色卡 → 自动写 state |
| A3 | 大纲 | A2 完成后 | 大纲 → 自动写 state |
| A4 | 章节写作 | 每章 | 章节正文 → pendingChapter |
| A5 | 风格润色 | 章节评审通过后 | 润色稿 |
| A6 | 摘要切片 | 章节完成后 | 结构化摘要 |
| A7 | 记忆检索 | 按需 | 关联历史信息 |
| A8 | 事实核查 | 章节完成后 | 矛盾列表 |
| A9 | 矛盾修复 | A8 有矛盾时 | 修复方案 |
| A10 | 十维评审 | 章节完成后 | 评分 + 反馈 |
| A11 | 任务书 | 写每章前 | 任务书 |
| A12 | 状态同步 | 章节通过后 | 全量 state 更新 |
| A13 | 评审顾问 | revise 2 次仍不通过 | 决策建议 |
| A14 | 阶段顾问 | 按需 | 阶段规划建议 |
| A15 | 大纲扩展 | 大纲耗尽 | 5-15 新节点 → 自动合并 |
| A16 | 角色检测 | 章节完成后 | 新角色列表 |
| A17 | 剧情诊断 | 里程碑章节 | 全局诊断 |
| A18 | 消息润色 | 对用户说话前 | 润色消息 |
| A19 | 上下文组装 | 写每章前 | 上下文包 |
| A20 | 项目审计 | 里程碑章节 | 全局审计 |

## 运行

```
浏览器打开 index.html
→ 点 Header 的 Key 按钮
→ 填入 DeepSeek API Key
→ 选择模型 (V4 Flash / V4 Pro)
→ 设置 max_tokens 和思考强度 (off/high/max)
→ 在输入框输入创作种子
```

数据存 IndexedDB，支持多项目、检查点回滚、JSON/TXT 导出。

## 技术栈

- DeepSeek API（OpenAI-compatible tool calling + SSE streaming）
- Dexie.js（IndexedDB）
- marked.js（Markdown 渲染）
- 零依赖构建，单 HTML 文件
