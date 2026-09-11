---
title: Vue3 中 ref 与 reactive 的区别
date: 2026-09-11 11:30:00
description: ref 是包装任意值的引用对象，reactive 是对象的 Proxy 代理。从实现原理、.value、基本类型支持、重新赋值、解构丢失等角度对比两者，并说明为什么官方推荐默认用 ref。
categories:
  - [Vue, 源码]
tags:
  - Vue
  - Vue3
  - 响应式
---

# Vue3 中 ref 与 reactive 的区别

Vue3 创建响应式数据有两个 API：`ref` 和 `reactive`。很多人只是记住了"基本类型用 ref、对象用 reactive"，但真到用时还是会踩坑：`.value` 忘了写、reactive 解构后不更新、整体替换后失去响应式……

这些坑的根源在于两者的**实现方式不同**。理解了原理，区别自然就清楚了。

---

## 一、实现原理不同

```javascript
const count = ref(0)        // RefImpl 实例：{ value: 0 }
const state = reactive({ count: 0 })  // Proxy 代理对象
```

- **ref**：返回一个 `RefImpl` 类实例，内部只有一个 `value` 属性，依赖收集和触发更新都挂在 `value` 的 getter / setter 上。访问 `count.value` 时 track，修改时 trigger。
- **reactive**：直接返回原对象的 Proxy 代理（Vue3 响应式系统的基础能力），拦截 get / set / deleteProperty 等所有操作，逐属性追踪。

一句话：**ref 是自己实现的"盒子"，reactive 是 Proxy 代理**。ref 的底层就是把 value 挂到类实例上做 getter/setter 拦截，没有用 Proxy（所以 `ref(undefined)` 也可以，不存在代理目标的问题）。

## 二、基本类型：只有 ref 能做

Proxy 只能代理对象，无法拦截 `let n = 0` 这种基本类型的读写。JavaScript 中基本类型按值传递，没有办法追踪它的修改。

所以：

```javascript
const count = ref(0)              // ✅ 基本类型只能用 ref
const state = reactive(0)         // ❌ 警告：value cannot be made reactive
```

ref 用"盒子 + `.value`"这种略显别扭的设计，就是为了把基本类型包进对象里再追踪——`.value` 就是访问这个盒子的代价。

## 三、.value：模板自动解包，JS 中必须手写

```html
<template>
  <div>{{ count }}</div>   <!-- 模板中自动解包，不写 .value -->
</template>

<script setup>
console.log(count)        // RefImpl {...}
console.log(count.value)  // 0
count.value++             // 修改必须 .value
</script>
```

模板中顶层 ref 会自动解包；但在 JS 逻辑里必须写 `.value`。忘写 `.value` 是 ref 最常见的坑——本质原因是**如果不经过 `.value`，JavaScript 无法感知你对这个值的读写**。

reactive 没有这个问题，`state.count++` 直接用。这是很多人偏爱 reactive 的原因。

## 四、重新赋值：ref 安全，reactive 会失去响应式

```javascript
const state = reactive({ list: [] })

function load(data) {
  state = data       // ❌ 报错：state 是 const；即使换成 let，新对象也不是响应式的
  state.list = data  // ✅ 修改属性，代理仍生效
}
```

reactive 返回的是**代理对象**，变量只是一个引用。整体替换后，变量指向了新的普通对象，与 Proxy 的联系断开，视图不再更新。所以 reactive 只能改属性，不能换整个对象。

ref 则随时可以：

```javascript
const list = ref([])
list.value = [1, 2, 3]  // ✅ 触发的是 value 的 setter，响应式不丢
```

## 五、reactive 的另一个坑：解构会丢失响应式

```javascript
const state = reactive({ count: 0 })
let { count } = state   // ❌ count 变成普通数字，与代理断开
count++                 // 视图不更新
```

解构相当于读了一次属性值赋给新变量，之后修改新变量与代理无关。需要解构时用 `toRefs` / `toRef` 把每个属性转成 ref 保持联系。

## 六、ref 传对象：内部自动用 reactive

ref 并不排斥对象，反而做了兼容：

```javascript
const state = ref({ count: 0 })
state.value.count++   // ✅ 深层响应式
```

`ref` 内部逻辑是：如果传入的是对象，就调用 `reactive` 包装后再存入 `value`（源码中就一行 `toReactive`）。所以 **ref(对象) = 对象的 reactive + 外层引用包装**，两层能力都有。

---

## 总结

| 对比项 | ref | reactive |
| --- | --- | --- |
| 实现方式 | RefImpl 类实例，getter/setter 拦截 `.value` | Proxy 代理整个对象 |
| 支持类型 | 任意（基本类型、对象） | 仅对象 / 数组等引用类型 |
| 取值写法 | JS 中必须 `.value`，模板自动解包 | 直接 `state.xx` |
| 整体重新赋值 | ✅ 安全，不丢响应式 | ❌ 失去响应式，只能改属性 |
| 解构 | ✅ 解构后仍是 ref（需 `.value`） | ❌ 解构即断开，需 `toRefs` |
| 深层对象 | 传入对象时内部自动调用 reactive | 默认深层响应 |

**实践建议**：默认用 `ref`，统一心智模型，不用纠结数据类型、也不用记 reactive 的两个坑；`reactive` 适合"一组相关数据的聚合体"这种不会被整体替换、不需要解构的场景。团队协作时保持风格统一，比选哪个更重要。
