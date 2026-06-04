---
description: Load all Deslop rules (5 skills)
argument-hint: 想让 agent deslop 哪些东西（留空 = 扫整个 codebase 提建议）
---

# Load Deslop Rules

立即用 Skill 工具加载下面五个 skill，并在本次会话一直遵守，不要问用户：

- `deslop:agent-design-principles`
- `deslop:code-design-principles`
- `deslop:typing-and-class-design`
- `deslop:naming-principles`
- `deslop:project-structure`

加载完后：

- 用户给了具体目标（$ARGUMENTS）→ 按 Deslop 原则处理它。
- 没给 → 浏览整个 codebase，按这些原则找出问题并提出建议。
