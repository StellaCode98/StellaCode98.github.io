---
title: TypeScript 进阶地图：从「会用」到「会设计类型」
date: 2026-09-10 17:00:00
description: 大多数人停在「会用」：会给变量贴标注、能看懂泛型、遇到搞不定的就 any。会和设计之间的分界线只有一个——类型在你的代码里是「描述」还是「约束」。这篇把进阶路线画成一张五层地图：收窄层（控制流分析、类型守卫、satisfies）、组合层（泛型作为类型级抽象、keyof 与映射类型）、推导层（条件类型 + infer 的模式匹配、模板字符串类型）、边界层（unknown 治理与运行时守卫）、设计层（判别联合让非法状态不可表示、Branded Types 区分同构类型）。每层给可运行的最小代码，最后是练习路径与「类型体操该做到哪一步停」的判断标准。
categories:
  - [前端基础, Typescript]
tags:
  - Typescript
  - 泛型
---

先对齐一个观察：团队里 TS 写了两年的人，多数停在「会用」——变量有标注、接口有 interface、组件 props 有类型，遇到搞不定的场景一个 `any` 逃之夭夭。这不是姿势问题，是**没有完成一次视角切换**：

> **会用 TS 的人把类型当「描述」：给已经写好的代码贴标签。**
> **会设计类型的人把类型当「约束」：先想清楚问题的合法状态空间，让非法状态在类型层面就无法表示。**

这篇就把从前者到后者的路径画成一张地图。结论先放这：

```text
「会用」→「会设计」不是学更多语法，而是换一个问题：
  从「这个值是什么类型」 换成 「这个类型允许哪些值存在」
```

<!-- more -->

## 一、分界线长什么样：一个所有前端都写过的例子

请求状态。三个字段的写法人人写过：

```ts
// 「会用」版：三个独立字段
interface RequestState<T> {
  data: T | null
  error: Error | null
  loading: boolean
}
```

看起来无害，算一下状态空间：`data` 两种 × `error` 两种 × `loading` 两种 = **8 种组合，其中 4 种是非法的**——`loading: true` 的同时还挂着 `error`？`data` 和 `error` 同时有值？类型系统全都放行，于是每个消费组件都得自己写防御：

```ts
if (state.loading && !state.error && state.data === null) { ... } // 防御性条件越堆越长
```

「会设计」的写法是把状态空间直接建进类型：

```ts
// 「会设计」版：判别联合（discriminated union）
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }
```

恰好 4 个成员、每个成员携带且仅携带自己需要的字段，非法组合**不是被运行时判断挡住，而是根本无法构造**。这就是那句名言的意思——*make illegal states unrepresentable*。

记住这个例子，下面每往上一层，都是在给这类设计提供工具。

## 二、地图总览

```text
┌─────────────────────────────────────────────────────────┐
│ ⑤ 设计层   判别联合 / Branded Types / 类型驱动开发        │ ← 让非法状态不可表示
│ ④ 边界层   unknown 治理 / 运行时守卫 / 声明合并           │ ← 给系统砌墙
│ ③ 推导层   条件类型 / infer / 模板字符串类型              │ ← 类型级模式匹配
│ ② 组合层   泛型抽象 / keyof / 映射类型                   │ ← 类型级函数
│ ① 收窄层   控制流分析 / 类型守卫 / satisfies / as const  │ ← 让推导跑起来
└─────────────────────────────────────────────────────────┘
```

五层的关系是**递进依赖**：不掌握收窄，判别联合的 switch 写不完整；不懂 keyof，泛型约束就没有表达力；不理解条件类型，Omit 这种日常工具类型永远是个黑盒。下面逐层展开。

## 三、① 收窄层：把类型推导当成队友，而不是对手

这一层的核心认知：**TS 的类型检查不是「逐行标注」，而是跟着控制流走的推导系统**。你写的每个 `if`、`return`、`??` 都在给编译器喂信息。

```ts
function format(value: string | number) {
  if (typeof value === 'string') {
    return value.trim()          // 这里已经被收窄为 string
  }
  return value.toFixed(2)        // 剩余分支自动收窄为 number
}
```

进阶者要掌握的四个抓手：

**1. 判别联合 + `never` 穷尽检查**。第一节的状态类型，消费端这样写：

```ts
function render(state: RequestState<User[]>) {
  switch (state.status) {
    case 'idle':    return placeholder()
    case 'loading': return spinner()
    case 'success': return list(state.data)      // 此分支里 data 一定存在
    case 'error':   return errorBox(state.error)
    default: {
      const _exhaustive: never = state           // 漏写任何分支，这一行编译报错
      return _exhaustive
    }
  }
}
```

`never` 是空集——只有所有可能都被前面分支吃掉，`state` 才可能被收窄成 `never`。这个模式的价值在于：**将来给联合加一个新状态，所有消费处的漏改会在编译期集体爆炸**，而不是上线后白屏一处。

**2. 自定义守卫：`x is T`**。当收窄逻辑复杂到编译器推不出来，就把判断封装成语义化的守卫函数：

```ts
function isApiError(x: unknown): x is { code: number; message: string } {
  return typeof x === 'object' && x !== null
      && 'code' in x && 'message' in x
}
```

**3. `satisfies`：校验但不拓宽**（TS 4.9+）。标注和推导之间的第三条路：

```ts
const theme = {
  dark:  { bg: '#111', fg: '#eee' },
  light: { bg: '#eee', fg: '#111' },
} satisfies Record<string, { bg: string; fg: string }>

theme.dark.bg   // ✅ 若写成「: Record<...>」标注，这里会因键被拓宽而报错
```

**4. `as const`**：把推导从「宽松的容器」切换到「精确的字面量」，是后面模板字符串类型和 branded 类型的前置条件。

## 四、② 组合层：泛型不是「类型的参数」，是类型级的函数

初学者对泛型的理解停在「调用时要传一个类型进去」，这是把它当语法糖。「会设计」的理解是：**泛型是抽象在类型世界的投影——函数值是值的抽象，泛型是类型的抽象**。

分水岭例子，一行看懂 `keyof` + 索引访问 + 泛型约束的组合拳：

```ts
function pick<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

const u = { id: 1, name: 'kxl', vip: true }
pick(u, 'name')   // 类型是 string，key 打错字直接编译报错
```

`K extends keyof T` 读作「K 只能是 T 的键」，`T[K]` 读作「按 K 查表取值」。三个零件拼起来，就实现了「属性名的类型安全」——这在 JavaScript 里只能靠运行时报错兜底。

这一层要建立的肌肉：

- **`keyof` / `T[K]` / `typeof x`**：类型的查表三件套。`typeof config` 从值反查类型，让配置对象和类型永远不漂移；
- **映射类型**：`{ [K in keyof T]: ... }` 是对类型的 for 循环，所有 `Partial`、`Readonly` 的本质；
- **泛型参数能省则省**：能被推导的就不要让调用者手传。`pick(u, 'name')` 不需要写 `pick<typeof u, 'name'>(...)`——**签名设计的目标是让调用方不可能传错，而不是让调用方传一堆**。

## 五、③ 推导层：条件类型 + `infer`，类型级别的模式匹配

这一层是「会设计」的分水岭，也是「类型体操」的重灾区。核心就一句话：

> **条件类型是类型层面的三元表达式，`infer` 是它在模式里声明的「捕获变量」。**

```ts
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T
type Awaited1 = UnwrapPromise<Promise<number>>   // number
type ElementOf<T> = T extends (infer E)[] ? E : never
type Item = ElementOf<string[]>                  // string
```

读法：`T extends Promise<infer U>` 不是在问「T 是不是 Promise」——`infer U` 相当于在匹配的**结构里挖一个洞**，把洞里的类型绑到 U 上。这是 `ReturnType`、`Parameters` 全家的原理。

理解了这两个零件，**标准库的工具类型就全透明了**，每个都不超过五行：

```ts
type MyPartial<T>  = { [K in keyof T]?: T[K] }
type MyPick<T, K extends keyof T> = { [P in K]: T[P] }
type MyReturnType<T> = T extends (...args: never[]) => infer R ? R : never
```

再进半步是**键重映射 + 模板字符串类型**（TS 4.1+），可以生成一族新键名：

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<K & string>}`]: () => T[K]
}

interface Point { x: number; y: number }
type PointGetters = Getters<Point>
// { getX: () => number; getY: () => number }
```

这一层还有个必踩的坑要提前知道：**裸类型参数遇到联合会分发**。`ElementOf<string[] | number[]>` 得到的是 `string | number`（逐成员求值再合并），不是 `string[] | number[]` 的元素。想关掉分发，用方括号包一层 `[T] extends [never[]] ? ... : ...`。很多「条件类型行为诡异」的 bug，拆开看都是分发在暗中生效。

## 六、④ 边界层：系统里最危险的地方，恰好是 TS 管不到的地方

类型只在编译期存在，而系统的边缘——网络响应、`JSON.parse`、`localStorage`、第三方 JS——**天然没有类型**。这里的抉择区分了三种人：

```text
any     = 双向放弃：既不检查我给它，也不检查它给我
unknown = 「我知道这里没类型，但我拒绝假装它有」→ 强制收窄后才能用
never   = 空集，穷尽检查的哨兵
```

「会设计」的边界一定是 `unknown` + 运行时守卫的组合：

```ts
async function request<T>(
  url: string,
  guard: (x: unknown) => x is T,          // 类型安全的最后一公里 = 一个守卫函数
): Promise<T> {
  const raw: unknown = await fetch(url).then(r => r.json())
  if (!guard(raw)) throw new Error(`响应结构不符: ${url}`)
  return raw
}

const isUser = (x: unknown): x is User =>
  typeof x === 'object' && x !== null && 'id' in x && 'name' in x

const user = await request('/api/user', isUser)   // user: User，且经过验证
```

注意这个设计的妙处：**守卫函数是唯一同时被「运行时」和「类型系统」信任的凭证**——运行时执行它，类型系统认它的返回值签名。后端改了字段，错误在守卫处当场爆炸，而不是渲染层读到 `undefined` 才炸。

配套的两个习惯：`catch (e)` 的 `e` 按 `unknown` 处理（`useUnknownInCatchVariables`）；给无类型的第三方包补 `.d.ts`，用模块扩充（`declare module`）给已声明类型打补丁。

## 七、⑤ 设计层：把「业务不变量」编码进类型

到这一层，类型不再是代码的附属品，而是**领域建模工具**。两个代表武器：

**1. 判别联合建模状态机**。第一节的 `RequestState` 推广开：向导表单的步骤、支付单的生命周期、组件的生命周期，凡是「状态 + 各状态的专属负载」都该是一个联合。写代码前先画状态转移表，再把表翻译成类型——这就是**类型驱动开发**：类型签名先于实现存在，它是给自己和同事的设计文档。

**2. Branded Types：给同构类型发身份证**。TS 是结构化类型系统——`string` 和 `string` 永远兼容，于是用户 ID 和订单 ID 可以互相赋值，编译器毫无意见：

```ts
declare const brand: unique symbol
type UserId  = string & { readonly [brand]: 'UserId' }
type OrderId = string & { readonly [brand]: 'OrderId' }

function getUser(id: UserId) { ... }

getUser(orderId)   // ❌ 编译报错：结构相同也不行
getUser('123')     // ❌ 裸字符串也不行
```

唯一合法的进入方式是走「铸造」出口，把校验收拢在一处：

```ts
function toUserId(raw: string): UserId {
  if (!/^\d+$/.test(raw)) throw new Error('非法的用户 ID')
  return raw as UserId            // 系统里唯一一个 as
}
```

从此「传错 ID」这类事故从运行时日志变成编辑器红线。经验法则：**同构类型一旦跨模块流动、且有自己的校验规则，就值得 brand 一下**——ID、毫秒时间戳、已消毒的 HTML、经纬度，都是典型候选。

## 八、练习路径，和「该在哪停」

每一层配一个刻意练习方式：

| 层 | 解锁能力 | 练习方式 |
| --- | --- | --- |
| ① 收窄 | 读懂并信任推导、穷尽检查 | 把旧代码里的防御性 if 换成判别联合，删掉 null 检查 |
| ② 组合 | 写出类型安全的工具函数 | 手写 pick/omit/restrictTo，不许查资料 |
| ③ 推导 | 读懂一切工具类型源码 | **通读 `lib.es5.d.ts`**——最被低估的教材，全部核心工具类型都在里面，每个不超十行 |
| ④ 边界 | any 的持续治理 | 给项目里每个 `any` 分类：能收窄的收窄、真没型的写守卫、第三方的补声明 |
| ⑤ 设计 | 用类型承载业务规则 | 找一个真实的「传错参数」事故，用判别联合或 brand 重构到它不可能再犯 |

最后是「度」的问题——这一层往上就是类型体操的深渊（四五行 `infer` 嵌套、递归类型展开成 AST 变换），要不要走下去？我的判断标准只有一条：

> **类型代码没有运行时开销，但有认知开销。一个高级类型值不值得写，看它能不能让调用方「不看实现就知道自己用错了」——做不到这点的类型体操，是在用团队的未来给自己的智商付费。**

团队项目里，⑤ 层的判别联合和 branded 值得推行，③ 层的炫技推导尽量封在底层库里并由注释护航。

## 结语

回到开头的分界线。五层地图爬完，变化的不是语法量，是动笔顺序：

> **「会用」是写完代码再补类型——类型是给编译器的交代；「会设计」是先画状态空间再写实现——类型是给问题的建模。前者让编译通过，后者让错误无处容身。**

这篇是地图，后面的坑位已经排好：泛型与条件类型的完整实战（一个类型安全的请求封装）、tsconfig 关键配置逐项讲清、存量项目的 any 治理。异步相关的类型边界（`Promise<T>` 的推导与 `await` 的 unwrap）可以配合[《JavaScript Promise》](/2026/09/09/javascript-promise/)食用。
