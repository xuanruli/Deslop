---
name: typing-and-class-design
description: 类型与类的设计原则（偏 TypeScript / 有类型系统的语言，思路也适用 Python typing）。覆盖：泛型参数化什么、用类型表达约束、discriminated union、Result vs 异常、class 还是 function/interface、状态封装与不变量、为成长而设计。当用户设计类型、定义 class / interface、做泛型或类型建模、纠结 class vs function 时使用。
---

# 类型与类设计

贯穿全篇的一条：**设计时就假设它会长大。** 别先写一个"功能小所以 limited"的版本，回头再重写——一开始就留好扩展的轴（泛型 / 空接口 / 注入点）。下面很多条都是这条的具体形态。

## 类型

## 1. 泛型只参数化"真正变化的那个轴"

一个泛型参数，然后用 conditional type 从它**选出依赖字段的合法形状**：

```ts
interface Model<TApi extends Api> {
  api: TApi;
  compat?: TApi extends "openai" ? OpenAICompat
         : TApi extends "anthropic" ? AnthropicCompat : never;
}
```

一个参数决定 sibling 字段，配错 provider 直接编译不过。别为了"看起来通用"加一堆用不上的泛型。

## 2. schema 当单一事实源，派生类型 + 运行时校验

数据形状定义一次（schema 值），用它**派生静态类型**（`Static<T>`）**也**编译成运行时校验器。编译期和运行期零漂移。

## 3. discriminated union + `Extract` 收窄 + 无 default 穷举

领域实体用 `type` 判别的 union；用 `Extract` 把 payload 收窄到合法子集（`done` 事件的 reason 不可能是 `"aborted"`）；switch **不写 default**，加了新 variant 不处理就编译失败。

## 4. 多类型 dispatcher 用 event→result map 类型

一个 dispatcher 处理多种 event、每种返回不同 → 把"event → result"编成一个 map 类型，用 discriminant 泛型索引 `Map[TType]`，而不是写 N 个 typed 方法。

## 5. 从 canonical union 派生相关类型，别重复声明

`PendingWrite = Omit<Entry, 存储层赋的字段>`。相关类型用 mapped / conditional 从同一个 union 推出来，源头改了它自动跟。

## 6. 用 typed 子接口的树组合，别让一个接口长成扁平大坨

东西变多时，把相关字段收进一个**命名的子接口**（`CompactionSettings` / `RetrySettings`），让顶层接口变成"子接口的树"：

```ts
interface Settings {
  compaction?: CompactionSettings;   // 各自一个接口
  retry?: RetrySettings;             // 里面可再嵌 ProviderRetrySettings
  terminal?: TerminalSettings;
}
```

加功能 = 加一个新子接口 + 顶层加一个字段引用它，**不是**往一个 god 接口平铺第 N 个字段。子接口能独立复用、独立带默认值、独立演进。这是"为成长设计"在类型层最常用的一招——config、state、event payload 都适用。

## 7. `Result<T,E>` 表达预期失败，`any` 只留信任边界

预期内的失败（文件不存在、解析错）返回 `Result`，不 throw；异常只留给真 bug。`any` 只允许出现在显式信任边界（外部 JSON 校验输出、跨系统类型擦除），别到处撒。

## 8. 空接口 + declaration merging 当零成本扩展点

```ts
interface CustomMessages {}  // 故意空
type Message = BuiltinMessage | CustomMessages[keyof CustomMessages];
```

下游 re-open 这个接口加自己的类型，核心不需要认识它们——对修改关闭、对扩展开放，全程类型安全、零运行时 registry。**这就是"为成长设计"的典型手法。**

## 9. `Known | (string & {})` 开放字面量

既要已知值的 autocomplete，又要接受任意字符串时，用 `KnownLiteral | (string & {})`，不要直接退化成 `string`。

## 类

## 10. class 只用于"状态 + 不变量"

无状态逻辑 → 函数；纯数据 / 能力契约 → interface；**只有"有可变状态且要保护不变量"时才用 class**。别给一堆无状态函数硬包个 class。

## 11. copy-on-assign + replace-don't-mutate

可被外部 assign 的字段，在 setter 里 `slice()` 拷贝（防 aliasing）；observer 可能持有的状态，**换新对象而不是原地改**（`pendingCalls = new Set(...)`），让旧快照一直安全。

## 12. 封装严格度跟着不变量走

纯内存对象 → 公开可变字段、同步 setter 没问题（图省事）。**一旦 mutate 必须同步外部系统**（持久化 / 广播）→ 只能走 method，每个 method 内部保证"改 + 存 + 通知"原子完成。同一个核心，封装强度按"有没有外部不变量"分两档。

## 13. 组合函数式核心 > 抽象基类继承

把核心写成无状态函数，几个有状态的壳**组合**它，而不是拉一条继承链（避免脆弱基类）。真要继承，用**构造参数特化**（`super(predicates)`），不靠 override 行为。

## 14. options-bag 构造器 + 少量 live setter

构造器收一个 options 对象、内部填默认值；之后要调的少数东西用 setter。别用望远镜式构造器（一长串位置参数）。

---

## 为成长而设计（核心心法）

设计类型 / 类时，问一句：**"这个东西长大 10 倍时，是加东西还是重写？"**

- 留**扩展轴**：泛型参数、空接口、注入点——以后加 provider / message 类型 / 行为时是"填进去"，不是"改签名"。
- 留**派生关系**：相关类型从一个源头推导，别复制——长大后改一处而非 N 处。
- 留**不变量的位置**：哪怕现在单机单会话，把"必须同步"的 mutate 收进 method，以后上多租户 / 持久化不用翻地基。

判断信号：你写下一个 class / 类型时，如果脑子里是"现在功能小，先这么对付"——停，按"它会长大"重想一遍接口形状。改接口形状是廉价的（现在），重写是昂贵的（以后）。
