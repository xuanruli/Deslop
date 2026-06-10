# Example：主函数从 procedure 重构成 orchestrator

一个"越写越长、靠抽 helper 补救"的主函数，怎么改成"薄编排 + 设计好的 stage"。

## Before —— 一个长 procedure

症状：混了配置解析 / IO / 编排 / 清理；`delete_after` 标志穿过嵌套 try/finally；`dry_run` 深埋；preuploaded vs build 靠两处 if + 两个变量拼。

```python
def run(dry_run: bool = False) -> int:
    _setup_logging()
    today = _dt.date.today().isoformat()
    yesterday = (_dt.date.today() - _dt.timedelta(days=1)).isoformat()
    _get_env("ANTHROPIC_API_KEY", required=not dry_run)

    preuploaded_file_id = os.environ.get("ENV_FILE_ID") or None
    if preuploaded_file_id:
        env_payload = b""
    else:
        env_payload = _build_env_payload()

    if dry_run:
        plan = { "today": today, "yesterday": yesterday, ... }   # 一大坨内联
        print(json.dumps(plan, indent=2, default=str))
        return 0

    client = Anthropic()
    agent_id, env_id = _resolve_agent_and_env(client)

    if preuploaded_file_id:
        file_id = preuploaded_file_id
        delete_after = False
    else:
        uploaded = _upload_env_file(client, env_payload)
        file_id = uploaded.id
        delete_after = True

    try:
        resources = [ {"type": "file", "file_id": file_id, ...} ]
        memory_store_id = _resolve_memory_store_id(client)
        if memory_store_id:
            resources.append({ "type": "memory_store", ... })
        session = client.beta.sessions.create(agent=agent_id, environment_id=env_id, resources=resources, ...)
        kickoff_text = ( f"今天是 {today}..." )   # 内联拼字符串
        _arm_timeout(HARD_TIMEOUT_SECONDS)
        try:
            with client.beta.sessions.events.stream(session.id) as stream:
                client.beta.sessions.events.send(session.id, events=[...])
                _drain(stream, client=client, session_id=session.id)
        except _Timeout:
            delete_after = False   # 翻标志保留文件 debug
        finally:
            _disarm_timeout()
    except Exception:
        delete_after = False
        raise
    finally:
        if delete_after:
            _try_delete_file(client, file_id)
    return 0
```

## After —— 薄编排 + 设计好的 stage

> 命名：脚本模块的函数没人从外部 import，不必每个都挂 `_`——`_` 只留给真正的内部 helper。

`run` 读起来像目录：收集配置 →（要 plan 就打印返回）→ 执行。

```python
def run(client: Anthropic, dry_run: bool = False) -> int:
    setup_logging()
    config = load_config(dry_run=dry_run)          # 纯：一次性收集输入
    if dry_run:
        print(json.dumps(plan(config), indent=2, default=str))   # plan / execute 分开
        return 0
    return execute(client, config)
```

env 来源做成 typed，`delete_after` 标志消失——"要不要删"是来源的属性：

```python
@dataclass(frozen=True)
class EnvSource:
    file_id: str | None       # 有值 = preuploaded（用完不删）
    payload: bytes | None     # 有值 = 待上传（用完删）

    @classmethod
    def resolve(cls) -> "EnvSource":
        if pre := os.environ.get("ENV_FILE_ID"):
            return cls(file_id=pre, payload=None)
        return cls(file_id=None, payload=build_env_payload())
```

文件生命周期用 context manager，不用 flag + 嵌套 try/finally：

```python
@contextmanager
def provisioned_env_file(client: Anthropic, source: EnvSource) -> Iterator[str]:
    if source.file_id:                 # preuploaded：用完不删
        yield source.file_id
        return
    uploaded = upload_env_file(client, source.payload)
    try:
        yield uploaded.id
    finally:
        delete_file(client, uploaded.id)
```

纯函数拼 plan / resources / kickoff（只收数据返数据），`execute` 在边界处组合：

```python
def plan(c: RunConfig) -> dict: ...
def build_resources(file_id, memory_id) -> list[dict]: ...
def build_kickoff(c: RunConfig) -> str: ...

def execute(client: Anthropic, c: RunConfig) -> int:
    agent_id, env_id = resolve_agent_and_env(client)
    with provisioned_env_file(client, c.env_source) as file_id:
        resources = build_resources(file_id, resolve_memory_store_id(client))
        session = client.beta.sessions.create(
            agent=agent_id, environment_id=env_id,
            title=f"Daily aggregation {c.today}", resources=resources,
        )
        with hard_timeout(HARD_TIMEOUT_SECONDS):   # arm/disarm 收成 CM
            run_session(client, session.id, build_kickoff(c))
    return 0
```

## 哪一步对应哪条原则

| 改动 | 原则 |
| --- | --- |
| `run` 只编排、三行 | 主函数是 orchestrator 不是 procedure |
| plan / execute 拆开，dry_run 不深埋 | early return + 结构化决策 |
| `EnvSource` typed，干掉 `delete_after` | 用类型表达约束（typing-and-class-design） |
| `provisioned_env_file` / `hard_timeout` CM | 资源管理用 context manager |
| `plan` / `build_resources` / `build_kickoff` 纯函数 | pure function 优先，IO 推边界 |
| `client` 作参数传进来 | 依赖注入，可测 |
| 命名 stage（`execute` / `run_session`） | 函数名透露契约，不是 `helper2` |

核心：helper 不是用来"缩长度"的，是设计出来的 stage。先把形状切成 `config → plan → execute` + 两个 RAII 抽象，主函数自然变薄。

---

# 同一份代码的另一处：事件 ladder → dispatch 表

`_drain` 消费 SSE 流。它本来是个 12 分支的 `if etype == ...` ladder，混了三件事：给每种事件打日志、reader-thread 生命周期（archive）、会话何时结束（`has_seen_running` 那个隐式状态机）。

## Before —— 12 分支 ladder + 三种关注点缠在一起

```python
def _drain(stream, client, session_id) -> None:
    has_seen_running = False
    reader_threads: set[str] = set()
    for event in stream:
        etype = getattr(event, "type", None)
        if etype == "agent.message":
            for block in getattr(event, "content", []) or []:
                if getattr(block, "type", None) == "text":
                    LOG.info("agent.message: %s", _truncate(block.text))
        elif etype == "agent.tool_use":
            LOG.info("agent.tool_use name=%s", getattr(event, "name", "?"))
        elif etype == "session.thread_created":
            agent_name = getattr(event, "agent_name", "?")
            thread_id = getattr(event, "session_thread_id", "?")
            if thread_id and agent_name in _AUTO_ARCHIVE_AGENTS:
                reader_threads.add(thread_id)
        elif etype == "session.thread_status_idle":
            thread_id = getattr(event, "session_thread_id", "?")
            if thread_id in reader_threads:
                reader_threads.discard(thread_id)
                _archive_thread_safe(client, session_id, thread_id, "...")
        elif etype == "session.status_running":
            has_seen_running = True
        elif etype == "session.status_idle":
            stop_type = getattr(getattr(event, "stop_reason", None), "type", None)
            if not has_seen_running:
                continue
            if stop_type == "requires_action":
                continue
            break
        elif etype == "session.status_terminated":
            break
        elif etype == "session.error":
            LOG.error("session.error: %s", ...)
            break
        # ... 还有 5 个 elif
        else:
            LOG.debug("event %s", etype)
```

## After —— 事件表 + 一个管 thread 的小 class + 收口的停止状态

```python
STOP = object()  # 哨兵：handler 返回它表示该结束

class ThreadArchiver:
    """owns reader-thread 生命周期 —— 状态 + 不变量，所以是 class。"""
    def __init__(self, client, session_id):
        self._client, self._session_id = client, session_id
        self._readers: set[str] = set()

    def on_created(self, ev):
        if ev.session_thread_id and ev.agent_name in _AUTO_ARCHIVE_AGENTS:
            self._readers.add(ev.session_thread_id)

    def on_idle(self, ev):
        tid = ev.session_thread_id
        if tid in self._readers:
            self._readers.discard(tid)
            self._archive_safe(tid, ev.agent_name)

class SessionDrainer:
    def __init__(self, client, session_id):
        self._archiver = ThreadArchiver(client, session_id)
        self._seen_running = False
        self._handlers = {            # 事件 → handler，取代 12 个 elif
            "agent.message":              _log_message,
            "agent.tool_use":             _log_tool_use,
            "session.thread_created":     self._archiver.on_created,
            "session.thread_status_idle": self._archiver.on_idle,
            "session.status_running":     self._on_running,
            "session.status_idle":        self._on_session_idle,   # 可能返回 STOP
            "session.status_terminated":  lambda ev: STOP,
            "session.error":              self._on_error,           # 返回 STOP
        }

    def run(self, stream) -> None:
        for ev in stream:
            handler = self._handlers.get(_event_type(ev), _log_unknown)
            if handler(ev) is STOP:
                return

    def _on_running(self, ev):
        self._seen_running = True
        LOG.info("status_running")

    def _on_session_idle(self, ev):
        # "看到 running 前忽略 idle / requires_action 继续" 这个状态机收在一处
        if not self._seen_running:
            return
        if _stop_reason(ev) == "requires_action":
            LOG.warning("unexpected requires_action; continuing")
            return
        return STOP
```

## 为什么更好

- **加事件类型 = 往表里加一行**，不是再 `elif` 一段（#24 dispatch map）。
- **三种关注点分开**：纯日志 handler、`ThreadArchiver`（状态+不变量 → class）、`SessionDrainer` 把"会话何时停"的状态机收在一处（#4）。
- **`getattr(event, ...)` 防御性访问收口**到 `_event_type` / `_stop_reason` 几个小 accessor，handler 内部干净。

和上半部分同一个道理：**长 ladder 背后是 `event → handler` 的 dispatch + 一个停止状态机**——认出结构、用对形状，长度自己就下来了。
