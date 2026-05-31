# Deslop

对抗 AI "slop" 代码的工程原则,给 coding agent(Claude Code / Cursor / Codex / Gemini CLI)当可复用指令包。

## Skills

| Skill | 作用 |
| --- | --- |
| [agent-design-principles](skills/agent-design-principles/SKILL.md) | 生产级 AI agent 架构:event-driven、工具调用、上下文管理、持久化 |
| [code-design-principles](skills/code-design-principles/SKILL.md) | 写代码的决策:控制流、数据结构、错误处理、抽象时机、可测性 |
| [naming-principles](skills/naming-principles/SKILL.md) | 变量 / 函数 / 类型 / 文件命名 |
| [project-structure](skills/project-structure/SKILL.md) | 目录拆分、文件命名、公开 API 设计 |

## Hooks

机械可判定的规则做成 hook,在工具调用前强制拦截——不占 skill context、不被 compact 影响。

| Hook | 拦截 |
| --- | --- |
| `check-bash` | `npm/yarn install` → pnpm;`pip/poetry` → uv;`mkdir` 顶层垃圾目录 |
| `check-write` | `Write` 到顶层 `utils/helpers/common/misc/shared/lib/` |
| `session-context` | 每次 session 启动 / compact 后注入 `session-rules.md` 里的铁律 |

判断标准:能 regex 机械判定 → hook;需要看语义 → 留在 skill。

## 安装

**Claude Code plugin**(完整功能:skills + hooks)

```text
/plugin marketplace add xuanruli/Deslop
/plugin install deslop@deslop
```

更新:`/plugin marketplace update deslop` → `/reload-plugins`。

**npx skills / gh skill**(仅 skills,跨客户端;不含 hooks)

```bash
npx skills add xuanruli/Deslop -g
gh skill install xuanruli/Deslop --scope user
```

## 结构

```
.claude-plugin/   plugin.json + marketplace.json
skills/           4 个 SKILL.md(自动发现)
hooks/            hooks.json + hooks-cursor.json + 脚本 + session-rules.md
```

## License

MIT
