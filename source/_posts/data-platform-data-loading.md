---
title: 数据大屏的数据加载体系：运行时配置、Adapter 级 Mock 与推送即拉取
date: 2026-09-09 21:00:00
description: 一个数据可视化大屏项目里数据加载的完整工程方案：config.js 运行时配置免构建切换后端，Axios 拦截器统一契约与解包，替换 adapter 的 Mock 方案让前端脱离后端开发，WebSocket 不直接渲染而是触发带筛选条件的重拉，再配上滚动懒加载与按需加载的完整清单。
categories:
  - 前端工程化
tags:
  - axios
  - mock
  - websocket
  - sse
  - 数据加载
  - 前端工程化
---

在一个数据可视化大屏项目里，「请求接口渲染数据」这件事被环境逼成了一套体系。三个现实约束：

- **后端是多微服务**（事件、设施、图层、用户……），联调排期永远对不齐，前端必须能在后端没就绪时把整个大屏跑起来；
- **部署环境经常换地址**——今天对接测试网关，明天切演示环境，后天要带去客户内网。每次改 `baseURL` 都要重新打包发版，不可接受；
- **数据要「实时」**——新事件发生后端会推送，但推送消息怎么变成列表和地图上的点，中间有讲究。

这篇文章按数据从后端到像素的路径，把这个项目的数据加载方案拆开讲：**运行时配置 → 请求封装 → Mock 拦截 → API 组织 → 数据流转 → 实时推送 → 按需加载**。技术栈是 Vue 3 + Pinia + Axios，方案本身与框架关系不大。

<!-- more -->

## 一、运行时配置：改地址不重新构建

先解决「换环境就重新打包」的问题。思路是把配置从「构建时」挪到「运行时」：

```html
<!-- index.html：同步加载，保证在任何业务代码执行前就绪 -->
<script src="/config.js"></script>
```

```js
// public/config.js：跟静态资源一起部署，不参与构建
window.__APP_CONFIG__ = {
  api: {
    baseURL: 'https://api.example.com',  // 网关地址
    timeout: 30000,
    mock: false,                          // Mock 总开关
  },
  ws:  { enabled: true, pushUrl: 'wss://api.example.com/ws/push', reconnectInterval: 5000 },
  sse: { enabled: true, url: '/api/ai/chat', mock: true },
  // 地图中心点、底图列表、瓦片服务前缀……也都在这里
}
```

读取时与代码里的**默认配置深度合并**，`config.js` 加载失败也有兜底：

```ts
export function getConfig(): AppConfig {
  const win = window as unknown as { __APP_CONFIG__?: Partial<AppConfig> }
  if (!win.__APP_CONFIG__) return defaultConfig
  return {
    api:       { ...defaultConfig.api, ...win.__APP_CONFIG__.api },
    ws:        { ...defaultConfig.ws, ...win.__APP_CONFIG__.ws },
    sse:       { ...defaultConfig.sse, ...win.__APP_CONFIG__.sse },
    // ...
  }
}
```

这个方案有三个细节值得抠：

1. `<script>` 必须放在应用 bundle **之前**同步加载（不用 `defer`），否则应用初始化时配置还没注入；
2. 深度合并只做了一层——够用的前提下不做通用 `deepMerge`，配置结构变了让 TS 类型直接报错，比静默合并错位更早暴露问题；
3. Vite 生态里更常见的做法是多个 `.env.xxx` 文件 + `import.meta.env`，但那是**构建时注入**：`vite build` 出来的产物里地址已经写死。运行时配置牺牲了一点点加载顺序上的心智负担，换来「同一份产物跑任意环境」——对内网交付项目，这笔账必须这么算。

## 二、请求封装：契约、解包、错误分类

Axios 封装是每个项目的保留节目，这里说三个设计点。

**统一响应契约**。后端所有接口遵循 `{ success, code, message, data }` 结构，响应拦截器负责校验和**解包**——业务代码拿到的永远是净数据：

```ts
service.interceptors.response.use(
  (response) => {
    const res = response.data as ApiResponse
    // HTTP 200 但业务失败（success=false 或 code 异常）也走 reject
    if (!res.success || (res.code !== 0 && res.code !== 200)) {
      return Promise.reject(new Error(res.message || '请求失败'))
    }
    return res.data   // 直接返回 data 字段
  },
  (error) => { /* 见下文错误分类 */ }
)
```

业务失败和 HTTP 失败统一成一条 `Promise.reject(Error)` 流，组件层 `catch` 到的永远是带中文 message 的 Error，不用每个调用点再判断一次 `res.success`。

**错误分类比统一提示更重要**。错误拦截器把「同一长得很像的 error」拆成用户能看懂的几种：

```ts
if (error.response) {
  const { status } = error.response
  if (status === 401) {                       // 登录过期：清凭证 + 跳登录页
    clearAuth()
    window.location.href = '/login'
    return Promise.reject(new Error('登录已过期，请重新登录'))
  }
  if (status === 403) return Promise.reject(new Error('没有权限执行此操作'))
  return Promise.reject(new Error(error.response.data?.message || `请求错误 (${status})`))
}
if (error.code === 'ECONNABORTED') return Promise.reject(new Error('请求超时，请稍后重试'))
if (axios.isCancel(error))          return Promise.reject(new Error('请求已取消'))
if (!window.navigator.onLine)       return Promise.reject(new Error('网络连接已断开'))
```

注意 401 里做了**副作用**（清 token、跳转）——这类「全局性、只该做一次」的处理放拦截器，比每个组件各写一遍可靠得多。

**Token 注入**放在请求拦截器，来源区分 localStorage（记住我）和 sessionStorage（会话级），这是登录模块的约定，请求层只管读。

## 三、Mock 拦截：换 adapter，而不是换函数

前端脱离后端开发，常见方案有三档：

| 方案 | 做法 | 问题 |
| --- | --- | --- |
| 注释切换 | 业务代码里 `if (mock) return fakeData` | mock 逻辑散落各处，上线前忘删就是事故 |
| 代理工具 | Mock.js 劫持 XHR / MSW 拦截 Service Worker | 引新依赖；MSW 需要注册 worker，环境受限时麻烦 |
| **adapter 替换** | 请求拦截器里把 axios 的 adapter 换成假实现 | 零依赖、零业务侵入，请求根本不发出 |

这个项目用的第三档。Axios 的 `adapter` 是真正发请求的底层函数，在**请求拦截器**里替换它：

```ts
if (mock) {
  service.interceptors.request.use((config) => {
    const mockData = mockAdapter(config.url, config.method, config.params, config.data)
    if (mockData !== undefined) {
      config.adapter = () =>                 // 命中 mock：替换 adapter，直接 resolve 假响应
        Promise.resolve({
          data: { success: true, code: 0, data: mockData, message: 'ok' },
          status: 200, statusText: 'OK', headers: {}, config,
        })
    }
    return config                            // 未命中：正常发出真实请求
  })
}
```

路由表用类似后端框架的写法，支持 `:param` 占位符：

```ts
const mockRoutes: MockRoute[] = [
  { pattern: 'event-service/event/list', method: 'post', handler: (_p, _q, body) => {
      // 与真实后端行为一致：category/severity 过滤 + pageNum/pageSize 分页
      let list = [...mockEvents]
      if (Array.isArray(body?.category) && body.category.length > 0) {
        list = list.filter(e => body.category.includes(e.category))
      }
      const start = ((Number(body.pageNum) || 1) - 1) * (Number(body.pageSize) || 10)
      return { records: list.slice(start, start + 10), total: list.length, /* ... */ }
  }},
  { pattern: 'event-service/news/list/:eventId', method: 'get', handler: (p) => findNewsBy(p.eventId) },
]
```

两个关键设计：

1. **mock 行为与真实后端对齐**（过滤、分页、响应结构都按接口文档实现），联调切换时前端逻辑一行不改，只有数据源变了——这正是 adapter 方案的价值：切换点在传输层，不在业务层；
2. **未命中的路由放行真实请求**并打 warn，允许「一半接口真实、一半接口 mock」的渐进联调状态，而不是全有或全无。

## 四、API 层组织：微服务前缀集中管理

后端按微服务拆分，网关按**路径前缀**路由。前缀常量集中一处定义，避免拼写漂移：

```ts
// api/commonApi.ts —— 服务前缀唯一事实来源
export const eventService = 'event-service'
export const facilityService = 'facility-service'
export const authService = 'auth-service'

// api/event.ts —— 按业务域拆文件
export async function getEventList(params?: EventListParams) {
  return post(`/${eventService}/event/list`, { pageNum, pageSize, ...filters })
}
```

API 文件只做三件事：拼路径、传参数、**把后端结构翻译成前端类型**。以事件列表为例，后端字段和前端模型并不一致——`description` → `summary`，关键词/实体字段是 PostgreSQL jsonb 的包装结构（`{ null: false, value: "[...]" }`，值是序列化字符串）：

```ts
/** PG jsonb 包装字段 → string[] */
function parsePgJsonArray(field: PgJsonField | string[] | undefined): string[] {
  if (!field) return []
  if (Array.isArray(field)) return field.map(String)   // 兼容后端已展开的形态
  if (field.null || !field.value) return []
  try {
    const parsed = JSON.parse(field.value)             // 值是 JSON 字符串，要二次解析
    return Array.isArray(parsed) ? parsed.map(String) : []
  } catch { return [] }
}

export function mapBackendEvent(raw: BackendEventInfo): EventItem {
  return {
    id: raw.id,
    title: raw.title,
    summary: raw.description,          // 字段改名收在这一层
    tags: parsePgJsonArray(raw.keywords),
    // ...
  }
}
```

这类「脏活」必须圈死在 API 层——store 和组件永远只见过滤干净的前端类型。后端分页结构在不同接口间还漂移（有的叫 `records`、有的叫 `list`、有的叫 `rows`），同样在 API 层做兼容：`res.records ?? res.list ?? res.rows ?? res.data`。

## 五、完整数据流：以事件列表为例

把前面的积木串起来，一个大屏面板的数据流长这样：

```text
EventPanel onMounted
  └─ Promise.all([loadCategories(), loadSeverities()])   // 筛选项也是接口下发的
       └─ loadEvents()
            └─ getEventList({ pageNum: 1, pageSize: 10, category, severity, keyword })
                 └─ axios（拦截器：mock 判断 → token → 响应解包）
                      └─ mapBackendEvent 字段映射 + jsonb 解析
                           └─ store: events = res.list
                                ├─ computed filteredEvents        // 多类别/严重度/关键词/视口过滤
                                │    └─ EventList v-for 渲染卡片
                                │         └─ 滚动距底 80px → loadMoreEvents() 追加下一页
                                └─ MapContainer watch(filteredEvents)
                                     └─ GeoJSON source.setData()  // 地图增量更新，图层不重建
```

三个细节：

- **筛选在哪层做？** 服务端筛选（接口参数）和客户端筛选（computed）并存：翻页和关键词搜索走接口（数据量大、要分页），视口过滤、会话态标记在 computed 里做（变化频繁、不值得发请求）；
- **地图更新用 `setData` 而不是重建图层**：GeoJSON source 支持 `setData` 原地换数据，图层定义只建一次，数据变化只触发一次重绘；
- **滚动懒加载的守卫**：`loadMoreEvents` 开头检查 `isLoadingMore || !hasMoreEvents`，防止滚动事件连打导致重复请求和页码错乱。

## 六、实时数据：推送即拉取

新事件由后端通过 WebSocket 推送。最容易想到的写法是「推送里带什么数据就渲染什么」——**恰恰是错的**。推送消息里往往只有摘要（比如 `{ totalCount: 5 }`），直接渲染会绕过 API 层的字段映射和 store 的筛选状态；用户此刻正筛选着「突发级事件」，一条不匹配的推送直接塞进列表就是脏数据。

这个项目的策略是**推送即拉取（push-then-pull）**：WS 消息只当「数据变了」的信号，收到后带着当前筛选条件重拉第一页：

```ts
function handleMessage(event: MessageEvent) {
  const parsed = JSON.parse(event.data)
  const totalCount = Number(parsed?.data?.totalCount ?? 0)
  if (totalCount > 0) {
    if (refreshTimer) clearTimeout(refreshTimer)
    refreshTimer = setTimeout(() => {        // 500ms 节流：连发多条推送只拉一次
      eventStore.refreshEvents()            // 重拉第一页，pageSize 取已加载数量
    }, 500)
    ElNotification({ title: `新事件 (${totalCount} 条)`, /* ... */ })
  }
}
```

这样推送数据永远走与手动刷新完全相同的链路（接口 → 映射 → store → computed → 视图），筛选一致性免费获得。代价是多一次 HTTP 往返，但大屏场景事件频率不高，这点延迟无所谓。**结论：推送当信号用，别当数据用。**

断线重连用**指数退避**，每次失败间隔 ×1.5，封顶 60s，连接成功后归零——避免后端重启时前端疯狂重连：

```ts
function scheduleReconnect() {
  if (manualClosed || !wsConfig.enabled) return
  const delay = Math.min(wsConfig.reconnectInterval * Math.pow(1.5, reconnectAttempts), 60000)
  reconnectAttempts++
  reconnectTimer = setTimeout(connect, delay)
}
```

组件卸载（`onUnmounted`）时置 `manualClosed = true` 再 close，防止主动关闭也触发重连。AI 对话的流式回复则走 SSE：原生 `fetch` + `ReadableStream.getReader()` 手动按行解析 `data:` 帧（POST 请求用不了原生 EventSource），同样由 config.js 的 `sse.mock` 开关一键切到「逐字输出」的假流。

## 七、按需加载清单

大屏首屏压力大，「什么时候才发起请求/渲染」需要刻意设计。这个项目的清单：

| 手段 | 实现 | 收益 |
| --- | --- | --- |
| 组件级懒加载 | 面板用 `defineAsyncComponent` 挂载 | 首屏 bundle 不含次要面板 |
| 初始化单例 | `_homeInitPromise` 缓存 Promise，首页与面板共享 | 两个组件同时触发初始化只发一次请求 |
| 勾选驱动拉瓦片 | 用户勾选图层才请求瓦片地址，取消勾选只隐藏不销毁 | 不看的图层零请求 |
| 滚动懒加载 | 列表距底 80px 触发下一页 | 首屏不拉全量 |
| 弹框懒加载 | 详情弹框 `watch(visible)` 打开时才请求关联新闻 | 关联数据用不到就不拉 |
| 防抖节流 | 搜索 400ms、视口同步 300ms + 变化阈值、WS 刷新 500ms | 高频交互不放大请求量 |
| 会话级缓存 | 全局样式配置进 sessionStorage | 刷新页面不重拉样式 |

其中「初始化单例」容易被忽略：首页 `onMounted` 和图层面板 `onMounted` 都会触发初始化请求，用 Promise 缓存挡住第二次——比「谁先挂载谁负责」的君子协定可靠得多。

## 总结

四种数据源的切换矩阵：

| 数据源 | 开关位置 | 生效方式 | 典型场景 |
| --- | --- | --- | --- |
| 真实 HTTP | `api.mock: false` | 正常请求 | 联调/生产 |
| Mock 数据 | `api.mock: true` + mockRoutes 命中 | 替换 axios adapter，请求不发出 | 后端未就绪 |
| WS 推送 | `ws.enabled` | 推送当信号，节流后带条件重拉 | 事件实时更新 |
| SSE 流 | `sse.mock` | fetch 流解析 / setTimeout 假流 | AI 对话 |

**一句话记住**：数据加载体系的目标是让「换环境、换数据源、上实时」都不侵入业务代码——配置在运行时注入，Mock 在传输层拦截，字段映射圈死在 API 层，推送只当信号不当数据，剩下的组件就只管对着干净的前端类型渲染。
