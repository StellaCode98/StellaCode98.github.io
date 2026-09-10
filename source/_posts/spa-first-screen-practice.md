---
title: SPA 首屏优化落地实录：大屏项目的路由分割、依赖分包与异步引擎加载
date: 2026-09-10 15:00:00
description: 上一篇讲了 SPA 首屏优化的理论清单——减体积、抢网络、缩白屏。这篇拿一个真实的数据可视化大屏项目逐条对照：路由全量懒加载、manualChunks 五路分包配 contenthash、数百 KB 的地图引擎动态 import 移出关键路径、面板组件 defineAsyncComponent 按需挂载、preconnect 提前握手底图 CDN、首屏接口 Promise.all 并行加单例防重，外加一个分阶段的加载进度反馈。每一条都有真实代码，没落地的也在文末如实记账。
categories:
  - [前端基础, 性能优化]
tags:
  - 性能优化
  - SPA
  - vite
  - vue3
---

上一篇[《SPA 首屏优化：减体积、抢网络、缩白屏》](/2026/09/10/spa-first-screen-optimization/)把理论清单列全了。这篇是它的落地篇——我拿手头一个数据可视化大屏项目（Vue 3 + Vite + MapLibre GL）逐条对照那份清单，**有代码支撑的讲透实现，没落地的在文末如实记账**。

先说这个项目的特殊性，它决定了优化的重心：大屏首屏最大的包袱不是业务代码，而是**地图引擎**——maplibre-gl 光 JS 就数百 KB，还有一份全量 CSS；其次才是 Element Plus 和一堆面板组件。所以整个优化围绕一句话展开：**首屏关键路径上只留「能让地图出现的最小集合」，其余全部异步化**。

先给核对结论，再逐条展开：

| 理论手段 | 项目落地 | 位置 |
| --- | --- | --- |
| 路由级代码分割 | ✅ 全部路由懒加载（含布局组件） | `router/index.ts` |
| 依赖分包 + 长缓存 | ✅ manualChunks 五路分包 + contenthash | `vite.config.ts` |
| 重型依赖按需/异步引入 | ✅ 地图引擎 JS/CSS/协议全部动态 import | `main.ts` / `MapContainer.vue` |
| 组件级懒加载 | ✅ defineAsyncComponent 挂面板 | `views/Home` |
| UI 库按需引入 | ⚠️ 图标按需注册，组件全量（见文末） | `main.ts` |
| 资源提示 preconnect | ✅ 底图 CDN 域名预连接 | `index.html` |
| 数据预取/缓存 | ✅ 底图样式预取 + force-cache | `stores/map.ts` |
| 首屏接口并行 | ✅ Promise.all + 单例防重 | `stores/layer.ts` |
| 白屏期反馈 | ✅ 分阶段加载进度（组件级） | `MapContainer.vue` |

<!-- more -->

## 一、减体积：三层异步化

### 1. 路由懒加载：连布局都不放进首屏

理论篇说过，路由分割的收益是「其他路由不再拖累首页」。这个项目做得比较彻底——**所有路由组件，包括多个路由共享的 MainLayout，全部走动态 import**：

```ts
const MainLayout = () => import('@/layouts/MainLayout.vue')

const routes: RouteRecordRaw[] = [
  {
    path: '/login',
    name: 'Login',
    component: () => import('@/views/Login/index.vue'),
    meta: { requiresAuth: false, title: '登录' },
  },
  {
    path: '/',
    component: MainLayout,
    children: [
      { path: '', name: 'Home', component: () => import('@/views/Home/index.vue') },
    ],
  },
  {
    path: '/system',
    component: MainLayout,
    children: [
      { path: 'user', name: 'SystemUser', component: () => import('@/views/System/User.vue') },
      // 设施管理、样式配置、分析报表……十余个路由，全部同款写法
    ],
  },
]
```

两个细节：MainLayout 被多个路由引用时，Rollup 会自动提取成共享 chunk，各路由并行预加载，不会重复下载；管理后台那十几个页面（用户/角色/日志/数据上传）对只看大屏首屏的用户来说**一个字节都不该下载**——懒加载后它们各自躺在独立的 chunk 里，访问到才拉取。

### 2. 依赖分包：五路 manualChunks + contenthash

```ts
// vite.config.ts
export default defineConfig({
  build: {
    cssCodeSplit: true,               // CSS 代码分割
    chunkSizeWarningLimit: 1000,
    rollupOptions: {
      output: {
        // 手动分包策略
        manualChunks: {
          'vue-vendor': ['vue', 'vue-router', 'pinia'],        // 框架核心
          'element-plus': ['element-plus', '@element-plus/icons-vue'],  // UI 库
          'maplibre': ['maplibre-gl'],                          // 地图引擎
          'chat-ui': ['vue-element-plus-x'],                    // AI 聊天组件
          'http': ['axios'],                                    // HTTP 工具
        },
        // 文件命名策略：contenthash 打头
        chunkFileNames: 'js/[name]-[hash].js',
        entryFileNames: 'js/[name]-[hash].js',
        assetFileNames: '[ext]/[name]-[hash].[ext]',
      },
    },
    minify: 'esbuild',
  },
})
```

分包的划分逻辑是**变更频率**：`vue-vendor` 几乎永不发版（缓存常年有效），`element-plus` 跟随 UI 库升级偶尔变，业务 chunk 天天变。配合 `[hash]` 文件名，理论篇那条「HTML 不缓存 → 拿最新 hash → 没变的 chunk 走强缓存」的闭环就有了物理基础——改一行业务代码，用户重新下载的只有业务 chunk 和 HTML。

`'http': ['axios']` 这种小分包值得单说：axios 本身只有十几 KB，单独拆出来不是为了体积，而是它被**入口和所有异步路由共同依赖**，不拆就会被打进某个大 chunk，别的 chunk 引用它时重复打包。拆出来变成公共依赖，谁加载都命中同一个文件（和同一份缓存）。

### 3. 地图引擎异步加载：本项目最值钱的一刀

maplibre-gl 是全项目最大的单体依赖。三处异步化把它彻底移出首屏关键路径：

**CSS 不阻塞首屏**——入口文件里用动态 import，样式变成异步 chunk，浏览器不会因为它阻塞渲染：

```ts
// main.ts
// maplibre-gl CSS 懒加载，避免阻塞首屏
import('maplibre-gl/dist/maplibre-gl.css')
```

**JS 在地图容器挂载时才加载**——用模块级变量做单例缓存，加载一次后续复用：

```ts
// MapContainer.vue
let maplibregl: typeof import('maplibre-gl').default | null = null

async function loadMapLibre() {
  if (!maplibregl) {
    const mod = await import('maplibre-gl')   // 数百 KB 的引擎，此刻才下载
    maplibregl = mod.default
  }
  return maplibregl
}

async function initMap() {
  if (!mapRef.value) return
  loadingStage.value = '加载地图引擎...'
  const maplibre = await loadMapLibre()        // ← 关键路径上的唯一大件
  await ensurePmtilesProtocol(maplibre)       // pmtiles 库同样是动态 import
  // ...创建地图实例
}
```

**配套的瓦片协议库也一样**——PMTiles 的 Protocol 也是 `await import('pmtiles')` 按需加载，不用瓦片协议的用户路径不触发。

这一组操作的净效果：首屏 HTML 之后的关键 JS 从「全量 bundle」变成「vue-vendor + 入口 + Home 页」，地图引擎在网络空闲期与地图初始化流程并行下载。用户感知到的顺序不再是「等引擎下载完才开始渲染页面」，而是「页面先出来，地图区域显示加载中，引擎到了地图补位」。

### 4. 面板组件：defineAsyncComponent 按需挂载

大屏首屏由地图容器 + 三个浮层面板（图层、事件、AI 助手）组成。面板不是首屏关键内容，全部异步：

```ts
// views/Home/index.vue
// 懒加载非首屏关键组件
const LayerPanel = defineAsyncComponent(() => import('@/components/layout/LayerPanel.vue'))
const EventPanel = defineAsyncComponent(() => import('@/components/layout/EventPanel.vue'))
const AIAssistant = defineAsyncComponent(() => import('@/components/ai/AIAssistant.vue'))
```

配合路由懒加载，这是两级异步：**路由级**（进首页才加载 Home）+ **组件级**（Home 渲染后才加载面板）。图层面板里还有整棵图层树的渲染逻辑和搜索过滤，拆出去后 Home 的首帧又轻了一截。

### 5. 图标按需注册

项目没有引 unplugin 自动按需，但图标走的是手动按需：从 `@element-plus/icons-vue` 具名导入需要的六十来个图标注册为全局组件，未引用的图标靠 tree shaking 摇掉：

```ts
import { Monitor, Camera, FullScreen, Plus, /* ... */ Grid, List } from '@element-plus/icons-vue'

const icons: Record<string, any> = { Monitor, Camera, FullScreen, Plus, /* ... */ }
for (const [name, component] of Object.entries(icons)) {
  app.component(name, component)
}
```

## 二、抢网络：预连接与预取

### preconnect：给底图 CDN 提前握手

大屏底图样式 JSON 和矢量瓦片全部来自 `basemaps.cartocdn.com`，首屏必然访问。在 HTML 里提前完成 DNS + TCP + TLS：

```html
<!-- index.html -->
<!-- DNS 预解析 + 预连接，加速外部资源加载 -->
<link rel="preconnect" href="https://basemaps.cartocdn.com" />
<link rel="dns-prefetch" href="https://basemaps.cartocdn.com" />
```

两行 HTML，省掉地图初始化时的握手串行等待。`dns-prefetch` 是 `preconnect` 的降级兜底，给老浏览器一份保险。选这个域名是有依据的——**只 preconnect 首屏确定要访问的域**，乱加反而挤占连接数。

### 预取底图样式 + 强缓存

切换底图要拉取新的 style JSON，这个项目在地图加载完成后顺手把**所有底图的样式预取**了一遍，且强制走 HTTP 缓存：

```ts
// stores/map.ts
async function preloadStyleJson(styleId: string): Promise<object | null> {
  if (_styleJsonCache.has(styleId)) return _styleJsonCache.get(styleId)!   // 内存缓存
  const target = availableStyles.value.find(s => s.id === styleId)
  if (target.type === 'raster') {
    const rasterStyle = buildRasterStyle(target)    // 栅格底图本地构造，零请求
    _styleJsonCache.set(styleId, rasterStyle)
    return rasterStyle
  }
  const resp = await fetch(target.url, { cache: 'force-cache' })  // 有缓存就不发请求
  const json = await resp.json()
  _styleJsonCache.set(styleId, json)
  return json
}

// 地图 style.load 之后，空闲期预取全部底图
mapStore.preloadAllStyles()
```

三层缓存各司其职：内存 Map 管会话内复用，`force-cache` 管跨会话的 HTTP 缓存，栅格底图直接本地构造 JSON 连请求都不发。用户点切换底图时样式已是本地数据，切换零延迟。

## 三、首屏数据：并行、防重、缓存

理论篇强调「别让接口排队」。首页初始化要同时拿图层树、全局样式、叠放顺序三份数据，全部 `Promise.all` 并行：

```ts
// stores/layer.ts
const [treeRes, , globalOrderResult] = await Promise.all([
  getLayerTree(),
  loadGlobalStyle(),                        // 内部还有 sessionStorage 缓存兜底
  getGlobalOrder().catch(() => null),       // 弱依赖：失败不阻塞主流程
])
```

两个额外细节：

- **弱依赖降级**：`getGlobalOrder().catch(() => null)`——排序接口挂了不该拖死整个图层树加载，拿到 null 就用默认排序；
- **单例防重**：首页和图层面板的 `onMounted` 都会触发初始化，用 Promise 缓存挡住第二次请求：

```ts
let _homeInitPromise: Promise<void> | null = null

async function initHomeLayerData() {
  if (_homeInitPromise) return _homeInitPromise   // 已经在跑，直接复用
  _homeInitPromise = (async () => {
    await loadLayers()
    await loadGraphLayers()
  })()
  return _homeInitPromise
}
```

全局样式还会写进 sessionStorage——刷新页面时跳过 `getGlobalStyle` 请求，直接从会话缓存构建样式缓存。首屏接口从 3 个降到 2 个。

## 四、白屏期：分阶段进度反馈

项目没有做 index.html 级的骨架屏，但在**地图容器内部**做了分阶段的加载反馈——地图区域不是干等，而是持续告诉用户「现在进行到哪了」：

```ts
const loadingProgress = ref(0)
const loadingStage = ref('正在初始化地图引擎...')

function startLoadingProgress() {
  loadingProgress.value = 5
  loadingStage.value = '加载地图样式...'
  return function advanceProgress(percent: number, stage?: string) {
    loadingProgress.value = Math.min(percent, 99)   // 封顶 99，完成交给真实事件
    if (stage) loadingStage.value = stage
  }
}

async function initMap() {
  loadingStage.value = '加载地图引擎...'
  const maplibre = await loadMapLibre()
  const advanceProgress = startLoadingProgress()
  // ... 底图加载中按节点推进：advanceProgress(40, '加载底图瓦片...')
  // style.load 后：loadingStage.value = '底图加载完成'
}
```

```html
<div class="loading-stage">{{ loadingStage }}</div>
<!-- 进度条按 loadingProgress 渲染 -->
```

一个小而讲究的设计：进度**封顶 99%**——异步加载的真实耗时不可预测，进度条冲到 100 却还卡在那儿，比停在 99 更让人焦虑。100% 只能由「地图真正加载完成」这个事件来置位。这是「体感优化」里典型的心理学小把戏，和骨架屏同一族：时间没变，等待变得可解释了。

另外详情弹窗里的图片用了原生懒加载，一行声明式：<img :src="img" loading="lazy" />，非首屏图片不占首屏带宽。

## 五、诚实的账：理论有、项目还没做的

- **web-vitals 上报**：没有。目前优化验收靠 Lighthouse 手跑，缺真实用户数据——LCP 在客户内网环境是什么水平，其实是笔糊涂账。这在监控篇补课；
- **Element Plus 组件按需引入**：`app.use(ElementPlus)` 是全量引入，全量 CSS 也进了首屏。manualChunks 把它隔离成了独立 chunk（不阻塞关键路径的并行下载），但**总量没减**。按 unplugin-vue-components 按需引入预计还能再砍掉一大块，属于已识别未执行；
- **服务端 gzip/brotli、CDN、缓存头**：前端仓库之外，需要部署侧配合，代码里无从体现；
- **index.html 骨架屏 / 内联关键 CSS / SSR**：未做。大屏场景白屏期被「地图加载进度」组件覆盖了大半，且内网部署暂无 SEO 需求，优先级排后——先吃便宜的收益，贵的刀最后动。

## 总结

这张大屏的首屏优化，本质是把理论清单按项目特点**排了优先级**：

| 手段 | 代码位置 | 对首屏的意义 |
| --- | --- | --- |
| 路由全懒加载 | `router/index.ts` | 管理页零字节进首屏 |
| manualChunks 五路分包 | `vite.config.ts` | 长缓存物理基础 + 公共依赖不重复打包 |
| 地图引擎三件动态 import | `main.ts` / `MapContainer.vue` | 数百 KB 最大件移出关键路径 |
| defineAsyncComponent 面板 | `views/Home/index.vue` | 首帧只渲染地图容器 |
| preconnect 底图 CDN | `index.html` | 握手与 JS 下载并行 |
| 样式预取 + force-cache | `stores/map.ts` | 切底图零网络等待 |
| Promise.all + 单例防重 | `stores/layer.ts` | 接口不排队、不重复 |
| 分阶段加载进度 | `MapContainer.vue` | 白屏期可解释 |

**一句话记住**：首屏优化落地时别平均用力——先找出项目里**最大的那块依赖**（这里是地图引擎），把它异步化；再用分包和缓存把「不变的」和「常变的」分开；最后让等待变得可解释。理论清单是地图，往哪走要看自己项目的地形。
