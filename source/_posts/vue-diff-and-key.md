---
title: 虚拟 DOM diff 算法与 key 的正确用法
date: 2026-09-09 17:20:00
description: diff 算法的同层比较与 key 匹配，Vue2 双端比较与 Vue3 头尾预处理 + 最长递增子序列的区别，以及为什么 index 作 key 会状态错位、随机数作 key 更糟。
categories:
  - [Vue, 源码]
tags:
  - Vue
  - diff
  - 性能优化
---

# 虚拟 DOM diff 算法与 key 的正确用法

`v-for` 总被编辑器警告要加 `:key`，加了警告就消失。但 key 到底参与了什么过程？为什么不能用 index 当 key？为什么 `Math.random()` 当 key 更糟？

这些问题都指向同一个东西：**diff 算法**。本笔记讲三件事：diff 的基本策略、Vue2 / Vue3 实现的区别、key 的正确用法。

---

## 一、diff 解决了什么问题

```text
数据变化 → 生成新 vnode → 与旧树对比 → 计算最小的真实 DOM 操作 → 更新页面
```

- diff 的目标不是"找出所有不同"，而是：**用尽可能少的真实 DOM 操作，把旧树变成新树**——用廉价的 JS 计算换昂贵的 DOM 操作。
- 完整对比两棵树（树编辑距离）是 O(n³)，完全不可用；Vue 基于经验假设把它简化到了线性 O(n)。

## 二、三条基本策略

1. **只做同层比较**：跨层级移动被当作"旧位置删除 + 新位置创建"，而不是移动。
2. **类型不同，直接重建**：`<div>` → `<span>` 不再深入对比，删旧子树、建新子树。
3. **用 key 标识节点**：判断两个 vnode 能否复用的依据：

```javascript
function sameVnode(a, b) {
  return a.key === b.key && a.tag === b.tag
}
```

key 是节点身份的一部分，而不是附加属性：key 不同，哪怕标签内容都一样也是两个节点；**不写 key 则按索引配对、就地复用（in-place patch）**。

就地复用对顺序不变的纯展示列表效率最高，但一旦头部增删、排序、过滤（如 `A B C` → `B C`），按索引配对会导致剩余节点全部 patch、还要多删一个。key 就是为了解决这个。

## 三、key 到底是什么

> **key 是 vnode 的身份标识，diff 用它在新旧两组子节点之间建立配对关系。**

```text
旧子节点         新子节点
key=1  A        key=2  B
key=2  B        key=3  C
key=3  C        key=1  A

diff 不看位置，看 key：
key=1 → key=1  复用 + 移动
key=2 → key=2  复用 + 移动
key=3 → key=3  复用 + 移动
```

两个能力：**复用**（相同 key 只做增量更新，不销毁重建）、**移动**（位置变化被识别为移动，而非删除 + 新建）。
两个条件：**唯一**（同列表内不重复）、**稳定**（同一数据每次渲染得到同一 key）。

---

## 四、Vue2 的 diff：双端比较

核心是 `updateChildren()`，维护四个指针：oldStartIdx / oldEndIdx / newStartIdx / newEndIdx，每轮按顺序尝试四种配对：

```text
① oldStart vs newStart  头对头：patch 后双头指针各前进一步
② oldEnd   vs newEnd    尾对尾：patch 后双尾指针各后退一步
③ oldStart vs newEnd    节点被移到末尾：patch 后移到 oldEnd 之后
④ oldEnd   vs newStart  节点被移到开头：patch 后移到 oldStart 之前
```

四种都没命中（乱序）时，用 **keyMap** 兜底：拿旧子节点建 `key → 索引` 的映射，用新头节点的 key 去查——查到 → patch 并移到 oldStart 之前；查不到 → 全新节点，创建插入。**没有 key，keyMap 无从建起，兜底路径整个失效。**

循环结束时一侧先用完：旧的先用完 → 批量创建新增节点；新的先用完 → 批量删除多余节点。

## 五、Vue3 的 diff：预处理 + 最长递增子序列

Vue3 重写为 `patchKeyedChildren()`（借鉴自 ivi / inferno）：

```text
① 预处理：从头部、尾部同步相同节点；一侧先用完，剩下的即纯新增（创建）/ 纯删除
② 乱序中间部分：建 keyToNewIndexMap（key → 新索引），遍历旧节点——
   查到 → patch 并记录"新索引 → 旧索引"；查不到 → 删除
③ 对该数组求最长递增子序列（LIS），决定哪些节点不用动
```

核心是第 ③ 步：

```text
新顺序：  D    E    A    B    C      （旧 [A,B,C,D,E] → 新 [D,E,A,B,C]）
旧位置：  4    5    1    2    3
LIS = [1, 2, 3] → 对应 A、B、C
```

A、B、C 在新旧列表中相对顺序没变 → **不动**；D、E 不在 LIS 中 → **移动**。LIS 内的节点一次都不动，把 DOM 移动次数压到理论最小。

## 六、Vue2 与 Vue3 的区别

**相同点**：匹配依据没有变，都是 `sameVnode = key 相同 + 标签相同`；变的是"找到匹配之后，如何安排移动"。

| | Vue2 | Vue3 |
| --- | --- | --- |
| 入口函数 | `updateChildren` | `patchKeyedChildren` |
| 核心策略 | 双端比较 | 头尾预处理 + key 映射 + LIS |
| 快速路径 | 头头 / 尾尾 / 交叉头尾四种尝试 | 头部同步 / 尾部同步 + 纯增删处理 |
| 乱序处理 | keyMap 逐个处理新头节点，命中即移到 oldStart 前 | 整体遍历建映射数组后，LIS 一次性求出"不用动的节点" |
| 移动次数 | 局部最优 | 理论最小 |

同一个乱序例子，两代的行为：

```text
旧 [A,B,C,D,E,F] → 新 [C,D,E,A,B,F]

Vue2：keyMap 兜底逐个移动 C、D、E       → 移动 3 次
Vue3：LIS 保留 C、D、E，只移动 A、B     → 移动 2 次
```

一句话版本：**Vue2 是"两端往中间比较，命中就移动"；Vue3 是"先剪头尾，再用 LIS 让不该动的节点一步不动"。匹配规则没变，变的是"怎么移动得更少"。**

---

## 七、key 用错的翻车现场

### index 作 key

```html
<li v-for="(item, index) in list" :key="index">
  {{ item.name }}
  <input type="text" />
</li>
```

用户在三个输入框里输入 111 / 222 / 333 后删除第一个元素。index 重新计算：李四/王五的 key 从 1/2 变成 0/1，配对变成"李四复用张三的节点、王五复用李四的节点、旧 key=2 删除"。

patch 会正常更新文本子节点（"张三"→"李四"），但 input 里的 value 是非受控的原生 DOM 状态，patch 不会去动它：

```text
李四  [111]   ← 复用了张三的输入框
王五  [222]   ← 复用了李四的输入框
             ← 王五的输入框（333）随节点被删
```

**文本是对的，输入框全错位。** 总结两个问题：

```text
① 性能：本可复用的节点被迫全部 patch
② 正确性：输入框、动画、组件内部状态错位到别的数据上
```

### Math.random() 作 key

每次渲染都生成新 key，新列表所有 key 在旧列表都查不到 → 整个列表被判定为全新 → 全部删除重建。复用能力归零，节点内部状态全部丢失（输入框清空、组件重新 mounted、过渡重播）。比 index 更糟。

| key 写法 | 唯一 | 稳定 | 结果 |
| --- | --- | --- | --- |
| `item.id` | ✓ | ✓ | 复用 + 最少移动，正确且高效 |
| `index` | ✓ | ✓ | 不指向数据：patch 放大、状态错位 |
| `Math.random()` | ✓ | ✗ | 每次渲染全量重建，状态全丢 |
| 不写 key | — | — | 就地复用：乱序场景问题同 index |

## 八、key 的正确用法

```html
<li v-for="item in list" :key="item.id">
```

- 首选后端返回的业务 id。
- 没有 id 时，在数据**创建时**生成一次（`nanoid()` 等），不要渲染时生成（等价于随机数）：

```javascript
list.value = rawData.map(item => ({ ...item, uid: nanoid() }))
```

- 只接受字符串 / 数字 / symbol；对象作 key 会变成 `'[object Object]'`，等于没写。
- Vue3 中 `v-for` 写在 `<template>` 上时，key 放在 `<template>` 上（Vue2 是写在子元素上）：

```html
<template v-for="item in list" :key="item.id">
  <dt>{{ item.name }}</dt>
  <dd>{{ item.desc }}</dd>
</template>
```

什么时候可以用 index？满足全部条件才可以：纯展示（无表单元素、无有状态组件）、不重排不过滤、增删只发生在尾部。index 作 key 的错误是隐性的——不报错不警告，只在特定操作后悄悄错位——所以工程上：**一律使用稳定 id**。

---

## 九、总结

```text
完整树对比 O(n³) 不可行
  ↓ 三条简化：同层比较 / 类型不同重建 / key 标识 → O(n)
key 是节点复用与移动的唯一依据：唯一 + 稳定，来自数据本身
```

两代 diff 的区别，只记三条：

```text
匹配：   完全一致（key + 标签），两代没变
策略：   Vue2 双端比较 + keyMap 兜底
         Vue3 头尾预处理 + keyToNewIndexMap + LIS
移动次数：Vue2 局部最优 → Vue3 理论最小（LIS 内的节点不动）
```

> key 是给 diff 算法看的节点身份证："能复用的不重建，能不动的少移动"。身份证的要点只有两个——**唯一，且始终属于同一个人**。
