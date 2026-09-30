---
name: ekko-memory-curation
description: "Ekko 结构化记忆卡写入 SOP：kind 判源、最小验证、原子批处理、声明式条目。写 Ekko 记忆卡前用；不用于任务进度或临时事实。"
version: 1.0.0
author: Ekko Agent
license: MIT
metadata:
  keywords:
    - "ekko"
    - "memory"
    - "curation"
    - "sop"
    - "kind"
---

# Ekko 结构化记忆卡写入 SOP

## kind 判源（先定 kind，不靠感觉）

- 「跟人走的偏好/习惯/个人信息」→ 对应 user 侧 kind（`general_preference`、`workflow_preference`、`habit_routine`、`personal_relationship` 等）。
- 「跟机器/工具走的环境事实/工具技巧/实测数据」→ 对应 machine 侧 kind（`environment_fact`、`tool_preference`、`project_context` 等）。
- 拿不准 → 查同 kind 下已有条目怎么归的，照同类；无同类，事实主体明显是用户意愿才进 user 侧。

## 写前检查（按序做）

1. 查现有条目：`memory_search(kind=...)` 读目标 kind 当前条目；被涵盖/矛盾/近似重复 → `supersede` 而不是 `create`；`create` 只给真没有的。
2. 最小验证：把已有条目数据当判断依据时，先验（读文件/跑命令/看一行日志）；与实测矛盾 → `supersede` 成实测值，条目内标注验证条件（版本/参数）。
3. 预算：目标 kind 条目数量已多时 → 同一次 `memory_write` 的 `operations` 数组里 `delete` 旧条目 + `create` 新条目（原子提交），不拆成两次。

## 条目格式

- 声明式事实，非祈使指令：「用户偏好 X」✓，「总是做 Y」✗。
- 一条一个事实；无日期、无叙事、无任务编号。
- 量化值只写实测过的，标测量条件；未实测的不写。
- 无法验证的条目标「(未验证)」，不把它当判断依据。
- `node.title` 与 `node.content` 语言跟随用户证据语言（中文证据用中文，英文证据用英文）。

## 不写（分流到别处）

- 任务进度/完成日志/临时 TODO → 工作区文件。
- 可复用流程/工作流/踩坑 → 技能管理工具 create。
- 易重新发现的事实、常识 → 不写。

## 写

- 单个 kind 一次 `memory_write` 调用；跨两个 kind 拆两次调用。
- 多个变更（删旧 + 加新）用一次 `operations` 数组批量，原子提交。
- `create` 时 `scope.type` 必须是 `profile` / `context` / `session` 之一，按宿主授权范围填。
- 写完读回结果确认；`create` 被拒（重复/超范围）→ 带上 delete 重发一次。

## 何时不用本技能

- 用户只是问记忆内容 → 直接答，不触发写入流程。
- 信息不满足「持久 + 不易重新发现」→ 不写。
- 纯会话内临时状态 → 不写，用工作区文件。
