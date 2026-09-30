---
name: ekko-mechanism-forensics
description: "When user disputes or questions Ekko internal mechanism, first read local source code + DB to verify before concluding. Not for general Q&A."
version: 1.0.0
author: Ekko Agent
license: MIT
metadata:
  keywords:
    - "ekko"
    - "mechanism"
    - "forensics"
    - "db"
    - "source"
    - "verify"
---

# Ekko 机制取证

用户说"以前不是这样"或凭印象质疑某机制时，先取证再下结论。记忆里的机制描述大概率过时或片面；本地源码和结构化记忆是权威来源。

## 取证顺序

1. **本地源码与文档**（最快、离线可靠）：`ekko-agent` 包内 `docs/` 目录是官方文档来源，先查再答。
   ```bash
   find <Ekko 数据目录> -name '*.md' | xargs grep -l "<关键词>" | head
   ```
2. **结构化记忆库**（验证实际数据）：记忆查询工具 确认该机制在真实操作里的表现。
3. **宿主 API 文档**（补充理解）：用 宿主 API 查询工具 查端点行为；单篇文档用 宿主 API 请求工具 读，失败就跳。

## 常见误判点

- **占位符 ≠ 最终结果**：`derived` 来源的字段是临时值，会被后续机制覆盖；`user` 来源永不被覆盖。看到陌生值先查来源列再判断。
- **工具轮次不计入"用户消息"计数**：模型调工具不会推进升级计数；只有真实用户发消息才触发。
- **Studio/桌面端会话不回写**：Ekko Studio 等平台的"新消息"不一定追加进 Ekko 会话的消息表，`user_msg_count` 可能恒为 1，升级永远等不到触发点。定因时区分"没收到第 2 条用户消息"（查该会话消息行数）与"收到但升级失败"（查日志）。
- **inode 三时间戳会被写操作整体重置**：patch/write 类工具按"换 inode"写盘，birth/change/modify 重置到同一刻——三戳相等只证明"那刻有写操作"，不等于创建时间。
- **升级通路会因 provider 鉴权整体挂掉**：阶段 2 模型调用走辅助 LLM 的 provider（`auto`=主 provider）。主 provider 401/不可用且没配 fallback 时直接失败，标题停留占位符。日志特征：`Title generation failed` + auth error。看到"标题卡占位符+以前能改"先 grep 日志确认升级调用是失败还是根本没触发，再定因。修法是修 provider 鉴权或配 fallback，不是改名机制。
- **`/title` 命令**：手动命名后，自动机制永久跳过该会话；配置 `auxiliary.title_generation.enabled=false` 可全局关闭自动命名。

## 标题语言

- **默认"跟随用户消息语言"，不是默认英文**：未设 `language` 时，标题提示词用"同用户消息语言"；设了则用固定语言。用户消息是中文却出英文标题 = 模型没遵守同语言规则（生成不稳定），不是机制故意切英文；修法 = 显式锁 `language: zh`，别指望默认跟随。
- **配置即时读取，改完无需重启**：每次起标题时重新读盘，不缓存进进程；改配置对新会话的下次起标题立即生效。
- **验证法（子代理冷跑）**：起 N 个子代理各自新开会话、各发一条中文消息（换话题避免标题撞车触发 `#N` 后缀混淆）、发完即止，各自回报 session id + title + title_source；父代理拿这批 id 回查 state.db 逐条核对中文，不直接信子代理自报的 `title_is_chinese`。

## 定因时查哪一层（逐层下钻）

标题/机制卡住时按此顺序定位，别凭记忆猜：

1. **查当前会话标题来源**：`derived`=占位符待升级，`llm`=已升级，`user`=手动定死。
2. **查该会话用户消息数**：若 user 行数=1，升级触发条件未满足（区分"没触发"与"触发后失败"）。
3. **grep 日志定升级是否失败**：有 `Title generation failed` + auth error = provider 通路挂；无日志 = 升级没被触发。
4. **给操作**：按定因给 `/title`、配 fallback、修鉴权或"等下一条真实用户消息"，每句带证据。

## 回复格式

定因后给：机制链路（表）+ 当前状态（哪一步卡住/为何没推进）+ 用户能做的操作。不凭记忆猜，每句有源码或 DB 证据支撑。

## 列名陷阱

- `messages` 表的会话时间戳列叫 `timestamp`（epoch 秒），不是 `created_at`——按直觉写 `created_at` 会 `no such column` 报错。
- 定位"当前"会话别靠猜：按 `sessions.last_activity_at` 倒序取最近几条与 UI 标题对照，再取该 `id` 做取证。
