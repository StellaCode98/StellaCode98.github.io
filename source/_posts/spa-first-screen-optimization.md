---
title: SPA 首屏优化：减体积、抢网络、缩白屏
date: 2026-09-10 11:00:00
description: SPA 首屏慢是结构性问题：HTML 里只有一个空挂载点，一切内容都要等 JS 下载、解析、执行完才能出现。所以优化只有三个方向——减体积（路由懒加载、依赖分包与按需引入）、抢网络（gzip/brotli 压缩、CDN 与强缓存、preconnect/modulepreload 资源提示）、缩白屏（内联关键 CSS、骨架屏、SSR/预渲染）。文末给出量化指标、实践顺序与一张收益总结表。
categories:
  - [前端基础, 性能优化]
tags:
  - 性能优化
  - SPA
  - 首屏
---

SPA 首屏慢的根源是结构性的：HTML 里只有一个空的 `<div id="app">`，**一切内容都要等 JS 下载、解析、执行完才能出现**。所以优化逃不出三个方向：

> **减体积**——让首屏只下载首屏需要的代码；**抢网络**——让必须传的字节更快到达；**缩白屏**——让 JS 接管之前，用户就有东西可看。

先把时间线摆出来，慢在哪一段，就用哪一段的手段：

```text
导航开始 ─► TTFB ─► HTML 到达 ─► JS/CSS 下载 ─► 解析执行 ─► 首屏渲染 ─► 首屏接口返回 ─► 内容完整
             │                    │               │            │              │
          服务端慢？           体积/网络？        长任务？      接口串行？
          (后端/SSR)          (减体积/抢网络)    (拆任务)     (并行/预取)
```

<!-- more -->

## 一、先量化，再优化

不测量就优化，等于蒙眼开药。三个核心指标：

- **FCP**（First Contentful Paint）：首次画出任何内容的时间，直接反映白屏期长短；
- **LCP**（Largest Contentful Paint）：最大内容元素出现的时间，**低于 2.5s 算合格**，首屏优化的主目标；
- **INP**：交互响应延迟，替代了旧的 TTI，主要衡量「能不能用」而非「能不能看」。

工具就两个：**Lighthouse** 跑总分和机会清单，**Chrome Performance 面板**看瀑布图定位具体瓶颈。另外 Performance 面板里的 **Coverage** 标签值得专门一提——它能统计首屏 JS 里「下载了但没执行」的代码占比，这个数字通常是 60% 以上，这就是代码分割的空间。

线上监控用 `web-vitals` 库上报真实用户数据：

```js
import { onLCP, onFCP, onINP } from 'web-vitals';

onLCP(metric => report('LCP', metric.value));
onFCP(metric => report('FCP', metric.value));
onINP(metric => report('INP', metric.value));
```

## 二、减体积：让首屏只背首屏的债

### 路由级代码分割（收益最大）

SPA 最常见的问题是**全站代码打进一个 bundle**——用户只看首页，却下载了设置页、订单页、编辑器组件。路由懒加载让每个路由变成独立 chunk，按需加载：

```js
// React
import { lazy, Suspense } from 'react';

const Settings = lazy(() => import('./pages/Settings'));

// 用 Suspense 包住路由出口，加载期间显示 fallback
<Suspense fallback={<PageSkeleton />}>
  <Settings />
</Suspense>
```

```js
// Vue Router
const routes = [
  { path: '/settings', component: () => import('@/views/Settings.vue') },
];
```

注意理解它对首屏的意义：首页 chunk 本身没变小，而是**其他路由不再拖累首页**。首屏 JS 从「全站体积」降到「首页体积」，大项目里常常是 1MB 到 300KB 的差距。

### 依赖分包 + 长缓存

把稳定的第三方依赖拆成单独的 vendor chunk，配合文件名 contenthash，**业务代码天天发版，vendor 缓存常年有效**：

```js
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vue: ['vue', 'vue-router', 'pinia'],
          vendor: ['axios', 'dayjs'],
        },
      },
    },
  },
};
```

Webpack 对应 `splitChunks.cacheGroups`，思路相同。拆的粒度别太细——chunk 过多在 HTTP/1.1 下反而拖慢，几十 KB 一个比较合理。

### 替换与按需引入重型依赖

几个经典的重灾区：

- **moment → dayjs**：moment 加上全量 locale 能有几百 KB，dayjs 核心 2KB，API 几乎一致；
- **lodash**：`import debounce from 'lodash/debounce'` 按文件引入，或直接用 `lodash-es` 让 tree shaking 生效；
- **UI 组件库按需加载**：Element Plus 配 `unplugin-vue-components`，antd v5 已默认支持 tree shaking。

### 让 Tree Shaking 真正生效

两个前提，缺一个都摇不动：**依赖必须是 ESM 格式**（CJS 的 `require` 是运行时动态的，静态分析不了）；库的 `package.json` 要正确声明 `sideEffects`。判断方法很简单——构建产物里搜一个你从没 import 过的导出，居然在，就是没摇掉。

## 三、网络层：让字节更快到达

### 压缩：gzip 是底线，brotli 更好

文本资源不压缩就上线属于事故。gzip 能砍掉约 70% 体积，brotli 再省 15%~20%：

```nginx
gzip on;
gzip_comp_level 6;
gzip_min_length 1k;
gzip_types text/css application/javascript application/json image/svg+xml;
```

更好的做法是**构建时预压缩**（`vite-plugin-compression` 生成 `.gz` / `.br` 文件），Nginx 用 `gzip_static` 直接发文件，省掉运行时压缩的 CPU。

### CDN + 分域部署 + 缓存策略

静态资源上 CDN 是基本盘。缓存策略记住一条铁律：

```nginx
# 带 hash 的静态资源：缓存一年，永不协商
location ~* \.(js|css|woff2|png|jpg)$ {
  add_header Cache-Control "public, max-age=31536000, immutable";
}

# HTML 是资源的"索引"，必须每次校验，否则发版用户看不到
location = /index.html {
  add_header Cache-Control "no-cache";
}
```

逻辑闭环是：HTML 不缓存 → 每次拿到最新 hash → hash 变了才下载新 JS，没变走缓存。这样**二次访问的首屏几乎只剩 HTML 一个请求**。

### 资源提示：把等待提前变成准备

```html
<!-- 对关键域名提前完成 DNS + TCP + TLS 握手，能省几百毫秒 -->
<link rel="preconnect" href="https://cdn.example.com" crossorigin>
<link rel="preconnect" href="https://api.example.com">

<!-- 提前加载 ESM 入口及其 import 依赖，浏览器并行去取 -->
<link rel="modulepreload" href="/assets/index-a1b2c3.js">

<!-- 关键字体必须 preload，且 crossorigin 不能少，否则加载两次 -->
<link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
```

`dns-prefetch` 是 `preconnect` 的降级版（只做 DNS 解析），给不重要域名用。`prefetch`（空闲时预取下一页 chunk）慎用——抢占移动端带宽。

## 四、白屏期：让用户尽早"看见点什么"

前面解决的是「快」，这一节解决「体感」——时间没变，但用户从空白变成看到布局，焦虑感完全不同。

### 骨架屏：性价比之王

SPA 的白屏来自 `#app` 里什么都没有。最简单的方案：**直接把骨架屏写进 `index.html`**，JS 挂载完成后会自然替换掉它：

```html
<div id="app">
  <div class="skeleton">
    <div class="sk-nav"></div>
    <div class="sk-banner"></div>
    <div class="sk-content"></div>
  </div>
</div>
```

```css
.sk-banner {
  height: 200px;
  border-radius: 8px;
  background: linear-gradient(90deg, #f2f2f2 25%, #e8e8e8 37%, #f2f2f2 63%);
  background-size: 400% 100%;
  animation: shimmer 1.4s ease infinite;
}
@keyframes shimmer {
  0%   { background-position: 100% 0; }
  100% { background-position: -100% 0; }
}
```

### 内联关键 CSS

CSS 不加载完，浏览器不渲染任何东西（这是渲染阻塞）。把**首屏必须的样式**（布局、header、骨架屏）内联进 HTML，其余 CSS 异步加载：

```html
<style>/* 首屏关键样式，可由 critical 等工具构建时抽取 */</style>
<link rel="preload" as="style" href="/css/main-hash.css"
      onload="this.rel='stylesheet'">
```

### 首屏数据：别让接口排队

很多项目的时间线是「等 JS 加载完 → 执行 → 才发第一个接口」——数据请求被串行在了资源加载后面。解法按侵入程度排序：

1. **接口并行**：多个首屏请求用 `Promise.all` 同时发，别嵌套 await；
2. **数据随 HTML 下发**：服务端把首屏数据注入 `window.__INITIAL_STATE__`，JS 拿来就用，省一轮 RTT；
3. **SSR**：连渲染都在服务端做完，见下。

### SSR / SSG / 预渲染：终极方案

| 方案 | 原理 | 适用场景 | 成本 |
| --- | --- | --- | --- |
| 预渲染（prerender） | 构建时用无头浏览器把首屏 HTML 生成好 | 纯静态站、营销页 | 低 |
| SSR（Nuxt / Next） | 每次请求服务端渲染 HTML 再注水 | 动态内容、强 SEO 需求 | 高（服务器、双端逻辑） |
| SSG | 构建时生成全部页面 | 博客、文档站 | 中 |
| 骨架屏 | 不减少实际时间，改善体感 | 所有 SPA 的兜底 | 极低 |

SSR 把白屏问题连根拔掉，但也引入服务器成本、注水（hydration）开销和双端兼容问题——**先把前两节的做了，SSR 往往就不需要了**。

## 五、别忽略图片

图片常常占首屏传输的大头，而且优化全是声明式的：

- **格式**：WebP 起步，CDN 支持 AVIF 更好（同画质再小 20%）；
- **懒加载**：非首屏图片一行搞定 `<img loading="lazy" decoding="async">`；
- **防 CLS**：写死 `width` / `height`（或 CSS `aspect-ratio`），图片加载才不会把内容顶下去；
- **按需尺寸**：用 CDN 的裁剪参数按屏幕宽度取图，手机别下 2000px 的图，配 `srcset` 让浏览器自己选。

## 六、总结：一张表 + 实践顺序

| 手段 | 解决的问题 | 收益量级 | 成本 |
| --- | --- | --- | --- |
| 路由懒加载 + 依赖分包 | 首屏 JS 体积 | ⭐⭐⭐ | 半天，纯代码 |
| 替换重型依赖（moment 等） | 体积 | ⭐⭐ | 视替换范围 |
| gzip / brotli | 传输体积（约 -70%） | ⭐⭐⭐ | 一行配置 |
| CDN + 强缓存策略 | 传输 + 二次访问 | ⭐⭐⭐ | 需运维配合 |
| preconnect / modulepreload | 连接与下载串行等待 | ⭐⭐ | 几行 HTML |
| 骨架屏 + 内联关键 CSS | 白屏体感 | ⭐⭐ | 半天 |
| 首屏数据预取 / 并行 | 接口串行 | ⭐⭐ | 视接口情况 |
| SSR / 预渲染 | 白屏期（根治） | ⭐⭐⭐ | 重构级 |

推荐的实践顺序：**先量化（Lighthouse + Coverage 找到瓶颈）→ 代码分割 + 按需引入（不依赖任何人）→ 压缩 + CDN + 缓存头（拉上运维）→ 骨架屏 + 关键 CSS → 最后才考虑 SSR**。绝大多数项目做完前四步，LCP 就能从 4s+ 进 2.5s 以内——先把便宜的收益吃完，再动贵的刀。

---

最后一个调试技巧：在 DevTools 的 Network 面板把网速限到 Slow 3G，再看一次瀑布图——很多「明明做了优化还是慢」的问题，答案都藏在那条被拉长的时间线里。
