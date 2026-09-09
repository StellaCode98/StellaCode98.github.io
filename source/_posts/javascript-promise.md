---
title: Promise 全景：从回调地狱、状态机到微任务与链式调用
date: 2026-09-09 15:30:00
description: Promise 不只是解决回调地狱的 API，而是一套「异步结果管理 + 后续任务组合」的抽象：状态机保存结果，then 注册后续并返回新 Promise 形成链，微任务决定执行时机。本文从状态机、执行时机、链式规则讲到组合 API 与 async/await。
categories:
  - [前端基础, javascript]
tags:
  - javascript
  - Promise
  - 异步
  - Event Loop
  - async/await
---

我以前对 Promise 的理解就一句话：**解决回调地狱的。** 这话没错，但远远不够——直到被一连串问题问住：

- `new Promise()` 里的代码为什么是**同步执行**的？
- 明明已经 resolve 了，`.then()` 为什么还是不立即跑？
- `then` 里 `return 123`，下一个 `then` 怎么就拿到了 `123`？`throw` 的错为什么后面的 `catch` 接得住？
- `async/await` 和 Promise 到底什么关系？

后来才想明白，Promise 本质是一套抽象：

> **用状态机表示「未来才产生的结果」，用 then / catch / finally 描述后续处理，用链式 Promise 传递值与错误，再交给微任务机制调度执行。**

```text
Promise
├── 状态管理    pending → fulfilled / rejected
├── 结果传递    resolve(value) / reject(reason)
├── 后续处理    then / catch / finally
├── 链式调用    then 返回新 Promise
├── 任务调度    reaction → 微任务
└── 组合能力    all / allSettled / race / any
```

<!-- more -->

## 一、从回调地狱说起：Promise 是「未来结果的容器」

回调时代把三个串行请求连起来，缩进和错误处理一起失控：

```js
getUser((user) => {
  getUserDetail(user.id, (detail) => {
    getOrders(detail.id, (orders) => {
      console.log(orders);   // 想统一处理错误？每层都要写一遍
    });
  });
});
```

Promise 的解法：**把「未来会得到的结果」封装成一个对象**。`fetch('/api/user')` 立刻返回一个 promise——现在还没有结果，未来要么 fulfilled（用户数据）、要么 rejected（错误原因）。此后的所有问题，都变成「怎么管理这个对象」。

## 二、状态机：Promise 的核心不是 then，是状态

```text
pending ──resolve──► fulfilled（保存 value）
   │
   └──reject───────► rejected（保存 reason）
```

最重要的一条规则：**状态只能从 pending 变一次，之后不可逆**：

```js
new Promise((resolve, reject) => {
  resolve('A');
  resolve('B');   // 无效：状态已定格
  reject('C');    // 无效
});
// 最终：fulfilled，value = 'A'
```

而 `resolve` / `reject` 做的事是完整的一条链：**改变状态 → 保存结果 → 触发已注册的 reactions → 把它们排进微任务队列**。最后一步，就是「then 为什么是异步的」的答案。

## 三、执行时机：executor 同步，then 是微任务

Promise 最容易误解的地方——「哪部分是异步的」：

```js
console.log('1');
new Promise((resolve) => {
  console.log('2');                    // executor：同步执行
  resolve('ok');
}).then(() => console.log('3'));       // 回调：进微任务队列
console.log('4');

// 输出：1 → 2 → 4 → 3
```

**executor 是同步执行的**，不会因为写在 `new Promise` 里就变异步；真正异步的是 `then` 的回调——即使 Promise 已经 settled，回调也要等当前同步代码跑完、轮到微任务时才执行。加上 setTimeout 就凑成经典面试题：

```js
console.log('1');
setTimeout(() => console.log('2'), 0);           // 宏任务
Promise.resolve().then(() => console.log('3'));  // 微任务
console.log('4');

// 输出：1 → 4 → 3 → 2（微任务在当前任务后清空，宏任务等下一轮）
```

## 四、链式调用：then 永远返回一个新 Promise

链式调用的基础：`then` 同时做两件事——注册当前 Promise 的后续处理，**创建并返回一个新的 Promise**。所以 `.then().then()` 连的从来不是同一个对象。新 Promise 的状态由回调的返回值决定，规则只有三条：

| then 回调里 | 新 Promise |
| --- | --- |
| `return 123`（普通值） | fulfilled，value = 123（≈ `Promise.resolve(123)`） |
| `return promise2` | 等待并跟随 promise2 的状态 |
| `throw new Error()` | rejected，reason = 该错误 |

两个高频推论：

- **值穿透**：`.then()` 不传回调时值不会被吃掉，`Promise.resolve(100).then().then(v => v)` 一路传到底——空 then 等价于 `v => v`；
- **错误沿链传播**：链中某一步 rejected 后，中间没有 onRejected 的 then 全部跳过，错误一路传到最近的 `.catch`。`catch` 就是 `then(undefined, onRejected)` 的语法糖——工程上推荐 `then / catch / finally` 分开写而不是用 then 的第二个参数，因为前者能兜住整条链。

`finally` 无论成败都执行且不改变值，典型用途是关 loading：

```js
showLoading();
fetch('/api/user')
  .then((res) => res.json())
  .catch(console.error)
  .finally(hideLoading);   // 成败都走到
```

## 五、微任务：Promise 和 Event Loop 的分工

Promise 本身不是事件循环。分工是：**Promise 负责状态、结果和 reactions；宿主的事件循环负责什么时候执行它们**——reaction 被排进微任务队列，每个宏任务结束后清空。

一个容易忽视的细节：链上的 then 是**逐个**入队的——第一个 then 的回调执行完、它返回的新 Promise settled 后，第二个 then 的回调才入队：

```js
Promise.resolve().then(() => console.log(1)).then(() => console.log(2));
console.log(3);

// 输出：3 → 1 → 2
// 同步打印 3 → 微任务 1 打印 1，第二个回调此时才入队 → 微任务 2 打印 2
```

所以链上每个 then 各占一个微任务位——「微任务清空过程中新入队的也要清」这条事件循环规则，日常来源就是它。

## 六、组合 API：批量管理多个 Promise

先看它们解决的真实问题。三个互不依赖的请求串行 await，总耗时是三者之和；`Promise.all` 并发发起，总耗时约等于最慢的那个：

```js
// 串行：T = T1 + T2 + T3
const user = await getUser();
const goods = await getGoods();
const orders = await getOrders();

// 并发：T = max(T1, T2, T3)
const [user, goods, orders] = await Promise.all([getUser(), getGoods(), getOrders()]);
```

| API | 成功条件 | 失败条件 | 典型场景 |
| --- | --- | --- | --- |
| `Promise.all` | 全部 fulfilled | 任意一个 rejected | 并发请求 |
| `Promise.allSettled` | 全部结束即 resolve | 永不因单个失败 reject | 批量任务要逐个结果 |
| `Promise.race` | 第一个 settle 的定结果 | 同左 | 请求超时控制 |
| `Promise.any` | 第一个 fulfilled | 全部 rejected | 多源容灾取最快成功 |

三个细节常被问：

1. `all` 的结果**按输入顺序**排列，不是完成顺序——B 先回来也排在第二位；
2. `all` 一个失败立即 rejected，但**不会自动取消**其他已发出的请求——要取消得配 `AbortController`；
3. `race` 同样只是「取先到的结果」，不会取消落败者。

## 七、async/await：Promise 之上的语法糖

async/await 不是 Promise 的替代品，而是**建立在 Promise 之上的语法机制**。两条核心规则，一个例子全占了：

```js
async function foo() {
  console.log('A');
  await Promise.resolve();
  console.log('B');   // ≈ 写在 then 里的代码
}
foo();
console.log('C');

// 输出：A → C → B
```

1. **async 函数一定返回 Promise**：`return 123` 会被包装成 `Promise.resolve(123)`；
2. **await 暂停的是当前 async 函数，不是 JS 线程**：函数在 await 处让出，把后面的代码当作「await 值的 then 回调」排进微任务，调用处继续同步执行——所以 B 在 C 之后。

串行陷阱也出自这里：循环里逐个 `await` 是串行，要并发就先 `Promise.all` 再取值。

## 八、手写 Promise：骨架与真正的难点

手写是检验理解的最好方式。内部最少需要这些：

```js
class MyPromise {
  constructor(executor) {
    this.status = 'pending';
    this.value = undefined;    // fulfilled 时保存结果
    this.reason = undefined;   // rejected 时保存原因
    this.callbacks = [];       // pending 期 then 注册的回调，先存起来
    // executor 同步执行；resolve/reject 里：改状态、存结果、依次异步触发 callbacks
  }
  then(onFulfilled, onRejected) { /* 返回新 Promise，难点见下 */ }
}
```

为什么需要 callbacks 数组：`p.then(f1); p.then(f2);` 在 pending 期注册的回调，要等 `resolve` 时再依次触发。

真正的难点不在状态机，而在 **then 的返回值解析**（规范叫 Promise Resolution Procedure）：回调可能 return 普通值、return 一个 Promise、return 一个 thenable（`{ then(resolve) {} }`）、甚至 throw——每种情况新 Promise 怎么变，全在这一个函数里。把这段写对，才算真懂 then 链。

## 九、常见误区速查

| 误区 | 事实 |
| --- | --- |
| Promise 的代码是异步的 | executor 同步执行；异步的是 then 的回调 |
| `Promise.resolve().then(fn)` 立即执行 | fn 进微任务，同步代码之后才跑 |
| Promise = 线程 / Web Worker | 只是结果抽象，干活的仍是宿主（网络、定时器……） |
| async/await 绕开了 Promise | 全部建立在 Promise 机制之上 |
| `Promise.all` 失败会取消其他请求 | 不会；取消需要 AbortController |

## 总结

一张图收束全文：

```text
pending ──resolve──► fulfilled(value)
   │                    │
   └──reject──────► rejected(reason)
                        │
             状态改变 → reaction → 微任务 → then 回调执行
                        │
          ┌─────────────┼─────────────┐
      return 值      return P       throw
          │             │             │
    新 P fulfilled  等待并跟随 P   新 P rejected
          └─────────────┼─────────────┘
                     then...（链继续）
```

<!-- 面试速答：

- **Promise 的本质？** 表示异步操作最终结果的对象：状态机保存结果，then / catch / finally 注册后续，reaction 排进微任务执行，then 返回新 Promise 形成链。
- **`new Promise` 里的代码同步还是异步？** 同步。异步的是 then 的回调，即使 Promise 已 settled 也要等微任务时机。
- **then 的三种返回值规则？** 普通值 → 新 Promise fulfilled；返回 Promise → 等待并跟随；throw → rejected，错误沿链传播到最近的 catch。
- **catch 为什么能接住前面的错？** then 回调 throw 会让它返回的新 Promise rejected，错误沿链向后传播直到遇到 onRejected；catch 就是 `then(undefined, onRejected)`。
- **async/await 和 Promise 的关系？** 语法糖：async 函数必返回 Promise，await 后的代码等价于 then 回调，暂停的是当前函数而非线程。 -->

**一句话记住**：Promise = 状态机 + 链式传递 + 微任务调度——状态存结果，链传值与错，微任务定时机；async/await 只是这套机制的衣服。

<!-- ## 参考资料

- [ECMAScript 规范：Promise Objects](https://tc39.es/ecma262/multipage/control-abstraction-objects.html)
- [MDN：Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [MDN：await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await)
- [Promises/A+ 规范](https://promises-aplus.promises-spec/) -->
