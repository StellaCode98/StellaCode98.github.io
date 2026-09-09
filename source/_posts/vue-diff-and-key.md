---
title: 虚拟 DOM diff 算法与 key 的正确用法
date: 2026-09-09 17:20:00
description: 从虚拟 DOM 的更新流程出发，理解 diff 算法为什么只做同层比较、Vue2 双端比较与 Vue3 最长递增子序列的实现思路，以及 key 在其中的作用——为什么 index 作 key 会翻车、随机数作 key 更糟，怎样才算 key 的正确用法。
categories:
  - [Vue, 源码]
tags:
  - Vue
  - diff
  - key
  - 虚拟DOM
  - 性能优化
---

# 虚拟 DOM diff 算法与 key 的正确用法

使用 Vue 或 React 时，几乎每个人都写过这样的代码：

```html
<li v-for="item in list" :key="item.id">
  {{ item.name }}
</li>
```

也几乎每个人都被编辑器警告过：

> Elements in iteration expect to have 'v-bind:key' directives.

于是 `key` 变成了一种"加上就不报错"的咒语。

但如果追问几个问题，很多人就答不上来了：

> key 到底参与了什么过程？
>
> 为什么不能用 index 当 key？
>
> 为什么用 `Math.random()` 当 key 反而更糟？

这些问题全部指向同一个东西：

```text
diff 算法
```

本文从虚拟 DOM 的更新流程开始，先理解 diff 为什么存在、它做了哪些简化，再看 Vue2 的双端比较和 Vue3 的最长递增子序列，最后回到 key 的正确用法。

---

## 一、diff 算法解决了什么问题？

框架普遍采用虚拟 DOM 的更新方式：

```text
数据变化
   ↓
重新执行渲染逻辑
   ↓
生成一棵新的虚拟 DOM 树
   ↓
和旧的虚拟 DOM 树对比
   ↓
计算出最小的真实 DOM 操作
   ↓
更新页面
```

其中"新旧对比"这一步，就是 diff：

```text
旧虚拟 DOM 树     新虚拟 DOM 树
     A                 A
   ┌─┴─┐             ┌─┴─┐
   B   C             B   D
                       └──→ C 去哪了？
```

diff 的目标不是"找出所有不同"，而是：

> **用尽可能少的真实 DOM 操作，把旧树变成新树。**

为什么如此在意 DOM 操作的数量？

因为创建和修改真实 DOM 的代价，远高于操作普通 JavaScript 对象：

```text
操作虚拟 DOM（普通 JS 对象）
        ↓
    速度快

操作真实 DOM
        ↓
可能引发样式计算、布局、绘制
        ↓
    代价高
```

所以一次典型的更新流程是：

```text
① 数据变化，生成新 vnode
② 新旧 vnode 做 diff（纯 JS 计算，便宜）
③ 根据 diff 结果，只对发生变化的部分做真实 DOM 操作（昂贵）
```

diff 本质上是一个**性能优化手段**：用廉价的 JS 计算换昂贵的 DOM 操作。

---

## 二、为什么不比较整棵树？

如果把两棵树当作任意的树结构做完整的"树编辑距离"对比，最优解法的时间复杂度是：

```text
O(n³)
```

1000 个节点意味着十亿级别的比较，完全不可用。

所以 Vue 和 React 都没有做"完整的树对比"，而是基于几个经验假设，把复杂度降到了线性：

```text
O(n³) → O(n)
```

这些假设来自对真实页面更新模式的观察。

---

## 三、diff 的三条基本策略

### 策略一：只做同层比较

diff 只会比较同一层级的节点：

```text
      旧树                新树
       ul                  ul
     ┌──┴──┐             ┌──┴──┐
     li    li            li    span
     ↓     ↓             ↓      ↓
   同层比较 ✓           同层比较：li vs span
                              ↓
                          类型不同
                              ↓
                        不再深入对比子树
```

跨层级的节点移动，会被当作：

```text
旧位置删除 + 新位置创建
```

而不是"移动"。

这看起来不够聪明，但背后的判断是：

> **真实页面中，跨层级移动节点是极少见的。**

为极少见的场景把复杂度抬回 O(n³)，不划算。

### 策略二：类型不同，直接重建

如果新旧节点的标签类型不同：

```html
<!-- 旧 -->
<div>hello</div>

<!-- 新 -->
<span>hello</span>
```

diff 不会去对比 `<div>` 和 `<span>` 内部的差异，而是：

```text
旧 div 及其整个子树
        ↓
      删除

新 span 及其整个子树
        ↓
      创建
```

理由同样来自经验：

> **不同类型的元素，渲染出的内容通常完全不同，细对比不如重建。**

### 策略三：用 key 标识节点

对于同一层级的列表：

```html
<li v-for="item in list" :key="item.id">
```

key 用来回答 diff 中最关键的问题：

> **新列表里的这个节点，是不是旧列表里那个节点的"同一个"？**

```text
key 相同 + 类型相同
        ↓
    认为是同一个节点
        ↓
    复用旧节点，只更新差异
    （必要时移动位置）
```

这三条策略合在一起，把"任意的树对比"简化成了"同层级的线性扫描"。

---

## 四、如何判断两个节点"相同"？

Vue 源码里有一个函数：

```javascript
sameVnode(a, b)
```

它决定两个 vnode 能否复用。简化后的核心判断是：

```javascript
function sameVnode(a, b) {
  return (
    a.key === b.key &&   // key 相同
    a.tag === b.tag      // 标签相同
  )
}
```

实际源码还包含一些细节，比如 `<input>` 会额外比较 type：

```javascript
(tag === 'input' && a.type === b.type)
```

但主线就两条：

```text
key 相同
  +
标签相同
    ↓
判定为同一节点
    ↓
进入复用流程（patchVnode）
```

注意一个容易忽略的点：

> **key 是节点身份的一部分，而不是附加属性。**

key 不同，哪怕标签、内容全都一样，也会被认为是两个不同的节点。

---

## 五、没有 key 的世界：就地复用

先看反例：如果不写 key，diff 会怎么做？

没有 key 时，Vue 对子节点的处理策略是**就地复用（in-place patch）**：

按索引一一配对：

```text
旧：  li[0]   li[1]   li[2]
       ↓       ↓       ↓
新：  li[0]   li[1]

索引 0 → 索引 0
索引 1 → 索引 1
索引 2 → 没有配对，删除
```

也就是：

```text
不看节点是谁，只看排在第几个
```

对于"纯展示、顺序永远不变"的列表，这样做的效率非常高——改哪个索引就 patch 哪个。

但一旦列表发生**头部的增删、排序、过滤**，问题就来了。

假设列表从：

```text
A  B  C
```

变成了：

```text
B  C
```

（头部删除了 A）

就地复用的处理方式是：

```text
索引 0：A 的节点 → patch 成 B 的内容
索引 1：B 的节点 → patch 成 C 的内容
索引 2：C 的节点 → 删除
```

本来只需要"删掉第一个节点"，现在变成了：

> **剩下每个节点都要 patch 一遍，还要多删一个节点。**

方向对了，但全在原地打补丁。key 要解决的，正是这个问题。

---

## 六、key 到底是什么？

有了前面的铺垫，可以给 key 一个准确的定义了：

> **key 是 vnode 的身份标识，diff 用它在新旧两组子节点之间建立配对关系。**

```text
旧子节点                新子节点
key=1  A               key=2  B
key=2  B               key=3  C
key=3  C               key=1  A

diff：不看位置，看 key

key=1 → key=1  复用，移动
key=2 → key=2  复用，移动
key=3 → key=3  复用，移动
```

key 带来的能力有两个：

```text
能力一：复用
  key 相同的节点只做属性/子节点的增量更新，
  而不是销毁重建

能力二：移动
  节点在列表中的位置变了，
  diff 能识别出"这是移动"，而不是"删除 + 新建"
```

这也是为什么 key 必须满足两个条件：

```text
① 唯一：同一列表内不能重复
② 稳定：每次渲染同一个数据应得到同一个 key
```

key 不是给开发者看的，是给 diff 算法做**节点匹配**用的。

接下来看 diff 算法具体怎么利用 key。

---

## 七、Vue2 的 diff：双端比较

Vue2 对两组子节点（oldChildren / newChildren）的 diff，核心函数是：

```javascript
updateChildren()
```

它维护了四个指针：

```javascript
let oldStartIdx = 0                 // 旧列表头部
let oldEndIdx = oldChildren.length - 1  // 旧列表尾部

let newStartIdx = 0                 // 新列表头部
let newEndIdx = newChildren.length - 1  // 新列表尾部
```

以及四个对应的节点：

```javascript
oldStartVnode   // 旧头
oldEndVnode     // 旧尾
newStartVnode   // 新头
newEndVnode     // 新尾
```

示意图：

```text
旧： [oldStart ... oldEnd]
新： [newStart ... newEnd]
```

---

## 八、双端比较的四次命中

每一轮比较，按顺序尝试四种配对：

```text
① oldStart vs newStart   （头对头）
② oldEnd   vs newEnd     （尾对尾）
③ oldStart vs newEnd     （旧头对 新尾）
④ oldEnd   vs newStart   （旧尾对 新头）
```

### ① 头对头命中

```text
旧： [A, B, C]
新： [A, B, D]

A vs A 命中
   ↓
patchVnode(A, A)
   ↓
双头指针各前进一步
```

### ② 尾对尾命中

```text
旧： [A, B, C]
新： [D, B, C]

C vs C 命中
   ↓
patchVnode
   ↓
双尾指针各后退一步
```

### ③ 旧头对上新尾：节点被移到了末尾

```text
旧： [A, B, C]
新： [B, C, A]

A（旧头）vs A（新尾）命中
   ↓
patchVnode
   ↓
把 A 移动到旧尾之后
   ↓
oldStartIdx++，newEndIdx--
```

### ④ 旧尾对上新头：节点被移到了开头

```text
旧： [A, B, C]
新： [C, A, B]

C（旧尾）vs C（新头）命中
   ↓
patchVnode
   ↓
把 C 移动到旧头之前
   ↓
oldEndIdx--，newStartIdx++
```

四种命中共用一个前提：

```text
sameVnode() 成立（key 相同 + 标签相同）
```

任何一种命中后，都回到循环开头，进行下一轮。

---

## 九、四头都配不上：keyMap 兜底

如果四次比较都没命中：

```text
旧： [D, E, A, B, C]
新： [E, C, D, A]

E 不在旧列表的头，也不在尾
        ↓
四种快速路径全部失效
```

这时 Vue2 会用 key 建立索引表：

```javascript
// 旧子节点：key → 索引
keyMap = {
  D: 0,
  E: 1,
  A: 2,
  B: 3,
  C: 4
}
```

然后拿**新头节点**的 key 去查：

```text
newStart = E
    ↓
keyMap['E'] = 1
    ↓
旧列表索引 1 的节点也是 E
    ↓
sameVnode 成立
    ↓
patchVnode + 把 E 移动到旧头之前
```

如果 keyMap 里查不到：

```text
newStart 的 key 在旧列表中不存在
        ↓
这是一个全新节点
        ↓
createElm 直接创建并插入
```

这一步是 key 最直接的用武之地：

> **没有 key，keyMap 无从建起，兜底路径整个失效。**

---

## 十、收尾：多退少补

指针不断靠近，循环结束的条件是某一侧的头指针越过尾指针：

```javascript
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx)
```

结束后只剩两种情况：

### 旧的先用完：新节点有剩余

```text
旧： []        （已全部处理）
新： [X, Y]    （还没处理）

        ↓
批量创建 X、Y 并插入
```

### 新的先用完：旧节点有剩余

```text
旧： [M, N]    （还没处理）
新： []        （已全部处理）

        ↓
批量删除 M、N
```

到这里，Vue2 的 diff 就完整了：

```text
updateChildren
   ↓
头头 / 尾尾 / 头尾 / 尾头 四次快速比较
   ↓
都不命中 → keyMap 兜底
   ↓
循环结束 → 多退少补
```

---

## 十一、Vue3 的 diff：预处理 + 最长递增子序列

Vue3 重写了这部分算法，思路变成了：

```text
① 预处理：从头、从尾同步相同的节点
② 中间"乱序部分"用 key 建映射
③ 用最长递增子序列决定哪些节点需要移动
```

### 第一步：从头部同步

```text
旧： [A, B, C, D, E]
新： [A, B, D, E, C]

A vs A ✓  patch，指针右移
B vs B ✓  patch，指针右移
C vs D ✗  停止
```

### 第二步：从尾部同步

```text
旧剩余： [C, D, E]
新剩余： [D, E, C]

E vs E ✓  patch，指针左移
D vs D ✓  patch，指针左移
C vs C ✗  停止
```

经过两轮预处理，真正需要"动脑子"的只剩：

```text
旧剩余： [C]
新剩余： [C]
```

很多真实场景（尾部追加、头部删除）在预处理阶段就直接消化完了，根本走不到复杂逻辑。

这两步预处理借鉴自 ivi / inferno 两个库。

### 第三步：处理中间乱序部分

对剩下的部分，Vue3 做三件事：

```text
① 为新节点建立 keyToNewIndexMap
    key → 新索引

② 遍历旧节点：
    能在 map 中查到 → patch，记录"新索引 → 旧索引"
    查不到          → 删除

③ 根据"新索引 → 旧索引"数组，
  求最长递增子序列（LIS）
```

---

## 十二、为什么需要最长递增子序列？

这是 Vue3 diff 最精妙的一步。

"新索引 → 旧索引"数组描述了：**新列表中每个位置上的节点，原来在旧列表的什么位置。**

举例：

```text
新顺序：  D    E    A    B    C
旧位置：  4    5    1    2    3
```

这个数组 `[4, 5, 1, 2, 3]` 的最长递增子序列是：

```text
[1, 2, 3]  （对应 A、B、C）
```

它的含义是：

> **A、B、C 在新旧列表中的相对顺序没有变化。**

于是 Vue3 的决策是：

```text
处于最长递增子序列中的节点 → 不动
不处于其中的节点          → 移动
```

```text
D、E → 需要移动
A、B、C → 保持不动
```

为什么这样最优？

因为**递增子序列内的节点相对顺序已经正确**，不需要任何移动；剩下的节点插入到正确位置即可。这样把 DOM 移动次数降到最少。

对比一下 Vue2 和 Vue3 的思路差异：

| | Vue2 | Vue3 |
| --- | --- | --- |
| 核心策略 | 双端比较 | 预处理 + key 映射 + LIS |
| 快速路径 | 头尾四次比较 | 头部 / 尾部同步 |
| 乱序处理 | keyMap 找到就移动 | LIS 保证只移动必须移动的 |
| 移动次数 | 较优 | 理论最优 |

注意：两者的**匹配依据没有变**，都是：

```text
sameVnode = key 相同 + 标签相同
```

变的是"找到匹配之后，如何安排移动"的策略。

---

## 十三、经典翻车现场：index 作 key

终于可以正面回答那个经典问题了。

```html
<li v-for="(item, index) in list" :key="index">
  {{ item.name }}
  <input type="text" />
</li>
```

```javascript
list = [
  { id: 1, name: '张三' },
  { id: 2, name: '李四' },
  { id: 3, name: '王五' }
]
```

用户在三个输入框里分别输入了：

```text
张三 → 输入了 "111"
李四 → 输入了 "222"
王五 → 输入了 "333"
```

现在删除第一个元素，list 变成：

```javascript
list = [
  { id: 2, name: '李四' },
  { id: 3, name: '王五' }
]
```

### 用 index 作 key 时发生了什么

删除前，节点的 key 是：

```text
张三 → key = 0
李四 → key = 1
王五 → key = 2
```

删除后，key 重新计算：

```text
李四 → key = 0
王五 → key = 1
```

于是 diff 做出的配对是：

```text
新 key=0（李四） ← 旧 key=0（张三的节点）
新 key=1（王五） ← 旧 key=1（李四的节点）
旧 key=2（王五的节点） → 没有配对 → 删除
```

patch 时发生的事情：

```text
① 文本子节点更新：
   "张三" → "李四" ✓
   "李四" → "王五" ✓

② input 是非受控的原生 DOM 状态：
   patch 只更新 vnode 描述的属性，
   不会去动用户输入的 value
```

最终页面呈现：

```text
李四  [111]   ← 张三的输入框，被复用了
王五  [222]   ← 李四的输入框，被复用了
              ← 王五的输入框（333）被删掉了
```

**文本是对的，输入框全错位了。**

### 用 id 作 key 时发生了什么

```text
新 key=2（李四） ← 旧 key=2（李四的节点）  完全相同，无需 patch
新 key=1（张三的节点） → 没有配对 → 删除
```

结果：

```text
李四  [222]  ✓ 复用原节点，连 patch 都省了
王五  [333]  ✓
```

除了问题本身，还有性能差距：

```text
index 作 key：所有剩余节点都要 patch + 删一个节点
id   作 key：只删除一个节点，其他节点原样复用
```

一句话总结 index 作 key 的两个问题：

```text
① 性能：本可复用的节点被迫全部 patch
② 正确性：节点内部的状态（输入框、动画、组件内部状态）
   会错位到别的数据上
```

---

## 十四、比 index 更糟：随机数作 key

既然 key 要唯一，那这样写行不行？

```html
<li v-for="item in list" :key="Math.random()">
```

不行，而且比 index 更糟。

`Math.random()` **每次渲染都会生成新值**：

```text
第一次渲染：  key = 0.8231, 0.1947, 0.5533
第二次渲染：  key = 0.2764, 0.9012, 0.4417
```

对 diff 来说：

```text
新列表的每个 key
        ↓
在旧列表中都找不到
        ↓
判定为"全新节点"
        ↓
整个列表：全部删除 + 全部重建
```

后果：

```text
① 性能：diff 的复用能力完全归零，
   每次更新都等价于重渲染整个列表

② 正确性：所有节点内部状态全部丢失
   ——输入框被清空、组件被重新 mounted、
   过渡动画从头播放
```

key 的两个要求"唯一、稳定"，随机数只满足唯一：

```text
index    → 稳定但不指向数据（会错位）
random   → 唯一但不稳定（全部重建）
正确的 key → 唯一且稳定，来自数据本身
```

---

## 十五、key 的正确用法

### 基本原则

```text
key 应该来自数据本身的、稳定唯一的标识
```

首选后端返回的业务 id：

```html
<li v-for="item in list" :key="item.id">
```

```javascript
list = [
  { id: 1001, name: '张三' },
  { id: 1002, name: '李四' }
]
```

### 没有 id 怎么办

在数据**创建时**（而不是渲染时）生成一次，之后保持不变：

```javascript
import { nanoid } from 'nanoid'

list.value = rawData.map(item => ({
  ...item,
  uid: nanoid()   // 创建时生成一次，之后不再变化
}))
```

```html
<li v-for="item in list" :key="item.uid">
```

关键区别：

```text
渲染时生成  → 每次渲染都变 → 等价于随机数
创建时生成  → 只生成一次   → 稳定
```

### key 的类型要求

key 只接受字符串、数字、symbol：

```html
:key="item.id"          ✓ 数字
:key="item.uuid"        ✓ 字符串
:key="item"             ✗ 对象（会被转成 '[object Object]'，全部相同）
```

### `<template v-for>` 中的 key

Vue3 中，`v-for` 写在 `<template>` 上时，key 要放在 `<template>` 上：

```html
<template v-for="item in list" :key="item.id">
  <dt>{{ item.name }}</dt>
  <dd>{{ item.desc }}</dd>
</template>
```

而不是放在子元素上（这是与 Vue2 不同的地方）。

---

## 十六、什么时候可以用 index？

index 并非绝对禁区。满足**全部**以下条件时，index 作 key 是安全的：

```text
① 列表是纯展示，不包含表单类元素或带状态的组件
② 列表不会重排（sort）、过滤（filter）
③ 增删只发生在尾部，或数据本身永不变化
```

比如完全静态的导航、配置驱动的纯文案列表：

```html
<nav>
  <a v-for="(item, index) in menus" :key="index">
    {{ item.label }}
  </a>
</nav>
```

但实际项目中，列表"以后会加搜索功能"的概率远超预期，而 index 作 key 的错误又是**隐性**的——不报错、不警告，只在特定操作后悄悄错位。

所以工程上的建议很简单：

> **一律使用稳定 id，把 index 留给"确定永远不会变"的静态列表。**

---

## 十七、React 中的 diff 与 key

同样的思想也适用于 React。

React 的 reconciliation 同样基于两条假设：

```text
① 不同类型的元素 → 推倒重建
② 同层比较，通过 key 标识子节点
```

使用方式也一致：

```jsx
{list.map(item => (
  <li key={item.id}>{item.name}</li>
))}
```

React 中 index 作 key 的后果和 Vue 一致：列表头部插入元素时状态错位、输入框内容张冠李戴。

实现层面的主要差异在于"乱序部分的移动策略"：

```text
Vue2：双端比较
Vue3：预处理 + 最长递增子序列
React：单向遍历 + 已放置节点索引（lastPlacedIndex）
```

React 不做双端比较，从左到右遍历新列表，用"最后一个已放置节点的位置"判断当前节点是需要移动还是原地复用；对乱序程度高的列表，React 的移动次数通常多于 Vue3。

对这些差异，理解到这一层就够了：

> **不同框架的 diff 策略有差异，但 key 的语义完全一致——节点身份标识。**

---

## 十八、总结

把全文的主线串起来：

```text
虚拟 DOM 更新需要对比新旧树
        ↓
完整对比是 O(n³)，不可行
        ↓
三条简化：同层比较 / 类型不同重建 / key 标识
        ↓
key 成为节点复用与移动的唯一依据
```

两代 Vue 的 diff 实现：

```text
Vue2 updateChildren
  四指针双端比较
  + keyMap 兜底
  + 多退少补

Vue3 patchKeyedChildren
  头尾预处理
  + keyToNewIndexMap
  + 最长递增子序列（最少移动）
```

key 的正确用法，浓缩成一张表：

| key 写法 | 唯一 | 稳定 | 结果 |
| --- | --- | --- | --- |
| `item.id` | ✓ | ✓ | 复用 + 最少移动，正确且高效 |
| `index` | ✓ | ✓ | 位置不指向数据：patch 放大、状态错位 |
| `Math.random()` | ✓ | ✗ | 每次渲染全量重建，状态全丢 |
| 不写 key | — | — | 就地复用：乱序场景下问题同 index |

最后回到那个根本问题——key 是什么？

> **key 是给 diff 算法看的节点身份证：diff 靠它在新旧列表之间配对节点，从而做到"能复用的不重建，能不动的少移动"。**

身份证的要点从来只有两个：**唯一，且始终属于同一个人。**

---
