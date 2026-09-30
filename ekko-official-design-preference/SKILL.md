---
name: ekko-official-design-preference
description: 记忆/技能/配置的设计决策须遵循 Ekko 官方规范（结构化卡片+kind 枚举），非 Hermes MEMORY.md/USER.md 纯文本设计。
version: 1.0.0
author: Ekko Agent
license: MIT
metadata:
  keywords:
    - "ekko"
    - "official design"
    - "memory card"
    - "kind enum"
    - "preference"
---

# Ekko Official Design Preference

## Core Rule
All memory/skill/config design decisions must follow **Ekko's official specification** (structured memory cards with kind enums, declarative short sentences, atomic write batches). This does NOT refer to Hermes's MEMORY.md/USER.md plain-text design.

## Evidence
- Ekko memory schema: `memory_nodes` table with domain, categoryPath, type (kind enum), key, revision, status fields
- Ekko skill schema: SKILL.md with YAML frontmatter, metadata.keywords (English ASCII), supported requests/boundaries
- The two systems (Ekko Agent vs Hermes Agent) run in parallel with separate asset domains; conflating their design specs is incorrect

## When to Apply
- Before creating/modifying memory entries → verify against Ekko's structured card schema, not Hermes
- Before creating/modifying skills → follow Ekko's SKILL.md frontmatter spec (keywords English ASCII only, no translations)
- Before configuring → check which system owns the config (Ekko config dir vs Hermes config dir)
