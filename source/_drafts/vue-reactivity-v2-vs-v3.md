---
title: Vue2 与 Vue3 响应式原理对比：从 defineProperty 到 Proxy
date: 2026-09-09 10:00:00
categories:
  - [Vue 进阶]
tags:
  - vue2
  - vue3
  - 响应式
  - 源码
description: 数组下标为什么监听不到、新增属性为什么要 $set——所有 Vue 的「反直觉 API」都能从响应式实现里找到答案。
---

> 本篇目标：手写两版迷你响应式（defineProperty 版与 Proxy 版），讲清 Vue 响应式的演进动机。这是「Vue 系列」第 1 篇，后续 diff、编译优化都会建立在这篇之上。

## 一、从两个「反直觉」问题切入

- `this.obj.newKey = 1` 视图不更新，为什么要 `this.$set`
- `arr[0] = x` 不触发更新，`arr.length = 0` 也不行

## 二、Vue2：Object.defineProperty 的能力与局限

- getter/setter 收集依赖、触发更新的核心流程（配图：Dep / Watcher 关系）
- 局限逐条对应源码的绕行方案：数组方法重写、$set、深层递归初始化的性能代价
- 手写 mini-reactivity（v1，约 60 行）

## 三、Vue3：Proxy 为什么是更优解

- Proxy 拦截的 13 种操作，`set/has/deleteProperty` 补齐了哪些洞
- 惰性代理：嵌套对象用到才代理，初始化性能对比数据
- WeakMap 的三层结构（target → key → effect）为什么不会内存泄漏
- 手写 mini-reactivity（v2，用 effect / track / trigger 三件套）

## 四、细节对比与迁移影响

- 表格：能力 / 性能 / 兼容性（Proxy 不可 polyfill → Vue3 放弃 IE）
- 响应式 API 的分层：reactive / ref / computed 的设计动机

## 五、总结

- 一句话记住：**Vue2 的坑不是 bug，是 defineProperty 的天花板；Vue3 换 Proxy 是「换了地基」**

## 参考

- Vue2 源码 `core/observer` 目录
- Vue3 源码 `reactivity` 包（可独立阅读）
- 尤雨溪相关分享：Vue3 设计过程
