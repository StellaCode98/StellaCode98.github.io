---
title: 小程序性能优化实战清单：分包、骨架屏、长列表
date: 2026-09-09 23:10:00
description: 小程序性能三板斧的实战清单：分包把主包压到 2M 以内（含 preloadRule、独立分包、分包异步化），骨架屏解决白屏感知（含手写 shimmer 动画代码），长列表治好 setData（含差量更新与 recycle-view 虚拟列表完整代码）。每条都是"结论 + 可直接抄的配置"。
categories:
  - [前端工程化, 小程序]
tags:
  - 微信小程序
  - 性能优化
---

小程序的性能问题，拆开就是一条启动链路上的三个阶段：

```text
下载代码包 → 注入 → 首屏渲染 → 运行时交互
     ↑           ↑         ↑           ↑
   分包         分包      骨架屏      长列表 / setData
```

三板斧各治一段：**分包治下载、骨架屏治感知、长列表治运行时**。本文以微信小程序为例，每条给结论和能直接抄的代码。动手前先跑一遍开发者工具的「体验评分」面板，让数据告诉你卡在哪一段。

---

## 一、分包：主包只留"进门必须的"

### 1.1 为什么要分包

小程序**先下载主包、再渲染首页**，主包越大，用户看到的第一个像素越晚。硬性限制：

```text
单个分包 / 主包 ≤ 2M        整个小程序（所有分包合计）≤ 30M
```

主包超 2M 直接没法上传，所以分包不是可选项，是必修课。

### 1.2 基础配置：app.json

```json
{
  "pages": [
    "pages/index/index",
    "pages/login/login"
  ],
  "subpackages": [
    {
      "root": "packageOrder",
      "pages": ["pages/list/index", "pages/detail/index"]
    },
    {
      "root": "packageActivity",
      "pages": ["pages/seckill/index"],
      "independent": true
    }
  ],
  "preloadRule": {
    "pages/index/index": {
      "network": "wifi",
      "packages": ["packageOrder"]
    }
  }
}
```

三个配置三件事：

- **subpackages**：订单、活动等非首屏页面全部搬出主包。跳转时才下载对应分包（有几百毫秒的加载态，所以要配合预下载）
- **preloadRule**：用户在首页闲着时，提前把"下一步大概率要去的分包"拉下来，跳转时零等待
- **independent（独立分包）**：不依赖主包、单独运行。用户从分享卡片直接进活动页时，**完全不下载主包**，秒开——适合活动、落地页

### 1.3 三条铁律

```text
① tabBar 页面和首页必须在主包，塞不进分包
② 分包可以引用主包的组件和工具，反之不行
   → 公共代码放主包，业务代码放分包
③ 图片、字体等静态资源走 CDN，不要打进包里
   → 主包体积超标，九成是图片干的
```

### 1.4 分包异步化：跨包用组件不拖累主包

偶尔需要"主包页面用分包里的组件/函数"，用分包异步化，用到时才加载：

```js
// 异步引用其他分包的 JS 模块
const utils = await require.async('../packageOrder/utils.js')
```

```json
// 主包页面的 json：引用分包组件 + 占位组件兜底
{
  "usingComponents": { "hello": "../packageActivity/components/hello" },
  "componentPlaceholder": { "hello": "view" }
}
```

组件没加载完时先渲染占位符，加载完自动替换。

---

## 二、骨架屏：把"白屏等待"变成"正在加载"

### 2.1 原则

感知性能比真实性能更值钱：**白屏超过 300ms 就该有骨架屏**。两条底线：

```text
① 骨架的结构、尺寸必须和真实内容一致——否则内容一回来页面跳动，比白屏更糟
② 骨架是纯静态的：不依赖任何接口数据，样式内联在包里，零网络请求
```

偷懒方案：开发者工具模拟器面板「…」菜单里有**一键生成骨架屏**，会按当前页面结构生成 `skeleton.wxml/wxss`，适合快速上线；结构精细的页面建议手写，可控得多。

### 2.2 手写骨架屏

```xml
<!-- pages/order/list.wxml -->
<block wx:if="{{loading}}">
  <view class="sk-card" wx:for="{{[1, 2, 3]}}" wx:key="*this">
    <view class="sk sk-avatar"></view>
    <view class="sk sk-line w-60"></view>
    <view class="sk sk-line w-90"></view>
  </view>
</block>

<block wx:else>
  <view class="card" wx:for="{{list}}" wx:key="id">
    <image class="avatar" src="{{item.avatar}}" />
    <view class="title">{{item.title}}</view>
    <view class="desc">{{item.desc}}</view>
  </view>
</block>
```

```css
/* shimmer 动画：一道高光从左到右扫过去 */
.sk {
  background: linear-gradient(90deg, #f2f2f2 25%, #e8e8e8 37%, #f2f2f2 63%);
  background-size: 400% 100%;
  animation: sk-shimmer 1.4s ease infinite;
}
@keyframes sk-shimmer {
  0%   { background-position: 100% 0; }
  100% { background-position: 0 0; }
}

.sk-card  { display: flex; flex-wrap: wrap; padding: 24rpx; }
.sk-avatar { width: 88rpx; height: 88rpx; border-radius: 50%; }
.sk-line  { height: 32rpx; margin: 8rpx 0; border-radius: 8rpx; }
.w-60 { width: 60%; }
.w-90 { width: 90%; }
```

```js
// pages/order/list.js
Page({
  data: { loading: true, list: [] },

  async onLoad() {
    try {
      const list = await this.fetchList()
      // 一次性切换：骨架 → 内容，避免半渲染状态
      this.setData({ list, loading: false })
    } catch (e) {
      this.setData({ loading: false })   // 失败也要撤掉骨架，展示错误态
    }
  }
})
```

一个容易忽略的点：请求要放在 `onLoad` 里尽早发出（骨架屏亮着的同时数据在路上），而不是 `onReady` 之后再发，否则骨架只是把白屏变漂亮了，总耗时一点没少。

---

## 三、长列表：瓶颈不在渲染，在 setData

### 3.1 先搞清楚卡在哪

小程序逻辑层（JS）和渲染层（WebView）是两个线程，`setData` 是两线程之间**序列化 + 桥接传输**：

```text
setData({ list })  →  序列化整个 list  →  跨线程传输  →  渲染层 diff
                          ↑
              list 有 1000 条时，每调一次全量传一次，页面必卡
```

所以优化方向就一句话：**让每次 setData 传输的数据尽量小、调用频率尽量低**。

### 3.2 第一层：分页加载 + 防抖守卫（必做）

```js
Page({
  data: { list: [], loading: false, finished: false },
  page: 1,

  async onReachBottom() {
    // 守卫：请求中 / 没有更多时，直接忽略触底事件
    if (this.data.loading || this.data.finished) return
    this.setData({ loading: true })

    const { list, hasMore } = await fetchPage(this.page++)
    if (!hasMore) this.setData({ finished: true })
    this.appendList(list)
    this.setData({ loading: false })
  }
})
```

### 3.3 第二层：差量更新，别全量推（核心）

```js
// ❌ 全量：整个 list 重新序列化、重新传输
this.setData({ list: this.data.list.concat(batch) })

// ✅ 追加：只传新增的段
appendList(batch) {
  const patch = {}
  batch.forEach((item, i) => {
    patch[`list[${this.data.list.length + i}]`] = item
  })
  this.setData(patch)
}

// ✅ 改单项：只传变化的字段（如给第 3 条设置加载态）
this.setData({ [`list[3].status`]: 'loading' })
```

配套的三条习惯：

```text
① data 只放渲染需要的数据，纯逻辑变量挂 this 上（不进 setData）
② 高频事件里别 setData：onPageScroll / bindscroll 每秒几十次，
   需要的话做节流，或改用 WXS 响应事件
③ 列表项封装成自定义组件，item 内部状态变化只 setData 组件自己，
   不惊动整页
```

### 3.4 第四位之后的内容不渲染：懒渲染

信息流里一屏只见 3~4 条，用户往下滚才需要后面的内容。用 `IntersectionObserver` 只渲染进入视口的 item，DOM 数量直接砍到 1/10：

```xml
<view wx:for="{{list}}" wx:key="id" class="item-observer item-{{item.visible ? 'show' : 'placeholder'}}">
  <block wx:if="{{item.visible}}">
    <order-card item="{{item}}" />
  </block>
  <view wx:else class="ph"></view>  <!-- 固定高度的占位，防止滚动抖动 -->
</view>
```

```js
onLoad() {
  this.observer = wx.createIntersectionObserver(this, { observeAll: true })
  // 比可视区域多观察上下三屏，滚动时提前渲染
  this.observer.relativeViewport({ bottom: 3000 }).observe('.item-observer', (res) => {
    const { index } = res.dataset
    if (res.intersectionRatio > 0 && !this.data.list[index].visible) {
      this.setData({ [`list[${index}].visible`]: true })   // 又是差量更新
    }
  })
}
```

给每个 item 加 `data-index="{{index}}"`，回调里就能拿到索引。

### 3.5 万级列表：官方虚拟列表 recycle-view

商品库、消息记录这种几千上万条的列表，用官方 `miniprogram-recycle-view`——**只渲染可视区域 ± 缓冲区的节点**，滚出屏幕的节点回收复用，无论数据多少条，渲染节点数恒定：

```bash
npm install miniprogram-recycle-view
```

```json
// 页面 json
{
  "usingComponents": {
    "recycle-view": "miniprogram-recycle-view/recycle-view",
    "recycle-item": "miniprogram-recycle-view/recycle-item"
  }
}
```

```xml
<!-- 页面 wxml -->
<recycle-view batch="{{batchSetData}}" id="recycleId" enable-back-to-top>
  <recycle-item wx:for="{{recycleList}}" wx:key="id">
    <order-card item="{{item}}" />
  </recycle-item>
</recycle-view>
```

```js
const createRecycleContext = require('miniprogram-recycle-view')

Page({
  onLoad() {
    this.ctx = createRecycleContext(this, {
      id: 'recycleId',            // 对应 wxml 里的 id
      dataKey: 'recycleList',     // 对应 wxml 里 wx:for 的数组名
      page: this,
      itemSize: { width: 710, height: 120 }   // 单位 rpx，定高才能算滚动
    })
    this.fetchPage(1).then(list => this.ctx.append(list))
  },

  async onReachBottom() {
    const list = await this.fetchPage(++this.page)
    this.ctx.append(list)              // 追加也只传增量
    // this.ctx.splice(start, n, ...items)   删改用 splice，同样差量
  }
})
```

限制要知道：`recycle-view` 要求**条目定高**（不定高的列表要先估算高度），这是它换性能的代价。几百条以内的列表，用 3.2~3.4 的组合就够了，不必上虚拟列表。

---

## 四、速查表

| 阶段 | 手段 | 关键点 |
| --- | --- | --- |
| 下载 | 分包 | 单包 ≤ 2M、tabBar 页在主包、图片走 CDN |
| 下载 | preloadRule 预下载 | 用户在首页时拉下一个分包，跳转零等待 |
| 下载 | 独立分包 | 分享直进的活动页不下载主包，秒开 |
| 下载 | 分包异步化 | `require.async` + `componentPlaceholder`，跨包不拖主包 |
| 首屏 | 骨架屏 | 尺寸与真实内容一致防跳动；纯静态零请求；onLoad 尽早发请求 |
| 运行时 | setData 差量 | 路径更新 `list[i].field`，单次 ≤ 1M，别高频调用 |
| 运行时 | 懒渲染 | IntersectionObserver，视口外加占位不渲染 |
| 运行时 | 虚拟列表 | recycle-view，节点数恒定，条目需定高 |
| 运行时 | 图片 | `<image lazy-load>`，列表图用缩略图尺寸 |

**落地顺序**：先跑体验评分定位阶段 → 分包保启动（硬性要求）→ 骨架屏保感知（半天工作量）→ 分页 + 差量 setData（性价比最高）→ 真有万级列表再上 recycle-view。

---

### 参考

- [微信官方文档 - 使用分包](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/basic.html)
- [微信官方文档 - 分包预下载](https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/preload.html)
- [微信官方文档 - setData 性能](https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips/runtime_setData.html)
- [miniprogram-recycle-view（官方虚拟列表组件）](https://github.com/wechat-miniprogram/recycle-view)
