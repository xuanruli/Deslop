---
name: agent-design-principles
description: 构建生产级 AI agent 的架构原则。覆盖：消息历史、工具调用、上下文管理、打断语义、外部资源恢复、持久化、流式多消费者、prompt cache、错误自纠正。当用户构建/重构 AI agent、设计 agent 框架、添加新工具、处理长对话、做 agent 可靠性工程时使用。
---

# Agent 架构原则

## 1. event-driven，不要 workflow

Agent 输入是异步信号流（用户说话、工具返回、context 超限、外部状态变化、定时器）。每个信号建模成 `Event`，每个 Event 配一个 handler。handler 之间通过共享 state 通信。

线性 workflow / DAG 描述不动"用户上一句话还没回答完时打断 + 新指令"这种场景。

## 2. handler 声明调度策略

每个 handler 显式声明对并发/打断的需求：

| 策略 | 何时用 |
|---|---|
| Interruptable | 新事件取消当前的（LLM 推理、context 压缩） |
| Sequential | 排队（写消息历史） |
| Parallel | 互不干扰（日志、metrics 上报） |
| Debounced(ms) | 抖动期合并（snapshot、UI 更新） |

判断：每个 handler 能不能用一句话描述它的并发行为？

## 3. State 与 Deps 严格分离

| 类别 | 含义 | 特征 |
|---|---|---|
| State | 业务事实 | 可序列化、要锁、要持久化 |
| Deps | 外部句柄 + 配置 | 含 socket / future、不持久化、重启重建 |

判断：能不能 `json.dumps(state)`？能 = 分对了。

## 4. 消息历史是 immutable + versioned

不要 `messages: list[Message]` 原地改。每次 reduce/compress 产生新版本，老版本保留：

```python
class Context:
    versions: dict[int, list[NodeRef]]
    def reduce(self, reducer) -> int:
        new_v = self.version + 1
        self.versions[new_v] = reducer(self.versions[self.version])
        return new_v
```

支持撤销摘要、对比策略、debug 重放、审计。

## 5. 工具错误是给 LLM 的反馈，不是异常

```python
if exit_code != 0:
    raise ModelRetry(stderr)     # LLM 看到错误，自己决定下一步
```

LLM 自动具备纠错能力（看到 "no such file" 会换路径）。判断：错误信息里 LLM 能不能看到足够上下文决定下一步？

## 6. 上下文压缩分级，廉价方案先

```python
if tokens > 100k:
    apply(compress_tool_outputs)   # 不调 LLM
if tokens > 200k:
    apply(summarize_history)       # 调 LLM
```

90% 情况下廉价方案就够。无脑调 LLM 摘要 = 丢细节 + 烧钱。

## 7. 外部资源用 id + fallback chain 恢复

外部资源（沙箱、容器、外部 session）会在 agent 不知情时消失。内存只持有 future，持久化的是 id：

```python
def get_or_create():
    if snapshot_id: return create_from(snapshot_id)   # 1. 最新快照
    if resource_id:                                    # 2. 复用
        try:
            r = attach(resource_id)
            if healthy(r): return r
        except NotFound: pass
    return create_fresh()                              # 3. 兜底
```

判断：进程被 kill 重启，能恢复到"用户感知不到中断"的状态？

## 8. mutation 后做 snapshot，事件驱动 + 防抖

不要定时 snapshot（没变化时浪费、变化太快又跟不上）。每次会改外部状态的工具调用后 `send(SnapshotRequired())`，handler 用 Interruptable 自动防抖（连续 N 次只跑最后一次）。

## 9. LLM 调用：短超时检测卡死 + 长超时检测异常 + 重试

```python
for attempt in range(3):
    task = run_llm(...)
    # 短超时：30s 内必须有第一个 token
    first_token = wait_first_delta(task, timeout=30)
    if not first_token:
        cancel(task); continue
    # 长超时：完整推理 5min
    try: return await wait_for(task, timeout=300)
    except TimeoutError: cancel(task); continue
```

只有一个 timeout 时，要么短杀掉慢但正常的请求，要么长拖住卡死请求。

## 10. 打断后保留"已发生的事实"

用户打断 LLM 推理：
- 用户输入 → 必须保留
- AI 已说出来的部分（哪怕没说完）→ 保留
- 已成功的工具调用 → 保留
- 调了一半的工具 → 清掉（孤儿 tool_call）

实现：handler 的 `try/finally` 在 `finally` 里把"已收到的部分"写回 state。

## 11. 给 LLM 发请求前 sanitize 历史

跨边界（→ LLM API）前最后一道关卡：

```python
def get_messages_for_llm(self):
    parts = self.collect()
    parts = remove_orphan_tool_calls(parts)        # 防 API 报 tool_use_id not found
    parts = remove_unavailable_tool_calls(parts)   # 工具配置变更后的清理
    parts = inject_cache_marker(parts)             # 自动 prompt cache
    return parts
```

每个写入点校验 = 容易漏。出口处一次 = 覆盖所有写入路径。

## 12. 自动注入 prompt cache marker

LLM provider 的 prompt caching（Anthropic CachePoint、OpenAI prefix cache）能省 50-90% 成本。**调用方不该手动管**——在序列化消息时自动在最后一个 user prompt 后注入。

判断：cache hit 率接近 0% → 没自动注入。

## 13. 工具集动态构建

不要把所有工具一股脑注册给 LLM。按 session 配置过滤：

```python
def build_tools(deps):
    tools = [read_file, grep]
    if deps.write_enabled: tools += [write_file, edit, shell]
    if deps.browser_enabled: tools += [attach_browser]
    return tools
```

无关工具浪费 token + LLM 会乱调用 + 失败浪费 turn。

## 14. 信息密度分层（LLM 看精简，UI 看完整）

工具返回的"给 LLM 看"和"给前端看"应该不同：

```python
async def read_file(ctx, path):
    data = await sandbox.read(path)
    output_collector[ctx.tool_call_id] = data    # 完整结构存
    return data.format_summary()                  # 给 LLM 看精简
```

UI 通过 RPC 用 tool_call_id 拉完整数据。LLM 省 token，UI 看完整内容。

## 15. 流式输出多路复用（partition）

一次推理被多个 consumer 用（TTS、UI、日志、metrics），按 partition 分发：

```python
handle.insert("voice_chunks", text)      # 给 TTS
handle.insert("ui_messages", formatted)  # 给 UI
async for c in handle.stream("voice_chunks"): tts.send(c)
async for m in handle.stream("ui_messages"): ui.show(m)
```

不要每个 consumer 都拉一次推理，也不要让所有 consumer 读同一 stream 自己过滤。

## 16. 多模态输入按时间戳合并 + 消费 pending

多路输入（用户麦克风、桌面音频、截图、外部事件）各自有时间戳。提交 turn 时：
- 收集所有路的 finalized + pending
- 按时间戳排序
- 用 `last_consumed_time` 去重

只看 finalized 不看 pending = 用户话还没说完就触发 → 截断。

## 17. 跨 turn 因果排序用 event flag

用户连发多条消息时，回复消息的写入顺序可能错乱。用 `asyncio.Event` 表达"上一轮是否完成"：

```python
class OnUserTurn:
    async def run(self, ctx):
        try:
            await wait_for(deps.inference_done.wait(), timeout=30)
        except TimeoutError:
            deps.inference_done.set()    # 兜底强制 unblock，不死锁
        deps.inference_done.clear()
        try: await invoke_llm()
        finally: deps.inference_done.set()
```

必须有 timeout 兜底——上一轮挂了/被 cancel 没 set 就死锁。

## 18. 心跳维持外部资源

外部资源（容器、连接）通常有 idle timeout（10-15min），但 agent 会话可能数小时且中间静默。每 60s 发心跳。

进程死了心跳停 → 资源自动回收 = 顺带的清理保险。

## 19. 劫持 framework 而不是 fork

framework 约束太多但音频/UI/调度有用 → 用 noop 实现劫持关键节点（STT/LLM 设为 noop，重写 hook），保留 framework 的其他部分。

fork 维护成本巨大。判断：你重写了 framework ≥80%？该考虑不用它了。

## 20. 工具不要直接 raise，所有异常翻译成 ModelRetry

```python
async def my_tool(ctx, ...):
    try:
        return await do_work()
    except Exception as e:
        raise ModelRetry(str(e))     # LLM 看得到，自己决定怎么办
```

普通 raise → agent 进程崩 → 整个会话死。`ModelRetry` → LLM 自纠 → 会话继续。

---

## 决策模板

设计/重构 agent 时按顺序问自己：

1. 输入信号有几路？每路的调度策略是什么？
2. State / Deps 分清了？State 能序列化？
3. 消息历史需要版本化？需要 → immutable DAG
4. context 超限的分级降级是什么？至少两级
5. 工具失败如何反馈给 LLM？ModelRetry 类机制
6. 外部资源用 id + 多级 fallback 恢复了？
7. mutation 后 snapshot + 防抖了？
8. LLM 调用有短超时 + 长超时 + 重试？
9. 打断后部分输出正确写回 state？
10. 跨 turn 因果顺序保证了？
11. prompt cache marker 在边界自动注入？
12. 多模态输入按时间戳合并 + 消费 pending？
13. 进程崩了能恢复到用户无感？

---

## 反模式速查

- `messages: list[Message]` 原地改（无版本）
- 上下文压缩只有"超了就调 LLM 摘要"
- 工具失败抛 Exception 让 agent 崩
- LLM 调用只有一个 timeout
- 外部资源句柄只在内存（重启就丢）
- 定时 snapshot / 完全不做 / 在错误时机做
- 用户打断后已收到的 token 没写入历史
- 用户连发消息回复顺序乱
- 多模态只看 finalized 不消费 pending
- 工具集固定不随 session 配置变化
- LLM API 偶发 "tool_use_id not found"
- UI 和 TTS 各自重复解析同一份 stream
- prompt cache hit 率接近 0%
- 沙箱 idle 后业务期望它还在线
- agent 崩了沙箱继续烧钱
- handler 的 wait/acquire 没有 timeout 兜底
- workflow 框架硬塞 event-driven 场景
- LLM 看到的工具描述和实际可用工具不一致
- 业务代码到处手写"准备 LLM 消息"的 boilerplate
