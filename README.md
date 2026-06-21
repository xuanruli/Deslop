# Deslop

对抗 AI "slop" 代码的工程原则,给 coding agent(Claude Code / Cursor / Codex / Gemini CLI)当可复用指令包。

## Skills

| Skill | 作用 |
| --- | --- |
| [agent-design-principles](skills/agent-design-principles/SKILL.md) | 生产级 AI agent 架构:event-driven、工具调用、上下文管理、持久化 |
| [code-design-principles](skills/code-design-principles/SKILL.md) | 写代码的决策:控制流、数据结构、错误处理、抽象时机、可测性 |
| [typing-and-class-design](skills/typing-and-class-design/SKILL.md) | 类型与类设计:泛型、discriminated union、Result、class vs function、为成长而设计 |
| [naming-principles](skills/naming-principles/SKILL.md) | 变量 / 函数 / 类型 / 文件命名 |
| [project-structure](skills/project-structure/SKILL.md) | 目录拆分、文件命名、公开 API 设计 |

## Hooks

机械可判定的规则做成 hook,在工具调用前强制拦截——不占 skill context、不被 compact 影响。

| Hook | 拦截 |
| --- | --- |
| `check-bash` | `pip/poetry` → uv;`mkdir` 顶层垃圾目录 |
| `check-write` | `Write` 到顶层 `utils/helpers/common/misc/shared/lib/` |
| `session-context` | 每次 session 启动 / compact 后注入 `session-rules.md` 里的铁律 |

判断标准:能 regex 机械判定 → hook;需要看语义 → 留在 skill。

## 安装

**Claude Code plugin**(完整功能:skills + hooks)

```text
/plugin marketplace add xuanruli/Deslop
/plugin install deslop@deslop
```

**建议开启自动更新**:进 `/plugin` → 选中 `deslop` marketplace → Enter → Enable auto-update。第三方 marketplace 默认是关的,开了之后每次启动自动拉最新。

手动更新:`/plugin marketplace update deslop` → `/reload-plugins`。

**Cursor**(本地装,含 skills + `/deslop` 命令)

Cursor 没有"指向任意 repo 一行装"的去中心化机制(官方 marketplace 要审核),用本地安装:

```bash
git clone https://github.com/xuanruli/Deslop ~/.cursor/plugins/local/deslop
```

然后重启 Cursor 或 `Developer: Reload Window`。skills 和 `/deslop` 命令即生效。

> hooks(uv 拦截、session 铁律注入)因 Cursor 与 Claude Code 的 `hooks.json` schema 不同,不随本地装自动生效。需要的话把 `hooks/hooks-cursor.json` 的内容合并进你的 `~/.cursor/hooks.json`。

**npx skills / gh skill**(仅 skills,跨客户端;不含 hooks / 命令)

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
