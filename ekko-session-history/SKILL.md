---
name: ekko-session-history
description: "By session ID or topic, recall Ekko historical conversation content across sessions. Not for current-session context. Triggers: 'find that session', 'before we discussed'."
version: 1.0.0
author: Ekko Agent
license: MIT
metadata:
  keywords:
    - "ekko"
    - "sessions"
    - "history"
    - "recall"
    - "topic"
---

# Ekko 会话历史检索

用户给出会话 ID 或按主题描述（"之前讨论过 X 的那次"、"找不到那次会话了"）要求找回对话时，按本流程走。

## 主题检索（用户没给 ID、按话题描述找）

1. 用 Ekko Studio 的 session 管理工具（`ekko_studio_use_toolset` 里查 session 相关操作）按关键词查——优先于直接读 state.db。
2. 复合主题拆 2-3 个同义查询各跑一遍（如"记忆 技能" + "记忆文件 规范"），取并集。
3. 给结果时把命中的会话写成会话 ID 链接（verbatim，句中内联），附会话标题与时间，让用户对认；命中不止一个时全部列出，按时间排。

## 步骤

1. **Ekko 工具优先**（经 `ekko_studio_use_toolset` 查操作）：
   - 查单个会话整读；`@session:<profile>/<id>` 按 `/` 拆成 profile + id，带 profile 传参。
   - 返回 "session not found" 时先核全存储再下结论：查是否有多个 profile，有则换 profile 重试；只有 default 时进入下一步。
2. **state.db 兜底**（`~/.ekko/state.db` 或等价的 Ekko 存储路径）：
   - 命中后读正文。
   - sessions 表**没有** `last_active` 和 `profile` 列，查了直接报列不存在。
3. **缺失判定**（全部查完才能说"不存在"）：
   - 全库文本匹配——注意误报。
   - 日志与看板残留。
   - 有备份/快照 DB 时对它们同样查。
4. **歧义消解**：全部查不到 → 如实说该 ID 不在任何存储（多半是记错/打错），并列出最近 ±14 天的候选会话表格（id / source / 开始时间 / message_count），让用户对认。

## Pitfalls

- **ID 查不到时不编造对话内容**：用户要数据说话；查不到就给出查过的位置清单 + 候选表，不合成历史。
- **文本命中 ≠ 会话记录**：全库 like 会命中"问这个 ID 的那条用户消息"和工具报错文本；权威判定只看 sessions 表是否有该 id。
- **工具报 not found ≠ 不存在**：可能是别的 profile 或记录已清理；必须走第 3 步全存储核查再下结论。
- **压缩接缝 = 一个对话窗口两个 DB 会话 ID**：上下文压缩时窗口会换新会话 ID 接续，库里前段、后段是两个不同 session。用户要"整个窗口/全部对话"时只查当前 ID 会漏掉前段：把与压缩点对接的那个前段 ID 找出来，两段一起导出并在交付时标注构成。
- **messages 表时间列叫 `timestamp`**（REAL epoch，UTC），没有 `created_at`；写 SQL 前先 `PRAGMA table_info(messages)` 核列名。
- **拉"用户全部发言"要全量合并，不按 compacted 过滤**：`messages.compacted=1` 的段才是长对话真正的主体。只查 `compacted=0` 会得到有损结果。要拉全部发言就两条都查并合并。

## 跨会话提炼（用户要求"查我全部发言，提炼某个概念/思维逻辑"）

1. **全量落盘再检索，别反复查库**：把全部用户发言 dump 成一个带行前缀的文件（每条含 会话ID|时间戳），落到 /tmp，再 grep 它。
2. **grep 必须带同义词族**：核心词 + 用户说过的近义表述，一轮 grep -E 扫完。
3. **过滤三类噪声再归纳**：① 行首是 `{` 的命中 = 注入的 JSON 载荷；② 每会话内发言常成对重复，按 (会话ID, 正文) 键去重；③ 伪 user 行家族——按已知前言家族正则批量剔除。
4. **归纳落盘，聊天只给概述**：提炼结果写成 MD，回复给结论 + 文件链接，不全文倒进聊天。
