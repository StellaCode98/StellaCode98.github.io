---
title: HTML 语义化总结：页面骨架、易混标签与无障碍实践
date: 2026-09-09 18:30:00
description: 语义化的本质是用标签表达「这是什么」而不是「这长什么样」，因为 HTML 除了浏览器还有两类读者——读屏软件和爬虫。本篇是技术总结：页面骨架标签、易混标签辨析（section/div、strong/b）、文本级语义速查、表单与 ARIA 的正确用法，最后给一张场景速查表。
categories:
  - [前端基础, html]
tags:
  - html
  - 语义化
  - 无障碍
  - SEO
  - 前端基础
---

语义化一句话：**用标签表达「这是什么」，而不是「这长什么样」。** 页面最终长什么样是 CSS 的事，但 HTML 还有一层身份——一份结构化文档，读者除了浏览器，还有读屏软件和搜索引擎爬虫。它们没有视觉，只能靠标签理解页面：`nav` 是导航、`h1` 是主题、`article` 是正文。用 `div` + class 描述一切，等于把这本书写成了「机器读不出来的图片」。

本篇是技术总结，直接进正题。

<!-- more -->

## 一、语义化解决什么

| 受益方 | 靠什么理解页面 | 语义化带来什么 |
| --- | --- | --- |
| 读屏软件用户 | 标签的角色（landmark、heading） | 可一键跳转导航/正文、标题可当目录朗读 |
| 搜索引擎爬虫 | 文档结构（h1、article、time） | 抓得住页面主题与内容权重 |
| 未来的维护者 | 结构本身 | 标签即文档，不用读 class 猜意图 |

反例长这样——人眼能看懂，机器什么都读不出来：

```html
<div class="top-header">
  <div class="nav-box">…</div>
</div>
<div class="main-wrap">
  <div class="big-title">为什么语义化重要</div>
</div>
```

## 二、页面骨架：结构级标签

一张典型的内容页结构：

```text
┌ <header> ── 页头：站名 / logo / 全局导航
│   └ <nav> ── 主要导航链接组
├ <main> ──── 主内容区（整页唯一）
│   ├ <article> ─ 独立成篇的内容
│   │   ├ <header> ─ 这篇文章的头（标题 / 时间）
│   │   ├ <section> ─ 有标题的主题分区
│   │   └ <footer> ─ 这篇文章的尾（作者 / 联系方式）
│   └ <aside> ── 与主内容间接相关：侧栏 / 相关阅读
└ <footer> ── 页脚：版权 / 备案 / 联系方式
```

对应的最小骨架：

```html
<body>
  <header>
    <h1>眉眼如初</h1>
    <nav aria-label="主导航">…</nav>
  </header>

  <main>
    <article>
      <header>
        <h2>HTML 语义化总结</h2>
        <time datetime="2026-09-09">9 月 9 日</time>
      </header>
      <section>
        <h3>第一节</h3>
        <p>…</p>
      </section>
      <footer>作者：<address>me@example.com</address></footer>
    </article>

    <aside>相关阅读…</aside>
  </main>

  <footer><small>© 2026 眉眼如初</small></footer>
</body>
```

三条硬规则：

1. **`main` 整页唯一**，且不能嵌在 `article` / `aside` / `nav` / `header` / `footer` 里；
2. **`header` / `footer` 不止页头页脚**：`article`、`section` 内部也可以有，表示「这一块的头和尾」；
3. **语义标签自带隐式 ARIA 角色**：顶层 `header` → `banner`、`footer` → `contentinfo`、`nav` → `navigation`、`main` → `main`——读屏软件据此生成「地标」，用户可以一键跳过导航直达正文（嵌套在 article 里的 header 则退化回普通节点，这正是「顶层」限定的含义）。

标题层级：`h1`–`h6` 不跳级、整页一个 `h1` 最稳。HTML5 曾允许每个 `section` 各配一个 `h1`，但那套 outline 算法已被规范移除，别再依赖。

## 三、易混标签辨析

| 问题 | 判据 | 结论 |
| --- | --- | --- |
| `section` 还是 `div` | 能不能给它起个像样的标题 | 有主题、配标题的分区用 `section`；纯样式/JS 挂钩的容器用 `div` |
| `article` 还是 `section` | 拿出来单独发（RSS、卡片）还成不成立 | 文章、评论、商品卡用 `article`；页面里的一章用 `section` |
| 链接组要不要包 `nav` | 用户会不会想「跳过 / 跳到」它 | 主导航、面包屑用；页脚的一串备案链接不用 |
| `aside` 是不是侧边栏 | 拿掉它正文是否依然完整 | 判据是「与主内容间接相关」，不是视觉位置——正文里插入的引申阅读框也是 `aside` |
| `strong` 还是 `b` | 是否要传达「重要」 | 重要/紧急/警告用 `strong`（读屏会加重语气）；只是视觉上吸引注意（关键词、引导句）用 `b` |
| `em` 还是 `i` | 强调是否改变句义 | 「我*就*是说…」这类改句义的强调用 `em`；术语、外语、内心戏用 `i` |

一句话版：**结构标签问「它是什么」，文本标签问「读者该怎么理解它」。**

## 四、文本级语义速查

| 标签 | 含义 | 典型场景 |
| --- | --- | --- |
| `code` / `pre` | 代码（行内 / 块级） | 行内片段；块级代码保持格式 |
| `abbr` | 缩写，`title` 给全称 | `<abbr title="Cascading Style Sheets">CSS</abbr>` |
| `cite` | 作品标题（**不是作者**） | 书名、影名、论文名 |
| `mark` | 当前语境中的「相关」高亮 | 搜索结果关键词标亮 |
| `del` / `ins` | 文档修订（删除 / 插入） | 价格变更、changelog |
| `s` | 不再准确但保留原文 | 原价划线（区别于 `del`：不是修订，是失效） |
| `small` | 附属细则 | 版权声明、免责条款 |
| `time` | 机器可读时间，值放 `datetime` | `<time datetime="2026-09-09">昨天</time>` |
| `figure` / `figcaption` | 被正文引用的独立单元及其说明 | 图表、截图、代码示例 |
| `address` | 最近 `article` / `body` 的作者联系方式 | 不是随便一个收货地址 |
| `dfn` / `var` / `kbd` / `samp` | 定义术语 / 变量 / 按键 / 程序输出 | 教程文档常用 |

## 五、表单：语义化的重灾区

表单是读屏用户受影响最大的区域，四条底线：

1. **每个输入框都有 `label`**，用 `for`/`id` 关联——点击 label 聚焦输入框，读屏才知道「这个框是干嘛的」；`placeholder` 对比度低、输入后消失，**不能替代 label**；
2. **`input type` 用对**：`email` / `url` / `tel` / `number` / `date` 会唤起对应的移动端键盘并带原生校验，白送的能力；
3. **相关选项用 `fieldset` + `legend` 分组**：单选组共用一个问题时，`legend` 就是那道题；
4. **`autocomplete` 写上 token**：`name` / `email` / `one-time-code`，浏览器自动填充对所有人都是效率，对读屏用户更是。

```html
<fieldset>
  <legend>订阅方式</legend>

  <label for="email">邮箱</label>
  <input id="email" type="email" autocomplete="email" required>

  <label><input type="radio" name="freq" value="w"> 每周</label>
  <label><input type="radio" name="freq" value="d"> 每天</label>
</fieldset>
```

## 六、ARIA：语义标签的兜底层

ARIA 第一法则：**能用原生语义标签，就不加 ARIA。** 原生标签自带键盘行为和焦点管理，ARIA 只改变读屏的「读法」，不补行为——这是假按钮最大的坑：

```html
<!-- 假按钮：读屏知道它是按钮了，但没有焦点、Enter/Space 不激活、不能提交表单，
     tabindex、keydown、样式全要手动补 -->
<div role="button" tabindex="0" onkeydown="handleKeys(event)">收藏</div>

<!-- 真按钮：以上全部原生自带 -->
<button type="button">收藏</button>
```

原生没有对应物时，ARIA 才登场，常用的就这几个：

- `aria-label`：无文字的图标按钮（按钮内只有 SVG）；
- `aria-expanded`：折叠菜单 / 弹层的展开状态；
- `aria-current="page"`：导航里的当前页高亮。

顺带 `img` 的 `alt` 三规则：信息图**描述内容**（不是「一张图片」）；装饰图 `alt=""`（空值表示跳过，不是省略属性）；在链接/按钮里的图**描述动作**。

## 七、常见误区

- **语义化 ≠ 消灭 `div`**：无语义容器正是 `div` 的本职，错的只是「用 div 表达一切」；
- **「长得一样」≠「语义一样」**：`b` 和 `strong` 默认都是粗体，但读屏读法不同；把 `h1` 样式改成 12px 不改变它是页面主题；
- **语义标签不自带样式**：`section` / `nav` 基本就是个 block，该写的 CSS 一行不少——语义化买的是含义，不是外观；
- **工作流上：先圈结构，再写样式**：拿到设计稿先标出 `header` / `main` / `aside` / `article`，再考虑 class 和样式，顺序反了必然 div 一把梭。

## 总结

场景速查：

| 要放的东西 | 用什么 |
| --- | --- |
| 页头 / 页脚 | `header` / `footer` |
| 导航链接组 | `nav` |
| 主内容区 | `main`（整页唯一） |
| 一篇独立内容 | `article` |
| 有标题的主题分区 | `section` |
| 纯样式 / 脚本容器 | `div` |
| 重要 / 强调的文字 | `strong` / `em` |
| 图 + 说明 | `figure` + `figcaption` |
| 时间 | `time[datetime]` |
| 图标按钮 | `button` + `aria-label` |
| 一组相关表单项 | `fieldset` + `legend` |

面试速答：

- **为什么需要语义化？** HTML 的读者除了浏览器还有读屏软件和爬虫；标签传达结构与重要性，决定它们如何理解和朗读页面，同时让人一眼看懂结构。
- **`section` 和 `div` 怎么选？** 有主题、能配上标题的分区用 `section`；只为样式或脚本挂钩的通用容器用 `div`。
- **`strong` 和 `b` 的区别？** `strong` 传达重要性（读屏加重语气），`b` 只是视觉上引起注意；`em`/`i` 同理。
- **ARIA 和语义标签的关系？** 原生优先——自带行为与焦点；ARIA 是没有原生对应物时的兜底，只改可读性不补行为。
- **`img` 的 `alt` 怎么写？** 信息图描述内容，装饰图写空 `alt=""`，链接/按钮内的图描述动作。

**一句话记住**：HTML 是写给三类读者的文档——人看结构、读屏读角色、爬虫抓重点；先选对标签，再谈样式，ARIA 只做兜底。

<!-- ## 参考

- [MDN：HTML 元素参考](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element)
- [HTML Living Standard](https://html.spec.whatwg.org/)
- [WAI-ARIA Authoring Practices Guide (APG)](https://www.w3.org/WAI/ARIA/apg/)
- [web.dev：Learn Accessibility](https://web.dev/learn/accessibility)
- [Using ARIA：First Rule of ARIA](https://www.w3.org/WAI/ARIA/apg/practices/read-me-first/) -->
