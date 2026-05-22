---
name: code-design-principles
description: 写代码时的具体决策原则。覆盖：动手前的探索、控制流、数据结构选择、错误处理、抽象时机、可测性、并发模型、依赖管理。当用户写新代码、做局部重构、review code、纠结"这个该怎么写"时使用。不涉及架构层（架构层看 agent-design-principles / project-structure）。
---

# 写代码原则

## 1. 动手前先探索，不凭印象写

写新功能前必做（按顺序）：

1. **grep / glob 找现有的相似实现** —— 大概率已经有，复用比新写好
2. **读相关文件 / 调用方 / 类型定义** —— 别盲猜签名
3. **查官方文档 / 看库源码** —— 不要靠记忆中的 API
4. **检查 lock 文件确认实际版本** —— 不要按"最新版"写

凭印象写 = 大概率写出"看起来对但实际过时"的代码。

## 2. 互斥状态用 state machine / enum，不用嵌套 if

```python
# ❌
if user.is_logged_in and not user.is_banned and user.email_verified:
    ...

# ✅
class UserStatus(Enum):
    ANONYMOUS = "anonymous"
    PENDING_VERIFICATION = "pending"
    ACTIVE = "active"
    BANNED = "banned"

match user.status:
    case UserStatus.ACTIVE: ...
```

判断标准：≥3 个 bool 字段相互约束 → 改 enum / sealed class。

## 3. Early return 替代深嵌套

```python
# ❌
def f(x):
    if a:
        if b:
            if c:
                return ...

# ✅
def f(x):
    if not a: return None
    if not b: return None
    if not c: return None
    return ...
```

任何函数嵌套 >3 层 → 立刻拆函数 / early return。

## 4. IO-bound 默认 async；CPU-bound 默认同步

`requests` → `httpx.AsyncClient`，`time.sleep` → `asyncio.sleep`，文件 IO → `aiofiles` 或 `anyio`。

阻塞 IO 在 async 上下文里 = 整个 event loop 卡死。新写网络/文件代码默认 async。

## 5. Pure function 优先，IO 推到边界

```python
# ❌
def calculate_price(user_id):
    user = db.get_user(user_id)        # IO 混在算法里
    rate = api.get_tax_rate(user.region) # 另一个 IO
    return user.cart_total * (1 + rate)

# ✅
def calculate_price(cart_total: float, tax_rate: float) -> float:
    return cart_total * (1 + tax_rate)

# 边界处组合
async def get_price_for_user(user_id):
    user = await db.get_user(user_id)
    rate = await api.get_tax_rate(user.region)
    return calculate_price(user.cart_total, rate)
```

纯函数好测、好缓存、好并行。

## 6. 不要过早抽象

第一次写：内联在 use case 里。
第二个 use case 出现：抽出来。
第三次出现：可以考虑泛化签名。

预先抽象往往**抽错维度**——你猜的"会复用的形状"和真实需求不一致。

判断标准：这段代码包含**业务知识**？→ 等第二个 caller。纯**技术工具**（日期格式化、字符串 escape）？→ 可提前抽。

## 7. 默认 immutability

- Python：`dataclass(frozen=True)` / `tuple` / `frozenset` 优于 `list` / `dict`
- TS：`readonly` / `Readonly<T>` / `as const`
- 函数不要修改入参，返回新对象

修改可变数据 = 隐式耦合。`x.sort()` vs `sorted(x)` 区别巨大。

## 8. 错误明示，不静默 catch

```python
# ❌
try:
    result = risky()
except Exception:
    pass                         # 沉默吞错

# ✅
try:
    result = risky()
except SpecificError as e:
    logger.error("...", exc_info=e)
    raise FallbackError(...) from e
```

只 catch **你知道怎么处理**的具体异常。`except Exception: pass` 是 bug 工厂。

## 9. 类型在编译期表达约束，运行时不重复 check

```python
# ❌
def process(items):
    if not isinstance(items, list): raise TypeError(...)
    if any(not isinstance(i, int) for i in items): raise TypeError(...)

# ✅
def process(items: list[int]) -> ...: ...
```

签名已经说明的不要再 if 一遍。运行时校验只在**真实边界**做（HTTP 入口、反序列化、用户输入）。

## 10. 依赖通过参数注入，不用全局 import

```python
# ❌
from app.db import db
def get_user(id): return db.query(...)   # 测试时换不了

# ✅
def get_user(db: Database, id) -> User:
    return db.query(...)
```

便于测试、便于配多个实例、依赖关系显式可见。

## 11. 资源管理用 context manager / with / RAII

```python
# ❌
f = open("x")
data = f.read()
f.close()                        # 中间 raise 就漏

# ✅
with open("x") as f:
    data = f.read()
```

锁、连接、临时目录、事务都同理。任何"获取 → 用 → 释放"用 context manager。

## 12. 小函数，单一职责

经验值：
- Python/TS 函数 > 50 行 → 大概率该拆
- 圈复杂度 > 10 → 必拆
- 一个函数有 ≥2 个 "and" 才能描述它做什么 → 拆

但**不要为拆而拆**。3 行函数被另一个地方调用 = 可以独立；只在一处用且无独立语义 = 留着。

## 13. Fail fast on invariant violation

不可能发生的状态如果发生了 → 立刻 `assert` / `raise`，不要"宽容地继续"。

```python
def divide(a, b):
    assert b != 0, "caller must check"     # 不变量
    return a / b
```

宽容 = bug 在远处的地方才崩，调试地狱。

## 14. 优先 composition，谨慎 inheritance

继承深度 ≤2 层。需要复用代码 → 优先：
1. **函数 / 模块** （最简）
2. **mixin / trait** （水平扩展）
3. **组合**（has-a）
4. 最后才考虑继承（is-a 且行为多态）

继承让"超类怎么改"成为"所有子类的 breaking change"，耦合最强。

## 15. 默认参数不要用 mutable 值

```python
# ❌ 经典 bug：默认值在所有调用间共享
def f(items=[]): items.append(1); return items

# ✅
def f(items=None):
    if items is None: items = []
    ...
```

TS 同理：`function f(opts = {})` 在某些情况下也有坑，确认引擎行为。

## 16. 优先返回数据，不返回 callback

```python
# ❌
def find_user(id, on_found, on_not_found): ...

# ✅
def find_user(id) -> User | None: ...
```

返回值可以 type-check、可以组合、可以测试。Callback 倾向于产生 callback hell。

## 17. 注释解释 "why"，不解释 "what"

```python
# ❌
# Increment counter by 1
counter += 1

# ✅
# Server uses 1-indexed pages, but our cache is 0-indexed
counter += 1
```

写不出 "why" 注释 = 代码本身已经清楚 = 不需要注释。

## 18. 配置和代码分离

魔法常量、超时值、URL、feature flag → 配置层（env / config file / DB）。

代码里硬编码 = 改值要重新部署 = 不同环境难切换。

## 19. 测试和被测代码同位

`channel.py` + `channel.test.py` 在同一目录。改实现立刻能看到对应测试。

反模式：仓库根 `tests/` 镜像所有 `src/` —— 改实现忘改测试。

## 20. Lint / typecheck / format 自动化，不靠人

CI 必须跑：
- formatter（black / prettier / rustfmt）
- linter（ruff / eslint / clippy）
- typechecker（mypy / tsc / pyright）

写代码前先 setup 这三个。代码风格争论是工程上最浪费时间的事。

## 21. 有行为的 dict 升级成 class

```python
# ❌
ctx = {"state": "running", "buffer": [], "retries": 0}
def feed(ctx, chunk): ctx["buffer"].append(chunk); ...
def flush(ctx): ...

# ✅
class StreamContext:
    state: State
    buffer: list[str]
    retries: int
    def feed(self, chunk): ...
    def flush(self): ...
```

判断标准：一个 dict **键 ≥3 个** 且**有专门读写它的函数** → 改成 class。state 变成 typed 属性，函数变成方法，IDE 跳转、type check、refactor 全到位。

## 22. Dispatch map / match 替代 if/elif 长链

```python
# ❌
if event.type == "message": handle_message(event)
elif event.type == "tool_call": handle_tool_call(event)
elif event.type == "error": handle_error(event)
elif event.type == "done": handle_done(event)
# ... 又来 8 个 elif

# ✅ 1：dict 查表
handlers = {
    "message": handle_message,
    "tool_call": handle_tool_call,
    "error": handle_error,
}
handlers[event.type](event)

# ✅ 2：method dispatch
getattr(self, f"_on_{event.type}")(event)

# ✅ 3：match (state, event)
match (self.state, event.type):
    case (State.IDLE, "start"): ...
    case (State.RUNNING, "tool_call"): ...
```

什么时候用：≥4 个分支按同一个 key 路由。非法转移（`(BANNED, "start")`）一眼能看见，不会静默落到 else。

## 23. 同一段 5 行代码出现 3 次 → 抽 helper

第 1 次：直接写。
第 2 次：复制，加 TODO。
第 3 次：抽。

抽出来要起一个**说清楚做什么的名字**（`upsert_file` / `update_assistant` / `make_event`），不是 `helper1` / `do_stuff`。

判断信号：你在 grep 第 3 个相同 5 行块的时候，停下来抽。

## 24. 类拥有自己的 state，依赖通过构造器注入

```python
# ❌
class Worker:
    def run(self):
        global_registry["agent"].run()        # 运行时从全局 dict 取
        config = global_registry["config"]    # 谁改了都不知道

# ✅
class Worker:
    def __init__(self, agent: Agent, config: Config):
        self.agent = agent                    # 依赖：注入
        self.config = config
        self.state = State.IDLE               # 自己的 state：自己 own
        self.buffer: list[str] = []

    def run(self):
        self.agent.run()
```

规则：
- **mutable state 归这个类自己 own**（不在外部 dict 里漂着）
- **外部对象（agent / config / clients）从构造器进来**（不在 method 里临时取）

效果：测试时构造器换 mock，依赖图一眼可见。

## 25. 同一种 pattern 跨栈复用

后端 `StreamContext.dispatch(event)` + 前端 `handlers[event.type](event)` 走同一种结构：**事件 → 查表 → 派发 → 处理**。

```python
# 后端 (Python)
class StreamContext:
    def dispatch(self, event):
        return self._handlers[event.type](event)
```

```ts
// 前端 (TS)
const handlers = {
  message: handleMessage,
  tool_call: handleToolCall,
};
handlers[event.type](event);
```

读懂一边就读懂另一边。跨服务 / 跨语言 / 跨进程的代码尤其值得这样做——降低 onboarding 成本、降低跨栈 bug 的认知开销。

## 26. 处理流水线用 flatmap chain：`T → list[T]`

```python
# ❌ 用 state 变量串起来
processed = []
for chunk in stream:
    if should_emit(chunk):
        processed.append(transform(chunk))
flush_remainder(processed)
send_all(processed)

# ✅ 每一层都是 T → list[T]，for 链起来
def stream():    yield from raw_events
def dispatch():  return [e for c in stream() for e in route(c)]
def emit():      return [out for e in dispatch() for out in to_output(e)]
for x in emit(): send(x)
```

每个 stage 是**纯函数 + generator**，数据自然流过去：没有进度变量、没有回调嵌套、没有"还差什么状态"的心智负担。`feed → handle → extend`、`stream → dispatch → yield` 都是这个 pattern。

## 27. 函数名要透露 input / output 契约

不要起 `process` / `handle` / `do` 这种"啥都能装"的名字。常用 shape 词汇：

| 名字 | 契约 |
| --- | --- |
| `feed(chunk)` / `flush()` | 有状态的流式处理：chunk 灌进来，结尾 flush 余量 |
| `dispatch(event)` | 一个输入 → 查表 → 路由到对应 handler |
| `emit(x) -> list[event]` | 一个东西转成 ≥0 个待发送事件 |
| `make_event(...) -> Event` | 纯工厂：参数 → 构造好的对象 |
| `drain(buffer)` / `flush()` | 清空 buffer，把剩下的处理掉 |
| `upsert(key, value)` | 不存在就 create、存在就 update，caller 不分支 |
| `iter_*(...)` | 懒 generator，一个个 yield，不一次性 build list |

看到名字就知道：能不能流式、有没有 state、返回单值还是列表、有没有副作用。命名是廉价的文档，远比加注释划算。

---

## 决策模板

写一段代码前问自己：

1. **有没有现有的类似实现？** 先 grep
2. **官方文档 / 库源码 怎么说？** 先查
3. **这是 IO 还是 CPU？** 选 async / sync
4. **签名能表达约束吗？** 用 type 而不是运行时 check
5. **真有第二个 use case 吗？** 没有就内联
6. **依赖能注入吗？** 不要 import 全局
7. **错误会怎么传播？** 只 catch 你知道怎么处理的

---

## 反模式速查

- 写新功能前没 grep 现有实现 / 没看 docs
- ≥3 个 bool 字段相互约束（应该用 enum）
- 函数嵌套 >3 层（应该 early return）
- 阻塞 IO 在 async 函数里（`time.sleep` 在 async 里）
- 算法函数里直接调 IO（应推到边界）
- `try: ... except Exception: pass`
- 运行时 isinstance check 已经有 type hint 的参数
- 全局 import 数据库/客户端单例（无法测试）
- `f = open(); ... f.close()`（应该 with）
- 函数 >100 行 / 圈复杂度 >15
- 默认参数是 `[]` / `{}` / `set()`
- 注释只是把代码翻译成英文
- URL / 超时 / 神秘数字硬编码
- 测试集中在仓库根 `tests/` 而不是源代码旁
- 没 lint / typecheck / format 自动化
- 三个月还没第二个用户的"通用工具类"
- 一个 dict ≥3 个键 + 多个函数读写它（应升级成 class）
- ≥4 个 `elif` 按同一个字段路由（应改 dispatch map / match）
- 同一段 5 行代码复制 ≥3 次（抽 helper）
- 类在 method 里从全局字典取依赖（应构造器注入）
- 函数叫 `process` / `handle` / `do_stuff`（名字不透露契约）
- 流水线里用进度变量 / 嵌套回调（应改 `T → list[T]` flatmap chain）
