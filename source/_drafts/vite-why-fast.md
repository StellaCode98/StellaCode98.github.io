---
title: Vite 为什么快：esbuild、按需编译原理与工程落地
date: 2026-09-09 10:00:00
categories:
  - [前端工程化]
tags:
  - vite
  - esbuild
  - 构建工具
description: 「快」只是结果。拆开 Vite 的 dev 与 build 两条链路，讲清它和 Webpack 的本质差异，以及迁移时要付出什么代价。
---

> 本篇目标：讲透 Vite 的核心机制，评估自己项目迁移的收益与成本。这是「前端工程化」分类第 1 篇。

## 一、Webpack 慢在哪：bundle 模式的先天约束

- dev 启动 = 完整打包：模块图遍历、transform、bundle 生成
- HMR 随项目体积线性劣化的原因

## 二、Vite dev 的两板斧

- **不打包**：浏览器原生 ESM，请求时按需 transform（配图：请求瀑布 → 模块图谱）
- **esbuild 预构建**：依赖预构建做了什么（CJS→ESM、合并请求），为什么用 esbuild 而不是 rollup（Go 单线程 vs JS）
- 源码缓存与浏览器缓存的强协商策略（`?v=xxx`）

## 三、Vite build：为什么又回到 Rollup

- 生产环境不用原生 ESM 的原因（瀑布请求、兼容性、优化空间）
- Rollup 的 tree shaking 与代码分割优势

## 四、与 Webpack 的差异对比表

- 启动速度 / HMR 粒度 / 生态（loader vs plugin）/ 配置复杂度 / 产物质量
- 常见迁移坑：CJS 依赖、环境变量注入、SSR 特殊处理

## 五、落地评估模板（实战占位）

- 用自己项目实测：冷启动时间、HMR 时间 before/after
- 迁移 checklist：依赖扫描 → 配置映射 → 产物 diff → 灰度上线

## 六、总结

- 一句话记住：**dev 用原生 ESM 换时间，build 用 Rollup 换体积；两头下注是架构的核心智慧**

## 参考

- Vite 官方文档：Why Vite / Dep Pre-Bundling
- esbuild FAQ（为什么快）
