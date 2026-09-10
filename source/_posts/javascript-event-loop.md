---
title: 事件循环全景：调用栈、宏任务、微任务与渲染帧
date: 2026-09-09 11:30:00
description: 事件循环不是 JavaScript 的特性，而是宿主环境（浏览器/Node）的调度机制：同步代码一口气跑完，每个宏任务之后微任务队列必清空，渲染只发生在任务之间。从调用栈讲到宏微任务的调度规则，再到 setTimeout(fn,0) 的 4ms 钳制、双 rAF 等帧技巧，最后覆盖 Node.js 的六个阶段与 setImmediate 的顺序之谜。
categories:
  - [前端基础, javascript]
tags:
  - javascript
  - 事件循环
  - node.js
---

我对事件循环的误解持续了很多年：最早以为「setTimeout 就是开个线程」，后来背下口诀「宏先微后」，直到连翻三次车：

1. **loading 不出现**：点击 → 显示转圈 → 同步计算 300ms → 隐藏转圈。转圈从头到尾没出现过。
2. **顺序背不动**：`Promise.then` 和 `setTimeout(fn, 0)` 谁先我能背，但代码里混进一个 `await`，顺序立刻说不清。
3. **Node 玄学**：同一段代码里 `setTimeout(fn, 0)` 和 `setImmediate`，先跑谁居然「看机器心情」。

后来才明白，这是同一个机制的三种表现。**事件循环不是 JavaScript 的特性，而是宿主环境（浏览器、Node）的调度机制**——JS 引擎只负责一块调用栈，「什么时候跑哪段异步代码」由宿主说了算。

<!-- more -->

## 一、发动机：调用栈、任务队列与循环算法

先立地基：JS 单线程，同一时刻调用栈里只有一段代码在跑，同步代码执行到栈空为止。「异步」从来不是并行，而是**推迟**。推迟出去的部分由宿主收着（定时器、DOM 事件、网络都不在 JS 线程上干等），时机一到，宿主把回调包装成一个**任务**（task，俗称宏任务）放进任务队列，事件循环按固定算法一个个调回 JS 线程：

```text
while (true) {
  task = 任务队列中最老的一个任务;      // 一轮只取一个
  执行 task;                           // 调用栈随之清空
  while (微任务队列非空) {
    执行最老的微任务;                   // 清空过程中新入队的也要执行
  }
  if (本轮有渲染机会) {                 // 只有浏览器有，大致对齐垂直同步信号
    执行 requestAnimationFrame 回调;
    样式计算 → 布局 → 绘制 → 合成;
  }
}
```

最小的例子：

```js
console.log('script start');
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('script end');

// 输出：script start → script end → promise → timeout
```

整段脚本是**第一个宏任务**：同步执行到末尾，`then` 回调入微任务队列、`setTimeout` 回调入宏任务队列；脚本任务结束后先清微任务（`promise`），下一轮循环才取宏任务（`timeout`）。

### 坑 1：「宏任务队列」其实不止一条

任务队列按**任务源**分多条（定时器一条、DOM 事件一条、网络一条），每轮从哪个源取由浏览器决定。所以「宏任务 vs 微任务」的准确区别不是「两个队列谁在前」，而是**调度方式**：宏任务一轮取一个、任务之间可插渲染；微任务每个宏任务之后**必清空**。后面所有的「玄学顺序」都是这条的推论。

## 二、微任务：每个宏任务后的「必清空」清单

微任务来源一张表列全：

| 来源 | 说明 |
| --- | --- |
| `then / catch / finally` | 回调进微任务队列 |
| `await` 之后的续体 | 本质就是 then 回调 |
| `queueMicrotask(fn)` | 显式入队 |
| `MutationObserver` | DOM 变更回调（Vue 时代前的 nextTick 实现） |

两条铁律：**每个宏任务结束后立刻清空**（清空过程中新入队的也要清）；**清完之前没有渲染，也没有下一个宏任务**。

### await 的本质：微任务的语法糖

```js
async function a() {
  console.log('a-1');
  await b();              // 暂停点：a 把「后面的代码」交给微任务，自己先返回
  console.log('a-2');
}
async function b() {
  console.log('b');
}

console.log('start');
setTimeout(() => console.log('timeout'), 0);
Promise.resolve()
  .then(() => console.log('then-1'))
  .then(() => console.log('then-2'));
a();
console.log('end');

// 输出：start → a-1 → b → end → then-1 → a-2 → then-2 → timeout
```

拆解：`a()` 同步执行到第一个 await——打印 `a-1`，`b()` 也是同步调用打印 `b`；`a-2` 及之后的部分作为「b 结果的 then 回调」进微任务队列；同步代码继续打印 `end`；清微任务：`then-1`（执行后挂出 `then-2`）→ `a-2` → `then-2`；最后宏任务 `timeout`。记法一句：**await 后面的代码 ≈ 写在 then 里的代码**——在哪个 await 暂停，就从那里进微任务队列。

### 坑 2：混进 await 之后顺序就背不动了？自测一题

不背口诀，纯用第一节的算法推：

```js
console.log('1');
setTimeout(() => {
  console.log('2');
  Promise.resolve().then(() => console.log('3'));
}, 0);
Promise.resolve()
  .then(() => {
    console.log('4');
    setTimeout(() => console.log('5'), 0);
  })
  .then(() => console.log('6'));
queueMicrotask(() => console.log('7'));
console.log('8');
```

按轮次推演：

| 轮次 | 发生什么 | 输出 |
| --- | --- | --- |
| 宏任务 1（脚本） | 打印 1、8；`2` 入宏任务；`then(4)`、`7` 入微任务 | 1、8 |
| 微任务批次 1 | `then(4)`：打印 4，`5` 入宏任务，返回后 `then(6)` 入队；依次执行 `7`、`6` | 4、7、6 |
| 宏任务 2（timeout-2） | 打印 2；`3` 入微任务 | 2 |
| 微任务批次 2 → 宏任务 3 | 执行 `3`；下一轮取 `5` | 3、5 |

最终输出：**1 → 8 → 4 → 7 → 6 → 2 → 3 → 5**。推的时候永远只问三个问题：栈空了吗？微任务队列还有吗？宏任务该取哪一个了？——比任何口诀都可靠。

## 三、setTimeout(fn, 0) 的真相与渲染帧

**「0 毫秒」从来不是 0**：HTML 规范规定定时器嵌套层级超过 5 后，小于 4ms 的延时一律按 4ms 算；后台标签页更狠，直接钳到 1 秒起步。所以 `setTimeout(fn, 0)` 的真实语义是「尽快，但至少让出本轮」。

### 坑 3：loading 不出现——渲染只发生在任务之间

回到开头翻的第一辆车：

```js
btn.addEventListener('click', () => {
  spinner.style.display = 'block';   // 想让用户先看见转圈
  heavyWork();                       // 同步计算 300ms，阻塞整个任务
  spinner.style.display = 'none';
});
```

事件回调是一整个宏任务，而样式计算、布局、绘制都发生在**任务之间**——浏览器只看得到这个任务的最终状态（`none`），中间那次「显示」从未被画出来。修复的关键是把任务切开，让一帧真正提交：

```js
btn.addEventListener('click', async () => {
  spinner.style.display = 'block';
  // 双 rAF：第一个 rAF 在本帧渲染步骤开始时执行（此时还没画），
  // 它里面排的 rAF 落在下一帧——那时上一帧已经画完提交了
  await new Promise(r => requestAnimationFrame(() => requestAnimationFrame(r)));
  heavyWork();
  spinner.style.display = 'none';
});
```

rAF 的准确位置：不是普通宏任务——回调跑在「更新渲染」这一步里、**样式计算和布局之前**，专为「绘制前最后一刻改样式」设计。这也是双 rAF 的原理：只等一帧不够，因为第一个 rAF 回调执行时那一帧还没画。

### 坑 4：微任务递归，页面冻结

```js
function loop() {
  Promise.resolve().then(loop);   // 无限续微任务
}
loop();
```

微任务清空之前没有渲染机会，这个循环让事件循环永远出不了当前轮次，页面直接冻结；换成 `setTimeout(loop, 0)` 就没事。引出一个容易被忽略的结论：**能「让出 UI」的从来不是异步，而是宏任务**——微任务让得再频繁，渲染和输入响应一帧都等不到。

## 四、Node.js：libuv 的六个阶段

Node 的事件循环由 libuv 实现，宏任务来源从「定时器 / DOM / 网络」换成六个阶段：

```text
   ┌────────────────────────────────┐
┌─►│ timers         setTimeout/Interval│
│  ├────────────────────────────────┤
│  │ pending callbacks  系统级回调    │
│  ├────────────────────────────────┤
│  │ idle, prepare     内部使用       │
│  ├────────────────────────────────┤
│  │ poll            I/O 事件、取新事件│
│  ├────────────────────────────────┤
│  │ check           setImmediate    │
│  ├────────────────────────────────┤
│  │ close callbacks  close 事件回调  │
└──┴────────────────────────────────┘
   每执行完一个宏任务回调：先清空 nextTick 队列，再清空 Promise 微任务队列
```

两个与浏览器的差异：

- **微任务清理时机**：Node 11 起与浏览器对齐——每个宏任务回调之后清一次（更早版本攒到阶段结束才清）；
- **process.nextTick 是独立队列**，优先级比 Promise 微任务还高，递归同样能饿死循环——绝大多数场景用 `queueMicrotask`（本轮后尽快）或 `setImmediate`（下轮 check 阶段）更好。

### 坑 5：setImmediate 和 setTimeout(0) 谁先？——「不一定」

```js
setTimeout(() => console.log('timeout'));
setImmediate(() => console.log('immediate'));
// 主模块里顺序不定：同一台机器两次运行都可能不同
```

原因在 timers 阶段的到期判断有约 1ms 阈值：进入事件循环时如果进程启动够快、0ms 定时器还没「到期」，循环就先绕过 timers 走到 check 阶段——**顺序取决于进程启动耗时，本质是竞态**。但放进 I/O 回调里顺序就恒定了：

```js
const fs = require('fs');
fs.readFile(__filename, () => {                  // 回调在 poll 阶段执行
  setTimeout(() => console.log('timeout'));      // 排进下一轮的 timers
  setImmediate(() => console.log('immediate'));  // 本轮 poll 之后的 check 阶段
});
// 恒定输出：immediate → timeout
```

poll 之后紧跟 check，而 timers 要等下一轮——这就是「I/O 回调内部想尽快执行用 setImmediate」的由来。

## 五、工程实践：把长任务切开

超过 50ms 的同步任务会明显伤害输入响应和帧率（RAIL 模型的经验值）。渲染只发生在任务之间，标准解法就是**主动切**：

```js
// 1) setTimeout：最常用，但有 4ms 钳制、优先级偏低
const yieldByTimeout = () => new Promise(r => setTimeout(r, 0));

// 2) MessageChannel：无钳制、触发快，React 调度器的同款思路
const channel = new MessageChannel();
const yieldByMessage = () => new Promise(resolve => {
  channel.port1.onmessage = () => resolve();
  channel.port2.postMessage(null);
});

// 3) scheduler.yield()：原生让出、保留任务优先级（Chrome/Edge 129+，用前检测）
const yieldToMain = () =>
  typeof scheduler !== 'undefined' && 'yield' in scheduler
    ? scheduler.yield()
    : yieldByMessage();
```

用法上按时间片切，而不是按条数切：

```js
async function process(items, work) {
  let chunkStart = performance.now();
  for (const item of items) {
    work(item);
    if (performance.now() - chunkStart > 40) {   // 每片约 40ms
      await yieldToMain();                       // 让出：给渲染和输入留窗口
      chunkStart = performance.now();
    }
  }
}
```

注意和 `await Promise.resolve()` 的区别：那样根本没让出——微任务清空之前既不渲染也不响应输入。**让出主线程 = 进入下一个宏任务**，这句是本节的全部内容。

## 六、总结

速查表：

| API / 来源 | 归属 | 执行时机 |
| --- | --- | --- |
| setTimeout / setInterval | 宏任务 | 一轮取一个；嵌套 ≥5 层后最小 4ms，后台页钳到 1s |
| DOM 事件 / I/O / MessageChannel | 宏任务 | 任务源之间的优先级由宿主定 |
| then / await 续体 / queueMicrotask | 微任务 | 每个宏任务之后，队列必清空 |
| requestAnimationFrame | 渲染步骤 | 渲染机会到来时、样式计算之前 |
| setImmediate（Node） | check 阶段宏任务 | 本轮 poll 之后 |
| process.nextTick（Node） | 独立微任务队列 | 比 Promise 微任务更早清空 |

<!-- 面试速答：

- **setTimeout(fn, 0) 是立即执行吗？** 不是。至少让出到下一个宏任务，且嵌套 5 层后被钳到 4ms。
- **await 后面的代码什么时候跑？** 作为微任务，在本轮宏任务结束后、渲染和下一个宏任务之前。
- **宏任务和微任务的本质区别？** 调度方式：宏任务一轮取一个、之间可渲染；微任务每轮后必清空、清完前不渲染。
- **为什么微任务读到的状态比 setTimeout 回调「新」？** 它跑在渲染和后续任务之前，中间没有任何人插队。
- **怎么让浏览器先画一帧再干重活？** 双 rAF 等上一帧提交，或把重活按时间片拆成多个宏任务。
- **Node 里 setImmediate 和 setTimeout(0) 谁先？** 主模块里是竞态、顺序不定；I/O 回调里 setImmediate 恒先。 -->

**一句话记住**：同步代码一口气跑完 → 清空微任务 → 也许渲染 → 下一个宏任务——浏览器如此；Node 只是换了一套宏任务的来源（六个阶段）和两条更早清空的队列（nextTick、微任务），主循环铁律不变。

<!-- ## 参考

- [WHATWG HTML Standard — Event loops](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops)
- [MDN：事件循环](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Event_loop)
- [MDN：使用 queueMicrotask()](https://developer.mozilla.org/zh-CN/docs/Web/API/HTML_DOM_API/Microtask_guide)
- [Jake Archibald：Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [Philip Roberts：What the heck is the event loop anyway?（JSConf EU 2014）](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
- [Node.js 官方文档：The Event Loop, Timers, and process.nextTick](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [web.dev：Optimize long tasks](https://web.dev/articles/optimize-long-tasks) -->
