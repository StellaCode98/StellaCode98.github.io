---
title: Vue3 编译器：模板如何变成渲染函数，以及它偷偷做了哪些优化
date: 2026-09-10 10:30:00
description: 从 parse、transform、generate 三个阶段理解 Vue3 编译器的工作流程，再看静态提升、patchFlag、block 树三大编译优化如何让运行时"只更新真正会变的部分"，最后串起编译产物与运行时 patch 的协作方式。
categories:
  - [Vue, 源码]
tags:
  - Vue3
  - patchFlag
  - 性能优化
---

# Vue3 编译器：模板如何变成渲染函数，以及它偷偷做了哪些优化

写 Vue 时，我们写的永远是模板：

```html
<div class="box">
  <h1>标题不会变</h1>
  <p>{{ message }}</p>
</div>
```

但浏览器不认识 `{{ }}`，也不认识 `v-if`。真正执行的其实是一段 JavaScript 函数——**render 函数**。

从模板到 render 函数的过程，就是 **编译**。Vue3 性能提升的大头，正是藏在编译器里：**编译时能算清楚的事，绝不留到运行时。**

本文先看编译的三个阶段，再看 Vue3 由此做出的三大优化。

---

## 一、编译三阶段：parse → transform → generate

Vue3 的编译器（`@vue/compiler-dom`）是一条三段流水线：

```text
模板字符串
   ↓  parse        —— 词法/语法分析
AST（抽象语法树）
   ↓  transform    —— 遍历并改写 AST
优化后的 AST
   ↓  generate     —— 代码生成
render 函数代码字符串
```

用一个最小例子走一遍。

### 1. parse：模板 → AST

parse 把字符串切成一个个标签和文本，组装成一棵描述结构的树：

```text
<div>
  <p>{{ message }}</p>
</div>
```

解析结果（简化）：

```js
{
  type: 'Element', tag: 'div',
  children: [
    { type: 'Element', tag: 'p',
      children: [
        { type: 'Interpolation', content: { content: 'message' } }
      ]
    }
  ]
}
```

和虚拟 DOM 很像，但 AST 是**给编译器看的**，节点上会挂大量编译期信息（是否静态、绑定了哪些指令等），编译完成后就丢弃。

### 2. transform：AST → 优化后的 AST

transform 阶段以"插件"的方式遍历 AST，做两类事：

**改写指令**——框架没有任何"指令"，全是语法糖：

```text
v-if="show"   →  三元表达式节点
v-for="..."   →  renderList() 调用
v-model       →  value 属性 + onInput 事件
```

**收集信息、打标记**——这是 Vue3 优化的核心，也是下面几节的主角：

```text
纯静态节点 → 标记 hoistable，可提升
动态文本   → 打 patchFlag = TEXT
动态 class → 打 patchFlag = CLASS
动态节点   → 挂到最近 block 的 dynamicChildren
```

### 3. generate：AST → 代码字符串

最后把 AST 打印成可执行的代码。开头的模板，编译结果大致是：

```js
const _hoisted_1 = createElementVNode("h1", null, "标题")

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", { class: "box" }, [
    _hoisted_1,
    _createElementVNode("p", null,
      _toDisplayString(_ctx.message),
      1 /* TEXT */
    )
  ]))
}
```

即使没读过源码，也能看出几个"不对劲"的地方：

> 为什么 `h1` 跑到了 render 函数外面？
>
> 为什么 `p` 节点后面多了一个 `1 /* TEXT */`？
>
> 为什么开头多了个 `_openBlock()`？

这三个问题，正好对应 Vue3 的三大编译优化。

---

## 二、优化一：静态提升（hoistStatic）——静态的只做一次

`<h1>标题不会变</h1>` 是纯静态节点：它的标签、属性、内容永远不会变。

如果写在 render 函数里，每次重新渲染都会重新执行 `createElementVNode("h1", ...)`，创建一个一模一样的新虚拟节点——纯属浪费。

编译器把它**提升到 render 函数外部**：

```js
// 模块顶层，只执行一次
const _hoisted_1 = createElementVNode("h1", null, "标题")

function render() {
  return createElementBlock("div", null, [
    _hoisted_1,  // 每次渲染直接复用同一个引用
    /* ... */
  ])
}
```

于是无论组件重渲染多少次，`h1` 的虚拟节点永远只有一份。

更进一步，**连续的多个静态节点会被合并预字符串化**：

```js
const _hoisted_1 = createStaticVNode('<h1>标题</h1><p>说明文字</p><span>...</span>')
```

三个节点变成一个字符串，连 vnode 结构都省了。

---

## 三、优化二：patchFlag——给动态节点"哪里会变"贴上标签

这是 Vue3 最核心的设计。

回顾传统 diff（Vue2）的问题：拿到一个节点，**不知道它哪里会变**，只能把 props、class、style、children……全比对一遍，大量比对发生在根本不可能变的地方。

Vue3 的答案：**编译时就知道 `{{ message }}` 里只有插值是动态的，那就把这个信息直接写进产物：**

```js
createElementVNode("p", null, toDisplayString(_ctx.message), 1 /* TEXT */)
```

最后的参数 `1` 就是 **patchFlag**，含义是"这个节点只有**文本**是动态的"。

运行时 patch 看到这个标记，就只比对文本，props、class 一律跳过。

常见的 flag 值（位运算，可组合）：

| flag | 值 | 含义 | patch 时只做的事 |
| --- | --- | --- | --- |
| TEXT | 1 | 动态文本 | 比对 children 中的文本 |
| CLASS | 2 | 动态 class | 只比对 class |
| STYLE | 4 | 动态 style | 只比对 style |
| PROPS | 8 | 动态属性 | 只比对标记过的那几个 prop |
| FULL_PROPS | 16 | 带动态 key | 完整比对所有 props |
| STABLE_FRAGMENT | 64 | 子节点顺序稳定 | 跳过乱序 diff |

对比一下：

```text
Vue2 patch 一个节点：
  比对 tag、props、class、style、children……（以防万一）

Vue3 patch 一个节点：
  读 flag → TEXT → 只 setText（有的放矢）
```

**从"全量比对"变成"靶向比对"**——性能差距就来自这里。

---

## 四、优化三：block 与 dynamicChildren——跳过整棵静态子树

patchFlag 解决了"单个节点比什么"，还剩一个更大的问题：**怎么找到需要 patch 的节点？**

传统方式是递归遍历整棵虚拟 DOM 树，逐个进入、逐个比对——哪怕子树全是静态内容，也必须走一遍。

Vue3 的做法：编译器在 transform 阶段发现，模板里动态的节点是**有限且已知**的。于是让动态节点把自己"上报"给最近的 block：

```js
(_openBlock(), _createElementBlock("div", { class: "box" }, [
  _hoisted_1,
  _createElementVNode("p", null, toDisplayString(_ctx.message), 1 /* TEXT */)
]))
```

`_openBlock()` 开启一个收集上下文，随后所有**带 patchFlag 的后代节点**（不论嵌套多深）都会被收集进当前 block 的 `dynamicChildren` 数组。

更新时：

```text
传统递归：
  从根节点向下，逐层逐个比对
        ↓
  静态子树也要走一遍

block 更新：
  只遍历 dynamicChildren
        ↓
  一整棵静态子树？直接不存在于这个数组里
```

也就是说，模板规模可以任意大，**动态节点只有 3 个，更新时就只看这 3 个**。树的深度、静态内容的多少，都不再影响更新成本。

> 结构会变的地方（`v-if` / `v-for`）会被编译成新的 block，形成 **block tree**，保证"动态"的收集范围始终正确。这也是为什么上一篇讲 diff 时说：带 key 的列表比对（`patchKeyedChildren`）只发生在 block 的直接子级这一层——更深层的内容早已被各自的 block 接管了。

---

## 五、顺手的小优化：事件缓存

内联事件处理器也会被缓存，避免每次渲染创建新函数：

```html
<button @click="count++">{{ count }}</button>
```

```js
function render(_ctx, _cache) {
  return createElementVNode("button", {
    onClick: _cache[0] || (_cache[0] = () => _ctx.count++)
  }, toDisplayString(_ctx.count), 1 /* TEXT */)
}
```

`_cache[0] || (_cache[0] = ...)`——第一次创建后存入缓存，之后每次渲染返回同一个函数引用。否则每次渲染 `onClick` 都是新函数，patch 判定 props 变了，牵连子组件无谓更新。

---

## 六、串起来：编译器与运行时的分工

现在可以完整回答"Vue3 为什么快"了：

```text
编译时（构建阶段，只做一次）
  静态提升    → 静态节点只创建一次
  预字符串化  → 连续静态节点合并为字符串
  patchFlag   → 记录每个动态节点"哪里会变"
  block       → 收集所有动态节点为扁平数组
  事件缓存    → 内联处理器保持稳定引用

运行时（每次更新，反复执行）
  不再递归整棵树
  只遍历 dynamicChildren
  按 patchFlag 只比对标记的部分
  静态内容零参与
```

一句话：**Vue2 的运行时在"猜"哪里变了，Vue3 的编译器直接"告诉"它。**

需要注意：这些优化依赖**模板静态可分析**。手写 render 函数 / `h()` 或使用运行时编译（如在浏览器里传字符串模板），编译信息要么没有、要么延迟到运行时，优化收益会打折扣——这也是"能用模板就别手写 render"的底层原因。

---

## 总结

编译三阶段，各自职责一句话：

| 阶段 | 输入 → 输出 | 干什么 |
| --- | --- | --- |
| parse | 模板字符串 → AST | 切词法、建树 |
| transform | AST → 优化后 AST | 改写指令 + 打标记（优化主战场） |
| generate | AST → 代码字符串 | 打印出 render 函数 |

三大优化，解决三个问题：

```text
静态内容反复创建   →  静态提升，只创建一次
不知道节点哪会变   →  patchFlag，靶向比对
不知道该更新谁     →  block，只遍历动态节点
```

最后回到那句话：

> **Vue3 编译优化的本质：把"哪些会变、哪里会变"在编译期算清楚、写进产物，让运行时的每一次更新都只做最少的事。**

---
