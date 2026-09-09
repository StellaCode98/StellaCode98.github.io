---
title: 被低估的 HTML：语义化、可访问性与那些你没用的标签
date: 2026-09-09 10:00:00
categories:
  - [前端基础, HTML]
tags:
  - html
  - 语义化
  - 无障碍
description: 框架写多了，HTML 只剩 div 和 span。这篇复盘语义化标签的价值，以及那些被遗忘但好用的元素。
---

> 本篇目标：把「写页面」的习惯从 div 堆砌拉回语义化，并能在团队里推动一份最低限度的 HTML 规范。适合结合自己项目里的真实代码截图举例。

## 一、为什么现在还在聊语义化

- SEO、可访问性、代码可维护性三条收益线
- 反面案例：一个全是 div 的页面在读屏器里的样子（配演示）

## 二、结构语义的最低配置

- `header / nav / main / article / section / aside / footer` 的使用边界
- 常见误用：section 当盒子、h 标题层级跳跃

## 三、那些被遗忘但好用的标签

- `dialog`：原生模态框（含 `showModal` 焦点管理与 `::backdrop`）
- `details / summary`：免 JS 折叠面板
- `picture / source`：响应式图片与格式降级
- `datalist / output / progress`：表单增强
- 每个标签给一个「替换掉手写组件」的对比 demo

## 四、可访问性速查

- 焦点管理、`aria-*` 的使用与滥用
- 键盘可达性 checklist（tab 顺序、esc 关闭、focus trap）

## 五、总结

- 一句话记住：**优先用浏览器已经实现的能力，div 是最后的选择**
- 附：团队 HTML 规范 checklist（可直接抄走）

## 参考

- MDN: HTML elements reference
- web.dev: Learn HTML / Accessibility
