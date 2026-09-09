---

title: CSS 中的层叠上下文与 BFC：那些年「莫名其妙」的样式问题
date: 2026-09-09 16:30:00
description: 从 z-index 失效、margin 溢出、浮动高度塌陷到父元素覆盖子元素，系统理解 CSS 中的层叠上下文与 BFC。
categories:
 - [前端基础, css]
tags:
 - CSS
 - 层叠上下文
 - BFC
 - z-index
 - 前端基础

---

在写 CSS 的过程中，经常会遇到一些非常奇怪的问题：

* 明明设置了 `z-index: 9999`，元素还是被盖住了；
* 子元素设置了 `z-index`，却无法超过另一个父元素；
* 两个元素都设置了 `position: relative`，为什么 `z-index` 还是不生效？
* 子元素设置 `margin-top`，为什么父元素也跟着往下移动？
* 父元素明明包裹着子元素，为什么设置背景色之后看不到完整高度？
* 一个元素设置了 `float`，父元素的高度突然变成了 `0`；
* 给元素加一个 `overflow: hidden`，莫名其妙很多布局问题就消失了。

这些问题表面上看起来毫无关系。

但往 CSS 的底层规则继续深入，会发现它们背后经常涉及两个非常重要的概念：

> **BFC（Block Formatting Context，块级格式化上下文）**

以及：

> **层叠上下文（Stacking Context）**

两者名字看起来很像，但解决的问题完全不同。

简单来说：

* **BFC 主要解决的是布局问题**
* **层叠上下文主要解决的是层级问题**

理解这两个概念之后，很多以前需要“试一个 CSS 属性看看”的问题，就可以直接从规则上分析。

---

# 一、先区分：BFC 和层叠上下文到底是什么？

可以先建立一个最简单的认识。

## BFC

BFC 全称：

```text
Block Formatting Context
块级格式化上下文
```

它可以理解成：

> **一个独立的块级布局环境。**

在这个环境内部，元素按照自己的规则进行布局，并且会和外部形成一定程度的隔离。

BFC 更多影响：

```text
普通文档流
浮动
高度计算
margin
元素之间的布局关系
```

---

## 层叠上下文

层叠上下文可以理解成：

> **一个独立的“图层世界”。**

元素在这个世界里进行 `z-index` 层叠。

例如：

```text
页面
│
├── 图层 A
│   ├── A1
│   └── A2
│
└── 图层 B
    ├── B1
    └── B2
```

A 内部的元素，即使：

```css
z-index: 999999;
```

也不一定能够跑到 B 的外面。

因为：

> **`z-index` 首先受到所属层叠上下文的限制。**

所以可以先记住：

| 概念    | 主要解决的问题         |
| ----- | --------------- |
| BFC   | 布局              |
| 层叠上下文 | 层级              |
| BFC   | 高度、浮动、margin 等  |
| 层叠上下文 | `z-index`、覆盖关系等 |

---

# 二、为什么 `z-index: 9999` 还是被盖住？

这是开发过程中非常经典的问题。

例如：

```html
<div class="box-a">
  <div class="child-a"></div>
</div>

<div class="box-b"></div>
```

CSS：

```css
.box-a {
  position: relative;
  z-index: 1;
}

.child-a {
  position: relative;
  z-index: 9999;
}

.box-b {
  position: relative;
  z-index: 2;
}
```

可能会发现：

```text
child-a
```

明明是：

```css
z-index: 9999;
```

却依然被：

```text
box-b
```

盖住。

为什么？

---

# 三、z-index 不是“全局排行榜”

很多刚接触 CSS 的时候，会产生一个误解：

```text
z-index 越大
↓
元素越靠上
```

实际上并不是这么简单。

更准确的理解应该是：

> **z-index 的比较通常发生在同一个层叠上下文中。**

这里实际上存在两个层级：

```text
页面层叠上下文
│
├── box-a
│   └── child-a
│       z-index: 9999
│
└── box-b
    z-index: 2
```

`child-a` 的 `9999`，并不是直接拿来和 `box-b` 的 `2` 进行全局比较。

首先要比较：

```text
box-a
vs
box-b
```

也就是：

```text
z-index: 1
vs
z-index: 2
```

因为：

```text
box-b > box-a
```

所以：

```text
box-b
```

所在的层叠上下文已经压在：

```text
box-a
```

之上。

此时：

```css
.child-a {
  z-index: 9999;
}
```

也无法突破：

```text
box-a
```

这个层叠上下文。

---

# 四、可以把层叠上下文理解成“图层组”

这个理解非常重要。

假设有：

```text
页面
│
├── A
│   z-index: 1
│   │
│   └── A-child
│       z-index: 9999
│
└── B
    z-index: 2
```

最终的关系并不是：

```text
9999 > 2
```

而是：

```text
A 这一组
↓
B 这一组
```

先比较：

```text
A: 1
B: 2
```

确定：

```text
B 在 A 上面
```

然后 A 内部再怎么调整：

```text
A-child: 9999
A-child: 999999
A-child: 99999999
```

都无法突破 A 这个“图层组”。

所以排查 `z-index` 问题时，不要只盯着：

```css
z-index: 9999;
```

而应该问：

> **这个元素到底属于哪个层叠上下文？**

---

# 五、哪些情况会创建层叠上下文？

常见的层叠上下文来源包括：

## 1. 根元素

HTML 根元素：

```html
<html>
```

本身就是最外层的层叠上下文。

---

## 2. 定位元素配合 z-index

例如：

```css
.box {
  position: relative;
  z-index: 1;
}
```

或者：

```css
.box {
  position: absolute;
  z-index: 1;
}
```

通常会形成新的层叠上下文。

---

## 3. fixed / sticky

例如：

```css
.box {
  position: fixed;
}
```

或者：

```css
.box {
  position: sticky;
}
```

也可能形成层叠上下文。

---

## 4. opacity 小于 1

例如：

```css
.box {
  opacity: 0.9;
}
```

会创建层叠上下文。

注意：

```css
opacity: 1;
```

不会因为这个原因创建新的层叠上下文。

---

## 5. transform

例如：

```css
.box {
  transform: translateX(10px);
}
```

也会创建层叠上下文。

这也是为什么有时候给父元素加了：

```css
transform: translate(...)
```

之后，某些：

```text
position: fixed
```

元素的表现会发生变化。

---

## 6. filter

例如：

```css
.box {
  filter: blur(10px);
}
```

也会影响层叠上下文。

---

## 7. isolation

```css
.box {
  isolation: isolate;
}
```

可以显式创建一个新的层叠上下文。

这个属性在处理复杂的层级关系时非常有用。

---

# 六、`z-index` 失效时，真正应该检查什么？

以后遇到：

```css
z-index: 999999;
```

仍然被覆盖，可以按照这个顺序排查：

```text
① 元素有没有定位 / 是否满足 z-index 生效条件

↓

② 元素属于哪个层叠上下文

↓

③ 父元素有没有创建新的层叠上下文

↓

④ 两个元素是不是属于不同的层叠上下文

↓

⑤ 比较的是哪个层叠上下文，而不是单纯比较两个 z-index
```

尤其是：

```css
transform
opacity
filter
position
z-index
isolation
```

这些属性值得重点检查。

---

# 七、再来看 BFC

理解了层叠上下文之后，再来看另一个非常容易混淆的概念：

```text
BFC
```

BFC 是：

```text
Block Formatting Context
```

它本质上是一套：

> **独立的块级格式化规则。**

可以简单理解成：

```text
普通页面
│
├── BFC A
│   ├── 元素 A
│   └── 元素 B
│
└── BFC B
    ├── 元素 C
    └── 元素 D
```

BFC 内部的布局不会完全按照外部环境随意影响。

因此，它特别适合解决：

```text
浮动
高度塌陷
margin 问题
布局互相影响
```

---

# 八、BFC 最经典的问题：浮动导致父元素高度塌陷

来看一个非常经典的例子。

```html
<div class="parent">
  <div class="child"></div>
</div>
```

```css
.parent {
  background: #eee;
}

.child {
  width: 200px;
  height: 100px;
  float: left;
}
```

可能会发现：

```text
parent
高度 ≈ 0
```

为什么？

因为：

```text
float
```

会脱离普通文档流。

于是父元素在计算普通文档流高度的时候：

```text
child
```

不再参与正常高度计算。

结果：

```text
parent
┌──────────────┐
│              │
└──────────────┘
```

父元素高度可能变成：

```text
0
```

而浮动元素跑到了外面。

---

# 九、为什么 BFC 可以解决浮动高度塌陷？

可以让父元素形成 BFC：

```css
.parent {
  display: flow-root;
}
```

或者：

```css
.parent {
  overflow: hidden;
}
```

此时父元素建立 BFC。

BFC 有一个非常重要的规则：

> **BFC 在计算高度时，会把浮动元素纳入高度计算。**

于是：

```text
parent
┌─────────────────┐
│                 │
│      child      │
│                 │
└─────────────────┘
```

父元素可以正确包裹浮动元素。

现代项目中，如果目的只是：

> **让父元素包含浮动子元素**

更推荐：

```css
.parent {
  display: flow-root;
}
```

而不是为了清除浮动随手写：

```css
overflow: hidden;
```

因为：

```css
overflow: hidden;
```

除了创建 BFC，还会改变溢出内容的处理方式。

---

# 十、BFC 解决的另一个经典问题：margin 塌陷

来看：

```html
<div class="parent">
  <div class="child"></div>
</div>
```

```css
.parent {
  background: #eee;
}

.child {
  margin-top: 50px;
}
```

很多时候会发现：

```text
child 的 margin-top
```

看起来像是把：

```text
parent
```

一起往下推了。

视觉上类似：

```text
       ↓ 50px

┌─────────────────┐
│                 │
│      child      │
│                 │
└─────────────────┘
```

而不是：

```text
┌─────────────────┐
│   50px           │
│                  │
│      child       │
└─────────────────┘
```

这就是经典的：

> **margin collapse（外边距折叠）**

---

# 十一、为什么会发生 margin 塌陷？

CSS 中存在一些情况下的垂直 margin 合并。

例如：

```text
父元素
│
└── 第一个子元素
    margin-top: 50px
```

如果父元素和子元素之间没有：

```text
border
padding
inline content
新的 BFC
```

等东西进行隔离，那么：

```text
父元素 margin
+
子元素 margin
```

可能发生合并。

这就是为什么：

```css
.child {
  margin-top: 50px;
}
```

会出现一种：

> “怎么子元素的 margin 把父元素也顶下去了？”

的感觉。

---

# 十二、创建 BFC，可以隔离 margin

例如：

```css
.parent {
  display: flow-root;
}
```

或者：

```css
.parent {
  overflow: hidden;
}
```

让父元素形成 BFC。

这样：

```text
parent BFC
│
└── child
    margin-top: 50px
```

就形成了一个独立的布局环境。

因此，在实际开发中，如果确实需要通过 BFC 隔离这种布局影响，可以考虑：

```css
.parent {
  display: flow-root;
}
```

---

# 十三、BFC 还有一个非常重要的作用：阻止元素被浮动元素覆盖

例如：

```html
<div class="left"></div>
<div class="content"></div>
```

```css
.left {
  width: 200px;
  height: 300px;
  float: left;
}

.content {
  height: 300px;
  background: #eee;
}
```

可能出现：

```text
┌───────┬─────────────────┐
│ left  │                 │
│ float │    content      │
│       │                 │
└───────┴─────────────────┘
```

但是普通块级元素的内容区域可能会和浮动元素产生环绕关系。

这时候可以让：

```css
.content {
  display: flow-root;
}
```

形成新的 BFC。

于是：

```text
┌───────┐ ┌─────────────────┐
│       │ │                 │
│ float │ │     content     │
│       │ │                 │
└───────┘ └─────────────────┘
```

可以把它理解为：

> **BFC 不允许自己的布局区域和外部浮动元素随意混在一起。**

---

# 十四、哪些属性可以创建 BFC？

常见方式：

## 1. `display: flow-root`

现代 CSS 中非常推荐：

```css
.box {
  display: flow-root;
}
```

语义非常明确：

> 我要建立一个新的块级格式化上下文。

---

## 2. `overflow` 非 `visible`

例如：

```css
.box {
  overflow: hidden;
}
```

或者：

```css
.box {
  overflow: auto;
}
```

或者：

```css
.box {
  overflow: scroll;
}
```

传统开发中经常利用：

```css
overflow: hidden;
```

来触发 BFC。

但它同时会产生裁剪效果，所以不要单纯把它当成：

> “BFC 开关”

---

## 3. `display: flex`

```css
.box {
  display: flex;
}
```

Flex 容器本身会建立一个独立的布局环境。

---

## 4. `display: grid`

```css
.box {
  display: grid;
}
```

同样会形成独立的布局上下文。

---

## 5. `float`

例如：

```css
.box {
  float: left;
}
```

浮动元素自身也会建立 BFC。

---

## 6. 绝对定位

例如：

```css
.box {
  position: absolute;
}
```

以及：

```css
.box {
  position: fixed;
}
```

也会建立独立的格式化上下文。

---

# 十五、为什么 `overflow: hidden` 经常“神奇地解决问题”？

前端开发中可能见过大量这样的代码：

```css
.parent {
  overflow: hidden;
}
```

然后：

> “诶，好了。”

但如果不知道 BFC 原理，就会变成：

```text
问题出现
↓
overflow: hidden
↓
好了
↓
不知道为什么
```

实际上：

```css
overflow: hidden;
```

做了不止一件事情。

其中一个重要效果就是：

```text
创建 BFC
```

于是它可以帮助解决：

```text
浮动高度塌陷
margin 塌陷
浮动元素影响布局
```

但同时：

```text
overflow: hidden
```

还意味着：

> 超出元素边界的内容可能会被裁剪。

所以有时候你会发现：

```text
下拉菜单
tooltip
popover
```

突然消失了。

原因可能就是：

```css
overflow: hidden;
```

把它裁掉了。

---

# 十六、这也是为什么不要“无脑 overflow: hidden”

比如：

```html
<div class="container">
  <button>菜单</button>

  <div class="dropdown">
    下拉菜单
  </div>
</div>
```

如果：

```css
.container {
  overflow: hidden;
}
```

而：

```css
.dropdown {
  position: absolute;
}
```

下拉菜单超出：

```text
container
```

的边界后，可能直接被裁剪。

于是又出现：

> “我的下拉菜单为什么显示不出来？”

实际上：

```text
不是 z-index 问题
不是 position 问题
而是 overflow 裁剪
```

因此：

```css
overflow: hidden;
```

虽然经常可以解决布局问题，但并不是万能答案。

---

# 十七、BFC 和层叠上下文千万不要混为一谈

这是最容易混淆的地方。

可以这样理解：

### BFC

关注：

```text
“怎么排？”
```

例如：

```text
这个元素多高？
margin 怎么处理？
float 怎么影响布局？
元素之间怎么排列？
```

---

### 层叠上下文

关注：

```text
“谁在上面？”
```

例如：

```text
A 和 B 谁覆盖谁？
z-index 为什么没效果？
为什么子元素 9999 还是被盖？
```

---

# 十八、一个非常典型的综合问题

例如页面：

```html
<div class="sidebar">
  <div class="menu"></div>
</div>

<div class="content"></div>
```

CSS：

```css
.sidebar {
  position: relative;
  z-index: 1;
}

.menu {
  position: absolute;
  z-index: 9999;
}

.content {
  position: relative;
  z-index: 2;
}
```

现在：

```text
menu
```

被：

```text
content
```

覆盖。

你可能会想：

```css
.menu {
  z-index: 999999999;
}
```

但是没有用。

原因是：

```text
sidebar
z-index: 1
```

和：

```text
content
z-index: 2
```

已经决定了两个层叠上下文之间的关系。

```text
sidebar
  └── menu
      z-index: 999999

content
  z-index: 2
```

实际上比较的是：

```text
sidebar: 1
content: 2
```

而不是：

```text
menu: 999999
content: 2
```

因此真正的解决方案可能是调整：

```css
.sidebar {
  z-index: 3;
}
```

而不是继续增加：

```css
.menu {
  z-index: 999999999;
}
```

---

# 十九、为什么有时候删除一个 transform，z-index 就正常了？

这是另一个很容易遇到的问题。

例如：

```css
.container {
  transform: translateZ(0);
}
```

然后内部：

```css
.modal {
  position: fixed;
  z-index: 9999;
}
```

结果发现：

```text
modal
```

表现异常。

原因之一就是：

```css
transform
```

会影响层叠上下文以及 `fixed` 定位的包含块行为。

所以开发过程中不要随便为了：

```text
GPU 加速
动画优化
```

就给各种父元素加：

```css
transform: translateZ(0);
```

因为它可能改变原本的 CSS 布局和层级关系。

---

# 二十、BFC 和层叠上下文的一个简单类比

可以把整个页面想象成一栋大楼。

## BFC = 房间

每个房间内部：

```text
桌子
椅子
柜子
```

按照自己的布局规则摆放。

房间之间存在一定隔离。

所以 BFC 主要解决：

```text
东西怎么摆
```

---

## 层叠上下文 = 楼层 / 图层

不同楼层之间存在明确的上下关系。

比如：

```text
3F
↑
2F
↑
1F
```

即使：

```text
1F 的某个东西 z-index = 999999
```

也不能直接理解成它可以跑到：

```text
3F
```

所以层叠上下文解决的是：

```text
谁覆盖谁
```

---

# 二十一、实际开发中的排查思路

以后再遇到“莫名其妙”的 CSS 问题，可以不要一上来就改 CSS。

先判断问题属于哪一类。

---

## 第一类：元素位置不对

例如：

```text
margin 不对
float 影响布局
父元素高度不对
元素被浮动影响
```

优先考虑：

```text
BFC
```

重点检查：

```text
float
margin collapse
overflow
display: flow-root
flex
grid
```

---

## 第二类：元素被覆盖

例如：

```text
z-index 不生效
弹窗被遮住
下拉菜单被盖住
tooltip 被覆盖
```

优先考虑：

```text
层叠上下文
```

重点检查：

```text
position
z-index
transform
opacity
filter
isolation
```

---

## 第三类：元素直接消失

例如：

```text
dropdown 不见了
tooltip 不见了
popover 不见了
```

不要只检查：

```css
z-index
```

还要检查：

```css
overflow: hidden;
```

因为它可能根本不是：

```text
“被别人盖住”
```

而是：

```text
“被父元素裁掉”
```

---

# 二十二、浏览器 DevTools 是非常重要的工具

遇到复杂 CSS 问题，不要只靠猜。

Chrome DevTools 中可以重点观察：

```text
Elements
Computed
Styles
Layout
```

尤其是检查：

```css
position
z-index
overflow
transform
opacity
display
float
margin
padding
```

对于层级问题，可以逐层检查父元素：

```text
当前元素
↓
父元素
↓
祖先元素
↓
更外层祖先
```

重点寻找有没有：

```css
z-index
transform
opacity
filter
isolation
```

等属性导致新的层叠上下文。

---

# 二十三、一个实用的 CSS 问题排查模型

可以把复杂 CSS 问题拆成三层。

```text
第一层：布局
        ↓
    BFC / Flex / Grid
        ↓
第二层：定位
        ↓
    position / containing block
        ↓
第三层：层叠
        ↓
    stacking context / z-index
```

例如一个弹窗：

```text
Modal
```

显示位置不对：

```text
先看 position
```

位置正确但是被盖住：

```text
再看 stacking context
```

完全看不到：

```text
再检查 overflow 裁剪
```

这样排查会比：

```text
z-index: 999999
z-index: 9999999
z-index: 99999999
```

有效得多。

---

# 二十四、现代 CSS 中，推荐优先使用明确的布局方式

过去经常看到：

```css
overflow: hidden;
```

解决各种布局问题。

但现代 CSS 已经有更明确的方式。

例如清除浮动：

```css
.parent {
  display: flow-root;
}
```

比：

```css
.parent {
  overflow: hidden;
}
```

语义更加明确。

如果是页面布局：

```css
.container {
  display: flex;
}
```

或者：

```css
.container {
  display: grid;
}
```

通常也比大量依赖：

```text
float
position
负 margin
```

更加清晰。

---

# 二十五、最后总结

CSS 中很多所谓的：

> “玄学问题”

其实并不是玄学。

只是我们没有看到浏览器背后的布局规则。

其中：

```text
BFC
```

可以重点理解为：

> **一个独立的块级布局环境。**

它主要影响：

```text
浮动
高度计算
margin 塌陷
布局隔离
```

而：

```text
层叠上下文
```

可以理解为：

> **一个独立的层级世界。**

它主要影响：

```text
z-index
元素覆盖
层级比较
```

最重要的几个认识可以浓缩成：

```text
布局问题
    ↓
先想 BFC

层级问题
    ↓
先想 stacking context

z-index 不生效
    ↓
不要只看 z-index 数值
    ↓
检查父元素的层叠上下文

overflow: hidden 解决问题
    ↓
很可能是创建了 BFC
    ↓
但同时要注意 overflow 的裁剪效果

子元素 margin 把父元素顶下去
    ↓
考虑 margin collapse

float 导致父元素高度塌陷
    ↓
考虑 BFC / display: flow-root
```

最终可以用一句话记住：

> **BFC 管“怎么布局”，层叠上下文管“谁在上面”。**

当你开始从这两个角度去分析 CSS 时，很多以前只能靠“不断改样式试出来”的问题，就会变成一个可以推导和解释的问题。
