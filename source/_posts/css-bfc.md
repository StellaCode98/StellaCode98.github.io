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

写 CSS 时总会撞上一些「莫名其妙」的问题：

- 明明 `z-index: 9999`，元素还是被盖住；
- 子元素 `z-index` 再大，也压不过隔壁的父元素；
- 子元素设 `margin-top`，反而是父元素被推下去了；
- 子元素一 `float`，父元素高度直接归零；
- 给父元素随手加个 `overflow: hidden`，上面的问题居然全好了——然后下拉菜单又神秘消失。

这些问题表面毫无关联，底层都指向两个概念：**层叠上下文（Stacking Context）** 和 **BFC（Block Formatting Context）**。名字很像，职责完全不同——一个管层级，一个管布局：

| 概念 | 一句话 | 类比 | 管什么 |
| --- | --- | --- | --- |
| BFC | 独立的块级布局环境 | 房间：东西怎么摆 | 高度塌陷、margin 折叠、浮动环绕 |
| 层叠上下文 | 独立的图层世界 | 楼层：谁在上面 | z-index、覆盖关系 |

<!-- more -->

## 一、层叠上下文：z-index 不是全局排行榜

### 坑 1：z-index: 9999 还是被盖住

```html
<div class="box-a">
  <div class="child-a"></div>
</div>
<div class="box-b"></div>
```

```css
.box-a   { position: relative; z-index: 1; }
.child-a { position: relative; z-index: 9999; }
.box-b   { position: relative; z-index: 2; }
```

结果：`child-a` 的 9999 输给了 `box-b` 的 2。

误解在于把 z-index 当成「全局排行榜」。实际规则是：**z-index 的比较只发生在同一个层叠上下文内部；不同上下文之间，先由各自的父级上下文分胜负**：

```text
页面（根层叠上下文）
├── box-a  z-index: 1   ← 先比这一层：1 < 2，box-a 整组输掉
│   └── child-a z-index: 9999   ← 只在 box-a 内部有效
└── box-b  z-index: 2
```

`child-a` 的 9999 从来没有和 `box-b` 的 2 同台比过——先把 box-a 和 box-b 比出胜负，输的那一组整体压在下面，组内 z-index 再大也只是「矮子里的将军」。

所以排查 z-index 的第一问不是「数值够不够大」，而是：**这个元素属于哪个层叠上下文？** 一个典型场景——侧边栏里的弹层被内容区盖住：

```css
.sidebar { position: relative; z-index: 1; }  /* menu 的父级 */
.menu    { position: absolute; z-index: 9999; }
.content { position: relative; z-index: 2; }
```

正解是提 `.sidebar` 的 z-index，而不是继续给 `.menu` 加 9。

### 谁创建了层叠上下文

| 来源 | 例子 |
| --- | --- |
| 根元素 | `html` 自带最外层上下文 |
| 定位 + z-index 非 auto | `position: relative/absolute` + `z-index: 1` |
| fixed / sticky | `position: fixed` 无条件创建 |
| opacity < 1 | `opacity: .9`（等于 1 不会） |
| transform / filter / perspective | 值非 none 即创建 |
| flex / grid 子项 + z-index 非 auto | 最常被忽略的一类 |
| 显式声明 | `isolation: isolate`（推荐：无副作用地建上下文） |

### 坑 2：随手加个 transform，fixed 失灵

容器上有 `transform: translateZ(0)`，里面的 `position: fixed` 弹窗表现异常——因为 transform 除了创建层叠上下文，还会成为 fixed 后代元素的**包含块**，fixed 从此不再相对视口定位。别为了「GPU 加速」随手给祖先元素加 transform。

z-index 不生效的排查顺序：

```text
① 生效条件：是定位元素或 flex/grid 子项吗？
② 两个元素在同一个层叠上下文里吗？
③ 沿祖先链找上下文创建者：z-index / transform / opacity / filter / fixed
④ 该比的是两个父级上下文，不是这两个数字
```

## 二、BFC：一个独立的块级布局环境

BFC 的心智模型：页面里一个个「房间」，房间内部的浮动、margin、高度计算自成一统，与外部隔离。三条核心规则，对应三个经典场景：

### 坑 3：float 让父元素高度塌陷

```html
<div class="parent">
  <div class="child"></div>
</div>
```

```css
.parent { background: #eee; }
.child  { width: 200px; height: 100px; float: left; }  /* 脱离普通文档流 */
```

`float` 脱离文档流，父元素计算高度时看不见它 → `parent` 高度 ≈ 0。

修复：让父元素建立 BFC——**BFC 计算高度时，会把内部的浮动子元素也纳入**：

```css
.parent { display: flow-root; }  /* 首选：为建 BFC 而生，无副作用 */
/* overflow: hidden 也能解决，但有裁剪副作用，见坑 6 */
```

### 坑 4：子元素的 margin 把父元素顶下去

```css
.parent { background: #eee; }
.child  { margin-top: 50px; }  /* 视觉效果：parent 整体下沉，child 紧贴顶边 */
```

这是 **margin collapse（外边距折叠）**：父子之间没有 border、padding、行内内容或 BFC 隔离时，两者的垂直 margin 合并成一个，作用到了父元素外侧。修复还是给父级建 BFC（或加 border/padding 隔离）：

```css
.parent { display: flow-root; }
```

补全折叠的另一半：**相邻兄弟元素**的上/下 margin 也会合并（取大者）——这是设计行为不是 bug，想要每个间隙都生效，用 padding 或 BFC 隔开。

### 坑 5：后面的元素被 float 兄弟环绕

```css
.left    { width: 200px; height: 300px; float: left; }
.content { height: 300px; background: #eee; }  /* 普通块级：内容环绕浮动元素 */
```

BFC 的第三条规则：**BFC 的区域不与外部浮动重叠**。给 `.content` 建 BFC（`display: flow-root`），两者即并排分立——经典「自适应两栏布局」的原理正是这一条。

### 创建 BFC 的方式

| 方式 | 备注 |
| --- | --- |
| `display: flow-root` | 首选：专门为建 BFC 设计，无副作用 |
| `overflow` 非 `visible` | 传统做法，附带裁剪 / 滚动条 |
| `float` / `position: absolute` / `fixed` | 元素自身建立 BFC |
| `display: flex` / `grid` | 容器行为上等价 BFC（包含浮动、隔离 margin） |

### 坑 6：overflow: hidden 为什么「包治百病」又「暗藏杀机」

它会创建 BFC，于是高度塌陷、margin 折叠、浮动环绕一起消失——「诶，好了」。但它同时**裁剪溢出内容**：dropdown、tooltip、popover 超出父级边界的部分直接不见——这类「元素消失了」的问题，九成不是 z-index，而是被某个祖先的 overflow 裁掉了。原则：要 BFC 就写 `flow-root`，真的要裁剪才写 `overflow: hidden`。

## 三、排查模型：三层漏斗

```text
元素表现不对
  ├─ 位置 / 高度 / margin 不对  → 布局层：BFC / float / margin 折叠
  ├─ 位置对，但被盖住           → 层叠层：stacking context / z-index
  └─ 压根看不见                 → 裁剪层：祖先 overflow；定位层：包含块
```

配套的 DevTools 手法：选中元素后沿祖先链逐层向上看，重点找 `transform`、`opacity`、`filter`、`z-index`、`overflow`——层叠上下文和裁剪的元凶都在这条链上。

## 总结

症状速查：

| 症状 | 元凶 | 解法 |
| --- | --- | --- |
| z-index 再大也被盖 | 父级层叠上下文输了 | 提父级上下文的 z-index，或把弹层挪出该上下文 |
| fixed 定位失灵 | 祖先有 transform | 去掉 transform，或把元素挪到外层 |
| 父元素高度塌陷 | float 脱流 | 父级 `display: flow-root` |
| 子 margin 顶飞父元素 | margin 折叠 | 父级建 BFC，或 border/padding 隔离 |
| 元素被 float 兄弟环绕 | 非 BFC 与浮动重叠 | 自身建 BFC（自适应两栏的原理） |
| dropdown 消失 | 祖先 overflow 裁剪 | 改 overflow，或把弹层挂到 body |

面试速答：

- **BFC 是什么？** 块级格式化上下文，一个独立的块级布局环境：内部浮动参与高度计算、垂直 margin 不与外部折叠、区域不与外部浮动重叠。
- **怎么创建 BFC？** 首选 `display: flow-root`；`overflow` 非 visible、float、绝对定位、flex/grid 容器也可以。
- **层叠上下文的比较规则？** z-index 只在同一层叠上下文内比较；上下文之间由各自的 z-index 分胜负，子上下文整体跟随父级，无法突破。
- **为什么 `overflow: hidden` 能修高度塌陷？** 它创建了 BFC，而 BFC 计算高度时纳入浮动子元素——但注意它还会裁剪溢出内容。
- **z-index 不生效怎么排查？** 先确认生效条件（定位元素 / flex、grid 子项），再沿祖先链找上下文创建者（transform / opacity / filter / z-index），最后确认比较对象是否在同一层上下文。

**一句话记住**：BFC 管「怎么布局」，层叠上下文管「谁在上面」——前者是房间，后者是楼层；楼层的胜负在楼层之间决定，房间里的 z-index 再大，也出不了自家那层楼。

<!-- ## 参考

- [MDN：层叠上下文](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_positioned_layout/Stacking_context)
- [MDN：块格式化上下文](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_display/Block_formatting_context)
- [MDN：掌握外边距折叠](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_box_model/Mastering_margin_collapsing)
- [CSS 规范：CSS Positioning / CSS Display](https://www.w3.org/Style/CSS/specs) -->
