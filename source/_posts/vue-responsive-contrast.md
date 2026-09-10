---

title: Vue2 与 Vue3 响应式原理对比：从 Object.defineProperty 到 Proxy
date: 2026-09-09 15:50:00
description: 从 Object.defineProperty 到 Proxy，理解 Vue2 与 Vue3 响应式系统的实现方式，以及 Vue3 响应式系统的设计变化。
categories:
 - [Vue, 源码]
tags:
 - Vue2
 - Vue3
 - Proxy

---

# Vue2 与 Vue3 响应式原理对比：从 Object.defineProperty 到 Proxy

Vue 最核心的能力之一，就是**响应式**：只需要修改数据 `state.count++`，页面就会自动更新，不需要手动查找和修改 DOM。

```text
修改数据 → 响应式系统感知变化 → 找到依赖该数据的逻辑 → 重新执行 → 视图更新
```

Vue2 和 Vue3 都实现了这套机制，但底层实现存在明显区别：

```text
Vue2：Object.defineProperty → getter / setter → Dep / Watcher
Vue3：Proxy → track / trigger → effect
```

从 Vue2 到 Vue3，并不只是把一个 API 换成另一个 API，而是响应式系统整体设计的一次变化。本文从 `Object.defineProperty` 开始，逐步理解 Vue2 的实现方式，再看 Vue3 为什么引入 `Proxy`，以及两套系统的区别。

---

## 一、响应式系统解决了什么问题

先抛开 Vue，只考虑一个最简单的问题：

```javascript
const state = { count: 0 }

function render() {
  console.log(state.count)
}
```

`render()` 明显依赖 `state.count`，但 JavaScript 本身并不知道这件事——它只能感知"属性被修改了"，却不知道**哪些代码依赖这个属性**。如果我们希望 `state.count++` 之后自动执行 `render()`，就需要建立一种关系：

> **数据和使用数据的副作用之间建立依赖关系。**

因此，任何响应式系统最核心的都是两个过程：

```text
读取数据 → 收集依赖（Vue3 中抽象为 track）
修改数据 → 触发依赖（Vue3 中抽象为 trigger）
```

---

## 二、Vue2：用 Object.defineProperty 劫持属性

`Object.defineProperty()` 是 JavaScript 提供的修改对象属性描述符的 API。Vue2 正是利用它，把普通对象属性转换成响应式属性。核心逻辑可以简化成：

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

const state = { count: 0 }
defineReactive(state, 'count', state.count)
// 此后 state.count 走 getter，state.count = 10 走 setter
```

仅仅知道数据被读取和修改还不够，Vue 还需要知道**谁读取了这个数据**。思路是：第一次执行 `render()` 时会读取 `state.count`，从而触发 getter——就在这个时机把当前正在执行的副作用记录下来：

```text
读取：render() → 读 state.count → getter → 记录 render
修改：state.count = 10 → setter → 找到依赖 → 重新执行 render()
```

这就是 Vue2 响应式的基本思路：getter / setter 提供了"监听数据变化"的入口，依赖关系在读取时建立、在修改时使用。

---

## 三、Dep 与 Watcher：依赖的保存与执行

上面思路落地为两个角色：

- **Dep**：一个属性对应的依赖管理器，负责保存依赖。属性被读取时调用 `dep.depend()` 收集当前正在执行的 Watcher；被修改时调用 `dep.notify()` 通知所有 Watcher。
- **Watcher**：一个具体的依赖。组件的渲染函数、用户的 watch 回调，都可以理解成一个 Watcher——它第一次执行时读取数据、被收集进 Dep，数据变化时被通知重新执行。

```javascript
get() {
  dep.depend()      // 收集当前 Watcher
  return value
}

set(newValue) {
  value = newValue
  dep.notify()      // 通知所有 Watcher 更新
}
```

整个 Vue2 的核心链路：

```text
读取 → getter → Dep.depend() → 收集 Watcher
修改 → setter → Dep.notify() → Watcher.update() → 重新执行 → 视图更新
```

---

## 四、Vue2 的局限

`Object.defineProperty` 的能力边界，带来了 Vue2 响应式的几个经典限制。

### 1. 初始化时需要递归遍历

`defineProperty` 针对具体属性进行劫持，而嵌套对象的深层属性（如 `state.user.name`）只有被逐层处理后才能被监听。因此 Vue2 必须在初始化阶段递归遍历整个数据对象，给每个属性都建立 getter / setter——这也是 Vue2 初始化开销较大的重要原因。

### 2. 新增 / 删除属性无法监听

初始化时只处理了已存在的属性。后来新增的 `user.age = 18` 没有对应的 getter / setter，不是响应式的；`delete user.name` 也不会触发 setter（setter 只负责"属性被重新赋值"这一种操作）。

所以 Vue2 不得不提供 `Vue.set()` / `this.$set()` 和 `Vue.delete()` 来手动处理这两种情况。

### 3. 数组需要特殊处理

通过索引修改数组元素、调用 `push` 等变异方法，同样绕过了属性劫持。Vue2 的做法是**重写七个数组变异方法**（push / pop / shift / unshift / splice / sort / reverse）：在执行原始操作后手动通知依赖。这也是 Vue2 的数组响应式比普通对象复杂得多的原因。

### 小结

这些限制并不是 Vue2 设计得不好，而是：

> **Object.defineProperty 本身的能力决定了这种实现方式存在边界。**

---

## 五、Vue3：用 Proxy 代理整个对象

Vue3 把响应式基础换成了 `Proxy`。两者最关键的区别：

```text
Object.defineProperty → 代理具体属性
Proxy                 → 代理整个对象
```

```javascript
const state = { count: 0 }

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

**因为 Proxy 代理的是整个对象而不是某个已存在的属性，之前的所有限制都自然解决了：**

```text
proxy.name = 'Tom'      → set             新增属性，天然被拦截
delete proxy.name       → deleteProperty  删除属性，天然被拦截
'name' in proxy         → has             in 操作，也能拦截
```

Proxy 还能拦截 `ownKeys`、`getOwnPropertyDescriptor` 等更多操作，为建立完整的响应式系统提供了更好的基础。Vue3 由此移除了 `Vue.set` / `Vue.delete`——新增和删除属性本身就走 `Proxy.set` / `deleteProperty`，无需额外 API。

---

## 六、track / trigger / effect 与依赖结构

### track 与 trigger

Vue3 中 `reactive()` 创建响应式对象，本质上就是返回一个 Proxy：

```javascript
const state = reactive({ count: 0 })

effect(() => {
  console.log(state.count)   // 一个副作用函数
})
```

两个核心过程对应到 Proxy 的 trap 上：

```text
读取 state.count   → Proxy.get  → track()  → 记录当前 effect
修改 state.count++ → Proxy.set  → trigger() → 找到对应 effect → 重新执行
```

`effect` 可以简单理解成**一个会根据响应式数据变化而重新执行的函数**。Vue 组件渲染、计算属性、watch 等能力，都建立在类似的副作用机制之上。

### 依赖保存在哪：WeakMap → Map → Set

一个对象的多个属性，可能分别被不同的 effect 使用（`state.name → effectA`、`state.age → effectB`），Vue3 用三层结构保存这种关系：

```text
WeakMap
│
└── target（响应式对象）
     │
     └── Map
          ├── name → Set(effectA)
          └── age  → Set(effectB)
```

三个数据结构各司其职：

- **WeakMap**：key 必须是对象，天然适合关联"响应式对象 → 依赖信息"；且对象被销毁后依赖自动可被回收。
- **Map**：一个对象有多个属性，需要按属性 key 分别记录依赖。
- **Set**：一个属性可能被多个 effect 使用，Set 保存它们且天然去重。

### 一个容易混淆的点：Proxy 并不等于响应式

"Vue3 使用 Proxy 实现响应式"这句话从原理上看并不完整——Proxy 只是 JavaScript 提供的代理机制，单独一个 Proxy 不会产生任何响应式。完整的系统是多个机制的组合：

> **Vue3 使用 Proxy 作为拦截基础，再通过 track、trigger 和 effect 建立完整的依赖追踪机制。**

---

## 七、Vue2 与 Vue3 的实现对比

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

这里最值得注意的并不是 API 名称变化，而是**观察粒度**的变化：

```text
Vue2：属性级别 —— 提前遍历，对每个属性做劫持（需要提前知道有哪些属性）
Vue3：对象级别 —— 运行时代理，访问到什么才处理什么
```

从"提前对属性进行处理"变成"运行时对对象操作进行拦截"，这正是 Vue3 响应式设计的核心演进。

---

## 八、总结

两套系统的核心结构：

```text
Vue2：Object.defineProperty → getter / setter → Dep → Watcher → update

Vue3：Proxy → get / set → track / trigger → effect
                              └─ 依赖存于 WeakMap → Map → Set
```

但两套系统最核心的思想其实从未变化：

```text
读取 → 收集依赖
修改 → 触发依赖
```

Vue3 的响应式升级并不是简单的 API 替换，而是从**属性级别的数据劫持**演进到**对象级别的运行时代理和依赖追踪**。这个更统一、更完整的响应式基础，也为 Vue3 的 `ref`、`reactive`、`computed`、`watch` 以及 Composition API 提供了支撑。

---
