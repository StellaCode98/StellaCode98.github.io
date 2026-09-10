---
title: 封装一个 Vue3 通用 ECharts 图表组件：options 驱动、容器自适应与事件透传
date: 2026-09-10 18:20:00
description: 每个图表页面都在复制粘贴 init / setOption / resize / dispose 四部曲，坑还都藏在细节里：容器没高度白屏、侧边栏折叠图不缩、实例被 Vue 深度代理、忘了 dispose 内存泄漏。把 echarts 封装成 options 驱动的通用图表组件：图表类型只是 options.series[].type 的字符串所以什么图都能画、ResizeObserver 做容器级自适应、事件白名单透传、暗黑主题销毁重建，最后附按需引入配置与完整代码。
categories:
  - Vue 进阶
tags:
  - vue3
  - echarts
  - 组件封装
  - typescript
  - 数据可视化
  - 中后台
---

写过数据大屏或中后台仪表盘的同学都有体感：一个页面四五张图，每张图都要 `init`、`setOption`、监听 `resize`、卸载时 `dispose`，四步曲抄四遍。更要命的是坑全藏在细节里——容器没有确定高度白屏、侧边栏折叠之后图不跟着缩、echarts 实例塞进 `ref()` 被 Vue 深度代理出幺蛾子、页面来回切换内存泄漏。

这篇把之前项目里沉淀的 ECharts 通用组件整理出来。组件很小，百来行，但设计目标一句话能说清：**使用者只关心一个 `options`**。柱状、折线、饼图、雷达、K 线……图表类型从来不是组件的维度，它只是 `options.series[].type` 里的一个字符串——所以一个组件就能装下所有类型的图表。

<!-- more -->

## 一、API 设计：一个必传 prop、九个可选、一组事件透传、一个 expose

老规矩，封装组件第一步是把对外契约定稳：

```text
props
  options    : EChartsOption      必传。图表配置，唯一数据源
  width      : string             容器宽度
  height     : string             容器高度（echarts 容器必须有确定高度）
  theme      : string | object    主题名或主题对象（'dark' 内置）
  renderer   : 'canvas' | 'svg'   渲染器
  autoResize : boolean            容器尺寸变化时自动 resize
  loading    : boolean            加载态（echarts 自带 showLoading）
  isEmpty    : boolean            空数据态（遮罩 + 清空图表）
  emptyText  : string             空数据态文案
  notMerge   : boolean            setOption 是否整图替换

emits
  click / dblclick / mouseover / …   echarts 常用事件白名单透传
  legendselectchanged / datazoom     图例开关、区域缩放

expose
  getInstance()   原生 echarts 实例（escape hatch）
  resize()        手动触发自适应
  setOption()     手动设置配置
  clear()         清空图表
```

props 明细（接口上也都写了 JSDoc，IDE 悬停可见）：

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `options` | `EChartsOption` | 必传 | 图表配置，**唯一数据源**：画什么图、什么样式、什么数据全在这里 |
| `width` | `string` | `'100%'` | 容器宽度，一般撑满父级 |
| `height` | `string` | `'400px'` | 容器高度。echarts 必须有确定高度，父级没高度时必须显式传 |
| `theme` | `string \| object` | — | 主题名（如 `'dark'`）或自定义主题对象；**变更时销毁重建** |
| `renderer` | `'canvas' \| 'svg'` | `'canvas'` | 渲染器。canvas 性能与交互均衡；svg 在超大数据量下更省内存、可矢量导出 |
| `autoResize` | `boolean` | `true` | 用 ResizeObserver 监听**容器**尺寸变化自动 resize |
| `loading` | `boolean` | `false` | 加载态，内部调 `showLoading` / `hideLoading` |
| `isEmpty` | `boolean` | `false` | 空数据态：显示遮罩并 `clear()` 图表，避免画一个空坐标轴 |
| `emptyText` | `string` | `'暂无数据'` | 空数据态文案 |
| `notMerge` | `boolean` | `true` | `setOption` 不合并模式，options 整图替换，见下文语义讨论 |

事件透传（回调参数都是 echarts 原生的事件对象 `params`）：

| 事件 | 触发时机 | 常用字段 |
| --- | --- | --- |
| `click` / `dblclick` | 点击/双击图表元素（柱子、扇区、数据点） | `params.name`、`params.value`、`params.seriesName` |
| `mousedown` / `mouseup` / `mouseover` / `mouseout` | 鼠标交互 | 同上 |
| `contextmenu` | 右键 | 同上 |
| `legendselectchanged` | 图例开关 | `params.selected`、`params.name` |
| `datazoom` | dataZoom 缩放 | `params.start`、`params.end` |
| `brushselected` | 框选 | `params.selected` |

expose：

| 方法 | 说明 |
| --- | --- |
| `getInstance()` | 拿原生 echarts 实例做组件没封装的事：`getDataURL` 导出图片、`convertToPixel` 坐标换算、`dispatchAction` 触发 tooltip…… |
| `resize()` | 布局变化时手动触发（`autoResize` 关掉的场景用） |
| `setOption(options)` | 手动设置配置（一般用不到，改 `options` prop 即可） |
| `clear()` | 清空图表 |

## 二、模板：确定高度的容器 + 空态遮罩

```html
<template>
  <div class="echarts-wrapper" :style="{ width, height }" v-bind="$attrs">
    <div ref="chartRef" class="echarts-container"></div>
    <div v-if="isEmpty" class="echarts-empty">
      <el-empty :description="emptyText" :image-size="80" />
    </div>
  </div>
</template>

<style scoped>
.echarts-wrapper {
  position: relative;
  width: 100%;
  height: 400px;
}
.echarts-container {
  width: 100%;
  height: 100%;
}
.echarts-empty {
  position: absolute;
  inset: 0;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--el-bg-color);
}
</style>
```

三个点：

**1. 两层 div，外层定尺寸、内层给 echarts。** `echarts.init` 会接管目标 DOM 的绘制，把宽高、定位、遮罩这些「布局职责」留在外层，职责干净：尺寸变化监听绑在内层，空态遮罩盖在外层，互不干扰。

**2. 默认高度 400px 不是随手写的。** echarts 容器高度为 0 或 `auto` 时 `init` 会得到一张 0 高度的图（白屏），这是图表组件的第一号坑。给一个兜底默认值，同时把「必须显式给高度」写进文档——`height: 100%` 在父级没有确定高度时同样是 0，这种情况要么父级定高，要么传具体像素。

**3. `v-bind="$attrs"` 照旧透传。** 和表格那篇同一个原则：父组件想加 `class`、`style` 调布局不设限，封装不是墙。

## 三、脚本：把 echarts 生命周期收进组件

完整脚本（Vue 3 `<script setup>` + TS）：

```ts
<script setup lang="ts">
import { nextTick, onActivated, onMounted, onUnmounted, ref, shallowRef, watch } from 'vue'
import * as echarts from 'echarts'
import type { EChartsOption, EChartsType } from 'echarts'

interface Props {
  /** ECharts 配置，图表的唯一数据源 */
  options: EChartsOption
  /** 容器宽度，默认撑满父级 */
  width?: string
  /** 容器高度，默认 400px（echarts 容器必须有确定高度） */
  height?: string
  /** 主题名（'dark' 内置）或主题配置对象，变更时销毁重建 */
  theme?: string | object
  /** 渲染器：canvas 性能均衡，svg 大数据量更省内存、可矢量导出 */
  renderer?: 'canvas' | 'svg'
  /** 跟随容器尺寸自适应（ResizeObserver），默认 true */
  autoResize?: boolean
  /** 加载态，走 echarts 自带 showLoading */
  loading?: boolean
  /** 空数据态：显示遮罩并清空图表 */
  isEmpty?: boolean
  /** 空数据态文案 */
  emptyText?: string
  /** setOption 不合并模式，默认 true：options 是唯一数据源，整图替换 */
  notMerge?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  width: '100%',
  height: '400px',
  theme: undefined,
  renderer: 'canvas',
  autoResize: true,
  loading: false,
  isEmpty: false,
  emptyText: '暂无数据',
  notMerge: true,
})

/** 只转发常用事件；冷门事件通过 getInstance() 自己 on */
type ChartEventName =
  | 'click' | 'dblclick' | 'mousedown' | 'mouseup'
  | 'mouseover' | 'mouseout' | 'contextmenu'
  | 'legendselectchanged' | 'datazoom' | 'brushselected'

const EVENTS: ChartEventName[] = [
  'click', 'dblclick', 'mousedown', 'mouseup',
  'mouseover', 'mouseout', 'contextmenu',
  'legendselectchanged', 'datazoom', 'brushselected',
]

const emit = defineEmits<{ (e: ChartEventName, params: any): void }>()

const chartRef = ref<HTMLDivElement>()
// 关键：实例必须放 shallowRef，不能放 ref
const chart = shallowRef<EChartsType>()
let resizeObserver: ResizeObserver | null = null

const initChart = () => {
  if (chart.value || !chartRef.value) return   // 防重复 init
  chart.value = echarts.init(chartRef.value, props.theme, { renderer: props.renderer })
  EVENTS.forEach((name) => {
    chart.value?.on(name, (params) => emit(name, params))
  })
  chart.value.setOption(props.options, { notMerge: props.notMerge })
  props.loading && chart.value.showLoading()
}

const renderChart = (options: EChartsOption) => {
  chart.value?.setOption(options, { notMerge: props.notMerge })
}

onMounted(() => {
  nextTick(initChart)
  if (props.autoResize) {
    resizeObserver = new ResizeObserver(() => chart.value?.resize())
    resizeObserver.observe(chartRef.value!)
  }
})

// options 变更：整图重设。deep 是为了照顾「原地改 series[0].data」的用法
watch(
  () => props.options,
  (val) => {
    if (props.isEmpty) return
    chart.value ? renderChart(val) : initChart()
  },
  { deep: true },
)

watch(
  () => props.loading,
  (val) => (val ? chart.value?.showLoading() : chart.value?.hideLoading()),
)

watch(
  () => props.isEmpty,
  (val) => (val ? chart.value?.clear() : renderChart(props.options)),
)

// 主题/渲染器是 init 时决定的，变更只能销毁重建（暗黑模式切换走这里）
watch(
  [() => props.theme, () => props.renderer],
  () => {
    chart.value?.dispose()
    chart.value = undefined
    nextTick(initChart)
  },
)

// keep-alive 缓存页切回来时容器尺寸可能已变，主动对齐一次
onActivated(() => chart.value?.resize())

onUnmounted(() => {
  resizeObserver?.disconnect()
  resizeObserver = null
  chart.value?.dispose()
  chart.value = undefined
})

defineExpose({
  /** 拿原生实例做组件没封装的事（getDataURL、convertToPixel、dispatchAction…） */
  getInstance: () => chart.value,
  /** 供父组件在布局变化时手动触发 */
  resize: () => chart.value?.resize(),
  setOption: renderChart,
  clear: () => chart.value?.clear(),
})
</script>
```

### 实例放 shallowRef：最隐蔽的坑

`const chart = ref()` 看起来人畜无害，但 Vue 3 的 `ref` 会深度递归代理对象——echarts 实例内部有大量私有状态和 canvas 引用，被 Proxy 包一层之后轻则无谓的性能开销（每次内部访问都走一遍代理），重则行为异常（一些内部逻辑用对象引用做 WeakMap key，被代理后取不到）。`shallowRef` 只代理 `.value` 这一层，实例本体原样透传。同类的正确姿势还有给实例套 `markRaw`。这个坑不报错、不警告，纯靠踩。

### ResizeObserver：侧边栏折叠才是 resize 的盲区

绝大多数示例代码用 `window.addEventListener('resize', ...)`，但中后台的真实布局变化有一半不触发 window resize：**侧边栏折叠、分栏拖拽、卡片收起展开**——窗口没变，是容器变了。`ResizeObserver` 监听的是容器元素本身，这些场景全能覆盖。挂载时 `observe`，卸载时 `disconnect`，一个不漏。唯一要注意它是观察内层 `chartRef`，而内层宽高 100% 跟随外层，所以外层布局怎么变都能捕捉到。

### notMerge 默认 true：options 是唯一数据源

`setOption` 默认是**合并模式**——新配置和旧配置按 key 合并。听起来贴心，实际是状态漂移的温床：把 series 从两条改成一条，旧的第二条还在；删掉的 `markLine` 残留半年；切换图表类型时旧系列的旧样式混进来。合并模式适合「手把手增量更新」的心智，而本组件的设计是「`options` 是唯一数据源」——组件无状态，渲染结果永远等于当前 options，**整图替换**才是最可预期的语义。

代价也要说清楚：`notMerge: true` 会重置交互状态，比如 dataZoom 缩放位置、图例开关状态会在每次 options 更新后回到配置值。需要「数据更新但交互状态保留」的场景（如实时推送的监控图），传 `:notMerge="false"` 即可——语义收在 prop 上，两种心智都有出口。

### dispose 与 keep-alive

`onUnmounted` 里 `disconnect + dispose` 是防止内存泄漏的标配，漏了 dispose，canvas、事件监听、定时器全套留在内存里，页面来回切几次堆就上去了。

keep-alive 是额外加分项：缓存的页面切回来时**组件不会重新 mount**，但容器尺寸可能已经变了（别的页面动过布局），`onActivated` 里补一次 `resize`，图不会以旧尺寸定格。

### 事件白名单透传

echarts 有几十种事件，不可能也不应该全转发。取常用的 10 个（鼠标六件套 + 右键 + 图例开关 + dataZoom + 框选）在 `initChart` 时统一 `on` 一遍，`emit` 出去的 `params` 就是 echarts 原生事件对象——父组件 `@click="onChartClick"` 拿到的和直接用 echarts 一模一样。冷门事件走 `getInstance()` 自己绑，不算阉割。

### 主题切换 = 销毁重建

`theme` 和 `renderer` 都是 `init` 的参数，实例创建后改不了。所以 watch 它们的策略是暴力但正确的：dispose 掉重建。暗黑模式切换一行搞定：

```ts
watch(isDark, (val) => (state.chartTheme = val ? 'dark' : undefined))
// 模板里 :theme="state.chartTheme"，所有图表同步换肤
```

## 四、实战：一个仪表盘页的四种姿势

组件注册后，一个典型的仪表盘页长这样（趋势图 + 类型切换 + 点击下钻）：

```html
<template>
  <div class="dashboard">
    <div class="card">
      <div class="card-header">
        <span>近 7 日访问趋势</span>
        <el-radio-group v-model="state.trendType" size="small" @change="buildTrendOption">
          <el-radio-button value="bar">柱状</el-radio-button>
          <el-radio-button value="line">折线</el-radio-button>
        </el-radio-group>
      </div>
      <Echarts
        :options="state.trendOption"
        height="300px"
        :loading="state.loading"
        :is-empty="!state.trend.length"
        @click="onPointClick"
      />
    </div>

    <div class="card">
      <div class="card-header"><span>访问来源</span></div>
      <Echarts :options="state.pieOption" height="260px" :is-empty="!state.pieList.length" />
    </div>
  </div>
</template>
```

```ts
import Echarts from '@/components/Echarts/index.vue'
import type { EChartsOption } from 'echarts'

const state = reactive({
  loading: false,
  trend: [] as { date: string; pv: number; uv: number }[],
  trendType: 'line' as 'bar' | 'line',
  trendOption: {} as EChartsOption,
  pieList: [] as { name: string; value: number }[],
  pieOption: {} as EChartsOption,
})

// 趋势图：同一份柱状/折线切换，options 重新生成就行
const buildTrendOption = () => {
  state.trendOption = {
    tooltip: { trigger: 'axis' },
    legend: { data: ['PV', 'UV'] },
    grid: { left: 40, right: 20, top: 40, bottom: 30 },
    xAxis: { type: 'category', data: state.trend.map((v) => v.date) },
    yAxis: { type: 'value' },
    series: [
      { name: 'PV', type: state.trendType, data: state.trend.map((v) => v.pv) },
      { name: 'UV', type: state.trendType, data: state.trend.map((v) => v.uv), smooth: true },
    ],
  }
}

const buildPieOption = () => {
  state.pieOption = {
    tooltip: { trigger: 'item' },
    legend: { bottom: 0 },
    series: [{ type: 'pie', radius: ['40%', '65%'], data: state.pieList }],
  }
}

const getDashboard = async () => {
  state.loading = true
  try {
    const { data } = await getDashboardApi()
    state.trend = data?.trend ?? []
    state.pieList = data?.source ?? []
    buildTrendOption()
    buildPieOption()
  } finally {
    state.loading = false
  }
}

// 点击数据点下钻：params 就是 echarts 原生事件对象
const onPointClick = (params: any) => {
  router.push({ name: 'Detail', query: { date: params.name } })
}

onMounted(getDashboard)
```

四个实践模式：

1. **数据变了就重新生成整个 options，不要去 patch 深层字段。** `buildTrendOption` 每次从原始数据全量构建配置——虽然组件的 deep watch 能感知深层修改，但「原始数据 → 纯函数 → options」的单向流更好排查：图不对就查 options 对不对，options 对就查数据对不对，没有中间态。
2. **切图表类型零成本。** `bar` ↔ `line` 切换只是 `series[].type` 换个字符串重新生成 options，`notMerge: true` 保证旧系列的样式、多余的配置全部被替换干净，不会有「折线图里残留柱状图 markPoint」的灵异事件。这就是「什么类型都能复用」的机理：**类型不是组件的参数，是 options 的参数**。
3. **空态是显式的。** 接口空数据时 `isEmpty` 一开，遮罩盖住的是一张被 `clear()` 掉的空图，而不是画着一个孤零零坐标轴的「假图表」。
4. **escape hatch 常备。** 导出图片这种组件没封装的能力，`getInstance()` 一步直达：

```ts
const exportPng = () => {
  const url = chartRef.value?.getInstance()?.getDataURL({ pixelRatio: 2, backgroundColor: '#fff' })
  const a = document.createElement('a')
  a.href = url!
  a.download = 'trend.png'
  a.click()
}
```

## 五、工程化：按需引入，体积砍半

上面组件里 `import * as echarts from 'echarts'` 是全量引入，min+gzip 后 300KB+ 往上。生产项目应该按需注册，且注册行为收敛到一个独立模块里，组件代码一行不用改：

```ts
// src/plugins/echarts.ts
import * as echarts from 'echarts/core'
import { BarChart, LineChart, PieChart, RadarChart } from 'echarts/charts'
import {
  DataZoomComponent,
  GridComponent,
  LegendComponent,
  TitleComponent,
  TooltipComponent,
} from 'echarts/components'
import { CanvasRenderer } from 'echarts/renderers'
import type { ComposeOption } from 'echarts/core'
import type {
  BarSeriesOption,
  LineSeriesOption,
  PieSeriesOption,
  RadarSeriesOption,
} from 'echarts/charts'
import type {
  DataZoomComponentOption,
  GridComponentOption,
  LegendComponentOption,
  TitleComponentOption,
  TooltipComponentOption,
} from 'echarts/components'

// 项目用到什么注册什么；新增图表类型 = 这里加一行
echarts.use([
  CanvasRenderer,
  BarChart, LineChart, PieChart, RadarChart,
  GridComponent, TooltipComponent, LegendComponent, TitleComponent, DataZoomComponent,
])

// 用到的 series/component 类型组装出项目的图表配置类型
export type ECOption = ComposeOption<
  | BarSeriesOption | LineSeriesOption | PieSeriesOption | RadarSeriesOption
  | GridComponentOption | TooltipComponentOption | LegendComponentOption
  | TitleComponentOption | DataZoomComponentOption
>

export default echarts
```

组件里只需把 `import * as echarts from 'echarts'` 换成 `import echarts from '@/plugins/echarts'`，其余代码原样工作。两个附带收益：

- **体积可控**：按项目实际用到的图表裁剪，gzip 体积一般能省下一半以上；
- **`ECOption` 比 `EChartsOption` 更严**：`EChartsOption` 是全量类型的并集，拼错字段不一定报错；`ComposeOption` 组装出来的类型只认注册过的东西，没用 `RadarChart` 却写 `type: 'radar'` 会直接类型报错——没用 TS 白板的原因之一。

## 六、设计复盘：好的与该改的

| 维度 | 做法 | 评价 |
| --- | --- | --- |
| 实例存储 | `shallowRef` | ✅ 避开 Vue 深度代理 echarts 实例的坑，不报错的那种坑最难排查 |
| 自适应 | ResizeObserver 监听容器 | ✅ 侧边栏折叠、分栏拖拽都能跟上，window resize 覆盖不了这些 |
| 生命周期 | mounted init / unmounted dispose | ✅ 无泄漏；keep-alive 用 onActivated 补 resize |
| 配置更新 | deep watch + `notMerge: true` 默认 | ✅ 「整图替换」最可预期；⚠️ 会重置 dataZoom 等交互状态，需要保留时传 `:notMerge="false"` |
| 事件 | 白名单转发 10 个常用事件 | ⚠️ 覆盖绝大多数场景，冷门事件走 `getInstance()` |
| 空态/加载 | `isEmpty` / `loading` prop | ✅ 视觉统一；loading 用 echarts 自带能力，不依赖 UI 库 |
| 主题切换 | watch theme → dispose + re-init | ✅ 暗黑模式一行切换；暴力但正确 |
| 按需引入 | 独立 `plugins/echarts.ts` 注册模块 | ✅ 体积减半，新增图表类型改一处 |
| 类型 | `withDefaults(defineProps<Props>())` | ✅ props 有完整类型和 JSDoc；⚠️ `EChartsOption` 偏宽松，严格收敛要靠 `ECOption` |
| SSR | 未处理 | ❌ 没有环境判断，服务端渲染会报 `document is not defined`，需要时得补客户端守卫 |

## 总结

```text
封装前（每张图）                      封装后
─────────────────                    ─────────────────
ref 拿 DOM + echarts.init             <Echarts :options="option" />
onMounted 里 setOption                数据变了重新生成 option
window resize 监听 + 手动 resize      容器变了自动 resize
onUnmounted dispose                   loading / 空态 / 主题全是 prop
每张图 ~40 行样板 + 一堆坑            0 行样板，坑在组件里修一次
```

**一句话记住**：图表组件封装的本质是把 **echarts 的生命周期**（init / setOption / resize / dispose）收进组件，把**图表长什么样**全部交给 options——类型只是 `options.series[].type` 的一个字符串，所以组件天然是全类型复用的；判断封装好坏的标准就一条：使用者是否只需要关心 options，其余一切（包括 `getInstance()` 这种后门）都有明确出口。
