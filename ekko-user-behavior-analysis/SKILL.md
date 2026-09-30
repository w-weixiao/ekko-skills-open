---
name: ekko-user-behavior-analysis
description: "Cross-session verification of user message style and claim patterns: full-DB scan + dedup + dual-granularity hypothesis testing, with original-text examples. Triggers: 'verify user pattern', 'cross-check style'."
version: 1.0.0
author: Ekko Agent
license: MIT
metadata:
  keywords:
    - "ekko"
    - "user-behavior"
    - "hypothesis"
    - "message-history"
    - "style-verification"
---

# 跨会话用户行为分析（Ekko 会话库）

群聊交叉验证、用户画像、声称行为模式核验时，用本技能扫全库数据，不做印象背书。

## 数据源与清洗

- Ekko 会话存储（state.db 或等价持久层）→ `messages` 表，取 `role='user' AND active=1 AND content IS NOT NULL`。
- 计数前必去重：同一条 user 消息在表内有多份写入（content 变体），对 content 做 MD5 去重，否则所有数字虚高。
- 排除伪 user 消息（role=user 但非用户原话），按 content 前缀判定：已知系统注入块前缀（`[ASYNC DELEGATION`、`[OUT-OF-BAND`、`[CONTEXT COMPACTION`、工具回报等），以及 `[...]` 批完成通知。
- 结果随口径一起报（会话数、去重后条数）；库随时间增长，不复用旧口径里的具体数字。

## 假设检验：永远测两个粒度

对声称的过程型模式（"先定框架再逐步收紧"、"禁止在前验收在后"这类）：把各条款类型（禁区/锚点/验收）各自正则化，然后两层都测——

1. **消息内层（任务书）**：同一条消息是否垂直叠放多类条款。
2. **会话层（轮次顺序）**：类型 A 是否只出现在前几条、类型 B 是否更晚才首现。

判据：会话层顺序不成立但消息内叠放成立 → 把声称改名为"单条任务书内垂直分层"再报告，不接受声称原话。分层扫描前先按消息长度分档：短到没有条款层的催办消息（先看长度分布定阈值）剔除，避免全量统一 regex 出假阳性。

## 报数纪律

- 每条结论 = 计数 + 占比 + 原文样例；样例直接从 DB 摘取，不转述。
- 其他 agent 采纳/提出的措辞，先全库复扫再表态；不扫全库不做背书。
- 假设不命中就明说不命中，并给出粒度降级或更便宜的验证路径，不用模糊话兜底。

## 注意

- 纯机械批处理任务：Python + sqlite3 合适，不违反"验证/判定不用 py"红线（那条针对人工验证判断，本任务是批量统计）。
- 条款正则按目标域调：先抽 20 条真用户消息对词再定 regex，不套固定词表。
