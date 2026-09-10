---

title: JavaScript 闭包：从作用域到数据私有化
date: 2026-09-10 14:30:00
description: 从作用域链、执行上下文出发，理解 JavaScript 闭包的形成原理、应用场景与常见问题
categories:
 - [JavaScript]
tags:
 - JavaScript
 - 闭包
---

# JavaScript 闭包：从作用域到数据私有化

> 闭包是 JavaScript 中一个非常重要的概念。
> 它看起来像是“函数记住了外部变量”，但如果只记住这句话，很容易在遇到实际代码时产生疑惑。
>
> 真正理解闭包，需要先理解 **作用域、作用域链以及函数执行时的变量查找过程**。

---

## 一、什么是闭包？

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

fn()
```

执行结果：

```text
Tom
```

这里有一个值得注意的地方：

```javascript
const fn = outer()
```

`outer()` 执行完成以后，按直觉来说，`outer` 的执行环境应该结束了。

但是：

```javascript
fn()
```

依然可以访问：

```javascript
const name = 'Tom'
```

这就是闭包最典型的表现。

可以简单理解为：

> **当一个函数能够访问并使用它定义时所在作用域中的变量，即使这个外部作用域已经执行结束，这种现象就形成了闭包。**

例如：

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

这里 `increment` 一直可以访问 `outer` 中的 `count`。

因此：

```text
outer()
  ↓
创建 count
  ↓
创建内部函数
  ↓
返回内部函数
  ↓
outer 执行结束
  ↓
increment 仍然可以访问 count
```

这就是闭包。

---

# 二、理解闭包之前，先理解作用域

闭包并不是一个孤立的概念，它建立在 JavaScript 的**词法作用域**之上。

例如：

```javascript
const name = 'global'

function outer() {
  const name = 'outer'

  function inner() {
    console.log(name)
  }

  inner()
}

outer()
```

输出：

```text
outer
```

为什么？

因为 `inner` 定义在 `outer` 内部。

当 `inner` 查找：

```javascript
name
```

时，会按照它定义时的作用域关系进行查找：

```text
inner
 ↓
outer
 ↓
global
```

这就是**作用域链**。

---

# 三、作用域链是什么？

可以把作用域链理解成：

> **变量查找时，一层一层向外寻找的路径。**

例如：

```javascript
const a = 1

function outer() {
  const b = 2

  function inner() {
    const c = 3

    console.log(a)
    console.log(b)
    console.log(c)
  }

  inner()
}

outer()
```

`inner` 中访问三个变量：

```javascript
a
b
c
```

查找过程大致可以理解为：

```text
inner 作用域
 ├── c ✅
 │
 ↓

outer 作用域
 ├── b ✅
 │
 ↓

全局作用域
 ├── a ✅
```

所以：

```javascript
inner
```

能够访问：

```javascript
inner 自己的变量
↓
outer 的变量
↓
更外层作用域的变量
↓
全局变量
```

但是反过来不成立。

例如：

```javascript
function outer() {

  function inner() {
    const name = 'Tom'
  }

  console.log(name)
}
```

这里：

```javascript
console.log(name)
```

无法访问 `inner` 中的 `name`。

因为作用域链的方向是：

```text
内层 → 外层
```

而不是：

```text
外层 → 内层
```

---

# 四、闭包是怎么形成的？

看一个经典例子：

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

此时发生了什么？

## 1. 执行 createCounter

创建：

```javascript
count = 0
```

同时创建内部函数：

```javascript
function () {
  count++
  return count
}
```

---

## 2. 返回内部函数

```javascript
return function () {
  count++
  return count
}
```

然后：

```javascript
const counter = createCounter()
```

此时：

```text
counter
  ↓
内部函数
  ↓
createCounter 的作用域
  ↓
count
```

---

## 3. createCounter 执行结束

虽然：

```javascript
createCounter()
```

已经执行结束，但是内部函数还在被：

```javascript
counter
```

引用。

因此它依赖的外部变量：

```javascript
count
```

不能被简单地回收。

所以：

```javascript
counter()
```

仍然能够访问：

```javascript
count
```

并修改它。

---

# 五、闭包的核心：函数 + 外部作用域

可以把闭包简单抽象成：

```text
闭包
=
函数
+
函数能够访问的外部词法环境
```

例如：

```javascript
function outer() {
  const name = 'Tom'

  return function inner() {
    console.log(name)
  }
}
```

这里：

```text
inner 函数
+
outer 作用域
```

形成了闭包关系。

因此不要把闭包理解成一个特殊的语法。

它实际上是：

> **函数与其词法作用域之间的一种关系。**

---

# 六、为什么函数执行结束后，变量还存在？

这是很多人第一次理解闭包时最容易产生疑惑的地方。

例如：

```javascript
function outer() {
  let count = 0

  return function () {
    return ++count
  }
}

const fn = outer()
```

有人会认为：

```text
outer 执行结束
↓
count 应该销毁
```

但实际上不能简单这么理解。

因为：

```javascript
const fn = outer()
```

得到的函数仍然需要：

```javascript
count
```

所以 JavaScript 的垃圾回收机制会判断：

> 这个变量是否还有对象引用？

如果闭包仍然引用它，就不能回收。

因此：

```text
outer()
  ↓
创建 count
  ↓
创建 inner
  ↓
inner 引用 count
  ↓
返回 inner
  ↓
fn 引用 inner
  ↓
count 仍然可达
```

所以 `count` 依然存在。

---

# 七、闭包最典型的用途：保存状态

闭包非常适合保存一些需要长期存在的状态。

例如计数器：

```javascript
function createCounter() {
  let count = 0

  return {
    increment() {
      count++
    },

    decrement() {
      count--
    },

    getCount() {
      return count
    }
  }
}

const counter = createCounter()

counter.increment()
counter.increment()

console.log(counter.getCount()) // 2
```

这里：

```javascript
count
```

没有暴露出去。

外部无法直接：

```javascript
counter.count = 100
```

修改真正的 `count`。

只有通过：

```javascript
increment()
decrement()
getCount()
```

操作。

因此闭包可以帮助实现一种简单的：

> **数据私有化。**

---

# 八、闭包实现数据私有化

例如：

```javascript
function createUser() {
  let username = 'Tom'
  let password = '123456'

  return {
    getUsername() {
      return username
    },

    changePassword(newPassword) {
      password = newPassword
    }
  }
}

const user = createUser()

console.log(user.getUsername())
```

外部只能通过提供的方法访问数据。

```text
                createUser
                    │
        ┌───────────┴───────────┐
        ↓                       ↓
    username                password
        │                       │
        └───────────┬───────────┘
                    ↓
                 闭包
                    ↓
          getUsername / changePassword
```

这和直接暴露：

```javascript
const user = {
  username: 'Tom',
  password: '123456'
}
```

相比，多了一层控制。

---

# 九、闭包与工厂函数

闭包经常和工厂函数一起使用。

例如：

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

person1.sayHello()
person2.sayHello()
```

输出：

```text
Hello, Tom
Hello, Jack
```

两个对象拥有不同的状态。

```text
person1
  ↓
sayHello
  ↓
name = Tom


person2
  ↓
sayHello
  ↓
name = Jack
```

这也是闭包非常典型的应用方式。

---

# 十、闭包与定时器

闭包在异步代码中也非常常见。

例如：

```javascript
function createTimer(name) {
  setTimeout(() => {
    console.log(name)
  }, 1000)
}

createTimer('任务完成')
```

这里：

```javascript
setTimeout(() => {
  console.log(name)
}, 1000)
```

中的回调函数使用了外部的：

```javascript
name
```

因此回调函数形成了对外部变量的引用。

即使：

```javascript
createTimer()
```

已经执行结束：

```javascript
name
```

依然需要被回调函数访问。

所以它会继续存在到不再需要为止。

---

# 十一、经典问题：循环 + 闭包

闭包最经典的坑之一，就是循环中的变量。

例如：

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i)
  }, 0)
}
```

结果：

```text
3
3
3
```

为什么？

因为 `var` 不会为每一次循环创建一个新的块级作用域。

这些回调函数访问的是同一个：

```javascript
i
```

循环结束以后：

```text
i = 3
```

所以定时器执行时，看到的都是：

```text
3
```

可以理解为：

```text
      i
      │
 ┌────┼────┐
 ↓    ↓    ↓
回调1 回调2 回调3
```

三个回调共享同一个 `i`。

---

# 十二、使用 let 为什么可以解决？

把：

```javascript
var
```

换成：

```javascript
let
```

```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i)
  }, 0)
}
```

结果：

```text
0
1
2
```

因为 `let` 具有块级作用域。

每一次循环都可以理解为拥有自己的变量绑定：

```text
第 1 次循环
i = 0
  ↓
回调1


第 2 次循环
i = 1
  ↓
回调2


第 3 次循环
i = 2
  ↓
回调3
```

因此每个回调访问的是对应的 `i`。

---

# 十三、也可以通过函数创建独立作用域

在 `let` 出现之前，可以利用闭包解决这个问题：

```javascript
for (var i = 0; i < 3; i++) {
  (function (index) {
    setTimeout(() => {
      console.log(index)
    }, 0)
  })(i)
}
```

输出：

```text
0
1
2
```

这里通过立即执行函数：

```javascript
(function (index) {

})(i)
```

每次创建一个新的函数作用域。

因此：

```text
第一次
index = 0


第二次
index = 1


第三次
index = 2
```

每个回调都闭包保存了自己的 `index`。

不过现代 JavaScript 中通常直接使用 `let`，代码更加简单。

---

# 十四、闭包不等于“变量永远不会被回收”

这是理解闭包时一个很重要的误区。

闭包确实可能延长变量的生命周期，但并不是：

> 只要形成闭包，变量就永远存在。

垃圾回收机制关注的是：

> **对象是否仍然可达。**

例如：

```javascript
function outer() {
  const data = new Array(1000000)

  return function () {
    console.log(data.length)
  }
}

let fn = outer()
```

此时：

```javascript
fn
```

仍然引用内部函数。

内部函数又引用：

```javascript
data
```

所以：

```text
fn
 ↓
内部函数
 ↓
data
```

`data` 仍然可达。

如果后来：

```javascript
fn = null
```

那么这条引用链可能被断开。

如果没有其他引用：

```text
data
```

就有机会被垃圾回收。

所以：

> **闭包不是内存泄漏，错误地长期持有闭包引用才可能导致内存无法及时释放。**

---

# 十五、闭包可能带来的内存问题

例如：

```javascript
let fn

function create() {
  const bigData = new Array(1000000).fill('data')

  fn = function () {
    console.log(bigData.length)
  }
}

create()
```

此时：

```text
fn
 ↓
匿名函数
 ↓
bigData
```

只要：

```javascript
fn
```

一直存在：

```javascript
bigData
```

就可能一直保持可达状态。

如果这个闭包被长期保存，尤其是：

* 全局变量
* 定时器
* 事件监听器
* 缓存
* 长生命周期对象

就需要注意引用是否应该继续存在。

例如事件监听：

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
```

如果组件销毁时仍然保留监听器，就可能让相关对象继续保持引用。

因此在合适的时候应该解除监听：

```javascript
button.removeEventListener('click', handler)
```

---

# 十六、闭包的常见应用场景

闭包并不是为了“炫技”，实际开发中非常常见。

## 1. 保存状态

```javascript
function createCounter() {
  let count = 0

  return () => ++count
}
```

---

## 2. 数据私有化

```javascript
function createUser() {
  let password = '123456'

  return {
    getPassword() {
      return password
    }
  }
}
```

---

## 3. 工厂函数

```javascript
function createPerson(name) {
  return {
    sayHello() {
      console.log(name)
    }
  }
}
```

---

## 4. 防抖

防抖函数本质上也大量使用了闭包保存定时器：

```javascript
function debounce(fn, delay) {
  let timer = null

  return function (...args) {
    clearTimeout(timer)

    timer = setTimeout(() => {
      fn.apply(this, args)
    }, delay)
  }
}
```

这里：

```javascript
timer
```

被返回的函数持续引用。

因此每次调用时，都可以访问之前的：

```javascript
timer
```

这就是闭包。

---

## 5. 节流

节流同样可以利用闭包保存状态：

```javascript
function throttle(fn, delay) {
  let lastTime = 0

  return function (...args) {
    const now = Date.now()

    if (now - lastTime >= delay) {
      lastTime = now
      fn.apply(this, args)
    }
  }
}
```

这里：

```javascript
lastTime
```

就是通过闭包保存的状态。

---

# 十七、闭包与模块化

闭包还可以实现简单的模块模式。

```javascript
const counter = (() => {
  let count = 0

  return {
    increment() {
      count++
    },

    getCount() {
      return count
    }
  }
})()
```

使用：

```javascript
counter.increment()
counter.increment()

console.log(counter.getCount())
```

但是外部无法直接访问：

```javascript
count
```

例如：

```javascript
console.log(counter.count)
```

得到：

```text
undefined
```

因为 `count` 并不是对象公开的属性。

---

# 十八、闭包和普通函数有什么区别？

并不是所有函数都需要特别强调“闭包”。

例如：

```javascript
function add(a, b) {
  return a + b
}
```

它当然也存在函数自己的作用域。

但我们通常不会特意把它称为闭包。

闭包真正值得关注的是：

> **函数离开了原来的作用域之后，仍然依赖并访问那个外部作用域中的变量。**

例如：

```javascript
function outer() {
  const name = 'Tom'

  return function inner() {
    console.log(name)
  }
}

const fn = outer()

fn()
```

这里的 `inner` 就具有典型的闭包特征。

---

# 十九、用一张图理解闭包

整个过程可以简化成：

```text
function outer() {
    const count = 0

    return function inner() {
        console.log(count)
    }
}
        │
        │ 执行
        ↓
┌─────────────────────┐
│ outer 作用域         │
│                     │
│ count = 0           │
│                     │
│ inner ──────────────┼────┐
└─────────────────────┘    │
                           │
                           ↓
                     ┌───────────┐
                     │ inner函数 │
                     └───────────┘
                           │
                           │ 引用
                           ↓
                         count
```

最终：

```javascript
const fn = outer()
```

得到：

```text
fn
 ↓
inner
 ↓
outer 作用域
 ↓
count
```

这就是闭包关系。

---

# 二十、如何真正理解闭包？

可以从三个层次去理解。

## 第一层：现象

函数可以访问定义位置外部的变量：

```javascript
function outer() {
  let count = 0

  return () => ++count
}
```

---

## 第二层：原因

JavaScript 使用词法作用域。

函数在定义的时候，就确定了自己能够访问哪些外部作用域。

因此内部函数即使之后被返回，也依然可以访问外部变量。

---

## 第三层：结果

当外部作用域中的变量仍然被内部函数引用时，这些变量的生命周期可能被延长。

因此闭包可以：

* 保存状态
* 实现数据私有化
* 创建独立作用域
* 实现防抖、节流等工具函数
* 构建工厂函数和模块模式

同时也需要注意：

* 不要无意义地长期保存闭包
* 注意事件监听器的解绑
* 注意定时器的清理
* 注意大型对象是否被闭包长期引用

---

# 二十一、总结

闭包可以浓缩成一句话：

> **闭包就是函数与其词法作用域之间形成的持续引用关系，使函数即使离开原来的执行环境，也仍然能够访问其中的变量。**

理解闭包时，不需要死记：

```text
函数 + 外部变量 = 闭包
```

更重要的是理解整个过程：

```text
词法作用域
    ↓
作用域链
    ↓
函数可以访问外部变量
    ↓
内部函数被返回 / 保存
    ↓
仍然引用外部作用域
    ↓
形成闭包
    ↓
外部变量生命周期可能延长
```

而闭包真正有价值的地方，就是：

```text
保存状态
    +
控制数据访问
    +
延长变量生命周期
```

因此，闭包并不是 JavaScript 中一个“特殊的语法”，而是**作用域、函数以及引用关系共同产生的一种语言机制**。

当能够从这个角度理解闭包之后，很多看起来独立的问题——例如防抖、节流、循环中的异步回调、数据私有化、工厂函数——其实都可以归结到同一个概念上。

---

