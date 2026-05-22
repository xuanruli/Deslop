# Deslop

一组对抗 AI "slop" 代码的工程原则 skills，给 coding agent（Claude Code / Cursor / Codex / Gemini CLI 等）当作可复用的指令包。

## 包含的 skills

| Skill | 作用 |
| --- | --- |
| [`agent-design-principles`](skills/agent-design-principles/SKILL.md) | 构建生产级 AI agent 的架构原则：event-driven、工具调用、上下文管理、持久化等。 |
| [`code-design-principles`](skills/code-design-principles/SKILL.md) | 写代码时的具体决策原则：控制流、数据结构、错误处理、抽象时机、可测性。 |
| [`naming-principles`](skills/naming-principles/SKILL.md) | 变量 / 函数 / 类型 / 文件的命名通用原则。 |
| [`project-structure`](skills/project-structure/SKILL.md) | 项目结构与代码组织通用原则：目录拆分、文件命名、公开 API 设计。 |

## 安装

### 1. 作为 Claude Code plugin（推荐，能被 `/plugins` 统一管理 + 可选 auto-update）

```bash
# 在 Claude Code 里
/plugin marketplace add xuanruli/Deslop
/plugin install deslop@deslop
```

之后想自动跟随上游：进 `/plugin` → 选中 `deslop` marketplace → Enter → Enable auto-update。

更新：

```text
/plugin marketplace update deslop
/reload-plugins
```

### 2. 作为 npx skills（跨 Cursor / Codex / Gemini CLI / Claude Code）

```bash
# user 级别（推荐，跨 project 复用）
npx skills add xuanruli/Deslop -g

# 或 project 级别
npx skills add xuanruli/Deslop
```

更新：

```bash
npx skills check
npx skills update
```

### 3. 作为 `gh skill`

```bash
gh skill install xuanruli/Deslop --scope user
```

更新：

```bash
gh skill update --all
```

## 结构

```
Deslop/
├── .claude-plugin/
│   ├── plugin.json          # Claude Code plugin 清单
│   └── marketplace.json     # 让 repo 自身成为 marketplace
├── skills/
│   ├── agent-design-principles/SKILL.md
│   ├── code-design-principles/SKILL.md
│   ├── naming-principles/SKILL.md
│   └── project-structure/SKILL.md
└── README.md
```

`skills/` 目录由 Claude Code plugin 按约定自动发现，无需在 `plugin.json` 里显式声明；`npx skills` 与 `gh skill` 直接扫描 `**/SKILL.md`，所以同一份内容三条安装路径同时生效。

## 版本

当前：`0.1.0`（初始版本，未正式发布）。

后续每次发布会打对应 git tag（例如 `v0.1.0`），用户可通过 tag pinning 锁定版本。

## License

MIT
