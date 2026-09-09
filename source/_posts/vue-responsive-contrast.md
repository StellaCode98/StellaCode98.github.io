---

title: Vue2 与 Vue3 响应式原理对比：从 Object.defineProperty 到 Proxy
date: 2026-09-09 15:50:00
description: 从 Object.defineProperty 到 Proxy，理解 Vue2 与 Vue3 响应式系统的实现方式，以及 Vue3 响应式系统的设计变化。
categories:
 - [Vue, 源码]
tags:
 - Vue2
 - Vue3
 - 响应式
 - Proxy
 - Object.defineProperty

---

# Vue2 与 Vue3 响应式原理对比：从 Object.defineProperty 到 Proxy

Vue 最核心的能力之一，就是**响应式**。

在使用 Vue 开发应用时，我们通常只需要修改数据：

```javascript
state.count++
```

页面就会自动发生变化。

开发者并不需要手动查找 DOM，再修改对应的节点。

```text
修改数据
   ↓
响应式系统感知变化
   ↓
找到依赖该数据的逻辑
   ↓
重新执行相关逻辑
   ↓
视图更新
```

这背后就是 Vue 响应式系统所解决的问题。

Vue2 和 Vue3 都实现了这套机制，但底层实现存在比较明显的区别：

```text
Vue2
Object.defineProperty
        ↓
   getter / setter
        ↓
   Dep / Watcher

Vue3
      Proxy
        ↓
  track / trigger
        ↓
      effect
```

从 Vue2 到 Vue3，并不只是简单地把一个 API 换成另一个 API，而是响应式系统整体设计的一次变化。

本文从 `Object.defineProperty` 开始，逐步理解 Vue2 的实现方式，再来看 Vue3 为什么引入 `Proxy`，以及两套响应式系统之间的区别。

---

## 一、响应式系统解决了什么问题？

先抛开 Vue，只考虑一个最简单的问题。

```javascript
const state = {
  count: 0
}

state.count++

console.log(state.count)
```

JavaScript 本身当然可以感知：

```text
state.count 被修改了
```

但是 JavaScript 并不知道：

> 哪些代码依赖 `state.count`？

例如：

```javascript
function render() {
  console.log(state.count)
}
```

这里 `render()` 明显依赖 `state.count`。

如果我们希望：

```javascript
state.count++
```

之后自动执行：

```javascript
render()
```

就需要建立一种关系：

```text
state.count
    ↓
    render
```

也就是：

> **数据和使用数据的副作用之间建立依赖关系。**

因此，一个响应式系统最核心的两个过程就是：

```text
读取数据
   ↓
收集依赖

修改数据
   ↓
触发依赖
```

Vue3 中把这两个过程抽象成了两个非常重要的概念：

```javascript
track()
trigger()
```

可以简单理解为：

```text
track   → 收集依赖
trigger → 触发更新
```

---

# 二、Vue2：Object.defineProperty

Vue2 的响应式系统主要建立在：

```javascript
Object.defineProperty()
```

之上。

这是 JavaScript 提供的一个用于修改对象属性描述符的 API。

例如：

```javascript
const user = {
  name: 'Tom'
}

Object.defineProperty(user, 'name', {
  get() {
    console.log('读取 name')

    return 'Tom'
  },

  set(value) {
    console.log('修改 name：', value)
  }
})
```

当执行：

```javascript
user.name
```

会进入：

```javascript
get()
```

而：

```javascript
user.name = 'Jerry'
```

则会进入：

```javascript
set()
```

Vue2 正是利用这个特性，把普通对象属性转换成响应式属性。

---

# 三、Vue2 如何将对象转换成响应式？

可以把 Vue2 的过程简化成：

```javascript
function defineReactive(obj, key, value) {
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,

    get() {
      return value
    },

    set(newValue) {
      value = newValue
    }
  })
}
```

例如：

```javascript
const state = {
  count: 0
}

defineReactive(state, 'count', state.count)
```

此时：

```javascript
state.count
```

会进入 getter。

而：

```javascript
state.count = 10
```

会进入 setter。

从这里开始，Vue 就获得了一个“监听数据变化”的入口。

---

# 四、getter 和 setter 为什么能够实现响应式？

仅仅知道数据被读取和修改还不够。

Vue 还需要知道：

> **谁读取了这个数据？**

例如组件：

```javascript
function render() {
  return `<div>${state.count}</div>`
}
```

第一次执行 `render()` 时：

```text
render()
  ↓
读取 state.count
  ↓
触发 getter
  ↓
记录 render
```

于是就可以建立：

```text
state.count
    ↓
  render
```

当数据发生变化：

```javascript
state.count = 10
```

执行：

```text
setter
  ↓
找到依赖
  ↓
重新执行 render()
```

最终：

```text
数据变化
   ↓
setter
   ↓
通知依赖
   ↓
重新执行相关逻辑
   ↓
视图更新
```

这就是 Vue2 响应式的基本思路。

---

# 五、Dep：依赖管理

Vue2 中有一个非常重要的概念：

```text
Dep
```

可以把它理解成：

> **一个属性对应的依赖管理器。**

例如：

```text
count
  ↓
Dep
  ↓
Watcher A
Watcher B
Watcher C
```

当 `count` 被读取时，Vue 会把当前正在执行的 Watcher 添加进去。

可以简化为：

```javascript
get() {
  dep.depend()

  return value
}
```

当 `count` 被修改时：

```javascript
set(newValue) {
  value = newValue

  dep.notify()
}
```

于是整个流程变成：

```text
                count
                  │
            Object.defineProperty
                  │
             ┌────┴────┐
             ↓         ↓
            get       set
             ↓         ↓
           depend    notify
             ↓         ↓
             Dep      Dep
             │         │
             └────┬────┘
                  ↓
               Watcher
                  ↓
                update
```

---

# 六、Watcher：具体的依赖

如果说：

```text
Dep
```

负责保存依赖，那么：

```text
Watcher
```

就可以理解成一个具体的依赖。

例如组件渲染：

```javascript
function render() {
  return state.count
}
```

可以把这个渲染逻辑看成一个 Watcher。

第一次执行：

```text
Watcher
   ↓
render()
   ↓
读取 count
   ↓
count.getter
   ↓
Dep 收集 Watcher
```

之后：

```javascript
state.count++
```

就会：

```text
setter
  ↓
Dep.notify()
  ↓
Watcher.update()
  ↓
重新执行 render
```

---

# 七、Vue2 响应式的完整流程

把前面的内容串起来：

```text
                数据对象
                   │
                   ↓
        Object.defineProperty
                   │
             getter / setter
               ↙         ↘
             get          set
              ↓            ↓
         依赖收集       派发更新
              ↓            ↓
             Dep          Dep
              ↓            ↓
           Watcher ←───────┘
              ↓
          重新执行
              ↓
          视图更新
```

因此 Vue2 的核心链路可以概括成：

```text
读取
 ↓
getter
 ↓
Dep.depend()
 ↓
收集 Watcher

修改
 ↓
setter
 ↓
Dep.notify()
 ↓
Watcher 更新
```

---

# 八、Vue2 为什么需要递归处理对象？

考虑一个嵌套对象：

```javascript
const state = {
  user: {
    name: 'Tom',
    age: 18
  }
}
```

如果只对第一层：

```javascript
state.user
```

进行响应式处理，那么：

```javascript
state.user.name
```

仍然无法完整地被监听。

因此 Vue2 会在初始化阶段递归处理对象：

```text
state
 ├── user
 │    ├── name
 │    └── age
 └── ...
```

每一个需要响应式处理的属性都需要建立对应的 getter / setter。

这也是 Vue2 初始化阶段需要遍历数据的重要原因之一。

---

# 九、Vue2 的局限：新增属性

`Object.defineProperty` 最大的设计限制之一，就是：

> **它需要针对具体属性进行定义。**

例如：

```javascript
const user = {
  name: 'Tom'
}
```

Vue 初始化时可能已经处理了：

```text
name
 ↓
getter / setter
```

但是后来：

```javascript
user.age = 18
```

`age` 是后来新增的属性。

Vue2 并没有提前对：

```javascript
age
```

执行：

```javascript
Object.defineProperty()
```

因此无法自动建立对应的响应式关系。

也就是说：

```text
初始化

user.name
    ↓
defineProperty
    ↓
响应式


运行过程中

user.age = 18
    ↓
新增属性
    ↓
没有对应 getter / setter
```

因此 Vue2 提供了：

```javascript
Vue.set()
```

以及：

```javascript
this.$set()
```

用于处理这种情况。

---

# 十、Vue2 的删除属性问题

类似地：

```javascript
delete user.name
```

也不会自然触发之前定义好的 setter。

因为 setter 只负责：

```text
属性被重新赋值
```

而：

```javascript
delete
```

是另外一种对象操作。

所以 Vue2 同样提供：

```javascript
Vue.delete()
```

来处理响应式对象中的属性删除。

---

# 十一、Vue2 对数组的特殊处理

数组是 Vue2 响应式系统中比较特殊的一部分。

例如：

```javascript
const list = [1, 2, 3]
```

如果执行：

```javascript
list.push(4)
```

Vue2 需要知道：

> 数组发生变化了。

因此 Vue2 对一些数组变异方法进行了特殊处理：

```text
push
pop
shift
unshift
splice
sort
reverse
```

例如：

```javascript
list.push(4)
```

Vue2 会通过重写后的数组方法，在执行原始操作后通知依赖。

可以理解为：

```text
list.push()
   ↓
Vue 重写的方法
   ↓
原始 push
   ↓
数组发生变化
   ↓
通知依赖
```

这也是为什么 Vue2 的数组响应式实现相比普通对象更加复杂。

---

# 十二、Vue2 响应式的整体限制

因此，Vue2 的响应式系统可以总结为：

```text
Object.defineProperty
        ↓
针对属性进行劫持
        ↓
需要初始化时遍历对象
        ↓
需要递归处理嵌套对象
        ↓
新增 / 删除属性存在限制
        ↓
数组需要特殊处理
```

这些限制并不是 Vue2 本身设计得不好，而是：

> **Object.defineProperty 本身的能力决定了这种实现方式存在边界。**

随着 Vue 的使用场景越来越复杂，这些问题也逐渐成为响应式系统进一步发展的限制。

---

# 十三、Vue3：Proxy

Vue3 的响应式系统换成了：

```javascript
Proxy
```

Proxy 和 `Object.defineProperty` 最大的区别是：

```text
Object.defineProperty
        ↓
代理具体属性

Proxy
        ↓
代理整个对象
```

例如：

```javascript
const state = {
  count: 0
}

const proxy = new Proxy(state, {
  get(target, key) {
    console.log('读取：', key)

    return target[key]
  },

  set(target, key, value) {
    console.log('修改：', key, value)

    target[key] = value

    return true
  }
})
```

现在访问：

```javascript
proxy.count
```

会进入：

```javascript
get()
```

修改：

```javascript
proxy.count = 10
```

会进入：

```javascript
set()
```

---

# 十四、Proxy 为什么能够解决新增属性问题？

关键就在于：

> **Proxy 代理的是整个对象，而不是某一个已经存在的属性。**

例如：

```javascript
const state = {}

const proxy = new Proxy(state, {
  set(target, key, value) {
    target[key] = value

    return true
  }
})
```

此时：

```javascript
proxy.name = 'Tom'
```

即使：

```text
name
```

之前根本不存在，也依然会进入：

```javascript
set()
```

因为 Proxy 代理的是整个对象。

所以：

```text
proxy.name = 'Tom'
        ↓
      Proxy
        ↓
       set
        ↓
    设置属性
        ↓
    触发响应式
```

不再需要额外的：

```javascript
Vue.set()
```

---

# 十五、Proxy 可以拦截更多操作

Proxy 不仅能够拦截：

```text
get
set
```

还可以处理很多其他操作：

```text
get
set
has
deleteProperty
ownKeys
defineProperty
getOwnPropertyDescriptor
...
```

例如：

### 读取属性

```javascript
proxy.name
```

触发：

```text
get
```

### 设置属性

```javascript
proxy.name = 'Jerry'
```

触发：

```text
set
```

### 删除属性

```javascript
delete proxy.name
```

触发：

```text
deleteProperty
```

### 判断属性是否存在

```javascript
'name' in proxy
```

触发：

```text
has
```

因此 Proxy 为 Vue3 建立更加完整的响应式系统提供了更好的基础。

---

# 十六、Vue3 的 reactive

Vue3 中：

```javascript
reactive()
```

就是创建响应式对象的重要 API。

例如：

```javascript
const state = reactive({
  count: 0
})
```

从概念上可以理解成：

```text
普通对象
   ↓
reactive()
   ↓
Proxy
   ↓
响应式对象
```

当：

```javascript
state.count
```

被读取时：

```text
Proxy.get
   ↓
track()
```

当：

```javascript
state.count++
```

被修改时：

```text
Proxy.set
   ↓
trigger()
```

---

# 十七、track：依赖收集

Vue3 将依赖收集抽象成：

```javascript
track()
```

例如：

```javascript
effect(() => {
  console.log(state.count)
})
```

执行 `effect` 时：

```text
effect()
  ↓
读取 state.count
  ↓
Proxy.get
  ↓
track()
  ↓
记录当前 effect
```

此时 Vue 就知道：

```text
state.count
     ↓
当前 effect
```

建立了依赖关系。

---

# 十八、trigger：触发更新

当：

```javascript
state.count++
```

发生时：

```text
Proxy.set
   ↓
trigger()
   ↓
找到 count 对应的依赖
   ↓
重新执行 effect
```

于是：

```text
读取：

state.count
    ↓
track
    ↓
收集 effect


修改：

state.count++
    ↓
trigger
    ↓
执行 effect
```

这就是 Vue3 响应式系统最核心的两个过程。

---

# 十九、effect：副作用函数

Vue3 中可以使用：

```javascript
effect()
```

来描述一个会受到响应式数据影响的副作用。

例如：

```javascript
const state = reactive({
  count: 0
})

effect(() => {
  console.log(state.count)
})
```

第一次执行：

```text
effect
 ↓
读取 count
 ↓
track
 ↓
收集依赖
```

之后：

```javascript
state.count++
```

就会：

```text
set
 ↓
trigger
 ↓
找到 effect
 ↓
重新执行
```

可以把 `effect` 简单理解成：

> **一个会根据响应式数据变化而重新执行的函数。**

Vue 组件渲染、计算属性以及部分其他响应式能力，都建立在类似的副作用机制之上。

---

# 二十、Vue3 如何保存依赖关系？

随着响应式数据变多，Vue 需要解决一个问题：

> 一个对象的多个属性，分别被哪些 effect 使用？

例如：

```javascript
const state = reactive({
  name: 'Tom',
  age: 18
})
```

可能存在：

```text
state.name → effectA
state.age  → effectB
```

因此 Vue3 使用了一种非常典型的依赖结构：

```text
WeakMap
   ↓
 target
   ↓
 Map
   ↓
 key
   ↓
 Set
   ↓
 effect
```

可以表示成：

```text
WeakMap
│
└── state
     │
     └── Map
          │
          ├── name → Set(effectA)
          │
          └── age  → Set(effectB)
```

---

# 二十一、为什么是 WeakMap → Map → Set？

三个数据结构分别解决不同的问题。

## WeakMap

保存：

```text
响应式对象
 ↓
依赖信息
```

例如：

```text
state → depsMap
```

WeakMap 的 key 必须是对象，非常适合用来关联响应式对象。

---

## Map

一个对象可能拥有多个属性：

```javascript
state.name
state.age
state.count
```

所以需要进一步记录：

```text
name
age
count
```

对应的依赖。

于是：

```text
Map
 ↓
属性 key
 ↓
依赖集合
```

---

## Set

一个属性可能同时被多个 effect 使用：

```text
state.count
   ↓
effectA
effectB
effectC
```

Set 非常适合保存这些 effect，并且可以避免重复依赖。

因此最终形成：

```text
WeakMap
  ↓
对象
  ↓
Map
  ↓
属性
  ↓
Set
  ↓
effect
```

---

# 二十二、Vue3 响应式完整流程

把这些概念组合起来：

```text
                    reactive()
                        ↓
                      Proxy
                        ↓
              ┌─────────┴─────────┐
              ↓                   ↓
             get                 set
              ↓                   ↓
           track()             trigger()
              ↓                   ↓
          依赖收集             查找依赖
              ↓                   ↓
       WeakMap → Map → Set       effect
                                  ↓
                              重新执行
```

完整过程就是：

```text
① 创建响应式对象

reactive()
    ↓
Proxy


② 读取数据

state.count
    ↓
Proxy.get
    ↓
track()
    ↓
记录 effect


③ 修改数据

state.count++
    ↓
Proxy.set
    ↓
trigger()
    ↓
找到对应 effect
    ↓
重新执行
```

---

# 二十三、Vue2 与 Vue3 的实现对比

将两套响应式系统放在一起看：

|       | Vue2                    | Vue3                     |
| ----- | ----------------------- | ------------------------ |
| 核心机制  | `Object.defineProperty` | `Proxy`                  |
| 代理范围  | 单个属性                    | 整个对象                     |
| 读取拦截  | getter                  | Proxy `get`              |
| 修改拦截  | setter                  | Proxy `set`              |
| 删除属性  | 需要额外处理                  | 可以拦截                     |
| 新增属性  | 需要额外处理                  | 可以拦截                     |
| 数组    | 重写部分数组方法                | Proxy 进行代理               |
| 依赖管理  | Dep / Watcher           | track / trigger / effect |
| 依赖结构  | 以 Dep 为核心               | WeakMap → Map → Set      |
| 响应式创建 | 初始化时遍历                  | 访问时通过 Proxy 代理           |

这里最值得注意的并不是 API 名称变化，而是**响应式系统的观察粒度发生了变化**：

```text
Vue2
属性
 ↓
getter / setter

Vue3
对象
 ↓
Proxy
```

---

# 二十四、为什么 Vue3 不再需要 Vue.set？

Vue2：

```javascript
this.$set(user, 'age', 18)
```

是因为：

```text
Object.defineProperty
        ↓
只能处理已经定义的属性
```

而 Vue3：

```javascript
user.age = 18
```

本身就可以被：

```text
Proxy.set
```

拦截。

所以：

```text
Vue2

新增属性
 ↓
需要额外 API
 ↓
Vue.set


Vue3

新增属性
 ↓
Proxy.set
 ↓
响应式系统处理
```

因此 Vue3 移除了 `Vue.set` 和 `Vue.delete` 这类 API。

---

# 二十五、Proxy 并不等于响应式

这里有一个容易混淆的地方。

很多时候会直接说：

> Vue3 使用 Proxy 实现响应式。

这句话从整体上来说没有问题，但从原理上看并不完整。

Proxy 本身只是 JavaScript 提供的**代理机制**。

例如：

```javascript
const proxy = new Proxy(target, {
  get(target, key) {
    return target[key]
  }
})
```

这并不会自动产生响应式。

Vue3 真正的响应式系统是多个机制组合起来的：

```text
Proxy
  ↓
拦截对象操作
  ↓
track / trigger
  ↓
依赖管理
  ↓
effect
  ↓
重新执行副作用
```

所以更准确地说：

> **Vue3 使用 Proxy 作为响应式系统的拦截基础，再通过 track、trigger 和 effect 建立完整的依赖追踪机制。**

---

# 二十六、从 defineProperty 到 Proxy

如果把 Vue2 到 Vue3 的变化抽象一下，可以看到一个非常清晰的演进过程。

### Vue2

```text
对象
 ↓
遍历属性
 ↓
defineProperty
 ↓
getter / setter
 ↓
Dep
 ↓
Watcher
```

这种方式的问题是：

> 需要提前知道有哪些属性。

---

### Vue3

```text
对象
 ↓
Proxy
 ↓
运行时拦截操作
 ↓
track / trigger
 ↓
effect
```

Proxy 让响应式系统从：

> **提前对属性进行处理**

变成了：

> **运行时对对象操作进行拦截。**

这也是 Vue3 响应式设计变化中非常重要的一点。

---

# 二十七、两套系统的核心思想其实没有改变

虽然 Vue2 和 Vue3 使用了不同的技术实现，但响应式系统最核心的思想其实一直没有变化：

```text
                 数据
                  │
              ┌───┴───┐
              ↓       ↓
             读取     修改
              ↓       ↓
          收集依赖   触发更新
              ↓       ↓
             依赖 ←───┘
```

变化的是实现方式：

```text
Vue2

Object.defineProperty
        ↓
      getter
        ↓
       Dep
        ↓
     Watcher
        ↓
      update
```

Vue3：

```text
Proxy
  ↓
get
  ↓
track
  ↓
effect

Proxy
  ↓
set
  ↓
trigger
  ↓
effect
```

因此，理解 Vue 响应式时，与其记住大量 API，不如先抓住这条主线：

```text
读取 → 收集依赖
修改 → 触发依赖
```

---

# 二十八、总结

Vue2 到 Vue3 的响应式变化，可以浓缩成一句话：

> **Vue2 基于 `Object.defineProperty` 对对象属性进行劫持，Vue3 则通过 `Proxy` 对整个对象进行代理，并重新设计了依赖追踪机制。**

Vue2 的核心结构：

```text
Object.defineProperty
        ↓
getter / setter
        ↓
Dep
        ↓
Watcher
```

Vue3 的核心结构：

```text
Proxy
 ↓
get / set
 ↓
track / trigger
 ↓
effect
 ↓
WeakMap → Map → Set
```

而两者最终解决的都是同一个问题：

```text
数据发生变化
      ↓
找到依赖该数据的逻辑
      ↓
重新执行
      ↓
完成视图更新
```

从这个角度来看，Vue3 的响应式升级并不是简单的 API 替换，而是从**属性级别的数据劫持**逐渐演进到了**对象级别的运行时代理和依赖追踪**。

这也为 Vue3 后续的 `ref`、`reactive`、`computed`、`watch` 以及 Composition API 等能力提供了更加统一的响应式基础。

---
