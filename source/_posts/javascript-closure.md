---
title: JavaScript 闭包：从作用域链到数据私有化
date: 2026-09-10 14:30:00
description: 闭包不是「函数记住外部变量」这句口诀，而是词法作用域与引用关系的自然结果：函数创建时就保存了定义处的词法环境，外部函数执行完毕后，该环境因被内部函数引用而保留。本文从作用域链讲到闭包的形成原理、引用而非快照、数据私有化与模块模式，再到循环陷阱与内存注意点。
categories:
  - [前端基础, JavaScript]
tags:
  - JavaScript
  - 闭包
  - 作用域
---

> 闭包是 JavaScript 中一个非常重要的概念。
> 它看起来像是「函数记住了外部变量」，但如果只记住这句话，很容易在遇到实际代码时产生疑惑。
>
> 真正理解闭包，需要先理解 **作用域、作用域链以及函数执行时的变量查找过程**。

一句话版本：

> **闭包 = 函数 + 它定义时所处词法环境的引用。外部函数执行结束后，只要内部函数还在被使用，这个环境就不会被回收。**

<!-- more -->

## 一、先看现象：函数「记住」了外部的变量

先看一个最简单的例子：

```javascript
function outer() {
  const name = 'Tom'

  function inner() {
    console.log(name)
  }

  return inner
}

const fn = outer()
fn() // Tom
```

值得注意的地方在于：`outer()` 执行完成后，按直觉它的执行上下文应该销毁了，`name` 也应该随之消失。但 `fn()` 依然打印出了 `Tom`。

再看一个更有「状态感」的版本：

```javascript
function outer() {
  let count = 0

  return function () {
    count++
    console.log(count)
  }
}

const increment = outer()

increment() // 1
increment() // 2
increment() // 3
```

`increment` 不仅还能访问 `count`，还能**持续修改**它——三次调用共享同一个 `count`：

```text
outer()
  ↓ 创建 count
  ↓ 创建内部函数（引用着 count 所在的作用域）
  ↓ 返回内部函数
  ↓ outer 执行结束
  ↓ increment 仍然可以读写 count
```

这就是闭包最典型的表现：**当一个函数能够访问它定义时所在作用域中的变量，即使那个外部作用域已经执行结束，这种现象就是闭包。**

## 二、前置知识：词法作用域与作用域链

闭包不是孤立的语法特性，它建立在 JavaScript 的**词法作用域**之上——变量的可见性由代码书写的位置决定，而不是由调用的位置决定。

```javascript
const name = 'global'

function outer() {
  const name = 'outer'

  function inner() {
    console.log(name)
  }

  inner()
}

outer() // outer
```

`inner` 打印的是 `outer`，因为它查找 `name` 时，沿着**定义时**的作用域关系一层层向外找：

```text
inner 作用域  → 没找到 name
  ↓
outer 作用域  → 找到 name = 'outer' ✅
  ↓
全局作用域
```

这条逐层向外的查找路径就是**作用域链**。再换个角度验证一下它的方向性——内层可以访问外层，反过来不行：

```javascript
function outer() {
  function inner() {
    const name = 'Tom'
  }

  inner()
  console.log(name) // ReferenceError: name is not defined
}
```

`outer` 永远看不到 `inner` 内部的变量，因为作用域链的方向是**内层 → 外层**，变量只能「向上」查找，不能「向下」穿透。

记住这一点，闭包的原理就已经讲完一半了：**内部函数天生就持有通向外层作用域的查找路径，无论它之后在哪里被调用。**

## 三、闭包是怎么形成的

用经典的计数器拆解整个过程：

```javascript
function createCounter() {
  let count = 0

  return function () {
    count++
    return count
  }
}

const counter = createCounter()
```

分三步看：

**第 1 步：执行 `createCounter`。** 创建变量 `count`，同时创建内部匿名函数。关键在于：**函数创建的那一刻，就把定义时所处的词法环境保存了下来**（ECMAScript 规范中函数通过内部槽 `[[Environment]]` 持有这个引用）——闭包的「记忆」不是魔法，就是这个引用。

**第 2 步：返回内部函数。** `counter` 拿到的是内部函数的引用，而这个函数又引用着 `createCounter` 的作用域：

```text
counter
  ↓
内部函数 ──[[Environment]]──▶ createCounter 的词法环境（count = 0）
```

**第 3 步：`createCounter` 执行结束。** 它的执行上下文出栈了，但它的**词法环境**不会销毁——因为还有一条从 `counter` 出发、经由内部函数指向它的引用链。所以 `counter()` 依然可以读写 `count`。

因此不要把闭包理解成一个特殊的语法，它只是：

> **函数与其词法作用域之间的一种持续引用关系。**

## 四、为什么外部函数结束了，变量还活着

这是初学闭包时最容易困惑的地方：「函数都执行完了，局部变量不是应该销毁吗？」

答案藏在垃圾回收的判定规则里。JavaScript 的垃圾回收关注的是：

> **对象是否仍然可达。**

从 `fn` 出发存在一条完整的引用链：

```text
fn
 ↓
inner 函数
 ↓ [[Environment]]
outer 的词法环境
 ↓
count          ← 仍然可达，不能回收
```

只要这条链还在，`count` 就一直活着。换句话说，不是「闭包强行留住了变量」，而是**「只要有对象还被引用着，垃圾回收就不动它」这条通用规则，恰好覆盖了被闭包引用的变量**。

## 五、闭包捕获的是变量本身，不是快照

一个常被忽略、但解释了很多现象的细节：闭包保存的是**变量的引用**，而不是创建函数那一刻的**值拷贝**。

```javascript
function outer() {
  let x = 1

  const getX = () => x

  x = 2
  console.log(getX()) // 2，不是 1
}
```

`getX` 拿到的永远是 `x` **当前**的值。理解了这一点，下一节的经典循环陷阱就不再神秘——它本质上就是「三个回调共享同一个变量的引用」。

## 六、典型应用：状态保存与数据私有化

闭包不是炫技，实际开发里到处都是。常见的几类应用：

### 1. 私有状态：计数器

```javascript
function createCounter() {
  let count = 0

  return {
    increment() { count++ },
    decrement() { count-- },
    getCount()  { return count }
  }
}

const counter = createCounter()

counter.increment()
counter.increment()
console.log(counter.getCount()) // 2
```

`count` 没有作为属性暴露出去，`counter.count = 100` 这样的修改是无效的——外部只能通过 `increment` / `decrement` / `getCount` 操作它。这就是**数据私有化**：数据只能通过受控的方法访问。

一个更实际的例子，用户对象不把密码直接挂在外面：

```javascript
function createUser() {
  let username = 'Tom'
  let password = '123456'

  return {
    getUsername() { return username },
    changePassword(newPassword) { password = newPassword }
  }
}

const user = createUser()
console.log(user.getUsername()) // Tom
```

对比直接暴露的 `{ username, password }`，多了一层访问控制。

### 2. 工厂函数：每个实例独立的状态

每次调用工厂函数都会产生一个全新的词法环境，所以每个实例的闭包状态互相独立：

```javascript
function createPerson(name) {
  return {
    sayHello() {
      console.log(`Hello, ${name}`)
    }
  }
}

const person1 = createPerson('Tom')
const person2 = createPerson('Jack')

person1.sayHello() // Hello, Tom
person2.sayHello() // Hello, Jack
```

### 3. 模块模式（IIFE）

在 ES Module 出现之前，这是实现模块的主要手段，至今仍有大量老代码在用：

```javascript
const counter = (() => {
  let count = 0

  return {
    increment() { count++ },
    getCount()  { return count }
  }
})()

counter.increment()
counter.increment()
console.log(counter.getCount()) // 2
console.log(counter.count)      // undefined —— count 不是公开属性
```

### 4. 防抖与节流

`debounce` / `throttle` 的实现核心就是用闭包保存 `timer` 和 `lastTime`，让多次调用之间共享状态：

```javascript
function debounce(fn, delay) {
  let timer = null

  return function (...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}
```

完整实现（立即执行版、合并版、应用场景）见 [手写防抖与节流](/2026/09/10/javascript-debounce-throttle/)。

### 5. 异步回调

定时器、事件监听里的回调函数同样形成闭包：

```javascript
function createTimer(name) {
  setTimeout(() => {
    console.log(name) // 1 秒后依然能访问 name
  }, 1000)
}

createTimer('任务完成')
```

`createTimer` 早就执行结束了，但 `name` 会被回调引用着，存活到回调执行为止。

## 七、经典陷阱：循环 + var + 异步

闭包最经典的坑，就是循环中的变量：

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i)
  }, 0)
}
```

直觉上应该输出 `0 1 2`，实际输出：

```text
3
3
3
```

原因正是第五节说的「捕获引用而非快照」：`var` 声明的 `i` 属于函数作用域，整个循环只有**一个** `i`，三个回调闭包引用的是同一个变量：

```text
      i（共享）
      │
 ┌────┼────┐
 ↓    ↓    ↓
回调1 回调2 回调3
```

等到定时器执行时循环早已结束，`i` 的值是 `3`，三个回调看到的都是 `3`。

### 解法一：`let`

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i)
  }, 0)
}
// 输出 0 1 2
```

`let` 具有块级作用域，规范还为 `for` 循环做了特殊处理：**每次迭代都会创建一个新的 `i` 绑定**。相当于每轮循环都有一个独立的 `i`，回调闭包各自引用各自的：

```text
第 1 次循环：i = 0 → 回调1 引用这个 i
第 2 次循环：i = 1 → 回调2 引用这个 i
第 3 次循环：i = 2 → 回调3 引用这个 i
```

### 解法二：IIFE 创建独立作用域

在 `let` 出现之前（以及需要兼容旧环境时）的写法，用立即执行函数把当前的值「接住」：

```javascript
for (var i = 0; i < 3; i++) {
  (function (index) {
    setTimeout(() => {
      console.log(index)
    }, 0)
  })(i)
}
// 输出 0 1 2
```

每次迭代执行一次 IIFE，参数 `index` 是值拷贝，各自形成独立的函数作用域，回调闭包保存的是自己的 `index`。

现代 JavaScript 中直接用 `let` 即可，更简单。

## 八、闭包与内存：不是泄漏，但可能被误用

一个重要的误区需要澄清：闭包**不等于**内存泄漏。

> 闭包确实会延长变量的生命周期，但「被闭包引用所以不回收」和「该回收却没回收」是两回事。垃圾回收关注的永远是**可达性**。

```javascript
function outer() {
  const data = new Array(1000000).fill('data')

  return function () {
    console.log(data.length)
  }
}

let fn = outer()
// fn → 内部函数 → data：可达，不回收

fn = null
// 引用链断开，data 有机会被回收
```

真正需要注意的是：**闭包被长生命周期的东西持有，且引用了大对象**。常见的「持有者」：

- 全局变量
- 未清理的定时器
- 未解绑的事件监听器
- 缓存、长生命周期的单例对象

典型场景是事件监听：

```javascript
const button = document.querySelector('#button')

function createHandler() {
  const data = new Array(1000000).fill('data')

  return function () {
    console.log(data.length)
  }
}

const handler = createHandler()
button.addEventListener('click', handler)

// 组件销毁时应解除监听，否则 data 会随 handler 一直存活
button.removeEventListener('click', handler)
```

所以结论是：

> **闭包本身不是内存泄漏；错误地长期持有闭包引用（尤其是引用了大对象的闭包），才可能导致内存无法及时释放。**

## 九、如何真正理解闭包：三个层次

**第一层：现象。** 函数可以访问定义位置外部的变量，即使外部函数已经执行结束。

**第二层：原因。** JavaScript 使用词法作用域，函数在**创建时**就通过 `[[Environment]]` 保存了定义处的词法环境；内部函数无论之后被传到哪里调用，都沿着这条引用找回外部变量。

**第三层：结果。** 被内部函数引用的变量，生命周期会被延长。由此衍生出一系列能力与代价：

- 能力：保存状态、数据私有化、独立作用域、防抖节流、工厂函数、模块模式
- 代价：注意事件监听器解绑、定时器清理、大对象不要被闭包意外长期引用

## 十、总结

闭包可以浓缩成一句话：

> **闭包就是函数与其词法作用域之间形成的持续引用关系，使函数即使离开原来的执行环境，也仍然能够访问其中的变量。**

理解闭包，不需要死记「函数 + 外部变量 = 闭包」，更重要的是串起整个过程：

```text
词法作用域
    ↓
作用域链（内层 → 外层查找）
    ↓
函数创建时保存词法环境的引用（捕获的是变量引用，不是快照）
    ↓
内部函数被返回 / 被保存
    ↓
外部函数执行结束，但词法环境因被引用而保留
    ↓
形成闭包，外部变量生命周期延长
```

闭包真正有价值的地方 = **保存状态 + 控制数据访问 + 延长变量生命周期**。

因此，闭包并不是 JavaScript 中一个「特殊的语法」，而是**作用域、函数以及引用关系共同产生的一种语言机制**。当能从这个角度理解它之后，很多看起来独立的问题——防抖、节流、循环中的异步回调、数据私有化、工厂函数——其实都可以归结到同一个概念上。

相关阅读：[手写防抖与节流](/2026/09/10/javascript-debounce-throttle/) · [事件循环全景](/2026/09/09/javascript-event-loop/)
