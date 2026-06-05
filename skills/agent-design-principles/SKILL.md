---
name: agent-design-principles
description: 构建 AI agent 的架构原则。覆盖：事件驱动的输入与实现、handler 的并发/打断、State 与 Deps 分离、分 section 的 prompt、runtime/sandbox/session 分离、注意力管理、工具失败处理、session 树。当用户构建/重构 AI agent、设计 agent 框架、处理并发打断、组织 system prompt 时使用。
---

# Agent 架构原则

## 1. prefer event-driven（input 和实现都是）

每个信号（用户输入、工具返回、超时、外部状态变化）建模成一个 event，agent 就是 `event → handler`。漂亮的副产物：**context 是 event 流的一个 projection**——历史、压缩、版本都从同一份 event 推导，不各存一份。

## 2. 每个 handler 显式声明并发 / 打断行为

新 event 来时这个 handler 该被取消、排队、还是并行？答案写进定义，别埋实现里（可被打断 / 串行 / 并行 / 防抖，按场景选）。说不清它的并发语义 = 还没设计它。

## 3. State 与 Deps 严格分离

| 类别 | 含义 | 特征 |
| --- | --- | --- |
| State | 业务事实 | 可序列化、要锁、要持久化 |
| Deps | 外部句柄 + 配置 | 含 socket / future、不持久化、重启重建 |

判断：能不能 `json.dumps(state)`？能 = 分对了。

## 4. prefer 分 section 的 prompt，而不是一整块模板

把 system prompt 拆成独立 section，别堆成大模板。拆开才能按 section 做 A/B testing、自由组合、按上下文动态注入或裁剪——一整块模板全做不了。

## 5. runtime / sandbox / session 三者分离

参考 Anthropic Managed Agents（"decouple the brain from the hands"）：

- **runtime（大脑）**：跑 loop，无状态，可随时重启。
- **sandbox（手）**：隔离环境跑 tools，不持凭证。
- **session**：append-only，独立持久化。

任一部件崩了不连累其它（sandbox 炸了重启即恢复），且 session 天然是 trace / 可重放。

## 6. 渐进式暴露注意力，而不是一次性 offload context

模型被 train 成用很多短 turn 思考。别一上来糊一大坨 context，用机制（skill、关键词触发的 instruction、system reminder、hook）让**注意力**一点点聚焦到当下该看的。重点是渐进暴露注意力，不是渐进暴露 context 本身。

## 7. 给能力：直接定义 tool，还是封装成 CLI + skill

两条路：① 直接定义一堆 tools；② 封装成一个 CLI，再用 skill 教 agent 用。tool 的 UI 展示 / traceability 更好；CLI + skill 能拼 pipeline、任意组合，且不受"工具数量不能太多"的限制。信号：过多 tool 做相似的事 → 收成一个 CLI 更划算。

## 8. 工具 / hook 失败是数据，不是异常

工具抛错 → 转成 `isError` 回灌模型让它自纠，别 crash loop；hook 异常同理 catch + 上报。约定：工具作者**直接 throw**，由 loop 统一转成模型可见的失败信号。把失败变数据的 agent 会自纠，让失败抛出去的会停摆。

## 9. session 存成 event-sourced tree，不是 message list

在 #5 基础上：每条历史是带 `parentId` 的不可变 entry，"当前位置"只是 `leafId` 指针，一切变更都是 append 一个 entry。undo / branch / fork / 编辑旧消息重跑全塌缩成"append + 移指针"，线性 context 只是这棵树的纯投影；fork = 选择性 replay，零拷贝零 aliasing。

## 10. transcript 消息类型 ≠ provider 消息类型

内部消息（带 UI / 持久化的富字段、自定义 role）和发给 LLM 的 provider 消息**分开**，中间一个**单点转换** seam。UI / 存储想带多少额外信息都行，模型永远看干净过滤版，两边各自演进不互相污染。
