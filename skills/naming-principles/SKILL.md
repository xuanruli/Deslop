---
name: naming-principles
description: 命名通用原则。指导 coding agent 给变量、函数、类型、文件起出"读名字就猜对意图"的命名。当用户问"叫什么好"、做命名 review、新建模块/类型/函数时使用。
---

# 命名原则

## 1. 描述意图，不描述实现

`persist_user()` 不是 `insert_user_to_postgres()`。换实现时不用改名 = 抽象层选对了。

## 2. 禁止 `Manager` / `Helper` / `Util` / `Service` / `Handler` 类无信息后缀

去掉后缀剩下的词应该已经表达职责。`UserManager` → `UserStore` / `UserRepository`，`AuthHelper` → `TokenSigner`。

例外：领域术语保留（`Scheduler`、`Reducer`、`Builder`）。

## 3. 布尔值用 `is_X` / `has_X` / `can_X` / `should_X`

`if user.permission` 是对象还是 bool？`if user.has_permission` 自解释。

## 4. 用领域语言

跟领域专家说话用什么词，代码就用什么词。AI agent 领域用 `turn / tool_call / message / context`，不用 `round / function_invocation / item / history`。

## 5. 函数名是动词短语，类名是名词

`User`、`Connection` 是类；`connect()`、`persist()` 是函数。`UserConnection` 是类，`HandleUser` 是函数（虽然名字本身违反原则 2）。

## 6. 名字长度 ∝ 作用域

| 作用域 | 长度 |
|---|---|
| 局部循环变量 | 1-2 字母（`i`、`ev`） |
| 函数参数 | 短词（`ctx`、`msg`） |
| 模块内类型 | 中等 |
| 全局可见的类型 | 完整描述性 |

全局类型缩写 = 别人 grep 不到。局部变量起超长名 = 阅读累。

## 7. 成对概念用对称命名

`start/stop`、`open/close`、`acquire/release`、`push/pop`、`subscribe/unsubscribe`。

不要 `start/end`（end 太通用）、`open/shutdown`（不对称）、`acquire/free`（acquire 配 release，allocate 配 free）。

## 8. 缩写只用行业公认

`ctx`、`db`、`url`、`http`、`api` 可以；`usr`、`prof`、`cfg`、`msg` 不行。判断标准：缩写在 Wikipedia 有词条 → 可以。

## 9. 互斥状态用 enum，不用多个 bool

```python
# ❌ 8 种组合，5 种非法
class Job:
    is_running: bool
    is_done: bool
    is_failed: bool

# ✅
status: Literal["pending", "running", "done", "failed"]
```

## 10. 区分"未设置"和"显式 None" 用 sentinel

业务里 `None` 是合法值时（"用户没设置 last_error" 也算一种状态），引入 `Unset` / `MISSING` 区分。

## 11. 类型与单例配对（适合 sentinel / sum type variants）

```python
class _Interrupt: pass
class _Parallel: pass
Interruptable = _Interrupt()      # 实例公开
Parallel = _Parallel()
RunMode = _Interrupt | _Parallel
```

调用方写 `return Interruptable`，读起来像 enum 值。

## 12. 事件用过去时，handler 用 `OnX`

事件 = 已发生的事实：`UserSignedUp`、`OrderPaid`、`FileSaved`。
Handler 名字配对：`OnUserSignedUp`、`OnOrderPaid`。grep `OnX` 立刻找到对应的事件流。

## 13. 文件名 = 它导出的核心概念（单数名词）

`channel.py` = 一个 Channel；`scheduler.py` = 一个 Scheduler 族。

`utils.py` / `helpers.ts` / `common.go` / `misc.*` = 设计失败信号（顶层时；子包内小作用域 utils 可接受）。

## 14. 否定式只在自然语义时用

`is_empty`、`not_found`、`unauthorized` 自然。`is_disabled` → 改 `is_enabled`，否则代码里出现 `if not is_disabled` 双重否定。

## 15. 常量化魔法字符串

任何在代码里出现 ≥2 次或"含义比值更重要"的字面量提常量。`PARTITION_VOICE = "voice_chunks"`。

## 16. 名字不要重复它所在的位置

`user/service.py: class Service` + 调用方 `user.Service` 优于 `user/user_service.py: class UserService`。

但当类型会被到处 import 进来失去 namespace 时，加领域前缀。**统一就行，别混用**。

## 17. 私有用语言机制（前缀 `_` / `#`），不靠注释

让 IDE 补全自动过滤、让 reviewer 一眼识别、让重构敢改。

## 18. 描述"现在是什么"，不描述"曾经是什么"

`state` / `phase` / `mode` 优于 `was_started + is_running`。多个布尔表达"过去/现在"是隐式状态机。

---

## 反模式速查

- 顶层文件叫 `utils` / `helpers` / `common` / `misc`（**目录形式由 Deslop hook 自动拦截**）
- 类名以 `Manager` / `Service` / `Handler` / `Util` 结尾且无领域前缀
- 全局变量叫 `data` / `obj` / `result` / `value`
- 布尔字段不带 `is_` / `has_` / `can_`
- 多个 bool 表达互斥状态
- 函数名暴露实现（`save_to_redis`、`fetch_via_grpc`）
- 否定式 + 调用处出现 `if not is_disabled`
- 全局类型缩写到 1-3 字母
- 局部循环变量起超长名
- 一个文件导出 5+ 不相关概念
- 成对概念命名不对称（`start` / `terminate`）
- 自创缩写（`usrSvc`、`msgHdlr`）
