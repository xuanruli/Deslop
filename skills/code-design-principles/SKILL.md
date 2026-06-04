---
name: code-design-principles
description: 写代码时的具体决策原则。覆盖：动手前的探索、控制流、数据结构选择、错误处理、抽象时机、可测性、并发模型、依赖管理。当用户写新代码、做局部重构、review code、纠结"这个该怎么写"时使用。不涉及架构层（架构层看 agent-design-principles / project-structure）。
---

# 写代码原则

## 1. 动手前先探索，不凭印象写

写新功能前按顺序做：

1. **grep / glob 找现有相似实现** —— 大概率已有，复用优先
2. **读相关文件 / 调用方 / 类型定义** —— 别盲猜签名
3. **查官方文档 / 库源码** —— 不靠记忆里的 API
4. **看 lock 文件确认实际版本** —— 不按"最新版"写

凭印象写 = 大概率"看起来对但实际过时"。

## 2. 不顺着烂结构堆补丁，该重构就提

为"赶快跑通"在烂 / 临时结构上继续叠加，是最常见的偷懒。新增前判断当前结构扛不扛得住：

- 扛得住 → 顺着写。
- 扛不住 → 先做个小重构再加，别 pile on workaround / 特例 / 复制粘贴。

信号：让它跑通的代价是又加一层绕路 → 多半该先重构。范围大或涉及业务取舍时，先跟用户说"建议先重构 X"让其拍板，别默默选最省事那条。

## 3. 互斥状态用 state machine / enum，不用嵌套 bool

判断标准：≥3 个 bool 字段相互约束（`is_logged_in` / `is_banned` / `email_verified`…）→ 改 enum / sealed class，用 `match` 分派。

## 4. Early return 替代深嵌套

函数嵌套 >3 层 → 立刻拆函数 / early return。

## 5. IO-bound 默认 async；CPU-bound 默认同步

`requests` → `httpx.AsyncClient`，`time.sleep` → `asyncio.sleep`，文件 IO → `aiofiles` / `anyio`。阻塞 IO 在 async 上下文里 = 整个 event loop 卡死。

## 6. Pure function 优先，IO 推到边界

算法函数只收数据、返数据；db / api / 文件等 IO 放到调用它的边界层组合。纯函数好测、好缓存、好并行。

## 7. 不要过早抽象

第一次写内联；第二个 use case 出现再抽；第三次才考虑泛化签名。预先抽象往往**抽错维度**。

判断：含**业务知识** → 等第二个 caller；纯**技术工具**（日期格式化、字符串 escape）→ 可提前抽。

## 8. 默认 immutability

- Python：`dataclass(frozen=True)` / `tuple` / `frozenset` 优于 `list` / `dict`
- TS：`readonly` / `Readonly<T>` / `as const`
- 函数不改入参，返回新对象

修改可变数据 = 隐式耦合。`x.sort()` vs `sorted(x)` 区别巨大。

## 9. 错误明示，不静默 catch

只 catch **你知道怎么处理**的具体异常，并 log + 转抛。`except Exception: pass` 是 bug 工厂。

## 10. 类型在编译期表达约束，运行时不重复 check

签名已说明的（`items: list[int]`）不要再 `isinstance` 一遍。运行时校验只在**真实边界**做（HTTP 入口、反序列化、用户输入）。

## 11. 依赖通过参数注入，不用全局 import

把 db / client / config 作为参数传进去，别在函数体里 `from app.db import db`。便于测试、便于多实例、依赖显式。

## 12. 资源管理用 context manager / with / RAII

任何"获取 → 用 → 释放"（文件、锁、连接、临时目录、事务）用 `with`，别手动 close（中间 raise 就漏）。

## 13. 小函数，单一职责

- 函数 > 50 行 → 大概率该拆
- 圈复杂度 > 10 → 必拆
- 要 ≥2 个 "and" 才能描述它做什么 → 拆

但**不要为拆而拆**：只在一处用且无独立语义的小片段，留着。

## 14. Fail fast on invariant violation

不可能发生的状态真发生了 → 立刻 `assert` / `raise`，别"宽容地继续"。宽容 = bug 在远处才崩，调试地狱。

## 15. 优先 composition，谨慎 inheritance

继承深度 ≤2 层。要复用代码优先级：**函数 / 模块 → mixin / trait → 组合（has-a）→** 最后才继承（is-a 且行为多态）。继承让"超类怎么改"变成"所有子类的 breaking change"。

## 16. 默认参数不要用 mutable 值

```python
# ❌ 默认值在所有调用间共享
def f(items=[]): items.append(1); return items

# ✅
def f(items=None):
    if items is None: items = []
```

经典 bug。TS 的 `function f(opts = {})` 在某些情况下也有坑。

## 17. 优先返回数据，不返回 callback

`find_user(id) -> User | None` 优于 `find_user(id, on_found, on_not_found)`。返回值可 type-check、可组合、可测试；callback 倾向 callback hell。

## 18. 注释与 docstring

- docstring 默认一行，确有必要才多行。
- 注释默认不写，只在别人可能误解逻辑时补一行。
- 禁止版本更替类注释（"原来是 X 改成 Y""unchanged"）—— 版本变化交给 git。

## 19. 不留 AI slop，与所在文件风格一致

写完 / review diff 时，删掉本次引入的、人类不会写、和周围格格不入的痕迹：

- **类型逃逸**：禁止为绕过类型报错而 `as any` / `# type: ignore` / `@ts-ignore`，要真正修类型。
- **多余防御**：别在可信 / 已校验的调用链上加冗余 try/catch（和 #9、#14 互补：该防的防，不该防的别瞎防）。
- **风格不一致**：新代码的命名、注释密度、写法匹配所在文件，不要鹤立鸡群。

判断就一句："**和这个文件的既有风格一致吗**"。

## 20. 配置和代码分离

魔法常量、超时值、URL、feature flag → 配置层（env / config / DB）。硬编码 = 改值要重新部署、多环境难切换。

## 21. 多来源的值用 resolver 抽象，别存死值

一个值可能来自字面量 / 环境变量 / shell 命令 / 运行时计算（典型：API key、header、路径）时，别在各处存死字符串 + 各处 if 判断来源。建模成"**一个待解析的字符串 + 一个 resolve 函数**":

- 输入统一(`"literal"` / `"$ENV_VAR"` / `"!command"`)，resolver 决定怎么解。
- 好处:密钥不进文件(指向 env / 命令)、值可动态、来源逻辑只有一处。
- 判断信号:你开始在第二个地方写"如果以 `$` 开头就读环境变量"——该抽 resolver 了。

## 22. 测试和被测代码同位

`channel.py` + `channel.test.py` 同目录。反模式：所有测试堆在仓库根 `tests/` —— 改实现忘改测试。

## 23. Lint / typecheck / format 自动化

CI 必须跑 formatter（black / prettier）+ linter（ruff / eslint）+ typechecker（mypy / tsc）。写代码前先 setup 这三个。

## 24. 有行为的 dict 升级成 class

判断标准：一个 dict **键 ≥3 个** 且**有专门读写它的函数** → 改成 class。state 变 typed 属性、函数变方法，IDE 跳转 / type check / refactor 全到位。

## 25. Dispatch map / match 替代 if/elif 长链

≥4 个分支按同一个 key 路由 → 用 dict 查表（`handlers[event.type](event)`）、method dispatch、或 `match (state, event)`。非法转移一眼可见，不会静默落 else。

## 26. 同一段 5 行代码出现 3 次 → 抽 helper

第 1 次直接写；第 2 次复制加 TODO；第 3 次抽。抽出来起一个**说清楚做什么**的名字（`upsert_file` / `make_event`），不是 `helper1`。

## 27. 类拥有自己的 state，依赖通过构造器注入

- **mutable state 归这个类自己 own**（不在外部 dict 里漂着）
- **外部对象（agent / config / clients）从构造器进来**（不在 method 里临时取）

效果：测试时构造器换 mock，依赖图一眼可见。

## 28. 同一种 pattern 跨栈复用

后端 `dispatch(event)` 和前端 `handlers[event.type](event)` 走同一种结构（事件 → 查表 → 派发）。读懂一边就读懂另一边，降低跨栈认知成本。

## 29. 处理流水线用 flatmap chain：`T → list[T]`

每个 stage 是纯函数 + generator（`stream → dispatch → emit`），用 for 链起来，数据自然流过去——没有进度变量、没有回调嵌套。

## 30. 函数名要透露 input / output 契约

不要起 `process` / `handle` / `do` 这种"啥都能装"的名字。常用 shape 词汇：

| 名字 | 契约 |
| --- | --- |
| `feed(chunk)` / `flush()` | 有状态流式：chunk 灌进来，结尾 flush 余量 |
| `dispatch(event)` | 一个输入 → 查表 → 路由到 handler |
| `emit(x) -> list[event]` | 一个东西转成 ≥0 个待发送事件 |
| `make_event(...) -> Event` | 纯工厂：参数 → 构造好的对象 |
| `upsert(key, value)` | 不存在就建、存在就更，caller 不分支 |
| `iter_*(...)` | 懒 generator，一个个 yield |

看名字就知道：能不能流式、有没有 state、返回单值还是列表、有没有副作用。

---

## 决策模板

写一段代码前问自己：

1. **有没有现有相似实现？** 先 grep
2. **官方文档 / 库源码怎么说？** 先查
3. **IO 还是 CPU？** 选 async / sync
4. **签名能表达约束吗？** 用 type 而非运行时 check
5. **真有第二个 use case 吗？** 没有就内联
6. **依赖能注入吗？** 不要 import 全局
7. **错误会怎么传播？** 只 catch 你知道怎么处理的

---

## 反模式速查

- 写新功能前没 grep / 没看 docs
- 为赶快跑通在烂结构上堆 workaround，而非该重构时提出重构
- ≥3 个 bool 字段相互约束（应 enum）
- 函数嵌套 >3 层（应 early return）
- 阻塞 IO 在 async 函数里
- 算法函数里直接调 IO（应推到边界）
- `try: ... except Exception: pass`
- 已有 type hint 还运行时 isinstance check
- 全局 import 数据库 / 客户端单例（无法测试）
- 手动 `open(); ... close()`（应该 with）
- 函数 >100 行 / 圈复杂度 >15
- 默认参数是 `[]` / `{}` / `set()`
- 多行 docstring 但一行就够
- 版本更替类注释（"原来是 X""unchanged"）写进代码而非 git
- `as any` / `type: ignore` 绕过类型而非真正修
- 可信调用链上加冗余 try/catch
- 新代码风格与所在文件不一致
- URL / 超时 / 神秘数字硬编码
- 测试堆在仓库根 `tests/` 而非源码旁
- 没 lint / typecheck / format 自动化
- 一个 dict ≥3 键 + 多函数读写（应升级 class）
- ≥4 个 elif 按同一字段路由（应 dispatch map）
- 同段 5 行复制 ≥3 次（抽 helper）
- 类在 method 里从全局字典取依赖（应构造器注入）
- 函数叫 `process` / `handle` / `do_stuff`（名字不透露契约）
