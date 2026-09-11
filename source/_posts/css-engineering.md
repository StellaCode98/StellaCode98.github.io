---
title: CSS 工程化演进：从命名地狱到原子化与原生回归
date: 2026-09-09 17:30:00
description: CSS 工程化不是堆工具，而是不断回应 CSS 的三个原生缺陷——全局作用域、级联失控、死代码。按命名规范→预处理器→CSS Modules→CSS-in-JS→原子化→原生新特性的演进链，拆解每个方案解决什么、留下什么，最后给一张选型地图。
categories:
  - [前端工程化, CSS]
tags:
  - CSS
  - 工程化
---

CSS 工程化的工具十几年代际更替，但它们都在回应同一件事：**CSS 没有模块系统。** 接手过老项目的人都见过 `.btn` 全站几十处引用、三层 `!important` 叠罗汉——不是谁的代码写得烂，是「所有样式共享一个全局命名空间」这个默认假设，在项目规模化之后必然崩塌。

把所有方案放进一个坐标系，整部演进史一目了然：

```text
CSS 的三个原生缺陷                 各时代的回应
──────────────────────────────────────────────────────
全局作用域（谁都能覆盖谁）     →   BEM / CSS Modules / scoped / @scope
级联失控（特异性军备竞赛）     →   ITCSS / @layer / :where()
死代码（删了组件不敢删样式）   →   原子化 CSS 的按需生成
```

<!-- more -->

## 一、问题的根源：CSS 生来没有「工程」这根筋

CSS 诞生于 1996 年，服务的是「文档」：几十个页面、一个作者。现代应用是几万个组件、几十个开发者——设计假设和现实差了三个数量级，于是有三个原生缺陷。

**全局作用域**：每条规则都注册进同一个命名空间，任何规则都可能命中任何元素。JS 2015 年就有了 ES Module，CSS 的「模块」至今仍靠社区方案模拟——这是整条演进史的主线。

**级联失控**：胜负由 `!important` > 特异性 > 源顺序三层博弈决定，而特异性没有上限，最后大家一起摆烂：

```css
.card .title { color: gray; }              /* 0-2-0 */
#app .title  { color: gray; }              /* 1-1-0，id 一出彻底失控 */
.card .title { color: red !important; }    /* 终局：全员 important */
```

**死代码**：CSS 与 DOM 是字符串匹配的松耦合——删了组件，没人敢确定样式是否还有别处在用；构建器也无法 tree-shaking（选择器命中谁只有运行时知道）。样式表体积只增不减。

一句话：**CSS 缺的不是功能，是模块系统——没有边界、没有依赖关系、没有可回收性。** 后面所有方案都是在给这三个洞打补丁。

## 二、靠纪律：命名规范（BEM）

思路很朴素：语言不给我作用域，就让命名承担唯一性。

```html
<article class="card">
  <h3 class="card__title">标题</h3>                            <!-- Element 用 __ -->
  <button class="card__btn card__btn--disabled">删除</button>  <!-- 状态用 -- -->
</article>
```

```css
.card                { padding: 16px; }                      /* Block */
.card__btn--disabled { opacity: .4; pointer-events: none; }  /* 永远只写单类选择器 */
```

两个机制层面的巧思：**扁平化特异性**（只写 0-1-0 的单类选择器，从机制上杜绝竞赛）、**命名即文档**（`card__btn--disabled` 不看上下文就知道来历）。OOCSS、SMACSS、ITCSS 同一思路，本质都是**没有任何强制力的约定**。

局限也就出在这儿：新人一句 `.card .card__btn`，特异性就涨了；跨项目复用还是复制粘贴。纪律方案的天花板是「团队永远不犯错」——这不是工程，是修行。

## 三、靠预处理：Sass——优化「写」，不改变「跑」

瞄准的是编写体验：没有变量、没有复用单元、几千行文件全靠人肉导航。

```scss
$primary: #1677ff;                        // 变量：设计值一处定义

.card {
  padding: 16px;
  &__title { color: $primary; }           // 嵌套 + &：与 BEM 天作之合
  &:hover  { box-shadow: 0 2px 8px rgba(0, 0, 0, .15); }
}

@use "components/button";                 // 文件级拆分，带命名空间引入
```

（另有 mixin 提供带参数的样式片段。）但要看清编译产物：嵌套、变量全部被摊平成普通的全局规则。**Sass 优化的是「编写时」的体验，完全不改变「运行时」的结果**——产物依旧是一张全局样式表，三大缺陷一个没解决；嵌套随手一写还是后代选择器，特异性反而悄悄上涨。它是舒适层，不是答案。

（后话：变量和嵌套如今都进了原生 CSS，见第七节——预处理器的增量价值正在归零。）

## 四、靠构建：CSS Modules——作用域终于有了

先定位 PostCSS：**CSS 界的 Babel**（解析成 AST → 插件转换 → 序列化回去），Autoprefixer、cssnano 都跑在它上面；CSS Modules 也构建在这层（webpack 的 css-loader、Vite 的 `.module.css` 文件名约定）。

CSS Modules 的核心思想：**语言层没有作用域，那就在构建层造一个——把每个文件的类名改写成全局唯一的名字。**

```css
/* Button.module.css */
.btn     { padding: 8px 16px; }
.primary { composes: btn; background: #1677ff; }  /* composes：组合而非复制 */
:global(.reset-all) { all: unset; }               /* 逃生舱：显式声明全局规则 */
```

```js
import styles from './Button.module.css';

// 类名被 hash 改写，import 拿到「原始名 → 改写名」的映射表：
// styles.primary => "Button_primary_3zK9 Button_btn_1xY2"
<button className={styles.primary}>确定</button>;
```

编译产物里 `.btn` 变成 `.Button_btn_1xY2`——两个文件各有一个 `.btn`，hash 后各管各的，**冲突在机制上不可能**；样式第一次有了 import 依赖关系，「删组件能不能删样式」变成可分析的问题；`composes` 用组合替代复制，比 Sass mixin 的摊平干净得多。

代价：类名变成黑话（调试靠 devtools 源码映射）、动态拼接要写 `styles[variant]`、CSS 与 JS 从松耦合变成强绑定。

## 五、靠 JS：CSS-in-JS 与 Vue scoped——样式跟着组件走

组件化时代的诛心之问：BEM 的「Block」和组件分明是一回事，为什么人肉维护两套对应关系？于是样式直接搬进组件。

**运行时派（styled-components）**：样式即组件，类名运行时生成（hash 串）注入 `<style>` 标签，样式拿到完整 JS 表达力（读 props、读主题：`background: ${props => …}`）、按组件注入零死代码。代价同样硬核：首屏要先执行 JS 才有样式（性能与 CLS 受拖累）、SSR 要服务端收集关键样式、与 RSC 不兼容——React 团队已明确不推荐新项目使用，此派退潮。

**零运行时派（vanilla-extract / linaria / PandaCSS）**：写法长得像 CSS-in-JS（TS 里写样式、传变量），但样式在**编译期**就被抽成静态 CSS 文件。思路完成回归：**表达力归 JS，交付物归 CSS。**

**Vue 的中庸答案（SFC scoped）**——给选择器加尾巴，给元素盖印章：

```text
源码                          编译产物
<style scoped>                .btn[data-v-7ba5bd90] { … }
  .btn { … }

<button class="btn">          <button class="btn" data-v-7ba5bd90>
```

和 CSS Modules 同代同思路（构建期作用域），只是隔离单位从「文件」变成「组件的 DOM 归属」。两个高频细节：

- **子组件的根元素会被同时打上父组件的 hash**——方便父级微调子组件根样式，也是「为什么子组件吃到了父组件样式」这个经典困惑的来源；
- **scoped 只隔离「出去」，不拦截「进来」**：全局样式照样命中你的组件，特异性竞赛依然存在——它是好的默认起点，不是完备的隔离方案。

动态样式交给 Vue 3.2+ 的 `v-bind()`，编译产物就是 CSS 变量（第七节）。

## 六、靠原子化：Tailwind / UnoCSS——不命名，就没有命名冲突

前几代方案有一个共同软肋：都需要「人为命名」，而冲突、死代码、不一致恰恰都从命名来。原子化掀了这个前提：**不命名**——每个类只做一件小事，类名即样式描述，由工具生成、全局唯一：

```html
<button class="px-4 py-2 rounded-md bg-blue-600 text-white hover:bg-blue-700">确定</button>
```

工程价值在三件事：

1. **JIT 按需生成，死 CSS 被机制性消灭**：只为源码中实际出现的类名字符串生成规则，源码里没有的类产物里就不存在——样式体积随「当页用到的原子类」收敛，而非随项目年龄增长；
2. **设计 token 内建为刻度**：`p-2`/`p-4` 来自间距刻度、`blue-600` 来自色板——没有「随手写 13px」的自由，一致性靠工具而不是靠 review 吼；
3. **与组件化天作之合**：HTML 住在组件里，样式写在 HTML 上，等于「删组件即删样式」，工程化追求的局部可推理性在这里闭环。

争议同样真实：类名串长、学习曲线。实践结论：原子类只写在**组件模板**里（组件本身就是封装边界），长类名串收敛进组件内部；散在页面级 HTML 上才是灾难。UnoCSS 是同思路的另一实现（preset 可组合）；Tailwind v4 底层已改用原生 `@layer` 保证 utilities 覆盖组件样式——正是第七节的主角。

## 七、原生反击：标准正在逐个收回工程化诉求

一条清晰的规律：**社区方案每验证一个需求，标准就吸收一个。**

**CSS 变量**——运行时的变量：可继承、可按选择器覆盖、可被 JS 读写，这是 Sass 变量没有的东西：

```css
:root { --primary: #1677ff; }
.btn { background: var(--primary); }
[data-theme="dark"] { --primary: #4096ff; }  /* 切主题 = 换一组变量值 */
```

**@layer**——把优先级从「特异性竞赛」变成「层声明顺序」：层序即胜负，与层内选择器写法无关。ITCSS 讲了十年的分层第一次成为语言机制：

```css
@layer reset, base, components, utilities;   /* 后声明的层胜出 */

@layer components { #app .btn { background: gray; } }    /* 1-1-0 也没用 */
@layer utilities  { .bg-blue { background: #1677ff; } }  /* 单类照样赢 */
```

**@scope**——原生作用域，还带一个「洞」（donut scoping）。「组件嵌套同款组件导致选择器越界」这个 BEM 时代的老大难，标准直接给了语法（2024 年起三大内核全绿）：

```css
@scope (.card) to (.card__content) {
  img { border-radius: 8px; }  /* 只命中 .card 内、.card__content 之外的 img */
}
```

**原生嵌套与 :where()**——Sass 的两大卖点进了标准（2023 年起主流全绿）；`:where()` 特异性归零，默认样式不怕被业务样式覆盖不了。

判断：2026 年的新项目，「CSS 变量 + `@layer` 分层 + 原生嵌套 + 一套 utility 约定」已能覆盖过去一整套工具链的大部分诉求；工具链的剩余价值收敛到两件事——**token 的约束力**（Tailwind）和**构建期作用域**（Modules/scoped），而后者 `@scope` 正在路上。

## 八、选型地图

| 方案 | 解决的核心问题 | 代价 / 风险 | 适用场景 |
| --- | --- | --- | --- |
| 命名规范（BEM/ITCSS） | 命名冲突（靠约定） | 无强制力，全靠人遵守 | 无构建链路的老项目、静态站 |
| 预处理器（Sass/Less） | 编写效率（变量/复用） | 产物仍是全局 CSS | 任何时代的舒适层，不再是关键决策 |
| CSS Modules | 作用域（构建期 hash） | 类名黑话、与 JS 强绑定 | React 生态、中大型应用 |
| Vue scoped | 作用域（组件 DOM 归属） | 只隔离「出去」，不拦截「进来」 | Vue SFC 的默认答案 |
| CSS-in-JS | 作用域 + 动态样式全表达力 | 运行时成本、SSR、RSC 摩擦 | 强主题组件库，优先零运行时实现 |
| 原子化（Tailwind/UnoCSS） | 死代码 + 一致性 token | 类名串、学习曲线 | 组件化项目、追求交付效率的团队 |
| 原生新特性 | 作用域 + 优先级 + 变量 | 需要较新的浏览器基线 | 新项目的默认基座 |

几条决策直觉：

- **先看框架再选方案**：Vue 默认 scoped（复杂设计系统再叠 UnoCSS）；React 在 CSS Modules 和 Tailwind 之间二选一基本不会错；做组件库才需要认真评估零运行时 CSS-in-JS；
- **团队越大越选「约束型」方案**：刻度、分层都是用机制换一致性——工程化的意义是让一致性不依赖个人自觉；
- **存量项目别推倒重来**：用 `@layer` 把旧样式垫在底层、新代码写新层，两套体系共存且优先级明确，按页面粒度推进；
- **要硬隔离只有 Shadow DOM / iframe**：微前端、嵌入第三方页面的组件（客服气泡、支付控件），类名 hash 不够——Shadow DOM 内外样式互相不可见（可继承属性除外），才是物理边界；
- **比工具更根本的是设计 token**：颜色、间距、字号的刻度统一了，用什么工具都乱不到哪去；刻度没统一，什么工具都救不回来。

## 总结

演进一览，也是全文的坐标系：

| 阶段 | 代表 | 靠什么 | 解决了 | 留下了 |
| --- | --- | --- | --- | --- |
| 纪律 | BEM / ITCSS | 人 | 命名冲突 | 无强制力 |
| 预处理 | Sass | 编译器 | 编写效率 | 产物仍全局 |
| 构建 | CSS Modules / scoped | 构建器 | 作用域 | 类名黑话 |
| 运行时 | styled-components | JS | 作用域 + 动态 | 性能 / SSR |
| 原子化 | Tailwind / UnoCSS | 生成器 | 死代码 + 一致性 | 可读性争议 |
| 原生 | @layer / @scope | 标准 | 以上全部（渐进） | 浏览器基线 |

面试速答：

- **CSS 工程化到底在解决什么？** CSS 没有模块系统，三个原生缺陷：全局作用域、级联失控、死代码。所有方案都是对这三个问题的不同回应。
- **CSS Modules 的原理？** 构建期把类名改写为「文件路径 + hash」的全局唯一名，JS import 得到映射表；冲突在机制上不可能，composes 提供组合能力。
- **Vue scoped 的原理和局限？** 编译期给组件内元素和选择器附加同 hash 的 `data-v-xxx` 属性；只隔离「出去」不拦截「进来」，子组件根元素会额外带父组件的 hash。
- **@layer 解决什么？** 把优先级从特异性竞赛变成层声明顺序——层序即胜负，与层内选择器写法无关。
- **为什么 Tailwind 没有死 CSS？** JIT 只为源码中实际出现的类名字符串生成规则，没出现就不生成。

**一句话记住**：CSS 工程化的每一步，都是在给一门没有模块系统的语言补模块系统——先靠人（命名规范），再靠编译（Modules/scoped），最后标准自己来（@layer/@scope）；具体工具会退潮，但「作用域、级联、死代码」这三个原生缺陷，是理解一切 CSS 工程化方案的坐标系。

<!-- ## 参考

- [MDN：级联层 @layer](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@layer)
- [MDN：@scope](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@scope)
- [MDN：使用 CSS 自定义属性](https://developer.mozilla.org/zh-CN/docs/Web/CSS/Using_CSS_custom_properties)
- [MDN：CSS 嵌套](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_nesting)
- [CSS Modules 规范](https://github.com/css-modules/css-modules)
- [Vue 文档：SFC CSS 功能（scoped / v-bind / CSS Modules）](https://cn.vuejs.org/api/sfc-css-features.html)
- [Tailwind CSS：Core Concepts](https://tailwindcss.com/docs/utility-first)
- [UnoCSS 文档](https://unocss.dev/) -->
