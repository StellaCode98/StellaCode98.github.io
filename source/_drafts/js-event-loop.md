---
title: 把事件循环讲透：宏任务、微任务与渲染时机
date: 2026-09-09 10:00:00
categories:
  - [前端基础, JavaScript]
tags:
  - javascript
  - 浏览器原理
  - 事件循环
description: 面试高频、实战常踩：用一张图和几段代码把宏任务、微任务与渲染的调度顺序彻底讲清。
---

> 本篇目标：读完能不看资料画出事件循环流程图，并解释 setTimeout/Promise/requestAnimationFrame 的执行顺序差异。这是「JS 原理系列」第 1 篇。

## 一、从一个面试题开始

- 一段混合 setTimeout / Promise / async-await 的执行顺序题
- 80% 的人答错的原因：把「任务队列」理解成了一条

## 二、调用栈与任务队列

- 单线程与调用栈：阻塞是怎么发生的
- 宏任务（script / timer / IO）与微任务（Promise / queueMicrotask / MutationObserver）

## 三、完整循环流程

- 一轮循环的步骤：执行同步代码 → 清空微任务 →（渲染时机）→ 取下一个宏任务
- 关键细节：微任务在**每个宏任务后**清空，而不是每轮循环结束
- 配图：事件循环流程图（用 mermaid 画）

## 四、渲染时机：requestAnimationFrame 在哪

- 渲染不发生在每个宏任务之后：rAF → style → layout → paint → rIC
- `setTimeout(fn, 0)` 为什么不适合做动画
- 用 Performance 面板验证一遍（配截图）

## 五、实战验证

- 5 段代码题：每段先预测再运行，配逐行解释
- 场景案例：微任务造成的「渲染前批量更新」与 Vue nextTick 的关系（埋个伏笔，Vue 系列展开）

## 六、总结

- 一句话记住：**同步先走、微任务清空、然后才轮到下一个宏任务；渲染插在两者之间**

## 参考

- HTML Spec: Event loops
- Jake Archibald: In The Loop (JSConf 演讲)
- MDN: queueMicrotask
