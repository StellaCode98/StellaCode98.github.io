---
title: TypeScript 进阶地图：从「会用」到「会设计类型」
date: 2026-09-09 10:00:00
categories:
  - [前端基础, TypeScript]
tags:
  - typescript
  - 类型系统
description: 会写 interface 和泛型只是起点。这篇梳理 TS 的能力地图，并用一个类型安全的请求封装串起所有进阶特性。
---

> 本篇目标：给「TS 用了几年但停留在 interface + 泛型」的自己画一张能力地图，通过一个贯穿全文的实战案例把进阶特性用起来。

## 一、自测：你在 TS 的哪一层

- L1 会标注 / L2 会泛型 / L3 会推导与约束 / L4 会设计公共类型 API
- 大多数业务代码停在 L2 的原因：类型只是「注释」，不是「约束」

## 二、类型系统核心概念串讲

- 结构化类型：为什么 TS 是「鸭子类型」
- 协变/逆变在函数参数上的表现（用一组合法/非法赋值演示）
- `unknown / any / never` 的正确分工

## 三、进阶工具箱

- 泛型约束（extends）、默认泛型参数
- 条件类型 + `infer`：从现有类型里「挖」出新类型
- `keyof / typeof / mapped types / template literal types` 各一个实战例子
- 内置工具类型的实现原理（`Pick / Partial / ReturnType` 手写）

## 四、贯穿案例：一个类型安全的请求封装

- 需求：接口返回类型自动推导、错误码收窄、路径参数校验
- 逐版演进：v1 any 满天飞 → v2 手动标注 → v3 泛型 + 条件类型自动推导
- 最终版代码全文 + 每个类型设计的理由注释

## 五、工程建议

- `strict` 全开的真实代价与收益
- 什么时候允许 any：团队规范的三条例外

## 六、总结

- 一句话记住：**类型是给调用方的 API，设计类型 = 设计使用体验**

## 参考

- TypeScript Handbook
- type-challenges（按 easy → medium 刷前 20 题）
