# Deslop

一组对抗 AI "slop" 代码的工程原则 skills，给 coding agent（Claude Code / Cursor / Codex / Gemini CLI 等）当作可复用的指令包。

## 包含的 skills


| Skill                                                                | 作用                                                 |
| -------------------------------------------------------------------- | -------------------------------------------------- |
| `[agent-design-principles](skills/agent-design-principles/SKILL.md)` | 构建生产级 AI agent 的架构原则：event-driven、工具调用、上下文管理、持久化等。 |
| `[code-design-principles](skills/code-design-principles/SKILL.md)`   | 写代码时的具体决策原则：控制流、数据结构、错误处理、抽象时机、可测性。                |
| `[naming-principles](skills/naming-principles/SKILL.md)`             | 变量 / 函数 / 类型 / 文件的命名通用原则。                          |
| `[project-structure](skills/project-structure/SKILL.md)`             | 项目结构与代码组织通用原则：目录拆分、文件命名、公开 API 设计。                 |


## 包含的 hooks

skills 描述"应该怎么做"，但有些纯 deterministic 的偏好（比如"用 pnpm 不用 npm"）不依赖判断、不需要让 agent 每次去 read skill —— 直接做成 `PreToolUse` hook 在工具调用前拦截，**省 context 而且强制生效，不会被 compact 影响**。

| Hook | 拦截条件 | 来源规则 |
| --- | --- | --- |
| `hooks/check-bash` | `npm install` / `yarn add` / `npm ci` / `npm update` … | `project-structure` 原则 13（→ pnpm） |
| `hooks/check-bash` | `pip install` / `poetry add` / `python -m pip install` … | `project-structure` 原则 13（→ uv） |
| `hooks/check-bash` | `mkdir utils\|helpers\|common\|misc\|shared\|lib`（顶层） | `project-structure` 原则 1 + `naming-principles` 反模式 #1 |
| `hooks/check-write` | `Write` 到顶层 `utils/helpers/common/misc/shared/lib/` 下 | 同上 |

**什么样的规则适合 hook？**

- ✅ 能用 regex / 字符串匹配机械判断的（命令名、文件路径、固定模式）
- ❌ 需要看语义、看上下文、需要"判断到底算不算违反"的 → 留在 skill 里

效果上：agent 写下 `npm install react` → hook 立刻 block 并提示 `pnpm add react`，agent 自动改命令重试；这些规则**不再占 skill 的 token**。

`hooks/hooks.json` 由 Claude Code 按约定自动发现；Cursor 用 `hooks/hooks-cursor.json`。

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
├── hooks/
│   ├── hooks.json           # Claude Code hook 注册
│   ├── hooks-cursor.json    # Cursor hook 注册（schema 不同）
│   ├── check-bash           # PreToolUse(Bash)：拦 npm/pip/mkdir 顶层垃圾目录
│   └── check-write          # PreToolUse(Write)：拦顶层垃圾目录下的写入
└── README.md
```

`skills/` 与 `hooks/` 都由 Claude Code plugin 按约定自动发现，无需在 `plugin.json` 里显式声明；`npx skills` 与 `gh skill` 直接扫描 `**/SKILL.md`，所以同一份内容三条安装路径同时生效（hook 仅在 plugin 安装时生效，npx skills 路径不带 hook）。

## 版本

当前：`0.1.0`（初始版本，未正式发布）。

后续每次发布会打对应 git tag（例如 `v0.1.0`），用户可通过 tag pinning 锁定版本。

## License

MIT