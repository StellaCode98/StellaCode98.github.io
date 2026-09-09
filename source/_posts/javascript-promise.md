---

title: Promise 全景：从回调地狱、状态机到微任务与链式调用
date: 2026-09-09 15:30:00
description: Promise 不只是一个解决回调地狱的 API，它本质上是一套异步状态管理与任务调度机制。本文从回调地狱讲起，逐步拆解 Promise 的三种状态、resolve/reject、then 链式调用、值穿透、异常处理、Promise.all、async/await，以及 Promise 为什么一定和微任务联系在一起。
categories:
 - [前端基础, javascript]
tags:
 - javascript
 - Promise
 - 异步
 - Event Loop
 - async/await

---

我以前理解 Promise，基本停留在一句话：

> **Promise 是用来解决回调地狱的。**

这句话没错，但远远不够。

真正让我把 Promise 想明白，是连续踩了几个坑：

1. `new Promise()` 里面的代码为什么是**同步执行**的？
2. `then()` 里面的代码为什么是**异步执行**的？
3. Promise 明明已经 `resolve` 了，为什么 `.then()` 还是不会立即执行？
4. 为什么 `.then()` 可以无限链式调用？
5. 为什么 `.then()` 里面 `return 123`，下一个 `.then()` 能拿到 `123`？
6. 为什么 `.then()` 里面 `throw new Error()`，后面的 `.catch()` 能接住？
7. `Promise.all()` 为什么一个失败就整体失败？
8. `async/await` 到底和 Promise 是什么关系？
9. 为什么 Promise 和微任务队列总是绑在一起？

后来才发现：

**Promise 并不是简单的“异步 API”。**

它真正解决的是：

> **如何用一个标准化的对象，表示一个未来才会产生的结果，并把“成功 / 失败 / 后续处理”组织成一套可组合的流程。**

如果把 Promise 拆开，其实就是：

```text
Promise
├── 状态管理
│   ├── pending
│   ├── fulfilled
│   └── rejected
│
├── 结果传递
│   ├── resolve(value)
│   └── reject(reason)
│
├── 后续处理
│   ├── then()
│   ├── catch()
│   └── finally()
│
├── 链式调用
│   └── then() 返回新的 Promise
│
├── 任务调度
│   └── Promise reaction → 微任务
│
└── 组合能力
    ├── Promise.all()
    ├── Promise.race()
    ├── Promise.allSettled()
    └── Promise.any()
```

这篇就从底层逻辑把它一次讲透。

<!-- more -->

## 一、为什么需要 Promise？

先从最原始的异步代码开始。

假设我们要：

```text
请求用户
 ↓
拿到用户 ID
 ↓
请求用户详情
 ↓
拿到用户详情
 ↓
请求订单
```

最原始的写法：

```js
getUser((user) => {
  getUserDetail(user.id, (detail) => {
    getOrders(detail.id, (orders) => {
      console.log(orders);
    });
  });
});
```

业务一复杂，就会出现：

```text
回调嵌套
    ↓
代码缩进越来越深
    ↓
错误处理分散
    ↓
流程难以组合
    ↓
回调地狱
```

Promise 的出现，本质上就是把：

> **“未来会得到一个结果”**

封装成一个对象。

例如：

```js
const promise = fetch('/api/user');
```

此时：

```text
promise
   │
   ├── 现在：还没有结果
   │
   └── 未来：
       ├── 成功 → 用户数据
       └── 失败 → 错误原因
```

所以 Promise 可以理解成：

> **一个代表未来结果的容器。**

ECMAScript 规范也把 Promise 描述为一个用于表示“延迟计算最终结果”的对象，并规定 Promise 具有 `pending`、`fulfilled`、`rejected` 三种状态。

---

# 二、Promise 的三个状态

Promise 最核心的东西其实不是 `then`。

而是：

> **状态机。**

Promise 一共有三个状态：

```text
pending
   │
   ├───────────────┐
   ▼               ▼
fulfilled       rejected
```

也就是：

```js
pending      // 进行中
fulfilled    // 成功
rejected     // 失败
```

例如：

```js
const promise = new Promise((resolve, reject) => {
  // ...
});
```

刚创建的时候：

```text
pending
```

调用：

```js
resolve('success');
```

之后：

```text
fulfilled
value = 'success'
```

调用：

```js
reject('error');
```

之后：

```text
rejected
reason = 'error'
```

---

## 2.1 Promise 状态只能改变一次

这是 Promise 非常重要的特性：

```js
const promise = new Promise((resolve, reject) => {
  resolve('A');
  resolve('B');
  reject('C');
});
```

最终：

```text
fulfilled
value = A
```

而不是：

```text
A
B
C
```

原因：

> Promise 一旦从 pending 进入 fulfilled 或 rejected，就不能再次改变状态。

Promises/A+ 对此也明确要求：fulfilled 和 rejected 都是最终状态，不能再次转换。

所以可以记：

```text
pending
   ↓
fulfilled
```

或者：

```text
pending
   ↓
rejected
```

但是不能：

```text
fulfilled → rejected
rejected → fulfilled
```

---

# 三、`new Promise()` 里面的代码到底什么时候执行？

这是 Promise 最容易产生误解的地方。

看代码：

```js
console.log('1');

const promise = new Promise((resolve, reject) => {
  console.log('2');

  resolve('success');
});

promise.then(() => {
  console.log('3');
});

console.log('4');
```

输出：

```text
1
2
4
3
```

为什么？

因为：

> **Promise 的 executor 是同步执行的。**

也就是说：

```js
new Promise((resolve, reject) => {
  console.log('2');
});
```

这里面的：

```js
console.log('2');
```

并不会自动变成异步代码。

真正异步的是：

```js
.then(...)
```

---

## 3.1 Promise 最重要的两个时间点

可以把 Promise 拆成：

```text
new Promise()
     │
     ├── executor：同步执行
     │
     ▼
resolve / reject
     │
     ▼
Promise 状态改变
     │
     ▼
then / catch 回调进入微任务
```

所以：

```js
new Promise(() => {
  console.log('A');
});

console.log('B');
```

输出：

```text
A
B
```

而：

```js
Promise.resolve().then(() => {
  console.log('A');
});

console.log('B');
```

输出：

```text
B
A
```

因为 Promise reaction 会进入微任务队列，而不是同步执行。

---

# 四、resolve 和 reject 到底干了什么？

看：

```js
new Promise((resolve, reject) => {
  resolve('hello');
});
```

可以把它理解成：

```text
resolve('hello')
        ↓
Promise 状态
pending → fulfilled
        ↓
保存结果
value = 'hello'
        ↓
触发对应的 Promise reactions
        ↓
把 then 回调加入微任务
```

reject 同理：

```js
reject(error)
```

表示：

```text
pending
   ↓
rejected
   ↓
reason = error
   ↓
触发 onRejected
```

ECMAScript 中，Promise 状态改变后会触发 Promise reactions，并为这些 reaction 创建 Job，再交给宿主进行 Promise Job 的调度。

---

# 五、Promise 为什么一定要有 then？

Promise 本身只负责：

```text
保存未来结果
```

但我们最终还是要：

> **拿到这个结果之后继续执行代码。**

所以有：

```js
promise.then(...)
```

例如：

```js
const promise = new Promise((resolve) => {
  resolve('hello');
});

promise.then((value) => {
  console.log(value);
});
```

这里：

```js
value
```

就是：

```text
hello
```

---

# 六、then 为什么不会同步执行？

这是 Promise 和普通回调非常重要的区别。

```js
const promise = Promise.resolve('hello');

promise.then(() => {
  console.log('then');
});

console.log('sync');
```

输出：

```text
sync
then
```

即使 Promise 已经 fulfilled：

```js
Promise.resolve('hello')
```

`then()` 的回调也不会立即同步执行。

它会进入微任务队列。

可以理解成：

```text
Promise fulfilled
       ↓
触发 Promise reaction
       ↓
加入 microtask queue
       ↓
当前同步代码执行结束
       ↓
执行微任务
```

MDN 也明确指出，即使 Promise 已经 settled，`then()` 注册的回调也不会同步调用，而是进入 microtask queue。

---

# 七、then 为什么可以链式调用？

这是理解 Promise 的核心。

看：

```js
Promise.resolve(1)
  .then((value) => {
    return value + 1;
  })
  .then((value) => {
    return value + 1;
  })
  .then((value) => {
    console.log(value);
  });
```

最终：

```text
3
```

为什么？

因为：

> **then() 本身会返回一个新的 Promise。**

也就是说：

```js
const p1 = Promise.resolve(1);

const p2 = p1.then(() => {
  return 2;
});

const p3 = p2.then(() => {
  return 3;
});
```

关系是：

```text
p1
 ↓
then()
 ↓
p2
 ↓
then()
 ↓
p3
```

所以：

```js
p1.then(...)
```

并不是简单地“注册一个回调”。

它实际上做了两件事：

```text
1. 注册当前 Promise 的后续处理
2. 创建并返回一个新的 Promise
```

这就是 Promise 链式调用的基础。

---

# 八、then 返回普通值会发生什么？

这是 Promise 链最重要的一条规则。

```js
Promise.resolve(1)
  .then((value) => {
    return 100;
  })
  .then((value) => {
    console.log(value);
  });
```

输出：

```text
100
```

因为：

```text
第一个 then
return 100
     ↓
新的 Promise fulfilled
     ↓
value = 100
     ↓
下一个 then 接收到 100
```

所以可以记：

```js
return 普通值
```

相当于：

```js
return Promise.resolve(普通值)
```

---

# 九、then 返回 Promise 会发生什么？

再看：

```js
Promise.resolve()
  .then(() => {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve('hello');
      }, 1000);
    });
  })
  .then((value) => {
    console.log(value);
  });
```

这里第二个 `then` 不会马上执行。

因为第一个 `then` 返回了一个 pending Promise：

```text
p1
 ↓
then
 ↓
返回 p2
 ↓
p2 pending
 ↓
等待 1 秒
 ↓
p2 fulfilled
 ↓
下一个 then
```

所以 Promise 链实际上形成的是：

```text
Promise
   ↓
Promise
   ↓
Promise
   ↓
Promise
```

每个 `then()` 都可能产生一个新的 Promise。

---

# 十、这就是 Promise Resolution Procedure

真正复杂的地方来了。

假设：

```js
p1.then(() => {
  return p2;
});
```

那么：

```text
p1 fulfilled
    ↓
执行 then 回调
    ↓
返回 p2
    ↓
新的 Promise 必须“跟随” p2
    ↓
p2 fulfilled
    ↓
新 Promise fulfilled

或者

p2 rejected
    ↓
新 Promise rejected
```

所以 Promise 不只是：

```js
return value
```

这么简单。

它还需要处理：

```js
return Promise
```

甚至：

```js
return thenable
```

例如：

```js
return {
  then(resolve) {
    resolve('hello');
  }
};
```

这也是手写 Promise 最难的地方之一。

---

# 十一、Promise 的值穿透

再看一个经常出现在面试中的问题：

```js
Promise.resolve(100)
  .then()
  .then()
  .then((value) => {
    console.log(value);
  });
```

输出：

```text
100
```

为什么？

因为：

```js
.then()
```

没有传处理函数。

它不会把值吃掉，而是继续向下传递。

可以理解成：

```js
.then()
```

相当于：

```js
.then(value => value)
```

对于 rejected：

```js
.catch()
```

也是类似的错误传递机制。

---

# 十二、Promise 的异常处理

Promise 的另一个巨大价值：

> **异步流程可以统一处理错误。**

例如：

```js
fetch('/api/user')
  .then((res) => res.json())
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error(error);
  });
```

如果链中的某一步：

```js
throw new Error('出错了');
```

后面的：

```js
.catch(...)
```

可以捕获。

例如：

```js
Promise.resolve()
  .then(() => {
    throw new Error('something wrong');
  })
  .then(() => {
    console.log('不会执行');
  })
  .catch((error) => {
    console.log('捕获:', error.message);
  });
```

输出：

```text
捕获: something wrong
```

其本质是：

```text
then 回调执行
      ↓
throw
      ↓
当前 then 返回的新 Promise
      ↓
rejected
      ↓
后续 catch
```

Promises/A+ 对 `then` 的定义也规定：如果处理函数执行过程中产生异常，对应的新 Promise 应当进入 rejected 状态。

---

# 十三、为什么 catch 能捕获前面的错误？

因为：

```js
.then(...)
.then(...)
.then(...)
.catch(...)
```

实际上就是一条 Promise 链：

```text
p1
 ↓
p2
 ↓
p3
 ↓
p4
 ↓
p5
```

如果：

```text
p3 rejected
```

而中间没有处理：

```text
p3 rejected
 ↓
继续向后传递
 ↓
p4 rejected
 ↓
p5 rejected
 ↓
catch
```

所以可以理解：

> **错误会沿着 Promise 链向后传播，直到遇到 rejected handler。**

---

# 十四、then 的两个参数

你可能见过：

```js
promise.then(
  (value) => {},
  (error) => {}
);
```

第一个：

```js
onFulfilled
```

处理成功。

第二个：

```js
onRejected
```

处理失败。

例如：

```js
Promise.resolve('success')
  .then(
    (value) => {
      console.log('成功:', value);
    },
    (error) => {
      console.log('失败:', error);
    }
  );
```

但是工程实践中更推荐：

```js
promise
  .then((value) => {
    // success
  })
  .catch((error) => {
    // error
  });
```

因为 `catch` 可以统一处理 Promise 链中前面传播过来的异常。

---

# 十五、finally 是干什么的？

`finally()` 表示：

> **无论成功还是失败，最终都要执行。**

例如：

```js
showLoading();

fetch('/api/user')
  .then((res) => res.json())
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error(error);
  })
  .finally(() => {
    hideLoading();
  });
```

典型用途：

```text
loading
    ↓
请求
    ↓
成功 ───┐
        │
失败 ───┤
        ↓
finally
        ↓
关闭 loading
```

所以：

```text
then    → 成功处理
catch   → 失败处理
finally → 无论如何都执行
```

---

# 十六、Promise 和 Event Loop 到底是什么关系？

这是理解 Promise 的关键。

先看：

```js
console.log('1');

Promise.resolve().then(() => {
  console.log('2');
});

console.log('3');
```

输出：

```text
1
3
2
```

原因：

```text
同步代码
│
├── console.log(1)
│
├── Promise.then()
│      ↓
│   微任务入队
│
└── console.log(3)

同步代码结束
      ↓
清空微任务
      ↓
console.log(2)
```

所以：

> **Promise 本身不是事件循环。**

Promise 负责：

```text
状态
+
结果
+
reaction
```

宿主环境负责：

```text
什么时候执行这些 Job
```

而 Promise 的 reaction 会被安排到微任务机制中执行。

MDN 也明确说明 Promise callbacks 会通过 microtask 机制执行，而 `setTimeout` callback 属于 task。

---

# 十七、Promise vs setTimeout

经典面试题：

```js
console.log(1);

setTimeout(() => {
  console.log(2);
}, 0);

Promise.resolve().then(() => {
  console.log(3);
});

console.log(4);
```

结果：

```text
1
4
3
2
```

画出来：

```text
同步任务
├── 1
├── setTimeout → Task
├── Promise.then → Microtask
└── 4

        ↓

Microtask
└── 3

        ↓

Task
└── 2
```

所以不要背：

> Promise 比 setTimeout 快。

应该理解成：

> **Promise reaction 属于微任务，而 setTimeout 回调属于 task；当前任务结束后，微任务会先于后续 task 得到执行机会。**

---

# 十八、Promise 链到底是怎么跑起来的？

来看一道经典题：

```js
Promise.resolve()
  .then(() => {
    console.log(1);
  })
  .then(() => {
    console.log(2);
  });

console.log(3);
```

输出：

```text
3
1
2
```

很多人会误以为：

```text
Promise.resolve()
 ↓
then 1
 ↓
then 2
```

然后觉得 `1、2` 一起执行。

实际上：

```text
同步阶段
    ↓
第一个 then reaction 入队
    ↓
打印 3
    ↓
同步结束

Microtask 1
    ↓
执行 then 1
    ↓
产生下一个 Promise
    ↓
第二个 then reaction 入队

Microtask 2
    ↓
执行 then 2
```

所以：

```text
3
1
2
```

而不是：

```text
3
1
2
```

“看起来一样”并不代表内部机制一样。

理解这一点之后，再去分析复杂的：

```js
Promise
  .then()
  .then()
  .then()
```

就不会靠背诵。

---

# 十九、Promise.all 到底解决什么问题？

假设同时请求：

```text
用户
商品
订单
```

如果串行：

```js
const user = await getUser();
const goods = await getGoods();
const orders = await getOrders();
```

耗时大概：

```text
T = T(user) + T(goods) + T(orders)
```

但三个请求互不依赖：

```js
const [user, goods, orders] = await Promise.all([
  getUser(),
  getGoods(),
  getOrders()
]);
```

就可以并发发起。

耗时更接近：

```text
T = max(
  T(user),
  T(goods),
  T(orders)
)
```

所以：

> **Promise.all 的核心价值不是“同时执行 Promise”，而是统一等待多个异步结果。**

---

# 二十、Promise.all 的特点

```js
Promise.all([
  Promise.resolve('A'),
  Promise.resolve('B'),
  Promise.resolve('C')
]);
```

结果：

```js
['A', 'B', 'C']
```

注意：

> **结果顺序按照输入顺序，而不是完成顺序。**

例如：

```text
A → 300ms
B → 100ms
C → 200ms
```

完成顺序：

```text
B
C
A
```

但：

```js
Promise.all([A, B, C])
```

最终仍然：

```text
[A, B, C]
```

---

# 二十一、Promise.all 为什么一个失败就失败？

例如：

```js
Promise.all([
  requestA(),
  requestB(),
  requestC()
]);
```

如果：

```text
A fulfilled
B rejected
C fulfilled
```

最终：

```text
Promise.all → rejected
```

因为：

> `Promise.all` 表示“全部成功”。

所以：

```text
全部成功
    ↓
fulfilled

任意一个失败
    ↓
rejected
```

但要注意：

**Promise.all 失败，并不会自动取消其他已经发出的请求。**

这是一个非常容易被忽略的点。

---

# 二十二、Promise.allSettled

如果我们想：

> 不管成功还是失败，全部执行完以后告诉我结果。

使用：

```js
Promise.allSettled([
  requestA(),
  requestB(),
  requestC()
]);
```

返回类似：

```js
[
  {
    status: 'fulfilled',
    value: 'A'
  },
  {
    status: 'rejected',
    reason: 'B error'
  },
  {
    status: 'fulfilled',
    value: 'C'
  }
]
```

所以：

```text
Promise.all
    ↓
全部成功才成功

Promise.allSettled
    ↓
全部结束才结束
```

---

# 二十三、Promise.race

`race`：

> **谁先 settle，结果就跟谁。**

例如：

```js
Promise.race([
  request(),
  timeout()
]);
```

可以实现：

```text
请求
  │
  ├── 先成功 → 返回请求结果
  │
  ├── 先失败 → 返回错误
  │
  └── timeout 先完成 → 超时
```

但是注意：

> `race` 只是决定返回哪个 Promise 的结果，并不会自动取消其他任务。

---

# 二十四、Promise.any

`Promise.any` 和 `Promise.race` 很像，但判断条件不同。

```text
race
 ↓
谁先 settle 就用谁

any
 ↓
谁先 fulfilled 就用谁
```

例如：

```js
Promise.any([
  requestServerA(),
  requestServerB(),
  requestServerC()
]);
```

只要：

```text
A 成功
```

就可以返回 A。

即使：

```text
B 失败
C 失败
```

也不影响。

只有：

```text
A 失败
B 失败
C 失败
```

才会最终 rejected。

---

# 二十五、四个组合 API 怎么记？

直接记这张表：

| API                  | 成功条件          | 失败条件               |
| -------------------- | ------------- | ------------------ |
| `Promise.all`        | 全部成功          | 任意一个失败             |
| `Promise.allSettled` | 全部结束          | 不会因为单个失败而 rejected |
| `Promise.race`       | 第一个 settle    | 第一个 settle         |
| `Promise.any`        | 第一个 fulfilled | 全部 rejected        |

一句话：

```text
all
→ 全部成功

allSettled
→ 全部结束

race
→ 第一个结束

any
→ 第一个成功
```

---

# 二十六、async / await 到底是什么？

现在来看 Promise 的最后一块拼图：

```js
async function getUser() {
  const user = await fetchUser();
  return user;
}
```

很多人会说：

> async/await 是 Promise 的替代品。

其实不准确。

应该说：

> **async/await 是建立在 Promise 之上的语法机制。**

MDN 也明确指出，`async/await` 建立在 Promise 之上，能够用更接近同步代码的形式表达 Promise 异步流程。

---

# 二十七、async 函数一定返回 Promise

例如：

```js
async function foo() {
  return 123;
}
```

实际上：

```js
foo()
```

得到的是：

```js
Promise
```

所以：

```js
foo().then((value) => {
  console.log(value);
});
```

输出：

```text
123
```

可以理解成：

```js
async function foo() {
  return 123;
}
```

类似：

```js
function foo() {
  return Promise.resolve(123);
}
```

---

# 二十八、await 做了什么？

例如：

```js
async function foo() {
  console.log(1);

  const result = await Promise.resolve(2);

  console.log(result);
}
```

可以理解为：

```text
执行 foo
 ↓
打印 1
 ↓
遇到 await
 ↓
暂停当前 async 函数
 ↓
Promise settled
 ↓
后续代码进入微任务机制
 ↓
恢复执行
 ↓
打印 2
```

重要的是：

> `await` 暂停的是当前 async 函数，不是整个 JavaScript 线程。

其他任务仍然可以继续运行。

---

# 二十九、await 为什么经常和 Promise 顺序题绑在一起？

例如：

```js
async function foo() {
  console.log('A');

  await Promise.resolve();

  console.log('B');
}

foo();

console.log('C');
```

输出：

```text
A
C
B
```

因为：

```text
foo()
 ↓
A
 ↓
await
 ↓
暂停 foo
 ↓
返回调用者
 ↓
C
 ↓
当前任务结束
 ↓
微任务
 ↓
B
```

所以记住：

> **await 后面的代码，不会继续在当前同步执行流中跑。**

---

# 三十、Promise 的本质到底是什么？

到这里，可以重新回答最开始的问题。

Promise 到底是什么？

我现在更愿意把它理解成：

> **Promise 是 JavaScript 中用于表示异步操作最终结果的对象，它通过状态机保存结果，通过 `then/catch/finally` 注册后续处理，并通过 Promise reaction 将这些处理安排到微任务机制中执行。**

拆开就是：

```text
Promise
│
├── ① 状态
│     pending
│     fulfilled
│     rejected
│
├── ② 结果
│     value
│     reason
│
├── ③ 状态转换
│     resolve
│     reject
│
├── ④ 后续处理
│     then
│     catch
│     finally
│
├── ⑤ 链式调用
│     then → new Promise
│
├── ⑥ 异常传播
│     throw → rejected
│
└── ⑦ 调度
      Promise reaction
           ↓
      Microtask
```

---

# 三十一、手写 Promise：先不要急着写代码

如果想真正理解 Promise，**手写 Promise 是非常好的方法**。

但不要一上来就写：

```js
class MyPromise {
  // 一堆代码
}
```

先想清楚 Promise 最少需要什么。

```js
class MyPromise {
  constructor(executor) {}

  then(onFulfilled, onRejected) {}

  catch(onRejected) {}

  finally(onFinally) {}
}
```

内部至少需要：

```js
this.status
this.value
this.reason
```

状态：

```js
const PENDING = 'pending';
const FULFILLED = 'fulfilled';
const REJECTED = 'rejected';
```

还需要保存：

```js
onFulfilledCallbacks
onRejectedCallbacks
```

为什么？

因为：

```js
const p = new MyPromise((resolve) => {
  setTimeout(() => {
    resolve('hello');
  }, 1000);
});

p.then(...)
p.then(...)
p.then(...)
```

Promise 还没完成的时候：

```text
pending
│
├── then callback 1
├── then callback 2
└── then callback 3
```

所以需要先保存。

等：

```js
resolve('hello');
```

之后：

```text
fulfilled
 ↓
依次触发 callbacks
```

这就是手写 Promise 的第一层。

---

# 三十二、手写 Promise 最难的是什么？

不是：

```js
resolve()
reject()
```

而是：

> **then 的返回值处理。**

例如：

```js
const p2 = p1.then(() => {
  return 100;
});
```

需要：

```text
p1
 ↓
then
 ↓
执行回调
 ↓
得到 100
 ↓
resolve(p2, 100)
```

如果：

```js
return Promise.resolve(100);
```

则：

```text
p1
 ↓
then
 ↓
得到 Promise
 ↓
等待这个 Promise
 ↓
p2 跟随它的状态
```

如果：

```js
throw new Error();
```

则：

```text
p1
 ↓
then
 ↓
throw
 ↓
p2 rejected
```

所以真正的 Promise 核心不是状态机本身，而是：

```text
状态机
+
then 链
+
返回值解析
+
异常传播
+
异步调度
```

---

# 三十三、Promise 最容易记错的几个点

## 误区 1：Promise 是异步的

不准确。

```js
new Promise(() => {
  console.log('A');
});
```

`executor` 是同步执行的。

真正通过 Promise reaction 延迟执行的是：

```js
.then(...)
.catch(...)
.finally(...)
```

---

## 误区 2：Promise.resolve() 会立即执行 then

错误。

```js
Promise.resolve().then(() => {
  console.log('A');
});

console.log('B');
```

输出：

```text
B
A
```

---

## 误区 3：Promise 就是线程

错误。

Promise：

```text
不是线程
不是 Web Worker
不是线程池
```

Promise 只是：

> **异步结果的抽象 + 后续处理机制。**

真正执行网络、文件、定时器等异步工作的，是宿主环境提供的能力。

---

## 误区 4：async/await 不用 Promise

错误。

```js
async function foo() {
  await bar();
}
```

底层抽象依然建立在 Promise 机制之上。

---

## 误区 5：Promise.all 会取消其他请求

错误。

```js
Promise.all([
  requestA(),
  requestB(),
  requestC()
]);
```

其中 B 失败：

```text
Promise.all → rejected
```

但：

```text
A / C
```

已经开始的任务不会因此自动取消。

如果需要取消请求，需要结合：

```js
AbortController
```

等机制。

---

# 三十四、面试时如何回答“Promise 的原理？”

如果面试官问：

> **“你能讲一下 Promise 的原理吗？”**

不要从：

```text
Promise.all
Promise.race
Promise.any
```

开始。

推荐按照这个顺序：

```text
第一层：Promise 是什么
        ↓
第二层：三种状态
        ↓
第三层：resolve / reject
        ↓
第四层：then
        ↓
第五层：then 返回新 Promise
        ↓
第六层：返回值解析
        ↓
第七层：异常传播
        ↓
第八层：Promise reaction → 微任务
        ↓
第九层：async/await
```

可以直接回答：

> Promise 本质上是一个表示异步操作最终结果的对象。它内部有 pending、fulfilled、rejected 三种状态，状态只能从 pending 转换到 fulfilled 或 rejected。通过 resolve 和 reject 改变状态并保存结果，通过 then、catch、finally 注册后续处理。then 不会同步执行回调，而是通过 Promise reaction 将任务安排到微任务机制中。更重要的是 then 会返回一个新的 Promise，因此可以形成链式调用；如果 then 返回普通值，新 Promise 会 fulfilled；如果返回 Promise，则会等待并跟随它的状态；如果回调抛出异常，则新 Promise 会 rejected。async/await 则是在 Promise 之上提供的更直观的异步代码写法。

这段基本就是：

**Promise 原理面试答案。**

---

# 三十五、最后把 Promise 串成一张图

如果把整篇文章压缩成一张图：

```text
                     Promise
                        │
                        ▼
                ┌───────────────┐
                │    pending    │
                └───────┬───────┘
                        │
             ┌──────────┴──────────┐
             │                     │
          resolve                reject
             │                     │
             ▼                     ▼
       ┌───────────┐         ┌───────────┐
       │ fulfilled │         │ rejected  │
       └─────┬─────┘         └─────┬─────┘
             │                     │
             └──────────┬──────────┘
                        ▼
                   Promise
                   Reaction
                        │
                        ▼
                   Microtask
                        │
                        ▼
                      then
                        │
             ┌──────────┼──────────┐
             │          │          │
        return value  return P    throw
             │          │          │
             ▼          ▼          ▼
         fulfilled    等待 P      rejected
             │          │          │
             └──────────┼──────────┘
                        ▼
                    新 Promise
                        │
                        ▼
                     then...
```

这张图基本就是 Promise 的核心。

---

# 三十六、最后的总结

如果只让我留下几个结论，我会留下这些：

### 1. Promise 是什么？

> **表示未来结果的对象。**

### 2. Promise 最核心的东西是什么？

> **三状态状态机。**

```text
pending
   ↓
fulfilled / rejected
```

### 3. `new Promise()` 里面的代码同步还是异步？

> **同步。**

### 4. `then()` 为什么异步？

> Promise reaction 会进入微任务机制，而不是同步执行。

### 5. `then()` 为什么可以链式调用？

> **因为 then 会返回一个新的 Promise。**

### 6. then 返回普通值？

```text
return value
 ↓
新 Promise fulfilled
```

### 7. then 返回 Promise？

```text
return Promise
 ↓
等待并跟随它的状态
```

### 8. then 里面 throw？

```text
throw
 ↓
新 Promise rejected
```

### 9. async/await 是什么？

> **建立在 Promise 之上的异步语法。**

### 10. Promise 和 Event Loop 什么关系？

```text
Promise 状态改变
      ↓
Promise reaction
      ↓
微任务
      ↓
事件循环调度执行
```

---

## 一句话理解 Promise

以前我会说：

> **Promise 是解决回调地狱的。**

现在我更愿意说：

> **Promise 是一套“异步结果管理 + 后续任务组合”的抽象：用状态表示结果是否完成，用 `then/catch/finally` 描述后续行为，用链式 Promise 传递结果和错误，再通过微任务机制把这些后续处理异步调度起来。**

当你真正理解这句话之后，`Promise`、`async/await`、`Promise.all`、`Event Loop`、`微任务`、甚至**手写 Promise**，其实就已经串成了一条线。

---

<!-- ## 参考资料

* ECMAScript Promise Objects：[ECMAScript Promise 规范](https://tc39.es/ecma262/2025/multipage/control-abstraction-objects.html?utm_source=chatgpt.com)
* MDN — Using Promises：[MDN Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises?utm_source=chatgpt.com)
* MDN — `await`：[MDN await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await?utm_source=chatgpt.com)
* Promises/A+：[Promises/A+ Specification](https://github.com/promises-aplus/promises-spec?utm_source=chatgpt.com) -->
