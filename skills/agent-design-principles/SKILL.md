---
name: agent-design-principles
description: 构建 AI agent 的架构原则。覆盖：事件驱动的输入与实现、handler 的并发/打断、State 与 Deps 分离、分 section 的 prompt。当用户构建/重构 AI agent、设计 agent 框架、处理并发打断、组织 system prompt 时使用。
---

# Agent 架构原则

## 1. prefer event-driven（input 和实现都是）

每个信号（用户输入、工具返回、超时、外部状态变化）建模成一个 event，agent 就是 `event → handler`。

一个漂亮的副产物：**context 是 event 流的一个 projection**——历史、压缩、版本都从同一份 event 推导出来，而不是各自维护一份状态。

## 2. 每个 handler 显式声明并发 / 打断行为

新 event 来时，这个 handler 是被取消、排队、还是并行跑？答案写进它的定义，别埋在实现里靠运气（可被打断 / 串行 / 并行 / 防抖，按场景选）。

说不清一个 handler 的并发语义 = 还没设计它。

## 3. State 与 Deps 严格分离


| 类别    | 含义        | 特征                          |
| ----- | --------- | --------------------------- |
| State | 业务事实      | 可序列化、要锁、要持久化                |
| Deps  | 外部句柄 + 配置 | 含 socket / future、不持久化、重启重建 |


判断：能不能 `json.dumps(state)`？能 = 分对了。

## 4. prefer 分 section 的 prompt，而不是一整块模板

把 system prompt 拆成独立 section（Claude Code 就是这么做的），别堆成一个大模板。

拆开才能：按 section 做 A/B testing、自由组合不同 section、按上下文动态注入或裁剪某些 section。一整块模板这些全做不了。

## 5. runtime / sandbox / session 三者分离

参考 Anthropic Managed Agents（"decouple the brain from the hands"）：

- **runtime（大脑）**：跑 loop，无状态，可随时重启。
- **sandbox（手）**：隔离环境跑 tools，不持凭证。
- **session**：append-only event log，独立持久化。

好处：任一部件崩了不连累其它（sandbox 炸了重启即恢复），且 session log 天然就是 trace / 可重放。

## 6. 渐进式暴露注意力，而不是一次性 offload context

模型被 train 成用很多短 turn 思考。别一上来糊一大坨 context，用机制让**注意力**一点点聚焦到当下该看的——skills, keyword triggered instruction, system reminder, hook等类似的设计。重点是渐进暴露**注意力**，不是渐进暴露 context 本身。

## 7. 给能力：直接定义 tool，还是封装成 CLI + skill

两条路：① 直接定义一堆 tools；② 把能力封装成一个 CLI，再用 skill 教 agent 用它。

- **tool**：UI 展示和 traceability 更好。
- **CLI + skill**：能拼 pipeline、任意组合参数，且不受"工具数量不能太多"的限制。

信号：当过多 tool 在做相似的事，把它们收成一个 CLI 往往更划算。