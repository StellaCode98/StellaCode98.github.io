---
title: Cesium 数据加载实战：从免 npm 引入到视口驱动的增量上球
date: 2026-09-09 19:30:00
description: 以一个真实落地的 Vue3 + Vite + CesiumJS 三维地图项目为例，完整复盘数据上球的全链路：免 npm 的 public 静态引入、配置驱动的多源影像底图、后端 JSON 到 Primitive 的渲染、视口驱动的增量加载与缓存治理，以及自定义 GLSL 流动线、球面弧线这些渲染亮点。
categories:
  - [GIS 与三维可视化]
tags:
  - Cesium
  - GIS
  - 三维可视化
  - Vue3
  - Vite
---

> 这篇文章不是 Cesium 入门教程，而是一次真实项目的落地复盘。最近负责的 Vue3 + Vite 项目里有一块三维地图：CesiumJS 1.123，渲染几十类基础设施（变电站、发电厂、线路……）的点线面数据，支持视口内动态加载、拓扑流动网络、实时事件告警。网上 Cesium 教程大多停留在「new 一个 Viewer + GeoJsonDataSource.load 一个文件」，而真实项目里你要回答的问题其实是：**内网离线环境怎么跑、几万个实体怎么不卡、拖动地图时数据怎么增量进出、内存怎么不涨爆**。本文按数据流动的顺序，把这条链路从头到尾拆开讲。所有代码都来自这个真实项目，文末附完整文件地图。

## 目录

- [0. 前言：一个真实的 Cesium 项目长什么样](#sec0)
- [1. 加载 Cesium 本体：不用 npm、不用 vite-plugin-cesium](#sec1)
- [2. Viewer 初始化：先拿到一个"干净"的球](#sec2)
- [3. 影像底图加载：配置驱动的多源图层](#sec3)
- [4. 业务数据上球：为什么放弃 Entity，改用 Primitive](#sec4)
- [5. 视口驱动的增量加载：只请求"看得见"的数据](#sec5)
- [6. 让数据"活"起来：实时推送与交互联动](#sec6)
- [7. 渲染亮点一瞥：GLSL 流动线、球面弧线、呼吸图标](#sec7)
- [8. 写在最后](#sec8)

<a id="sec0"></a>

## 0. 前言：一个真实的 Cesium 项目长什么样

先交代背景。项目是一个基础设施三维地图应用：Vue 3.3 + Vite 6 + Pinia 3 + CesiumJS 1.123，部署在**内网离线环境**——这个前提直接决定了后面很多技术选型：不能用 Cesium Ion 的在线资产、不能依赖任何公网瓦片服务、后端接口返回的是**带坐标数组的业务 JSON 而不是标准 GeoJSON 文件**。

数据的流动方向可以用一张图概括，后文所有章节都是对这张图的展开：

```text
┌────────────────── 静态层（public/）──────────────────┐
│  /cesium/          CesiumJS 1.123 构建产物            │
│  /config.js        window.baseConfig（运行时配置）     │
└────────────┬─────────────────────────────────────────┘
             ▼  index.html 里 <script> 全局引入
┌────────────── Viewer 初始化（cesiumMap.vue）──────────────┐
│  禁用全部默认控件 · 不加载 Ion 默认底图 · 阻断 Ion 请求     │
│  camera.setView(baseConfig.cesiumCameraView)              │
└────────────┬─────────────────────────────────────────┘
             ▼  dispatchEvent('viewer-ready')
┌────────────── 影像底图（home-header）───────────────────────┐
│  config.js layers[] → layerFactory.createProvider()      │
│  XYZ / WMS / WMTS / ArcGIS → addImageryProvider()       │
└────────────┬─────────────────────────────────────────┘
             ▼
┌────────────── 业务数据管线（use-load-data.ts，核心）──────────┐
│  camera.moveEnd(400ms 防抖) → 相机高度过滤(1km~400km)     │
│    → computeViewRectangle() 算视口经纬度边界               │
│    → POST 接口 { tableIds, boundary }                     │
│    → processCoordinates() 坐标解析                        │
│    → Primitive 工厂（点/线/面）→ 增量 diff 上球             │
└────────────┬─────────────────────────────────────────┘
             ▼
┌────────────── 渲染与交互 ────────────────────────────────┐
│  globalPrimitiveCollection + globalBillboardCollection   │
│  悬停/点击/右键状态机 · WebSocket 事件 → 呼吸图标 · flyTo   │
└─────────────────────────────────────────────────────────┘
```

一个值得先说结论的事：这个项目**没有用**地形（`CesiumTerrainProvider`）、**没有用** 3D Tiles、**没有用** `GeoJsonDataSource`，也没有 CZML/KML。不是不会用，而是数据形态和性能要求决定了「后端 JSON + Primitive API」这条更底层的路线。第 4 节会讲清楚为什么。

<a id="sec1"></a>

## 1. 加载 Cesium 本体：不用 npm、不用 vite-plugin-cesium

### 1.1 常规路线的问题

Cesium 官方推荐的 npm 接入是 `npm i cesium` + 打包插件（`vite-plugin-cesium` 或 Vite 官方文档里 copy-webpack-plugin / vite-plugin-static-copy 那套）。但 Cesium 的构建产物非常特殊：`Build/Cesium/` 下除了一个 3MB+ 的 `Cesium.js`，还有 **Workers、Assets、Widgets、ThirdParty 四个静态目录**，运行时按需加载，一个都不能少。这带来两个痛点：

- 构建器要处理的文件数量暴涨（Workers 目录下几百个文件），dev 冷启动和 HMR 都会被拖慢；
- 每次构建都要复制一遍静态资源，CI 时间和 `dist` 体积都难看。

而 `vite-plugin-cesium` 的原理说穿了也很朴素：**把 node_modules 里的构建产物复制到输出目录 + external 掉 cesium 包**。既然如此，不如直接自己来。

### 1.2 项目的做法：public 目录 + 全局变量

项目的 `package.json` 里**没有 cesium 依赖**（node_modules 里那份只是为了取构建产物），只留了一个复制脚本：

```json
{
	"scripts": {
		"copy-cesium": "cp -r node_modules/cesium/Build/Cesium/* public/cesium/"
	}
}
```

装包后手动执行一次 `npm run copy-cesium`，把产物复制进 `public/cesium/`。之后 `index.html` 用最原始的方式引入：

```html
<head>
	<link rel="stylesheet" href="/cesium/Widgets/widgets.css" />
	<script src="/cesium/Cesium.js"></script>
	<script src="/config.js"></script>
</head>
```

这两行 script 各有分工：

- `/cesium/Cesium.js` 把整个库挂到全局变量 `window.Cesium` 上；
- `/config.js` 是**运行时配置**，往 `window.baseConfig` 上写后端地址、图层列表、初始相机视角、token——部署后改这一个文件就能换环境，不用重新构建。对内网项目来说这比任何环境变量方案都直观。

代价是失去了 ESM 导入，所以补一个全局类型声明文件 `global.d.ts`（记得在 `tsconfig.json` 的 `include` 里加上它）：

```typescript
import type * as Cesium from 'cesium';

declare global {
	interface Window {
		Cesium: typeof Cesium;
		viewer?: Cesium.Viewer;
		baseConfig: {
			baseUrl: string;   // 请求 ip
			baseWsUrl: string; // ws 请求 ip
			cesiumToken: string;
			layers?: any;      // 地图的图层
		};
	}
}
```

注意第一行 `import type * as Cesium from 'cesium'`——类型可以继续用 npm 包的（devDependencies 里装着，只参与类型检查不参与打包），运行时用的才是 `window.Cesium`。**类型走 npm，运行走全局**，两头的好处都占了。

### 1.3 vite.config.ts 里必须加的三个配置

```typescript
export default defineConfig({
	// 解决 knockout-3.5.1.js:159 Uncaught ReferenceError: global is not defined
	define: {
		global: 'globalThis',
		CESIUM_BASE_URL: JSON.stringify('/cesium/')
	},
	worker: {
		format: 'es'
	}
	// ...
});
```

三条各有来历，全是踩坑换来的：

1. **`global: 'globalThis'`**：Cesium 依赖的老版本 knockout 是为 Node/Browserify 时代写的，引用了裸的 `global` 变量，Vite 的 ESM 环境里没有这个东西，直接报 `ReferenceError`。用 define 把 `global` 替换成标准的 `globalThis`。
2. **`CESIUM_BASE_URL`**：告诉 Cesium 去哪找 Workers/Assets 这些静态资源。因为我们不是从 npm 包解析的，必须显式指到 `/cesium/`，否则几何计算（`PolygonGeometry` 创建时会甩给 Worker）会 404。
3. **`worker.format: 'es'`**：Cesium 的 Worker 需要 ES Module 格式，Vite 默认的 IIFE 会导致 Worker 加载失败。

这套方案的净收益：业务代码和 Cesium 彻底解耦，构建产物里不再有 cesium 的 chunk（`manualChunks` 不用管它），dev server 秒起；升级 Cesium 版本 = 覆盖一个目录 + 回归测试，`package.json` 纹丝不动。

<a id="sec2"></a>

## 2. Viewer 初始化：先拿到一个"干净"的球

`new Cesium.Viewer()` 默认给你的不是一个空球，而是一个"全家桶"：时间轴、动画控件、Ion 默认影像、地理编码搜索框……在离线内网里，这些东西不仅多余，而且**每一个都在偷偷发请求**。所以初始化的第一原则是：全部关掉。

```javascript
// src/pages/home/components/cesium-map/cesiumMap.vue（有删减）
const initMap = () => {
	// 用 Cesium 的 Ion 服务进行认证
	window.Cesium.Ion.defaultAccessToken = cesiumToken;
	window.Cesium.Ion.defaultServer = window.baseConfig?.defaultPreventServer; // 阻止所有 ion 请求

	window.Cesium.buildModuleUrl.setBaseUrl('/cesium/'); // 👈 关键！指向 public/cesium/

	const viewer = new window.Cesium.Viewer(cesiumContainer.value, {
		timeline: false,            // 时间轴
		animation: false,           // 动画控制器
		geocoder: false,            // 地名查找控制器
		homeButton: false,          // Home 按钮
		sceneModePicker: true,      // 投影方式控制器（保留 2D/3D 切换）
		baseLayerPicker: false,     // 图层选择控制器
		navigationHelpButton: false,// 帮助按钮
		fullscreenButton: false,    // 全屏按钮
		infoBox: false,             // 信息框
		selectionIndicator: false,  // 选择指示器
		imageryLayers: false,       // 不加载默认底图
		imageryProvider: false,
		scene: { msaaSamples: 4 }   // 多重采样抗锯齿
	});

	// 高分屏下保持清晰度
	if (window.Cesium.FeatureDetection.supportsImageRenderingPixelated()) {
		viewer.resolutionScale = window.devicePixelRatio;
	}
	// 再加一层 FXAA 抗锯齿
	viewer.scene.postProcessStages.fxaa.enabled = true;

	window.viewer = viewer;
	// 初始视角：全部来自运行时配置
	window.viewer.camera.setView({
		destination: window.Cesium.Cartesian3.fromDegrees(...window.baseConfig.cesiumCameraView.destination),
		orientation: {
			heading: window.Cesium.Math.toRadians(window.baseConfig.cesiumCameraView.orientation.heading),
			pitch: window.Cesium.Math.toRadians(window.baseConfig.cesiumCameraView.orientation.pitch),
			roll: window.Cesium.Math.toRadians(window.baseConfig.cesiumCameraView.orientation.roll)
		}
	});
};
```

几个点单独说：

**`Ion.defaultServer = 'undefined'`——字符串，不是值**。这是全项目最"邪门"也最有效的一行。`config.js` 里 `defaultPreventServer: 'undefined'`，赋给 `Ion.defaultServer` 后所有指向 `api.cesium.com` 的资产请求都变成了对 `undefined/...` 的请求，直接快速失败。内网环境没有外网，与其让每个请求等超时，不如让它们立刻死掉。配合 `imageryProvider: false`，球上是干干净净的椭球面——底图完全由我们自己的图层系统接管（第 3 节）。

**初始视角全部配置化**。`config.js` 里：

```javascript
cesiumCameraView: {
	destination: [121.1, 23.75, 400000], // 经度、纬度、高度（米）
	orientation: { heading: 0, pitch: -90, roll: 0 }, // -90 = 垂直朝下
	VIEW_2D_HEIGHT: 5000000,   // 2D 模式预设高度
	VIEW_3D_HEIGHT: 10000000   // 3D 模式预设高度
}
```

换部署区域只改这三个数字，不碰代码。

**2D/3D 切换的视角适配**。保留了 `sceneModePicker`，但 2D 和 3D 需要的视野高度差着一个量级，所以用 `postRender` 监听场景模式变化，切完自动 `flyTo` 到各自的预设高度：

```javascript
let lastMode = window.viewer.scene.mode;
window.viewer.scene.postRender.addEventListener(() => {
	const currentMode = window.viewer.scene.mode;
	if (currentMode !== lastMode) {
		lastMode = currentMode;
		const targetView = currentMode === Cesium.SceneMode.SCENE3D ? globalView : customView;
		window.viewer.camera.flyTo({ ...targetView, duration: 0.5 });
	}
});
```

**`depthTestAgainstTerrain = false`**。教程里这句通常教你怎么开（配合地形），我们项目反过来是**显式关掉**：没有地形也没有 3D Tiles 时开启深度检测，实体的空间坐标反而会出现"消失/漂移"的诡异问题。没有地形，就别装作有地形。

最后，初始化完成时广播一个自定义事件，让其它组件（图层管理、树组件）等到 viewer 就绪再干活：

```javascript
onMounted(async () => {
	await nextTick();
	initMap();
	window.dispatchEvent(new CustomEvent('viewer-ready'));
});
```

<a id="sec3"></a>

## 3. 影像底图加载：配置驱动的多源图层

底图的需求看起来简单，实际约束不少：要支持多种协议（内网自建瓦片的 XYZ、OGC 标准的 WMS/WMTS、ArcGIS 服务）、要能勾选开关、透明度亮度可调、层级可控、**用户勾选状态要记住**（下次打开还是上次的组合）、并且全部要配置化——实施人员改 `config.js` 就能加图层，不用发版。

### 3.1 图层即配置

`config.js` 里每个图层是 `layers` 数组的一项：

```javascript
layers: [
	{
		id: "wms_demo",           // 唯一 id，删除/排序都靠它
		title: "WMS-GeoServer",   // 界面显示名
		type: "WMS",              // XYZ / WMS / WMTS / ArcGIS 四选一
		visible: false,           // 初始是否加载
		options: {
			url: "https://gibs.earthdata.nasa.gov/wms/epsg4326/best/wms.cgi",
			layers: 'BlueMarble_NextGeneration',
			parameters: { transparent: false, format: 'image/jpeg' },
			minimumLevel: 0,
			maximumLevel: 20
		},
		display: { alpha: 0.75, brightness: 1.2, zIndex: 1 } // 显示控制
	},
	{
		id: "xyz_osm",
		title: "台湾",
		type: "XYZ",
		visible: true,
		options: { url: "http://your-tile-server/maps/tw/{z}/{x}/{y}.jpg", maximumLevel: 20 },
		display: { zIndex: 2 }
	}
]
```

### 3.2 Provider 工厂

四种协议对应 Cesium 四种 ImageryProvider，一个 switch 分发完事：

```typescript
// src/utils/layerFactory.ts
import type { LayerConfig } from '@/types/cesium';

export function createProvider(cfg: LayerConfig) {
	switch (cfg.type) {
		case 'XYZ':
			return new window.Cesium.UrlTemplateImageryProvider(cfg.options);
		case 'WMS':
			return new window.Cesium.WebMapServiceImageryProvider(cfg.options);
		case 'WMTS':
			return new window.Cesium.WebMapTileServiceImageryProvider(cfg.options);
		case 'ArcGIS':
			return new window.Cesium.ArcGisMapServerImageryProvider(cfg.options);
		default:
			throw new Error(`未知图层类型 ${(cfg as any).type}`);
	}
}
```

顺带放一份各家的瓦片地址模板，都是项目里实际用过/验证过的，抄作业就能用：

```text
# 天地图 WMTS（注记层 cia，需换成自己的 tk）
https://t0.tianditu.gov.cn/cia_w/wmts?tk=你的token
  layer: 'cia', style: 'default', tileMatrixSetID: 'w', format: 'tiles'
# 天地图 XYZ 写法（等价）
https://t0.tianditu.gov.cn/DataServer?T=cia_w&x={x}&y={y}&l={z}&tk=你的token

# ArcGIS World_Imagery（注意是 {z}/{y}/{x}，y 在 x 前面！）
https://services.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}

# 高德（卫片 style=6 / 路网+地名 style=8）
http://webst01.is.autonavi.com/appmaptile?style=6&x={x}&y={y}&z={z}
```

### 3.3 添加、删除与排序

图层管理的核心逻辑在头部组件里。添加时把配置里的 `display` 逐项应用到 `ImageryLayer` 上，并给图层实例**打两个私有标记**——这是整个模块的小机灵：

```javascript
// src/pages/home/components/home-header/index.vue（有删减）
const addLayer = (cfg: LayerConfig) => {
	const provider = createProvider(cfg);
	const layer = viewer.imageryLayers.addImageryProvider(provider);
	if (cfg.options.minimumLevel !== undefined) layer.minimumLevel = cfg.options.minimumLevel;
	if (cfg.options.maximumLevel !== undefined) layer.maximumLevel = cfg.options.maximumLevel;

	const d = cfg.display;
	if (d?.alpha !== undefined) layer.alpha = d.alpha;
	if (d?.brightness !== undefined) layer.brightness = d.brightness;

	/* 关键：打标记，用于后续精准删除 / 排序 */
	layer['__layerId'] = cfg.id;
	layer['__zIndex'] = d?.zIndex ?? 0;
};
```

- **删除**：`removeLayer(id)` 倒序遍历 `imageryLayers`，按 `__layerId` 精确匹配后 `imageryLayers.remove(layer, true)`。Cesium 的图层对象没有业务 id，不打标记就只能按下标删，一排序就全乱。
- **排序**：`imageryLayers` 是个栈式结构（越后添加越在上层），没有直接的 zIndex 概念。项目用两步模拟：先把所有带标记的图层 `lowerToBottom` 沉底，再按 zIndex 升序依次 `raiseToTop` 抬起——两轮循环后层级就是 zIndex 的顺序：

```javascript
const reorderLayersByZIndex = () => {
	const list = [];
	for (let i = 0; i < viewer.imageryLayers.length; i++) {
		const layer = viewer.imageryLayers.get(i);
		if (layer['__layerId']) list.push({ layer, z: layer['__zIndex'] ?? 0 });
	}
	list.sort((a, b) => a.z - b.z); // Cesium 里越后越上
	list.forEach(({ layer }) => viewer.imageryLayers.lowerToBottom(layer));
	list.forEach(({ layer }) => viewer.imageryLayers.raiseToTop(layer));
};
```

- **持久化**：每次勾选变化，把整个图层配置数组（含最新的 `visible`）写进 `localStorage('layerConfig')`；页面初始化时优先读 localStorage、没有才用 `config.js` 的默认值。这里有个真实的坑，`config.js` 第一行注释就在警告它：**改了 config.js 的图层配置但浏览器没清缓存，是不会生效的**——localStorage 里的旧配置永远优先。

图层组件初始化同样要等 viewer：`onMounted` 时如果 `window.viewer` 已存在就直接加载，否则监听 `viewer-ready` 事件后再按 `visible: true` 的配置逐个 `addLayer`。

<a id="sec4"></a>

## 4. 业务数据上球：为什么放弃 Entity，改用 Primitive

### 4.1 数据长什么样

后端接口不返回 GeoJSON 文件，返回的是按设施表分组的业务 JSON：

```json
{
	"TRANSFORMER_STATION": [
		{ "id": "xx1", "name": "XX变电站", "geoType": "points",  "geo": [120.98, 23.64] },
		{ "id": "xx2", "name": "XX线路",   "geoType": "lines",   "geo": [[120.9, 23.6], [121.0, 23.7]] },
		{ "id": "xx3", "name": "XX园区",   "geoType": "polygons", "geo": [[[120.9, 23.6], [121.0, 23.7], [120.95, 23.8]]] }
	]
}
```

`geoType` 决定渲染成点/线/面，`geo` 的坐标格式并不统一（多面、单面、GeoJSON 对象、lon/lat 字段都有）。这决定了两件事：`GeoJsonDataSource.load()` 这条捷径走不通；坐标解析必须自己写兼容层。

### 4.2 Entity 还是 Primitive

Cesium 渲染业务数据有两条 API 路线：

| | Entity API | Primitive API |
| --- | --- | --- |
| 抽象级别 | 高，声明式（`viewer.entities.add({polygon: ...})`） | 低，手动组装 GeometryInstance + Appearance |
| 渲染时机 | 每帧根据属性动态更新 | 一次性 GPU 资源，静态高效 |
| 性能 | 几千个实体就开始吃力 | 数万级实体无压力 |
| 修改样式 | 直接改属性，立即生效 | 通常要销毁重建 |

项目早期用的就是 Entity + DataSource，实体量上来之后（单视口几千个设施、后面还有拓扑网络叠加）帧率撑不住，于是整体迁移到 Primitive——项目源码里还留着那次迁移的注释痕迹（"换成 primitives"）。迁移的代价是样式交互全要自己管：选中高亮、hover 恢复这些 Entity 白送的能力，都得围绕 `scene.pick` 重新设计。换来的是渲染性能和完全的细节控制（`renderOrder`、自定义材质）。

### 4.3 坐标解析：一层兼容各种后端历史格式

```typescript
// src/pages/home/composables/handleDatas.ts（有删减）
export const processCoordinates = (dataItem: any, geometryType: LayerType) => {
	let positions: number[][] = [];
	const rawGeo = dataItem.geo || dataItem.properties.geo;

	if (geometryType === 'polygons') {
		// [[[lon, lat]]] → 多面，取第一个面；[[lon, lat]] → 单个面
		if (Array.isArray(rawGeo) && Array.isArray(rawGeo[0]) && Array.isArray(rawGeo[0][0])) {
			positions = rawGeo[0][0];
		} else {
			positions = rawGeo;
		}
	} else if (geometryType === 'lines') {
		positions = rawGeo[0];
	} else {
		// 点：兼容 [lon, lat]、[[lon, lat]]、{coordinates: [...]}、{lon, lat} 四种
		if (Array.isArray(rawGeo)) {
			positions = typeof rawGeo[0] === 'number' ? [rawGeo] : rawGeo;
		} else if (rawGeo?.coordinates) {
			positions = Array.isArray(rawGeo.coordinates[0])
				? rawGeo.coordinates
				: [rawGeo.coordinates];
		} else if (dataItem.lon && dataItem.lat) {
			positions = [[dataItem.lon, dataItem.lat]];
		}
	}
	return { positions };
};
```

解析出的经纬度统一转 `Cartesian3.fromDegrees`，再用 `BoundingSphere.fromPoints` 求几何中心（面和线需要中心点挂图标/标签）：

```typescript
// BoundingSphere 包围所有顶点，球心即几何中心
const boundingSphere = window.Cesium.BoundingSphere.fromPoints(cartesianPositions);
const cartographic = window.Cesium.Cartographic.fromCartesian(boundingSphere.center);
```

### 4.4 三种几何的 Primitive 工厂

**点 → BillboardCollection**。点不用 Primitive，用 `BillboardCollection` 里的一条 billboard：图标 URL 来自后端样式接口（`baseApiUrl + style.image`），`disableDepthTestDistance: Number.POSITIVE_INFINITY` 让图标永远不被地形/其它几何裁剪——电力设施图标被山体挡住是不能接受的：

```typescript
// src/pages/home/composables/pointPrimitive.ts（有删减）
export const createPointPrimitive = (data: any, position: window.Cesium.Cartesian3, style: any) => {
	return {
		billboardOptions: {
			id: { customAttributes: data },          // 业务数据全塞进 id，拾取时取回
			position: position,
			image: baseApiUrl + _style.image,
			width: _style.width,
			height: _style.height,
			verticalOrigin: window.Cesium.VerticalOrigin.TOP,
			heightReference: window.Cesium.HeightReference.NONE,
			disableDepthTestDistance: Number.POSITIVE_INFINITY // 👈 避免远处被裁剪
		}
	};
};
```

注意 `id` 字段被用来挂 `customAttributes`——Cesium 拾取（`scene.pick`）返回的对象会带上这个 id，业务数据就在这一刻"回到"前端手里。这是 Primitive 路线里传递业务上下文的标准手法。

**面 → PolygonGeometry + EllipsoidSurfaceAppearance**。面创建完还要在中心点补一个 billboard 图标，并且和面建立**双向关联**（点图标 → `associatedPrimitive` 指回面，面 → `associatedBillboard` 指向图标），hover 图标时高亮整个面：

```typescript
// src/pages/home/composables/polygonPrimitive.ts（有删减）
const instance = new window.Cesium.GeometryInstance({
	geometry: new window.Cesium.PolygonGeometry({
		polygonHierarchy: new window.Cesium.PolygonHierarchy(positions),
		vertexFormat: window.Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
		perPositionHeight: false,
		height: 20 // 离地高度，避免和底图 z-fighting
	}),
	id: { customAttributes: data }
});

const polygon = new window.Cesium.Primitive({
	geometryInstances: instance,
	appearance: new window.Cesium.EllipsoidSurfaceAppearance({
		material: window.Cesium.Material.fromType('Color', {
			color: window.Cesium.Color.fromCssColorString(style?.material || '#97FFFF')
		}),
		translucent: true
	}),
	allowPicking: true,    // ✅ 显式启用拾取
	asynchronous: false,   // ✅ 同步创建，立即可拾取
	renderOrder: 0         // 面在底层
});
```

`asynchronous: false` 值得展开一句：默认为 true 时几何在 Web Worker 里分帧构建，创建后的一两帧内 `scene.pick` 拾取不到它，用户刚画完就点会"点空"。数据量不大时关掉异步，换来即时可交互。

**线 → PolylineGeometry + PolylineMaterialAppearance**。线有两个特殊处理：`renderOrder: 100` 保证线画在面之上；`customAttributes.defaultStyle` 把当前样式快照存进 primitive，hover 恢复时不用再查样式表：

```typescript
// src/pages/home/composables/linePrimitive.ts（有删减）
const polyline = new window.Cesium.Primitive({
	geometryInstances: [new window.Cesium.GeometryInstance({
		geometry: new window.Cesium.PolylineGeometry({
			positions,
			width: lineWidth,
			vertexFormat: window.Cesium.PolylineMaterialAppearance.VERTEX_FORMAT
		})
	})],
	appearance: new window.Cesium.PolylineMaterialAppearance({ material }),
	renderOrder: 100 // 线永远在面之上
});
(polyline as any).customAttributes = { ...data, defaultStyle }; // ✅ 保存默认样式用于恢复
```

### 4.5 全局集合：实体的"户口"统一管理

所有 primitive 进出两个全局集合，而不是直接 `scene.primitives.add()`。集合挂在 scene 上一次，之后只操作集合，销毁、清空、遍历都有唯一入口：

```typescript
// src/pages/home/composables/use-load-data.ts
export let globalPrimitiveCollection: window.Cesium.PrimitiveCollection | null = null;
export let globalBillboardCollection: window.Cesium.BillboardCollection | null = null;

export function initCollections() {
	if (!globalPrimitiveCollection) {
		globalPrimitiveCollection = new window.Cesium.PrimitiveCollection();
		globalPrimitiveCollection.destroyPrimitives = true; // 移除时自动销毁内部 primitive
	}
	if (!globalBillboardCollection) {
		globalBillboardCollection = new window.Cesium.BillboardCollection();
		globalBillboardCollection.depthTest = false; // 图标始终显示在最上层
	}
}
```

<a id="sec5"></a>

## 5. 视口驱动的增量加载：只请求"看得见"的数据

这一节是整个项目的核心。设施全量有几十万条，一次性上球既不现实也没必要——**用户看到什么，就加载什么**。

### 5.1 触发时机：相机静止 400ms 后

拖动地图的过程中相机在连续变化，直接监听 `moveEnd` 会连环触发请求。两层过滤：

```typescript
// use-load-data.ts · setupClickHandler 内
let loadDebounce: number | null = null;
window.viewer?.camera.moveEnd.addEventListener(() => {
	if (loadDebounce) clearTimeout(loadDebounce);
	loadDebounce = window.setTimeout(() => {
		handleCameraMoveEnd();
	}, 400);
});
```

```typescript
// src/pages/home/composables/handleEventPicks.ts
export const handleCameraMoveEnd = () => {
	if (cesiumlStore.isTopologyMode) return; // 拓扑展示模式不打扰

	const height = window.viewer?.camera.positionCartographic.height;
	if (1000 < height && height <= 400000) { // 只在 1km~400km 高度之间加载
		loadData();
	}
};
```

高度窗口是个业务决策：低于 1km 视口太小查不出东西还频繁触发；高于 400km（约卫星视角）视口覆盖太广，后端空间查询代价大，干脆不加载。

### 5.2 视口边界：computeViewRectangle 及其三个分支

请求参数需要"当前视口的经纬度矩形"。Cesium 提供了现成的 `camera.computeViewRectangle()`，但 2D/3D/Columbus 三种模式下它的行为和可靠性不同，所以 store 里做了分支处理：

```typescript
// src/store/cesium.ts（有删减）
async getCartographicLimitsData() {
	const currentMode = window.viewer?.scene.mode;
	const is2DOrCVMode = currentMode === 1 /* SCENE2D */ || currentMode === 2 /* COLUMBUS_VIEW */;

	if (is2DOrCVMode) return this.get2DViewportBounds();
	return this.get3DViewportBounds();
}

get3DViewportBounds() {
	let rectangle = window.viewer?.camera.computeViewRectangle();
	if (!rectangle) return this.calculateViewportBoundsFallback(); // 计算失败 → 兜底

	this.cartographicLimitsData = {
		lowerLeftJing: window.Cesium.Math.toDegrees(rectangle.west),
		lowerLeftWei: window.Cesium.Math.toDegrees(rectangle.south),
		upperRightJing: window.Cesium.Math.toDegrees(rectangle.east),
		upperRightWei: window.Cesium.Math.toDegrees(rectangle.north)
	};
	return this.cartographicLimitsData;
}
```

2D 分支除了同样的转换，还多两道修正：视口范围太小（< 30°）时以中心为准扩到 30°，太大（> 180°）时压回 180°——2D 模式下 `computeViewRectangle` 在极端缩放时会给出不合理的边界。

**兜底方案**是按相机高度查表估一个范围（0.05° ~ 30° 共九档），当 `computeViewRectangle` 返回 undefined（相机朝天上、贴地等极端姿态）时顶上：

```typescript
calculateViewportBoundsFallback() {
	const { longitude, latitude, height } = window.viewer.camera.positionCartographic;
	let range = 1.0;
	if (height < 10000) range = 0.05;         // 非常近的视角
	else if (height < 100000) range = 1.0;    // 中近视角
	else if (height < 1000000) range = 3.0;   // 远视角
	else if (height < 10000000) range = 15.0; // 全球视角
	else range = 30.0;
	// 以相机正下方为中心，range 为半径构造矩形……
}
```

### 5.3 请求与组装

边界 + 勾选的设施类型（tableIds）作为参数发给后端，由后端做空间过滤：

```typescript
// src/api/cesiumApi.ts
export async function getFacilityPrimitives(data: any) {
	return request<any, ResponseData>({
		url: `/${managementModel}/aggregation/get/range/table`,
		method: 'POST',
		data // { tableIds: [...勾选的设施类型], boundary: { lowerLeftJing, lowerLeftWei, upperRightJing, upperRightWei } }
	});
}

// use-load-data.ts
export const loadData = async () => {
	await cesiumlStore.getCartographicLimitsData(); // 算视口边界
	const _data = {
		tableIds: facilitiesManage.facilitiesCheckedList,
		boundary: cesiumlStore.cartographicLimitsData
	};
	if (_data.tableIds.length > 0 && Object.values(cesiumlStore.cartographicLimitsData).some(v => v !== 0)) {
		const res = await getFacilityPrimitives(_data);
		if (res.code == 200) await loadDataAsPrimitives(res.data || {});
	}
};
```

### 5.4 增量 diff：不闪、不重、不漏

`loadDataAsPrimitives` 是整个管线里最精华的函数。它的前身有两个，都在源码注释里留着"尸体"，正好构成一部踩坑进化史：

- **v1：全量加载 + 清理**——每次把缓存里"不在本次结果中"的实体移除，新实体加入。问题是每次请求间隔里实体有增有减，界面出现可见的闪烁；
- **v2：清空重载**——干脆每次清空所有实体重新创建。闪烁更严重（线实体创建瞬间还会"抖动"），而且反复创建销毁 GeometryInstance 性能浪费；
- **v3（现行）：增量 diff**——三步走，只动真正变化的部分：

```typescript
// use-load-data.ts（有删减）
export const loadDataAsPrimitives = async (data: Record<string, any>) => {
	if (!window.viewer || window.viewer.isDestroyed()) return;
	initCollections();

	// ① 构建本次视口应该显示的 ID 集合（按设施类型分组）
	const currentIdsByType = new Map<string, Set<string>>();
	for (const [key, list] of Object.entries(data)) {
		if (Array.isArray(list)) {
			const ids = new Set(list.filter(item => item.id).map(item => item.id));
			currentIdsByType.set(key, ids);
		}
	}

	// ② 移除"已离开视口/被取消勾选"的实体
	for (const [key, existingMap] of rawDataMap) {
		const currentIds = currentIdsByType.get(key) || new Set();
		for (const [id, obj] of existingMap.entries()) {
			if (!currentIds.has(id)) {
				if (obj.primitive && globalPrimitiveCollection?.contains(obj.primitive)) {
					globalPrimitiveCollection.remove(obj.primitive);
				}
				if (obj.billboard && globalBillboardCollection?.contains(obj.billboard)) {
					globalBillboardCollection.remove(obj.billboard);
				}
				existingMap.delete(id); // 缓存同步删除
			}
		}
	}

	// ③ 处理新数据 —— processGeoToPrimitive 内部按 id 跳过已存在的，只有新 id 会走创建流程
	const processed = await processGeoToPrimitive(data);
	processed.forEach(item => {
		if (item.primitive && globalPrimitiveCollection) globalPrimitiveCollection.add(item.primitive);
		if (item.billboardOptions && globalBillboardCollection) {
			const billboard = globalBillboardCollection.add(item.billboardOptions);
			if (item.primitive) {
				(item.primitive as any).customAttributes.associatedBillboard = billboard; // 双向关联
			}
			// billboard 引用回填缓存……
		}
	});

	treeObj.treeData = await processTreeData(); // 左侧设施树同步更新
};
```

去重的关键在 `processGeoToPrimitive` 里的这一行——缓存结构 `rawDataMap: Map<表名, Map<id, {primitive, billboard}>>` 天然就是去重表：

```typescript
for (const dataItem of value) {
	if (!dataItem.id) continue;
	// ⚠️ 关键：如果已存在，跳过创建（避免重复）
	if (existingData.has(dataItem.id)) continue;
	// ……解析坐标 → 求 center → 按 geoType 分发到三个工厂函数
}
```

效果：相机平移时，离开视口的实体被精准移除、新进入的实体被创建、视口内已有的实体**一个都不动**——没有闪烁，没有重复请求渲染，内存里也永远只有"看得见"的那批。

### 5.5 缓存治理：勾选、卸载与定时清理

设施类型在左侧树上可勾选/取消，联动逻辑同样走增量化：新增的 typeCode 立即请求加载，取消的 typeCode 从两个全局集合移除并记入 `loadedFacilitiesCache`（记下取消时间）。因为"取消勾选只是隐藏，用户很可能马上又勾回来"，数据并不立刻销毁；真正释放靠定时任务——每分钟扫一遍缓存，超过 5 分钟（`MAX_CACHE_DURATION`）没被重新勾选的类型才真正清掉：

```typescript
// handleDatas.ts（有删减）
export const startCleanupTask = () => {
	setInterval(() => {
		const now = Date.now();
		cesiumlStore.loadedFacilitiesCache.forEach((value, typeCode) => {
			if (now - value.loadTime > cesiumlStore.MAX_CACHE_DURATION) {
				// 释放该类型数据 & 从设施树移除 & 删除缓存记录
			}
		});
	}, 1 * 60 * 1000);
};
```

这是典型的**空间换时间 + 延迟回收**：勾选切换是高频操作，5 分钟内恢复就免请求；长时间不用则彻底回收，内存不无限增长。

<a id="sec6"></a>

## 6. 让数据"活"起来：实时推送与交互联动

### 6.1 WebSocket 事件 → 呼吸图标

后端通过 WebSocket 推送事件变更，前端收到任何消息都重新拉取"进行中事件列表"，把事件关联的设施渲染成**呼吸告警图标**：

```typescript
// src/pages/home/index.vue（有删减）
const { messages } = useWebSocket(window.baseConfig?.baseWsUrl + '/ws/event');

const getEventList = async () => {
	const res = await getRunningEventList();
	if (res.code === 200) {
		eventList.value = res.data;
		createEventPrimitive(eventList.value); // 事件设施上球
	}
};

watch(messages, () => getEventList(), { deep: true }); // 收到推送 → 刷新
```

`createEventPrimitive` 的实现有几个细节值得说：图标用独立的 `BillboardCollection` 承载并 `raiseToTop` 保证压在所有设施图标之上；每个 billboard 打上 `type: 'eventFlag'` 标记，点击事件的处理器里优先识别它、走事件信息框分支；多个设施同时告警时动画起始时间错开（`animationTime: facilityIndex * 0.5`），避免"齐步走"的呆板感。

呼吸动画本体就是一个 `requestAnimationFrame` 循环 + 正弦函数驱动 scale：

```typescript
// src/pages/home/composables/eventPrimitive.ts（有删减）
function startAnimation(billboardArray) {
	const animate = () => {
		const time = performance.now() * 0.001;
		billboardArray.forEach(item => {
			// sin 波形映射到 [0,1]，再放大成 [scaleBase, scaleBase + scaleRange]
			const progress = Math.sin(time * item.animationSpeed + item.animationTime) * 0.5 + 0.5;
			item.billboard.scale = item.scaleBase + item.scaleRange * progress;
		});
		animationFrameId = requestAnimationFrame(animate);
	};
	animate();
}
```

### 6.2 拾取与高亮状态机

`scene.pick` 拿到 primitive 后，样式状态由一个四态状态机管理：`normal / moveIn / click / rightclick`。悬停高亮、移出恢复、点击选中（保持高亮，右键取消）、右键弹菜单。线的样式切换因为 Primitive 不可变，走的是**销毁重建**：改线宽 = 用缓存的原坐标 + 新样式新建一个 GeometryInstance 替换旧的——这也是 Primitive 路线的代价之一，好在有 `defaultStyle` 快照，恢复不难。

### 6.3 flyTo 与屏幕坐标联动

Vue 弹窗（信息框、右键菜单）要贴着三维实体显示，靠的是 `SceneTransforms.wgs84ToWindowCoordinates`（新版叫 `worldToWindowCoordinates`）把实体坐标投到屏幕坐标，再交给 Vue 层绝对定位。飞行定位则是 `camera.flyTo` 的 complete 回调里做这一次投影——飞行结束后实体才在视口内，此时算屏幕坐标才是准的。

<a id="sec7"></a>

## 7. 渲染亮点一瞥：GLSL 流动线、球面弧线、呼吸图标

这三个特效是拓扑展示模式的门面，各讲一个知识点。

### 7.1 自定义 GLSL 流动线材质

Cesium 的 `Material` 体系支持注入自定义 shader：把 GLSL 源码和默认 uniforms 注册进 `Material._materialCache`，之后就能像内置材质一样 `Material.fromType('PolylineFlow', {...})` 使用：

```typescript
// src/pages/home/composables/linePrimitive.ts（有删减）
export function registerFlowLineMaterial() {
	window.Cesium.Material.PolylineFlowType = 'PolylineFlow';
	window.Cesium.Material.PolylineFlowSource = `
		uniform vec4 color;
		uniform vec4 glowColor; // 光影颜色
		uniform float speed;
		uniform float percent;
		uniform float gradient;

		czm_material czm_getMaterial(czm_materialInput materialInput) {
			czm_material material = czm_getDefaultMaterial(materialInput);
			float st = materialInput.st.s;             // 沿线方向的纹理坐标 0→1
			float time = czm_frameNumber * speed / 1000.0;
			float currentPos = fract(time - st);       // 流动的"光头"位置
			float trailPos = smoothstep(0.0, percent, currentPos);          // 拖尾
			float glowPos = smoothstep(0.0, gradient * percent, currentPos)
			              * smoothstep(percent, percent * (1.0 - gradient), currentPos); // 辉光

			material.diffuse = color.rgb;
			material.alpha = 0.6;
			material.emission = glowPos * glowColor.rgb;
			return material;
		}
	`;

	window.Cesium.Material._materialCache.addMaterial('PolylineFlow', {
		fabric: {
			type: 'PolylineFlow',
			uniforms: {
				color: new window.Cesium.Color(0.1, 0.8, 0.4, 0.8),
				glowColor: new window.Cesium.Color(1.0, 1.0, 0.0, 1.0),
				speed: 5.0, percent: 0.15, gradient: 0.4
			},
			source: window.Cesium.Material.PolylineFlowSource
		},
		translucent: function () { return true; }
	});
}
```

流动感的核心就两行：`czm_frameNumber`（Cesium 内置的帧计数 uniform，每帧自增）驱动时间，`fract(time - st)` 让"亮段"沿线坐标 st 周期性扫过。注册一次全局可用，拓扑模式下的线全部换成这个材质，网络"能量流动"的观感就出来了。

### 7.2 球面抛物线弧线（航线）

两点间的航线不能画直线（会穿过地球），要在球面上撑一条弧线。做法：对起点终点的**单位方向向量**做球面插值得到路径点，再用抛物线公式 `h * 4t * (1 - t)` 沿弧顶法线抬升——t=0.5 时恰好达到最大高度 h：

```typescript
// linePrimitive.ts（有删减）
export function createArcPositions(start, end, segments = 50, heightFromEllipsoid = 20000) {
	const positions = [];
	const startDir = Cesium.Cartesian3.normalize(start, new Cesium.Cartesian3());
	const endDir = Cesium.Cartesian3.normalize(end, new Cesium.Cartesian3());
	let midDir = Cesium.Cartesian3.add(startDir, endDir, new Cesium.Cartesian3());
	Cesium.Cartesian3.normalize(midDir, midDir); // 弧顶法线方向
	const earthRadius = Cesium.Ellipsoid.WGS84.maximumRadius;

	for (let i = 0; i <= segments; i++) {
		const t = i / segments;
		let dir = Cesium.Cartesian3.lerp(startDir, endDir, t, new Cesium.Cartesian3());
		Cesium.Cartesian3.normalize(dir, dir);
		const surfacePoint = Cesium.Cartesian3.multiplyByScalar(dir, earthRadius, new Cesium.Cartesian3());
		const arcHeight = heightFromEllipsoid * 4.0 * t * (1.0 - t); // 抛物线
		positions.push(Cesium.Cartesian3.add(
			surfacePoint,
			Cesium.Cartesian3.multiplyByScalar(midDir, arcHeight, new Cesium.Cartesian3())
		));
	}
	return positions;
}
```

工程上还有两个防御性细节：分段数按两点距离自适应（`distance / 100000`，夹在 20~200 之间），近距离航线不至于棱角分明、远距离不至于顶点爆炸；两点几乎对跖（midDir 趋近零向量）时用叉乘另取法线，避免 normalize 除零。

### 7.3 交互式画多边形

区域圈选功能用的是 Entity API 的另一件武器——`CallbackProperty`：把预览面的 `hierarchy` 属性定义成一个回调函数，鼠标每点一个新顶点，回调返回的形状立即变化，**不用重建实体**就得到"橡皮筋"跟手效果。绘制完成后把顶点 `Cartesian3` 转回 WGS84 经纬度数组交给上层业务，预览用的临时实体随即销毁。绘制期间会置一个全局 `isDrawingPolygon` 标志，右键菜单等交互全部让路——两个交互系统共存时，状态互斥一定要显式管理。

<a id="sec8"></a>

## 8. 写在最后

把整条链路的选型收成一张表：

| 需求 | 方案 | 关键文件 |
| --- | --- | --- |
| 内网离线 + 快速构建 | public 静态引入 + 全局变量，`global: 'globalThis'` + `CESIUM_BASE_URL` | `index.html`、`vite.config.ts`、`global.d.ts` |
| 阻断外网请求 | `Ion.defaultServer = 'undefined'` + 不加载默认底图 | `cesiumMap.vue`、`config.js` |
| 多源底图配置化 | layers 配置 + Provider 工厂 + `__layerId`/`__zIndex` 标记 + localStorage 持久化 | `config.js`、`layerFactory.ts`、`home-header/index.vue` |
| 海量业务数据渲染 | 后端 JSON → 坐标兼容解析 → 点 Billboard / 线面 Primitive 工厂 | `handleDatas.ts`、`pointPrimitive.ts`、`linePrimitive.ts`、`polygonPrimitive.ts` |
| 只加载看得见的数据 | `computeViewRectangle` 视口边界（2D/3D/兜底三分支）+ 高度窗口 + 400ms 防抖 | `store/cesium.ts`、`handleEventPicks.ts` |
| 拖动不闪烁、内存不涨 | 增量 diff（rawDataMap 按 id 去重）+ 取消勾选延迟回收（5 分钟定时清理） | `use-load-data.ts`、`handleDatas.ts` |
| 实时告警 | WebSocket 推送 → requestAnimationFrame 正弦呼吸图标 | `home/index.vue`、`eventPrimitive.ts` |
| 拓扑/航线特效 | 自定义 GLSL 材质注入 `_materialCache`、球面插值 + 抛物线抬升 | `linePrimitive.ts` |

以及这个项目**刻意没用**的东西和原因，同样是选型的一部分：

- **地形**：没有地形数据源，`depthTestAgainstTerrain` 显式关闭，避免无地形时的坐标漂移；
- **3D Tiles**：没有倾斜摄影/建筑模型需求；
- **GeoJsonDataSource**：数据是业务 JSON 不是 GeoJSON，且 Entity/DataSource 路线扛不住实体量；
- **CZML/KML**：没有时序驱动需求。

**一句话记住**：Cesium 项目的数据加载，本质是回答"**数据从哪来、什么时候来、来了怎么高效上球、走了怎么干净下球**"——视口驱动回答了"什么时候"，增量 diff 和缓存治理回答了"怎么高效"和"怎么干净"，剩下的才是"从哪来"的工程问题。

### 文件地图

```text
infrastructure-data-plus/
├── index.html                        # 全局引入 /cesium/Cesium.js 与 /config.js
├── vite.config.ts                    # define(global/CESIUM_BASE_URL) + worker es
├── global.d.ts                       # window.Cesium / viewer / baseConfig 类型
├── public/
│   ├── config.js                     # 运行时配置：图层、相机视角、后端地址
│   └── cesium/                       # CesiumJS 1.123 构建产物（copy-cesium 脚本复制）
└── src/
    ├── utils/layerFactory.ts         # XYZ/WMS/WMTS/ArcGIS Provider 工厂
    ├── api/cesiumApi.ts              # 视口范围查询 / 样式 / 拓扑接口
    ├── store/cesium.ts               # 视口边界计算、选中实体、缓存状态
    ├── types/cesium.ts               # 样式常量、图层类型、图标映射
    └── pages/home/
        ├── index.vue                 # WebSocket → 事件列表 → 呼吸图标
        ├── components/
        │   ├── cesium-map/cesiumMap.vue    # Viewer 初始化 / 2D-3D 切换
        │   └── home-header/index.vue       # 图层勾选 / 排序 / 持久化
        └── composables/
            ├── use-load-data.ts      # ★ 数据管线核心：loadData → 增量 diff
            ├── handleDatas.ts        # 坐标解析 / 中心点 / 勾选联动 / 定时清理
            ├── pointPrimitive.ts     # 点 → Billboard
            ├── linePrimitive.ts      # 线 → Polyline + GLSL 流动材质 + 弧线
            ├── polygonPrimitive.ts   # 面 → Polygon + 中心图标双向关联
            ├── eventPrimitive.ts     # 告警呼吸图标
            ├── handleEventPicks.ts   # 悬停/点击/右键 + 高度过滤
            ├── use-map-style.ts      # 四态样式状态机
            ├── use-flyTo-map.ts      # 飞行定位 + 屏幕坐标投影
            └── drawPolygon.ts        # CallbackProperty 交互画多边形
```

### 参考

- [CesiumJS 官方文档 - Quickstart（构建产物的构成）](https://cesium.com/learn/cesiumjs-learn/cesium-for-absolute-beginners/)
- [CesiumJS 官方文档 - ImageryProvider 体系](https://cesium.com/learn/cesiumjs/ref-doc/global.html#ImageryProvider)
- [Vite 官方文档 - define 与 Worker 配置](https://vitejs.dev/config/shared-options.html)
