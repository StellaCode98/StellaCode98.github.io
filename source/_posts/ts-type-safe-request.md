---
title: 泛型与条件类型实战：写一个类型安全的请求封装
date: 2026-09-10 12:40:00
description: 以一个真实演进过程讲透 TypeScript 类型编程：从 any 满天飞的 request 封装出发，用泛型让返回值有类型，用 keyof + 索引访问把 API 定义表变成唯一数据源，用条件类型 + infer 自动提取 params/data/resp，用「条件类型 + 可变元组」实现有参数必传、无参数不传，再用模板字面量类型递归提取 REST 路径参数，最后收束 never、satisfies 与分布式条件类型三大坑。
categories:
  - [TypeScript, 实战]
tags:
  - TypeScript
  - 泛型
---

这篇文章只做一件事：**把一个 `any` 满天飞的请求封装，一步步改造成调用处零手写类型、零类型断言的类型安全封装**。所有「类型体操」都不为炫技——每一步都对应一个真实的工程痛点。

> **先说结论**：类型安全的请求封装，核心思路不是「在调用处标注类型」，而是**让 API 定义表成为唯一数据源**——`url`、`method`、参数类型、返回类型都写在一张表里，然后用 `typeof + keyof + 索引访问`取到某个 API 的定义，用**条件类型 + `infer`** 从定义里把参数和返回值「解构」出来，用**条件类型 + 可变元组**表达「有 params 必传、没有就不能传」。调用处一个尖括号都不写，类型全部自动流动，而且**写错参数名、传错类型、忘记写返回类型，都在编译期报错**。

<!-- more -->

## 一、痛点：一个典型的 any 封装

几乎所有项目都有这个文件：

```ts
// utils/request.ts —— 祖传版本
import axios from 'axios'

export function get<T = any>(url: string, params?: any): Promise<T> {
  return axios.get(url, { params }).then((res) => res.data.data)
}

export function post<T = any>(url: string, data?: any): Promise<T> {
  return axios.post(url, data).then((res) => res.data.data)
}
```

用它的时候：

```ts
// 类型是我「自称」的 —— 运行时到底返回什么，编译器一无所知
const user = await get<User>('/users/1')

// 这些全部静默通过，编译期毫无反应：
await get<User>('/userss/1')            // url 拼错，靠肉眼
await get<User>('/users/1', { id: 'x' }) // params 是 any，传什么都不报错
const u = await get('/users/1')          // 忘写泛型 → T = any → user.name 也变成 any
```

问题的根源有三个：

1. **类型声明和接口是两份东西**：`get<User>` 里的 `User` 是调用时的「口头声明」，与后端接口没有绑定关系，改了后端 TS 不知道；
2. **url、参数、返回值三者无关联**：同一个接口的信息散落在调用处，没有单一数据源；
3. **泛型默认值是 `any`**：忘了写就静默退化，类型安全形同虚设。

目标形态先亮出来：

```ts
const user = await request('getUser', { id: 1 })   // ✅ Promise<User>，自动推导
await request('logout')                            // ✅ 没有 params，不传也不许传
await request('getUser', { id: '1' })              // ❌ 编译错误：id 应为 number
await request('getUser')                           // ❌ 编译错误：缺少参数 { id: number }
await request('getUserr', { id: 1 })               // ❌ 编译错误：API 名不在表里
```

注意调用处**没有一处尖括号、没有一处 `as`**。下面开始一步步推。

## 二、十五分钟前置：类型层面的编程语言

后面要用的类型语法就五组，先各用一个最小例子讲清。已经熟的可以直接跳到第三节，忘了再回来查。

### 2.1 泛型：类型的「函数参数」

值有函数（`x => x + 1`），类型也有「函数」——泛型：

```ts
function identity<T>(x: T): T { return x }
// identity<number>(1) —— T 是「传入的类型实参」

type Box<T> = { value: T }
type NumBox = Box<number> // { value: number }
```

**泛型的本质是类型的参数化**：把类型当作输入，输出新的类型/函数签名。

### 2.2 约束、keyof 与索引访问

`extends` 给泛型加约束——「T 至少得是什么」：

```ts
// K extends keyof T：K 只能是 T 的属性名之一，且与 obj 的键关联
function pick<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

pick({ id: 1, name: 'k' }, 'name') // ✅ 返回 string
pick({ id: 1, name: 'k' }, 'age')  // ❌ 'age' 不在 keyof 范围内
```

三个关键工具：`keyof T` 取属性名**联合**（`'id' | 'name'`）；`T[K]` 是**索引访问类型**，取属性的类型；`K extends keyof T` 把两个参数的类型关联起来——**关联，是类型安全的全部秘密**。

### 2.3 条件类型与 infer：类型层面的三元与模式匹配

```ts
// 条件类型：类型层面的 A ? B : C
type IsString<T> = T extends string ? true : false
type A = IsString<'hi'>  // true
type B = IsString<42>    // false

// infer：在「匹配」的同时声明一个待推断的类型变量
type ElementOf<T> = T extends Array<infer E> ? E : never
type C = ElementOf<string[]>   // string

type Unwrap<T> = T extends Promise<infer V> ? V : T
type D = Unwrap<Promise<User>> // User
```

`infer E` 的含义：**如果 T 能看成 `Array<某个类型>`，就把那个类型捕获为 E**。这就是类型层面的「解构赋值」，是本文的主角。

### 2.4 分布式条件类型

最后一个容易被忽略的规则：**当条件类型的判断对象是「裸类型参数」且传入联合类型时，会把联合拆开逐个判断，再合并结果**：

```ts
type ToArray<T> = T extends unknown ? T[] : never
type R = ToArray<'a' | 1> // 'a'[] | 1[]，不是 ('a' | 1)[]

type IsNever<T> = T extends never ? true : false
type X = IsNever<never> // never！空联合分配后还是空 —— 所以判断 never 要用 [T] extends [never]
```

想阻止分配，把两边都包上方括号：`[T] extends [U] ? X : Y`。第三节先记住结论，第九节讲它在我们封装里的实际影响。

## 三、v1：泛型让返回值有类型——但类型写了两遍

第一步改造，起码让返回值不要 `any`：

```ts
interface ApiEnvelope<T = unknown> {
  code: number
  message: string
  data: T
}

export async function request<T>(
  url: string,
  config?: { params?: object; data?: object },
): Promise<T> {
  const res = await axios.post<ApiEnvelope<T>>(url, config?.data ?? config?.params)
  if (res.data.code !== 0) throw new Error(res.data.message)
  return res.data.data
}

const user = await request<User>('/users/1') // 有类型了，但 User 和 '/users/1' 仍是两张皮
```

进步有限：`User` 仍然是调用处的口头声明，url 仍然是裸字符串。**类型没有和「接口定义」绑定，一切标注都可以骗人**。真正的问题浮出水面：需要一个地方，把 url、参数、返回值**写在一起**。

## 四、v2：API 定义表——唯一数据源

把所有接口收进一张表，用「值」承载「类型」：

```ts
interface User {
  id: number
  name: string
}

interface ApiDef {
  url: string
  method: 'GET' | 'POST' | 'PUT' | 'DELETE'
  params?: object  // query / 路径参数的类型占位
  data?: object    // 请求体的类型占位
  resp?: unknown   // 返回数据的类型占位
}

const api = {
  getUser: {
    url: '/users/:id',
    method: 'GET',
    params: {} as { id: number }, // 值是空对象，类型是标注 —— 纯粹的「类型占位符」
    resp: {} as User,
  },
  updateUser: {
    url: '/users',
    method: 'PUT',
    data: {} as { name: string },
    resp: {} as User,
  },
  login: {
    url: '/login',
    method: 'POST',
    data: {} as { username: string; password: string },
    resp: {} as { token: string },
  },
  logout: {
    url: '/logout',
    method: 'POST',
    // 没写 params/data/resp —— 没有参数、没有返回值
  },
} satisfies Record<string, ApiDef> // ⚠️ 用 satisfies，不要用冒号标注，原因见下
```

三个关键点：

**1. `{} as { id: number }` 是类型占位符。** 值永远是空对象（运行时无意义），类型才是我们想要的。它是把「类型」存进「值世界」的容器——后面用 `infer` 再把类型取出来。

**2. `satisfies` 而不是 `: Record<string, ApiDef>`。** 如果写冒号标注，`keyof typeof api` 会塌缩成 `string`（因为 `Record<string, ...>` 的键是任意 string），后面所有按 API 名的索引全部失去约束；而 `satisfies` 只**校验**形状（每个条目确实是 ApiDef），**不改变推导出的字面量类型**——`keyof typeof api` 精确地是 `'getUser' | 'updateUser' | 'login' | 'logout'`。这是 TS 4.9 引入该操作符最典型的使用场景。

**3. 可选属性 + 缺席的键，是后面条件类型分支的依据。** `logout` 没写 `params`，就是「无参数」的表达。

现在，「getUser 长什么样」这个问题有了唯一答案：`(typeof api)['getUser']`。**类型层面**的索引访问：

```ts
type ApiOf<K extends keyof typeof api> = (typeof api)[K]

type GetUserDef = ApiOf<'getUser'>
// { url: '/users/:id'; method: 'GET'; params: { id: number }; resp: User }
```

## 五、v3：条件类型 + infer，把 params / resp「解构」出来

定义表有了，接下来写三个工具类型，从定义里提取参数和返回值——`infer` 的主战场：

```ts
// 定义里有 resp 键 → 提取它的类型；没有 → void（表示「没有返回值」）
type Resp<T> = T extends { resp: infer R } ? R : void

// 同理提取 params 与 data
type Param<T> = T extends { params: infer P } ? P : undefined
type Body<T> = T extends { data: infer D } ? D : undefined

type R1 = Resp<ApiOf<'getUser'>> // User ✅
type R2 = Resp<ApiOf<'logout'>>  // void ✅
type R3 = Param<ApiOf<'getUser'>> // { id: number } ✅
```

逐行看 `Resp` 在 `'getUser'` 上的求值过程：

```text
Resp<{ url: '/users/:id'; method: 'GET'; params: { id: number }; resp: User }>
  → { url: ...; resp: User } extends { resp: infer R } ? R : void
  → 匹配成功，R 捕获为 User
  → User
```

为什么 `logout` 走 `void` 分支？因为 `{ url: '/logout'; method: 'POST' }` **不能**赋给 `{ resp: 任意 }`——缺键即不匹配。**「键的缺席」就这样变成了「类型的分支」**，这就是 API 表里「没写就是没有」的类型学解释。

先写一个能用的版本（参数统一可选，下一节再收紧）：

```ts
async function request<K extends keyof typeof api>(
  name: K,
  payload?: Param<ApiOf<K>> & Body<ApiOf<K>>,
): Promise<Resp<ApiOf<K>>> {
  const def = api[name]
  const res = await axios.post<ApiEnvelope<Resp<ApiOf<K>>>>(def.url, payload)
  if (res.data.code !== 0) throw new ApiError(res.data.code, res.data.message)
  return res.data.data
}

const user = await request('getUser', { id: 1 }) // ✅ Promise<User>，无尖括号
```

返回值类型已经完全自动了。但还差一口气：`request('logout', { 任意垃圾 })` 现在也能过——参数是可选的 `undefined & {...}`，约束太松。

## 六、v4：条件类型 + 可变元组——「有参数必传，没参数不许传」

目标：`getUser` 必须传 `{ id: number }`；`logout` 一个参数都不许传。用**可变元组 + 条件类型**把「要不要参数」变成类型的一部分：

```ts
// 「参数列表」本身是一个条件类型：
// 定义里有 params → 参数列表是 [P]；否则是空元组 []
type Args<T> =
  T extends { params: infer P } ? [payload: P] :
  T extends { data: infer D }   ? [payload: D] :
  []

async function request<K extends keyof typeof api>(
  name: K,
  ...args: Args<ApiOf<K>>            // ⭐ 剩余参数的类型由 API 定义决定
): Promise<Resp<ApiOf<K>>> {
  const def = api[name]
  const payload = args[0] as Record<string, unknown> | undefined // 实现细节，见第七节
  const url = fillPath(def.url, payload)
  const res = await axios.post<ApiEnvelope<Resp<ApiOf<K>>>>(url, payload)
  if (res.data.code !== 0) throw new ApiError(res.data.code, res.data.message)
  return res.data.data
}
```

调用处效果：

```ts
const user = await request('getUser', { id: 1 })    // ✅ Promise<User>
await request('getUser')                             // ❌ 缺少参数 payload: { id: number }
await request('getUser', { id: '1' })                // ❌ id 应为 number
await request('getUser', { userId: 1 })              // ❌ 对象字面量只能声明已知属性
await request('logout')                              // ✅ 参数列表是 []，不传
await request('logout', {})                          // ❌ Expected 1 arguments, but got 2（只剩 API 名这一个参数位）
```

原理：TS 4.0 的可变元组允许 `...args: SomeTupleType`，而**元组类型可以是条件类型的求值结果**。当 K 被推断为 `'getUser'` 时，`Args<ApiOf<'getUser'>>` 求值为 `[payload: { id: number }]`，函数签名「长出」一个必选参数；K 是 `'logout'` 时求值为 `[]`，签名就是零参数函数。**同一个函数，按第一个参数（API 名）的不同，呈现不同的参数列表**——这就是重载的「类型级实现」，而且是自动维护的：往表里加一个 API，它就自动获得正确的签名，不需要手写 overload。

> 约定：一张表里的条目要么走 `params`（GET 类），要么走 `data`（POST/PUT 类），不混用；真要同时用，把 `Args` 第一行换成 `[params: P, body: D]` 的双元素元组即可。

## 七、进阶：模板字面量类型，提取 REST 路径参数

表里 `url: '/users/:id'` 还有一个没兑现的类型承诺——`:id` 应该被替换。先用**模板字面量类型 + 递归条件类型**把路径变量名提取成联合：

```ts
// 递归扫描字符串：`${前缀}:${变量}/${剩余}` → 拆出变量继续扫右边
type PathVars<S extends string> =
  S extends `${string}:${infer V}/${infer Rest}`
    ? V | PathVars<Rest>
    : S extends `${string}:${infer V}`
      ? V
      : never

type P1 = PathVars<'/users/:id'>          // 'id'
type P2 = PathVars<'/users/:id/posts/:pid'> // 'id' | 'pid'
type P3 = PathVars<'/users'>               // never —— 没有路径变量
```

用它能给「填路径」函数做精确签名——**参数的形状由路径字符串决定**：

```ts
function fillPath<S extends string>(
  path: S,
  vars: Record<PathVars<S>, string>,
): string {
  return path.replace(/:(\w+)/g, (_, key: string) => {
    const v = (vars as Record<string, string | undefined>)[key]
    if (v === undefined) throw new Error(`[request] 缺少路径参数: ${key}`)
    return encodeURIComponent(v)
  })
}

fillPath('/users/:id', { id: '1' })  // ✅
fillPath('/users/:id', {})           // ❌ 缺少 id
fillPath('/users/:id', { foo: '1' }) // ❌ 多余属性
fillPath('/users', { id: '1' })      // ✅ 通过（见下）
```

一个实测过的边界：`Record<never, string>` 会求值为 `{}`——全局最宽的对象类型，**连多余属性检查都不会触发**。所以「没有路径变量的接口传了 vars」拦不住，只拦得住反方向（有变量没传、传错名）。这是模板字面量守卫的真实局限：类型约束做不到的地方，靠 `fillPath` 实现里那行 `v === undefined` 的运行时检查兜底——**编译期安全网和运行时兜底是互补关系，不是替代关系**。

在 `request` 实现里调用它时，`payload` 是 `{ id: number }` 而 `fillPath` 要 `Record<'id', string>`，实现处用了一个 `as` 桥接（顺手 `String(v)`）。这里要坦诚一个工程事实：**类型安全的封装内部一定有几处断言，价值在于把不安全关进笼子——边界处（调用方）绝对安全，笼子里（实现方）集中审查**。这与运行时校验（zod / valibot）互补：TS 管编译期，schema 管网络另一端的不确定性。

## 八、失败路径：让错误也变成类型的一部分

成功路径安全了，失败路径还是 `throw new Error(message)`——调用方只能拿到 `string`。给错误建类型：

```ts
class ApiError extends Error {
  readonly code: number
  constructor(code: number, message: string) {
    super(message)
    this.name = 'ApiError'
    this.code = code
  }
}
```

调用方用 `instanceof` 收窄：

```ts
try {
  const user = await request('getUser', { id: 1 })
} catch (e) {
  if (e instanceof ApiError) {
    e.code       // number —— 401 跳登录、429 提示稍后再试
    if (e.code === 401) router.push('/login')
  }
  // 不是 ApiError 的（网络层、取消请求），交给全局兜底
}
```

另外两个与 `never` / `void` 相关的取舍：

- **`Resp` 的兜底用 `void` 而不是 `any`/`unknown`**：表里忘写 `resp` 时，返回 `Promise<void>`，调用方一旦取值立刻编译报错——错误被「放大」而不是被吞掉。用 `any` 则静默通过，等于给忘写返回类型开绿灯。
- **穷尽检查**：如果后续按 `method` 分发（GET 拼 query、POST 走 body），在 switch 里用 `never` 兜底，新增 method 常量时漏分支会直接编译失败：

```ts
function send(method: 'GET' | 'POST' | 'PUT' | 'DELETE') {
  switch (method) {
    case 'GET': return /* ... */
    case 'POST': return /* ... */
    case 'PUT': return /* ... */
    case 'DELETE': return /* ... */
    default: {
      const _exhaustive: never = method // 漏掉任何一个分支，这里编译报错
      throw new Error('unreachable')
    }
  }
}
```

## 九、三个最容易踩的坑

**坑一：手滑把 `satisfies` 写成类型标注。**

```ts
const api: Record<string, ApiDef> = { ... }          // ❌ keyof typeof api → string，全部塌缩
const api = { ... } satisfies Record<string, ApiDef> // ✅ 校验形状，保留字面量推导
```

塌缩后的症状很典型：`request('任意字符串')` 都不报错。看到这个现象先检查表的定义方式。

**坑二：分布式条件类型在意料之外拆开联合。**

本文的 `Resp<T>`/`Args<T>` 之所以安全，是因为 `ApiOf<K>` 在调用时总是**单个**对象类型。但如果哪天写出 `Resp<ApiOf<keyof typeof api>>`（想「批量」处理所有 API），裸 `T` 会把四个定义的联合拆开逐个判断、再合并成联合——`Args` 会得到 `[payload: ...] | []`，可变元组变成「或」语义，约束直接失效。**想禁止分配，写 `[T] extends [{ params: infer P }]`**。判断标准就一条：条件类型的检查对象是不是「裸的类型参数」。

**坑三：可选属性 + `infer` 的 `undefined` 污染。**

如果表里写的是 `params?: {} as { id: number }`（可选键），`T extends { params: infer P }` 推出的 `P` 是 `{ id: number } | undefined`，`Args` 会要求你传一个「可能是 undefined 的参数」，约束变糊。这就是第四节坚持**要么写键、要么不写键**（不用 `?`）的原因。若必须处理可选键，用守卫分支把 undefined 剥掉：

```ts
type Param<T> =
  T extends { params: infer P }
    ? undefined extends P    // P 里掺了 undefined（可选键）吗
      ? never               // 当作「不强制传」
      : P
    : never
```

## 十、完整实现与总结

```ts
// ── types.ts ──
interface ApiEnvelope<T = unknown> {
  code: number
  message: string
  data: T
}

interface ApiDef {
  url: string
  method: 'GET' | 'POST' | 'PUT' | 'DELETE'
  params?: object
  data?: object
  resp?: unknown
}

type ApiOf<K extends keyof typeof api> = (typeof api)[K]
type Resp<T> = T extends { resp: infer R } ? R : void
type Args<T> =
  T extends { params: infer P } ? [payload: P] :
  T extends { data: infer D }   ? [payload: D] :
  []

type PathVars<S extends string> =
  S extends `${string}:${infer V}/${infer Rest}`
    ? V | PathVars<Rest>
    : S extends `${string}:${infer V}`
      ? V
      : never

class ApiError extends Error {
  constructor(readonly code: number, message: string) {
    super(message)
    this.name = 'ApiError'
  }
}

// ── api.ts ──（表即文档：url / method / 参数 / 返回值一处可见）
const api = {
  getUser:   { url: '/users/:id', method: 'GET',  params: {} as { id: number }, resp: {} as User },
  updateUser:{ url: '/users',     method: 'PUT',  data: {} as { name: string }, resp: {} as User },
  login:     { url: '/login',     method: 'POST', data: {} as { username: string; password: string }, resp: {} as { token: string } },
  logout:    { url: '/logout',    method: 'POST' },
} satisfies Record<string, ApiDef>

// ── request.ts ──
async function request<K extends keyof typeof api>(
  name: K,
  ...args: Args<ApiOf<K>>
): Promise<Resp<ApiOf<K>>> {
  const def = api[name]
  const payload = args[0] as Record<string, unknown> | undefined
  const url = fillPath(def.url, payload as Record<string, string>)
  const res = await axios.post<ApiEnvelope<Resp<ApiOf<K>>>>(url, payload)
  if (res.data.code !== 0) throw new ApiError(res.data.code, res.data.message)
  return res.data.data
}
```

回头看这趟演进，每一步都是一个可迁移的思路：

| 痛点 | 用到的类型能力 | 迁移到别处的场景 |
| --- | --- | --- |
| 返回值是 `any` | 泛型 `request<T>` | 所有容器 / 工具函数 |
| 类型与接口两张皮 | `typeof + keyof + 索引访问`，表为唯一数据源 | 配置表、路由表、i18n key |
| 参数可传可不传 | 条件类型 + `infer` 提取 | 从现有类型解构出想要的片段 |
| 有参数必传 / 无参禁传 | 条件类型 + 可变元组 | 任何「按前者决定后者」的签名 |
| `:id` 无约束 | 模板字面量 + 递归条件类型 | 事件名、路由 path、CSS 变量名 |
| 错误是裸 `string` | `ApiError` + `instanceof` 收窄 | 所有可预期的失败路径 |

**类型安全的本质不是「多写类型」，而是「建立关联」**：把 API 名和它的定义关联、把定义和参数列表关联、把 url 字符串和路径变量关联。`infer`、条件类型、模板字面量只是建立关联的胶水。想清楚了这一点，再看 `Partial`、`Pick`、`Awaited` 这些内置工具类型的源码（它们全是十几行的条件类型），就能读出每一条「关联」是为哪个痛点服务的——也就到了随手给自己写工具类型的时候。
