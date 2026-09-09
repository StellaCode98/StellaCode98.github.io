---
title: Webpack 性能优化速查：构建速度与产物体积，一张表讲完
date: 2026-09-09 22:30:00
description: Webpack 优化只围绕两件事——构建速度（开发体验）和产物体积（用户首屏）。不铺原理，只留结论：每条优化给「一句话为什么 + 最小配置」，按收益/成本排序，最后汇总成一张速查表。以 webpack5 为准，webpack4 差异单独标注。
categories:
  - [前端工程化, webpack]
tags:
  - Webpack
  - 性能优化
  - Tree Shaking
  - 代码分割
  - 构建优化
---

Webpack 的优化手段几十条，但目标只有两个：

```text
构建速度  → 优化的是开发体验（等多久能跑起来）
产物体积  → 优化的是用户首屏（下载多少 JS 才能看）
```

本文不铺原理，每条只留**结论 + 最小配置**，按「收益 / 成本」排序。以 webpack5 为准，webpack4 的差异随文标注。

---

## 0. 先测量，再优化

没有数据的一切优化都是猜。两件工具先装上：

```javascript
// 体积构成：谁把包撑大的
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer')

// 各阶段耗时：时间花在哪了
const SpeedMeasurePlugin = require('speed-measure-webpack-plugin')
```

90% 的项目分析完会发现：体积大头是 `node_modules` 里的重复依赖和没被 tree shaking 掉的组件库；耗时大头是 babel 对 `node_modules` 的无谓转译。下面的手段就是逐个干掉它们。

---

## 一、构建速度（开发体验）

### 1. 持久化缓存 —— webpack5 收益最大的一条

```javascript
// webpack.config.js
cache: { type: 'filesystem' }   // webpack5 默认开发环境已开启
```

二次构建从分钟级降到秒级。**webpack4 对应物**：`babel-loader` 的 `cacheDirectory: true`、`cache-loader`、以及已过时的 DllPlugin——webpack5 一个配置全部替代。

### 2. 别让 loader 碰 node_modules

```javascript
{
  test: /\.js$/,
  exclude: /node_modules/,        // 依赖通常已是编译产物
  include: path.resolve(__dirname, 'src')
}
```

`node_modules` 往往是业务代码的几十倍体积，babel/ESLint 多看它一眼都是浪费。

### 3. resolve 提效

```javascript
resolve: {
  extensions: ['.js', '.ts', '.jsx', '.tsx', '.json'],  // 每多一个，import 无后缀时多试一轮
  alias: { '@': path.resolve(__dirname, 'src') },
}
```

### 4. oneOf：命中即止

`module.rules` 默认每个文件把所有规则试一遍；包一层 `oneOf` 后命中一条就停：

```javascript
module: { rules: [{ oneOf: [ /* js / css / 图片规则 */ ] }] }
```

一个 `.js` 文件不该再被 css 规则白测一遍。

### 5. 换编译器：babel → esbuild / SWC

```javascript
{ test: /\.js$/, exclude: /node_modules/, loader: 'esbuild-loader', options: { target: 'es2015' } }
```

转译速度差一个数量级以上。项目越大收益越明显。

### 6. 其他两条

```text
devtool：开发环境用 'eval-cheap-module-source-map'，
        生产环境按需，别全量 source-map
多进程：thread-loader 只对转译量大 + 多核的项目有效，
        小项目进程通信开销反而更慢 —— 别无脑加
```

---

## 二、产物体积（用户首屏）

### 1. Tree Shaking 的三个前提

```text
① 源码用 ESM（import/export）—— CommonJS 摇不动
② package.json 标注副作用：
   "sideEffects": false           // 全部无副作用
   "sideEffects": ["*.css"]       // css 这类不能摇
③ production mode —— 自动开启 usedExports + 压缩删除
```

组件库引入后体积没变小，八成是这三条里缺了一条。

### 2. 路由懒加载 —— 体积优化里最立竿见影的一条

```javascript
const Home = () => import(/* webpackChunkName: "home" */ './views/Home.vue')
```

首屏只加载首屏的代码，其他页面按需。SPA 不做这一条，谈别的都是细节。

### 3. splitChunks 拆公共依赖

```javascript
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendors: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        priority: -10,
      },
    },
  },
}
```

业务代码天天改，vendor 单独成块才能吃到浏览器长效缓存（见第三节）。

### 4. 大而稳的库走 CDN（externals）

```javascript
externals: { vue: 'Vue', echarts: 'echarts' }   // 打包时排除，运行时读全局变量
```

权衡：externals 会**失去 tree shaking 和版本锁定**，只适合"确实用得多且稳定"的大库；偶尔用的库老老实实打包。

### 5. 传输压缩：gzip / brotli

```javascript
new CompressionPlugin({ algorithm: 'brotliCompress' })  // 构建时预压缩
```

```nginx
# nginx 直接吐预压缩文件，省掉实时压缩的 CPU
brotli_static on;
gzip_static on;
```

brotli 比 gzip 再小 15%~20%，两者可以并存做兜底。

### 6. 静态资源

```javascript
// webpack5 asset modules：file-loader/url-loader 已无需安装
{ test: /\.png$/,
  type: 'asset',
  parser: { dataUrlCondition: { maxSize: 4 * 1024 } } }  // 4KB 内联 base64
```

```text
小图标  → 内联或雪碧图 / iconfont
大图    → 压缩 + webp/avif，首图考虑懒加载
```

### 7. 依赖瘦身 —— 常被忽略的大头

```text
组件库按需引入（unplugin-vue-components / babel-plugin-import）
lodash        → lodash-es（配合 tree shaking）或按路径引入
moment（~70KB）→ dayjs（~2KB）
重复依赖      → npm ls <pkg> 查多版本，resolve.alias 收敛到一份
```

`BundleAnalyzerPlugin` 里看到同一个库出现两份，就是这条没做。

---

## 三、浏览器缓存与加载

### 1. contenthash + 长效缓存

```javascript
output: {
  filename: '[name].[contenthash:8].js',   // 内容变才变 hash
  clean: true,                              // 每次构建清旧文件
}
```

配合 nginx 对带 hash 的资源给 `Cache-Control: max-age=31536000, immutable`。

### 2. runtimeChunk：别让业务代码污染 vendor 的 hash

webpack 的运行时（模块加载器）默认混在入口 chunk 里，业务代码一改，runtime 变，vendor 引用跟着变，hash 全失效。抽出来：

```javascript
optimization: { runtimeChunk: 'single' }
```

改业务代码时，`vendors.js` 的 hash 纹丝不动，用户不用重新下载几百 KB 依赖——这是长效缓存真正生效的前提。

### 3. 资源提示：preload / prefetch

```javascript
import(/* webpackPreload: true */ './critical-module')   // 与主包并行，当前页就要用
import(/* webpackPrefetch: true */ './next-page')        // 浏览器空闲时偷偷加载下一页
```

一句话区分：**preload 给当前导航，prefetch 给下一步操作**。滥用 prefetch 会浪费用户流量。

### 4. 分包不是越细越好

HTTP/2 下请求数不再昂贵，但每个 chunk 有压缩下限和请求开销；几百个碎 chunk 不如适度聚合。`splitChunks.maxSize` 按需用，别为拆而拆。

---

## 四、速查表

| 目标 | 手段 | 关键配置 | 备注 |
| --- | --- | --- | --- |
| 构建 | 持久化缓存 | `cache: { type: 'filesystem' }` | webpack5 最大杀器，wp4 用 cache-loader/Dll |
| 构建 | 缩小 loader 范围 | `exclude / include` | 无成本，默认就该做 |
| 构建 | 命中即止 | `oneOf` | 无成本 |
| 构建 | 换编译器 | `esbuild-loader` / SWC | 大项目收益显著 |
| 体积 | Tree Shaking | ESM + `sideEffects` + production | 三个前提缺一不可 |
| 体积 | 路由懒加载 | `() => import()` | 立竿见影，必做 |
| 体积 | 拆 vendor | `splitChunks.cacheGroups` | 配合长效缓存 |
| 体积 | CDN 外置 | `externals` | 失去 tree shaking，慎用 |
| 体积 | 预压缩 | `CompressionPlugin` + `brotli_static` | 比 gzip 再小 15%+ |
| 体积 | 依赖瘦身 | 按需引入 / dayjs / 去重 | 最容易被忽略的大头 |
| 加载 | 长效缓存 | `contenthash` + `runtimeChunk: 'single'` | 两者配套才生效 |
| 加载 | 资源提示 | `webpackPreload/Prefetch` | preload 给当前页，prefetch 给下一页 |

**落地的顺序**：先上 analyzer 看数据 → 构建侧做缓存 + 范围（几乎零成本）→ 体积侧做懒加载 + tree shaking + 依赖瘦身 → 最后补 hash 缓存与资源提示。对绝大多数项目，做完这张表就够了，剩下的交给 CDN 和 HTTP/2。
