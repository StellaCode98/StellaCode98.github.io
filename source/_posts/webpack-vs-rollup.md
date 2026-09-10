---
title: Webpack 与 Rollup：一个为应用而生，一个为库而生
date: 2026-09-10 14:00:00
description: 两者都是打包器，但回答的问题不同：Webpack 面向「应用」——万物皆模块 + loader 流水线 + HMR，产物自带运行时和模块注册表；Rollup 面向「库」——原生 ESM 静态分析，scope hoisting 把模块摊平成一份接近手写的代码，多格式产物让下游还能继续摇树。本文用打包产物对比讲清本质区别，逐项分析 tree shaking、代码分割、生态扩展、产物格式，最后给出选型口诀与 Vite/Rspack 时代的新答案。
categories:
  - [前端基础, 工程化]
tags:
  - 构建工具
  - Webpack
  - Rollup
  - 工程化
---

这两个工具经常被放在一起比，但它们出生时回答的是**两个不同的问题**：

> **Webpack（2012）：怎么把一个「应用」跑进浏览器？** 于是万物皆模块（JS/CSS/图片/字体），loader 负责转换，dev-server + HMR 负责开发体验，产物自带运行时。
> **Rollup（2015）：怎么把一个「库」发布得干净？** 于是原生 ESM 静态分析，产物几乎没有运行时，多格式输出（ESM/CJS/UMD），下游打包器还能继续 tree shaking。

选型口诀可以先记住：**写应用用 Webpack，发库用 Rollup**。下面解释为什么。

```text
Webpack 产物 = 运行时(__webpack_require__) + 模块注册表(每个模块包一层函数)
Rollup 产物 = 所有模块摊平进同一个作用域，看起来像一份手写的代码
```

<!-- more -->

## 一、出身决定性格

Webpack 诞生于 CommonJS 时代，核心洞见是「**任何资源都是模块**」+「**动态依赖可以切分出按需加载的代码块**」。它的战场是应用开发：入口一堆、资源一堆、要热更新、要按路由分包。

Rollup 由 Rich Harris（Svelte 的作者）在 2015 年创建，比 Webpack 晚三年，正好踩在 ES Module 标准落地之后。它的洞见是：**ESM 是静态的——import/export 在编译期就能确定，不需要运行时**。于是它可以做到 Webpack 当时做不到的事：把模块直接摊平合并，产出干净得像手写的代码。tree shaking 这个词就是随 Rollup 火起来的。

后来的事实也印证了分工：React、Vue 3、D3 的构建产物都出自 Rollup；而 create-react-app、Vue CLI 时代的应用脚手架清一色是 Webpack。

## 二、产物对比：一眼看懂本质区别

同样的源码，看两家的产物（Webpack 产物做了简化，保留结构）：

```js
// ---------- 源码 ----------
// utils.js
export function a() { return 1; }
export function b() { return 2; }

// main.js
import { a } from './utils.js';
console.log(a());
```

```js
// ---------- Webpack 产物（简化） ----------
(() => { // webpackBootstrap 运行时
  const modules = {
    './src/utils.js': (module, exports) => {
      exports.a = () => 1;
      exports.b = () => 2;
    },
    './src/main.js': (module, exports, __webpack_require__) => {
      const { a } = __webpack_require__('./src/utils.js');
      console.log(a());
    },
  };
  // ...模块缓存、require 实现、执行入口
})();
```

```js
// ---------- Rollup 产物 ----------
function a() { return 1; }

console.log(a());
```

差异一目了然：

- **Webpack 给每个模块包一层函数**，用字符串 id 索引成一张注册表，再配一个运行时（模块缓存、require 实现、异步加载逻辑）负责按需执行——这是为了让**任何模块格式（包括 CommonJS、AMD）在任何环境**都能跑，代价是体积和一层间接性。
- **Rollup 把所有模块摊平到同一个作用域**（scope hoisting，作用域提升），没有模块边界、没有运行时。`b` 这种没用的导出直接被摇掉了。

一个比喻：Webpack 的产物像**带发动机的整车**，开箱能跑但多了一堆底盘；Rollup 的产物像**精密加工过的零件**，本身几乎零冗余。

## 三、Tree Shaking：都能摇，干净程度不同

Webpack 2（2016）之后就支持 tree shaking，但它是「半路出家」——必须先把 CommonJS 转成兼容格式，静态分析天然打折。实践中有两个前提，缺一个就摇不动：

- 依赖必须是 **ESM**（CJS 的 `require` 是运行时动态调用，无法静态分析）；
- 库要在 `package.json` 里正确声明 **`sideEffects`**，否则 Webpack 不敢删「看起来没用到」的文件（万一有 `polyfill` 这类副作用呢）。

Rollup 从第一天就围绕 ESM 静态分析设计，配合 scope hoisting，摇得更快更彻底，也不需要 `sideEffects` 这类补救字段。同样的库源码，Rollup 的 ESM 产物通常更小。

顺带一提，Webpack 4 之后也做了 scope hoisting（`ModuleConcatenationPlugin`，生产模式默认开启），差距在缩小——但「以 CJS 兼容为底座」和「以 ESM 纯净为底座」的出身差异，决定了它永远要多处理一层兼容逻辑。

## 四、代码分割：Webpack 的主场

应用开发的核心需求是**按路由/按需分包**，这正是 Webpack 的看家本领：

- 多入口、`import()` 动态导入、`splitChunks` 公共依赖抽取，策略丰富且成熟；
- 和 HMR、懒加载失败重试、preload/prefetch 魔法注释等应用级能力配套完整。

Rollup 也支持代码分割（多入口 + 动态导入会生成共享 chunk），但它的分割策略是**面向多入口的库**设计的——比如同时打包出 `core` 和 `full` 两个入口，共享部分自动提取。拿它做复杂应用的分包（按路由切、vendor 策略、runtime chunk），配置别扭，优化空间也远不如 Webpack。Rollup 文档自己都说：它不是主要为应用构建设计的。

开发体验同理：webpack-dev-server 的 HMR（改代码不丢状态、页面不整刷）是应用开发刚需；Rollup 只有 watch 重新打包，没有 HMR——因为**库开发根本不需要跑一个完整应用，写个 demo 页用 ESM 直接 import 源码就够了**。

## 五、生态与扩展模型：全能 vs 纯净

**Webpack 的扩展是「双轨制」**：

```js
// loader 管「转换」：让 webpack 认识非 JS 资源
module: {
  rules: [
    { test: /\.css$/, use: ['style-loader', 'css-loader'] },
    { test: /\.(png|woff2)$/, type: 'asset/resource' },
  ],
},
// plugin 管「介入构建生命周期」：基于 Tapable 的事件钩子
plugins: [new HtmlWebpackPlugin()],
```

loader 让 CSS、图片、字体都成为一等公民，plugin 钩子覆盖构建的每个阶段——这套体系庞大到「没有 webpack 做不到的事，只有你没配对的 loader」。

**Rollup 只有一种插件接口**（`load`/`transform`/`resolveId` 等钩子），设计干净得多，但能力边界也明显：资源处理要自己拼插件；最经典的痛点是**依赖里全是 CommonJS 时，必须配 `@rollup/plugin-commonjs` 做转换**，遇到写法刁钻的 CJS 依赖还可能踩坑——毕竟把不纯净的东西洗纯净，本来就是它的逆命题。

## 六、多格式产物：发库用 Rollup 的根本原因

一个库的用户可能用 ESM import、可能用 CJS require、可能直接 `<script>` 引入。Rollup 一份配置输出全家桶：

```js
// rollup.config.js
export default {
  input: 'src/index.js',
  output: [
    { file: 'dist/index.esm.js', format: 'es' },   // 给打包器：可继续 tree shaking
    { file: 'dist/index.cjs',    format: 'cjs' },  // 给 Node.js
    { file: 'dist/index.umd.js', format: 'umd', name: 'MyLib' }, // 给 <script>
  ],
};
```

```json
{
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.cjs"
    }
  }
}
```

关键洞察是：**库的产物会被下游的打包器再构建一次**。ESM 产物进了用户的 bundle 后还能被继续摇树；而 Webpack 产物里那层 `__webpack_require__` 包装会阻止下游分析，等于把树「焊死」了——库体积有多大，用户就得全量背多重。这就是 React、Vue 都选 Rollup 的根本原因：**库的产物不是终点，而是下游构建的原料**。

（Webpack 也能通过 `output.library.type` 输出 umd 等格式，能做，但不优雅。）

## 七、选型决策 + 新时代的变化

| 维度 | Webpack | Rollup |
| --- | --- | --- |
| 设计目标 | 应用打包 | 库打包 |
| 模块机制 | 运行时 + 模块函数包装 | ESM 静态合并（scope hoisting） |
| 产物体积 | 较大（含运行时） | 接近手写，几乎零冗余 |
| Tree shaking | 支持，依赖配置与依赖格式 | 更彻底 |
| 代码分割 | 强，策略成熟 | 较弱，面向多入口库 |
| 资源处理 | loader 全家桶，一等公民 | 需插件拼装 |
| HMR | 内置（dev-server） | 无，只有 watch |
| 产物格式 | 以浏览器 bundle 为主 | es / cjs / umd / iife 多格式 |
| CJS 依赖 | 原生丝滑 | 需 commonjs 插件，有坑 |
| 适用场景 | 复杂应用 | 组件库 / 工具库 |

最后是 2026 年的补充答案——这个问题的「现代默认选项」其实是 **Vite**：开发态用 esbuild 预打包依赖 + 浏览器原生 ESM 按需编译，生产构建的内核恰恰就是 Rollup。所以新项目很少直接手写 Webpack 配置了，但底层还是这两个工具的理念在分工：**Vite 开发体验是新的，生产构建的纯净依然姓 Rollup**。另外 Rust 军团（Rspack 走 Webpack 兼容路线、Rolldown 用 Rust 重写 Rollup 理念）正在接管性能问题，可以理解为：**Webpack 的全能与 Rollup 的纯净这场理念之争没有结束，只是换了更快的引擎继续打**。

对面试和实战，记这句话就够：**应用要的是「开发爽 + 分包灵活」，库要的是「产物纯 + 可被下游摇树」——前者选 Webpack（或 Vite 的开发体验），后者选 Rollup。**
