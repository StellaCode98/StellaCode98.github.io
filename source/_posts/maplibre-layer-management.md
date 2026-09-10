---
title: 大屏地图的图层管理实战：MapLibre GL 动态样式、两阶段渲染与叠放排序
date: 2026-09-09 20:30:00
description: 一个数据可视化大屏项目里几十类设施图层的管理方案：用 Pinia 组合式工厂拆解图层 Store，用「样式缓存 + 两阶段渲染」解决瓦片样式延迟，用质心图层解决面要素图标重复，再理清显隐、叠放、透明度与 Vue 集成的那些坑。
categories:
  - Vue 进阶
tags:
  - vue3
  - Pinia
  - Maplibre
  - 前端架构
---

最近在一个数据可视化大屏项目里负责地图模块，需求乍看很朴素：左侧一棵图层树，勾选哪类设施，地图上就渲染哪类要素。但做下去发现坑一个接一个：

- 设施有**几十类**，点、线、面混杂，样式（颜色/图标/线宽）全部由后端配置下发，前端写死不可能；
- 矢量瓦片是**异步加载**的，「瓦片里到底有哪些要素类型」要等加载完才知道——直接等它，图层会白屏好几百毫秒；
- 大面积面要素挂图标，放大后**每个瓦片上都长出一个重复图标**；
- 一切换底图样式，所有业务图层**集体消失**（MapLibre 的 `setStyle` 会重置整个样式）；
- 把 MapLibre 实例塞进 Vue 的 `ref`，地图**随机报错**，内部数据结构被 Proxy 拆得面目全非。

这篇文章把这些问题的解法按主题拆开讲清楚。技术栈是 **Vue 3 + Pinia + MapLibre GL（配 PMTiles 协议）**，但思路对任何 WebGL 地图引擎（Mapbox GL / MapLibre / 高性能 Canvas 地图）都通用。

<!-- more -->

## 一、整体架构：不写图层基类，用组合式工厂

先回答一个设计问题：几十种图层，要不要抽象一个 `BaseLayer` 基类，子类重写 `addToMap` / `removeFromMap`？

我们的答案是不写 class 继承，而是把图层系统拆成**职责单一的工厂函数**，在 Pinia setup store 里组合：

```text
useLayerStore (图层主 Store)
 ├── state      : layers / graphLayers 两棵树 + activeTab + 搜索词
 ├── computed   : filteredLayers / checkedIds / activeLayerCount
 ├── styleCache   = createStyleCache()     全局样式解析缓存
 ├── persistence  = createPersistence()    localStorage 勾选/展开持久化
 ├── mapOps       = createMapLayers()      MapLibre 图层操控（核心）
 └── searchFilter = createKeywordFilter()  关键词过滤（纯函数）
```

`createMapLayers` 通过**依赖注入**拿到它需要的东西，而不是自己 `import` store——这让它可以独立单测：

```ts
const mapOps = createMapLayers({
  layers,                    // 图层树（响应式）
  graphLayers,               // 拓扑树（响应式）
  getCache: styleCache.getCache,          // 样式缓存快照
  findNodeInBoth,            // 两棵树按 ID 查节点
  getActiveTab: () => activeTab.value,
  getFacilityOrderIds: () => facilityOrderIds.value,
  ready,                     // 两棵树加载完成的 Promise
})
```

选择组合而非继承的理由很实际：图层系统的变化维度是「样式解析 / 持久化 / 地图操作 / 过滤」四个**正交方向**，继承树只能沿一条轴展开，最终必然长出 `BaseLayer → VectorLayer → StyledVectorLayer` 这种每次改样式都要动基类的结构。组合模式下每个工厂只对自己的 Map 负责，核心纯函数（瓦片 URL 解析、排序合并）还能直接跑单测。

## 二、数据模型：一棵树跑通 UI 和地图两个世界

后端返回的是三级树：一级分类（电力/交通）→ 二级分组 → 三级叶子设施。前端转成统一的 `LayerNode`：

```ts
export interface LayerNode {
  id: string
  name: string
  nodeType: 'group' | 'layer'
  visible: boolean          // 勾选状态
  opacity: number
  children?: LayerNode[]
  sourceLayer?: string      // 矢量瓦片的 source-layer 名
  // ---- 运行时：MapLibre 绑定字段（渲染后由 store 写回） ----
  _mapLayerIds?: string[]   // 该节点在地图上的所有 MapLibre layer ID
  _mapSourceAdded?: boolean // source 是否已添加（隐藏时不销毁）
  _centroidMeta?: { ... }   // 面要素质心图层的运行时状态
}
```

关键设计是那几个下划线开头的**运行时绑定字段**：一棵树同时服务图层面板（勾选、搜索、计数）和地图（真正的 layer ID 列表）。「用户勾选的是树节点，地图操作的是 layer ID」这层映射关系就收在节点自己身上，不用再维护一张 id 映射表。

瓦片协议上同时支持两种源，按后端下发的字段自动分流：

```ts
if (item.pmtiles) {
  // PMTiles：单文件瓦片，注册自定义协议后直接当 vector source 用
  map.addSource(sourceId, { type: 'vector', url: `pmtiles://${base}${path}` })
} else {
  // GeoServer TMS PBF 瓦片
  map.addSource(sourceId, { type: 'vector', tiles, scheme: 'tms', tileSize: 512 })
}
```

PMTiles 值得一提：它是把整包瓦片打成一个文件的格式，配合官方库注册 `pmtiles://` 协议即可无缝接入 MapLibre，私有化部署时比维护一套瓦片服务省事得多：

```ts
let registered = false
export async function ensurePmtilesProtocol(maplibre: typeof maplibregl) {
  if (registered) return          // 幂等注册
  const { Protocol } = await import('pmtiles')  // 动态 import，不进首屏包
  maplibre.addProtocol('pmtiles', new Protocol().tile)
  registered = true
}
```

## 三、动态样式：样式不写死，按 type 过滤建图层

几十类设施样式由后端「全局样式配置」下发，结构大致是 `Record<typeKey, 样式规则[]>`——key 是瓦片要素 `properties.type` 的值（比如 `transformer_station`），值是渲染规则（`code` 决定渲染成点/线/面，`config` 决定颜色图标线宽）。

拿到配置后先**预编译**成 MapLibre 的 layer spec（`createStyleCache` 的职责）：point + 图标 → symbol 图层、point 无图标 → circle、line → line、polygon → fill + 边界线 outline，同时把 SVG 图标预渲染成 `ImageData` 备用。渲染时按 typeKey 建**带 filter 的独立图层**：

```ts
for (const [typeKey, entries] of styleCache) {
  const filter = ['==', ['get', 'type'], typeKey]
  const layerId = `${nodeId}__${typeKey}`
  map.addLayer({ id: layerId, type: 'fill', source: sourceId,
                 'source-layer': sourceLayer, filter, ...layerSpec })
}
```

图层 ID 用 `${nodeId}__${typeKey}` 这样的命名约定串起来，一个设施图层勾选后实际产生的所有 layer（多条样式、面边界线 `-outline`、质心 `_centroid`）都能靠前缀反查回来——后面处理显隐、排序、销毁时全靠这份「户口」。

## 四、两阶段渲染：先上菜，再校菜

这是整个模块里最值得写的一笔。矛盾在于：

- 瓦片是**按需异步加载**的，`addSource` 之后要等一会儿才能查到里面的要素；
- 但样式缓存的 key 集合我们是**预先知道**的（全局样式配置就摆在那）；
- 用户勾选图层后盯着地图等，白屏超过几百毫秒体验就很差。

方案是两阶段：

```text
Phase 1（立即渲染，~100ms）
  addSource → 注册图标 → 用样式缓存的全部 typeKey 建 filter 图层
  特征先按缓存假设渲染，用户立刻看到带样式的要素

Phase 2（后台精炼，requestIdleCallback）
  探测瓦片里实际存在的 type 集合与几何类型
  与缓存假设 diff：一致 → 什么都不做；不一致 → 拆掉重建
```

探测用 `map.querySourceFeatures` 轮询，带**指数退避**和超时兜底：

```ts
const MAX_ATTEMPTS = 10
const TIMEOUT_MS = 3000

const getBackoffInterval = () => {
  if (attempt <= 2) return 150
  if (attempt <= 5) return 300
  return 500
}

const tryCollect = () => {
  const features = map.querySourceFeatures(sourceId, { sourceLayer })
  if (features.length > 0) {
    // 收集 properties.type 集合 + 每个 type 的几何类型（Point/LineString/Polygon）
    resolve({ types, typeGeometries })
    return
  }
  if (attempt < MAX_ATTEMPTS) timer = setTimeout(tryCollect, getBackoffInterval())
  else resolve(null)   // 超时兜底：维持 Phase 1 的渲染结果
}
```

精炼阶段的 diff 逻辑克制到「多一步都不做」：类型集合一致、几何类型与渲染 code 匹配（`GEOM_TO_CODE` 映射：`Point → {point, circle}`、`Polygon → {polygon, fill}`）、且渲染图层确实有要素命中——三条全过就**不重建**，避免无谓的移除/重加造成闪烁。

这个模式的本质是把「**乐观渲染 + 后台校验**」搬到了地图场景：与其等真相出来再画，不如先按最可信的假设画出来，真相到了只在偏差时纠正。类似的思路在离线优先的编辑器、React 的并发渲染里都能看到影子。

## 五、面要素的图标重复：质心图层方案

面要素（比如一片行政区、一个园区）想标注图标，直接在 vector source 上加 symbol 图层会踩坑：**MapLibre 会在每个包含该要素的瓦片里各放一个图标**。低缩放时一个瓦片能罩住整个面，看不出来；一放大，面横跨 6 个瓦片，图标就排成了 6 个。

解法是把图标从瓦片渲染里拿出来，放到**独立 GeoJSON source** 上：

1. `querySourceFeatures` 取出面要素几何，计算每个面的**质心**（外环顶点坐标平均）；
2. 质心点写进独立的 GeoJSON source，symbol 图层挂在质心 source 上——一个面永远只有一个图标；
3. 监听 `sourcedata` 事件，**新瓦片到达时增量补充**新出现的面质心（150ms 防抖，`seenIds` 集合去重）。

```ts
map.on('sourcedata', (e) => {
  if (e.sourceId !== sourceId || !e.isSourceLoaded || !e.tile) return
  // 防抖后：querySourceFeatures → 算新面质心 → geoSource.setData(prev.concat(newFeats))
})
```

## 六、显隐、叠放、透明度：三个小专题

### 显隐：隐藏不销毁

取消勾选只把图层 `visibility` 设为 `none`，**source 原地保留**：

```ts
function setLayerVisibility(map, layerIds, visible) {
  for (const id of layerIds) {
    map.setLayoutProperty(id, 'visibility', visible ? 'visible' : 'none')
  }
}
```

再次勾选时直接切回 visible，瓦片零请求、秒出。代价是内存里留着 source，但大屏场景图层总量有限，这笔账划算。显示时配一段 350ms 的 rAF 渐入动画（easeOutCubic 缓动），并用 `_previouslyShown` 集合记录「首次加载才动画，再次显示直接出」，避免每次勾选都闪一下。

### 叠放：MapLibre 没有 z-index

WebGL 地图的图层顺序 = **addLayer 的先后顺序**，想调整只有 `map.moveLayer(id)`（不传第二个参数即置顶）一条路。我们的设施叠放顺序是后端下发的全局配置（`orderIds[0]` = 最上层），应用方式是「**从最底层开始逐个置顶**」：

```ts
function applyFacilityOrderToMap(map) {
  const orderedLeaves = getOrderedFacilityLeaves(layers, graphLayers, orderIds)
  for (const node of orderedLeaves) {          // 数组顺序 = 从下到上
    for (const layerId of collectNodeMapLayerIds(node)) {
      if (map.getLayer(layerId)) map.moveLayer(layerId)  // 逐个置顶
    }
  }
}
```

坑在于图层是**异步**创建的——两阶段渲染的 Phase 2 可能随时重建图层，质心层还会延迟 200ms 追加，每次都会破坏顺序。所以所有会新增图层的路径末尾统一调一个 **50ms 防抖的重排**，保证最终状态收敛到全局排序。

### 透明度：别覆盖样式默认值

面图层样式里常写着 `fill-opacity: 0.25` 这种「设计师调好的半透明」。如果用户拖滑块设 0.8，直接 `setPaintProperty(0.8)` 就把原设计覆盖了；再拖回 1.0，图变成实心——原始基准丢了。

解法是**基准透明度缓存**：首次读某图层时把它的原始 opacity 记下来，之后永远写 `userOpacity × base`：

```ts
const _baseOpacityCache = new Map<string, number>()

function applyOpacity(map, layerId, userOpacity) {
  const base = ensureBaseOpacity(map, layerId)  // 首次读取并缓存原始值
  const effective = Math.max(0, Math.min(1, userOpacity * base))
  map.setPaintProperty(layerId, `${layer.type}-opacity`, effective)
}
```

图层销毁时同步清缓存（`clearLayerBaseOpacityCache`），否则重建后的图层会拿旧基准。

## 七、Vue 与地图实例的相处：markRaw + toRaw

MapLibre 的 Map 实例内部有大量 `Set`/`Map` 和 WebGL 资源。放进 Vue 的 `ref` 会被 **深度 Proxy 代理**，读写属性都过一遍 Proxy，轻则性能劣化，重则 MapLibre 内部依赖引用相等性的逻辑直接失效（Proxy 包装后的对象 `!==` 原对象）。

处理三件套：

```ts
// 1. 存储用 shallowRef + markRaw
const mapInstance = shallowRef<MapLibreMap | null>(null)
function setMapInstance(map: MapLibreMap | null) {
  mapInstance.value = map ? markRaw(map) : null   // markRaw 阻止深度代理
}

// 2. 使用前 toRaw 兜底（防中间环节被 reactive 污染）
const map = toRaw(mapInstance.value)

// 3. 传给地图 API 的节点对象同样 toRaw
renderNodeLayers(map, toRaw(node))
```

一句话记住：**地图实例和一切会进地图 API 的对象，都要待在响应式系统之外**。真的需要响应式的（中心点、缩放级别），单独用基本类型 ref 同步一份。

## 八、底图切换：setStyle 是一次「大清洗」

`map.setStyle(newStyle)` 会**替换整个样式**，所有手动 `addLayer/addSource` 的业务图层全部蒸发。正确姿势是把它当成一次受控的销毁重建：

```ts
function switchMapStyle(map, styleId) {
  const snapshot = layerStore.layers
  layerStore.removeAllFromMap(map)          // ① 主动清干净业务层（含事件监听）
  map.setStyle(newStyle, { diff: true })    // ② diff 模式：能复用的底层不动
  map.once('style.load', () => {
    requestAnimationFrame(() => {
      registerLayerIcons(map, flattenLeafNodes(snapshot))  // ③ 重注册图标
        .then(() => layerStore.syncAfterLoad(map))         // ④ 按勾选状态重渲染
    })
  })
}
```

两个细节：`{ diff: true }` 让 MapLibre 尽量增量替换而非全部重载；底图 style JSON 提前用原生 `fetch` 预取并缓存（`force-cache`），切换时省一次网络往返。另外图标也要重注册——`addImage` 注册的图片同样活在 style 里，`setStyle` 后一样会丢。

## 总结

整套图层管理的生命周期一图流：

```text
初始化     并行拉取：图层树 + 全局样式 + 叠放顺序（样式进 sessionStorage 免重复请求）
  ↓
勾选图层   树节点 visible = true（localStorage 持久化）
  ↓         ↓
首次加载   fetchLayerTiles → addSource → Phase 1 立即渲染（缓存 key）
  ↓         Phase 2 idle 探测 → diff → 必要时重建 → 质心图层 → 防抖重排
  ↓
再次勾选   visibility 直接切回，渐入动画，零瓦片请求
  ↓
样式变更   保留 source，只拆重建 layer
  ↓
切底图     removeAll → setStyle(diff) → style.load 后重注册图标 + 全量恢复
```

问题速查表：

| 问题 | 根因 | 解法 |
| --- | --- | --- |
| 图层样式要等瓦片加载完才出来 | 渲染 key 依赖瓦片数据 | 两阶段：缓存 key 乐观渲染 + idle 探测校验 |
| 面要素图标放大后重复 | symbol 按瓦片放置 | 质心 GeoJSON source + sourcedata 增量刷新 |
| 图层顺序被异步加载打乱 | moveLayer 顺序 = 添加顺序 | 全局排序数组 + 新增图层后 50ms 防抖重排 |
| 透明度滑块毁掉样式默认值 | setPaintProperty 覆盖基准 | 基准透明度缓存，写 userOpacity × base |
| 切底图业务图层全丢 | setStyle 重置整个样式 | 主动清除 → diff 切换 → style.load 后恢复 |
| 地图实例放 ref 随机报错 | 深度 Proxy 破坏内部结构 | shallowRef + markRaw + toRaw 三件套 |

**一句话记住**：大屏图层管理的核心不是「往地图上加图层」，而是管理**树节点与 MapLibre layer 之间的映射**——渲染可以乐观、校验放后台，但每个 layer 的出生（ID 命名）、户籍（`_mapLayerIds`）、销毁（source 保留策略）都必须在掌控之中。
