---
layout: post
title: "前端工程师的 AI 入门：从流式接口到落地一个 AI 应用"
date: 2026-09-11 12:30:00 
description: "以一个真实落地的 AI 聊天模块为例，讲透前端视角下的两大核心：SSE 流式对话与 Function Calling 工具调用——从协议、代码到工程化的所有坑。"
categories: [AI 应用开发]
tags: [AI, SSE, Function Calling, Vue3, NodeJs, LLM]
---

> 这篇文章不是概念科普，而是一次真实的落地复盘。最近我在自己的 Vue3 后台管理项目（vue3_admin）里完整实现了一个 AI 智能问答模块：支持 SSE 流式打字机输出、多轮对话记忆、Function Calling 工具调用、可视化组件渲染、会话持久化。本文把这条链路从头到尾拆开讲——**为什么需要流式、前端怎么手写 SSE 解析器、Function Calling 的两轮循环到底转了什么、以及从 demo 到可用应用之间隔着哪些工程细节**。所有代码都来自这个真实模块，文末附完整文件地图。

## 目录

- [0. 前言：前端工程师离 AI 应用有多远](#sec0)
- [1. 认识流式接口：AI 的回复为什么是"蹦"出来的](#sec1)
- [2. 服务端：把模型的流转发给浏览器](#sec2)
- [3. 前端：手写一个 SSE 解析器](#sec3)
- [4. 从"流"到"聊天界面"：状态编排层](#sec4)
- [5. 让 AI 用上你的函数：Function Calling](#sec5)
- [6. 落地成应用：demo 和工程之间隔着什么](#sec6)
- [7. 展望：MCP 离我们还有多远](#sec7)
- [8. 写在最后](#sec8)

<a id="sec0"></a>

## 0. 前言：前端工程师离 AI 应用有多远

打开 ChatGPT 或者任何 AI 对话产品，仔细观察它的交互，你会发现两个本质特征：

1. **回复是流式的**——文字一个字一个字"蹦"出来，而不是等几秒后一次性出现；
2. **AI 会用工具**——问"今天天气"它去调天气 API，问"帮我算一下"它去调计算器，最后把结果组织成人话告诉你。

这两件事分别对应两套技术：**SSE（Server-Sent Events）流式传输**和 **Function Calling（工具调用）**。它们也是前端工程师切入 AI 应用开发的最佳入口——不需要懂模型训练，不需要会调参，需要的恰恰是我们最熟悉的东西：HTTP 协议、异步编程、状态管理、组件渲染。

本文按照我实际搭建这个模块的顺序展开，技术栈：

- **前端**：Vue 3 + TypeScript + Vite（`src/views/aiChatroom/`）
- **服务端**：Node.js + Express 5 + OpenAI 官方 SDK（`server/`）
- **模型**：任何 OpenAI 兼容网关（官方 API、DeepSeek 等均可，只需改一个 `baseUrl`）

先给一张全链路图，后文的所有章节都是对这张图的展开：

```text
┌─────────────────────── 浏览器 ────────────────────────┐
│  chatRoom.vue        （UI 编排：组件接线 + 页面骨架）  │
│      ↓                                               │
│  useAgentChat.ts     （状态：气泡 / 块 / 竞态防护）    │
│      ↓                                               │
│  useSSEChat.ts       （传输：fetch 发请求 + 读流解析） │
└────────┬──────────────────────────────────────────────┘
         │  POST /api/chat-room-agent
         │  body: { messages: [...对话历史], stream: true }
         ▼
┌─────────────────────── Node 服务 ─────────────────────┐
│  routes/chat-room-agent.js   （HTTP 进出 + SSE 翻译）  │
│      ↓                                               │
│  chatRoomAgentService.js     （Agent 工具循环 ≤4 轮）  │
│      ↓  stream: true                                │
│  OpenAI 兼容网关（官方 / DeepSeek / 中转站…）          │
└──────────────────────────────────────────────────────┘
```

<a id="sec1"></a>

## 1. 认识流式接口：AI 的回复为什么是"蹦"出来的

### 1.1 LLM 的生成方式决定了交互形态

大语言模型的本质是"逐 token 预测下一个 token"——一段 500 字的回答，是模型一个词一个词生成出来的。假设生成完整体需要 8 秒，那么：

- **非流式**：用户盯着加载动画 8 秒，然后"啪"地出现一大段文字；
- **流式**：首个 token 通常几百毫秒就到，用户第 1 秒就开始阅读，文字边生成边展示。

体验差距是量级性的。这就是为什么几乎所有 AI 产品都用流式——**不是炫技，是把模型的生成过程原样暴露给用户**。

### 1.2 SSE：为"服务器单向推流"而生的协议

实现"服务器不断推、客户端不断收"，标准答案就是 SSE。它不是一个新协议，就是**普通 HTTP 响应**，只是约定了三件事：

1. 响应头声明 `Content-Type: text/event-stream`，告诉客户端"后面的 body 是事件流，别等它结束"；
2. 每条消息以 `data:` 开头，以**空行（两个换行 `\n\n`）结尾**；
3. 连接一直保持，直到服务端主动关闭。

我的模块里，一次流式响应的原始报文长这样：

```text
Content-Type: text/event-stream; charset=utf-8
Cache-Control: no-cache, no-transform
Connection: keep-alive

data: {"type":"text","content":"你"}

data: {"type":"text","content":"好"}

data: {"type":"agent_status","phase":"start","name":"get_current_time"}

data: [DONE]
```

有一个容易误解的点：**`data: [DONE]` 不是 SSE 标准的一部分**，它是 OpenAI 风格的流结束约定（DeepSeek 等兼容网关都沿用了）。SSE 标准里流结束就是连接关闭，但实践中"显式结束标记 + 关连接"更可靠，前端也不需要额外处理"连接断了到底是答完了还是出错了"的歧义。

### 1.3 SSE vs WebSocket vs 轮询

| 方案  | 方向  | 协议成本 | 适用场景 |
| --- | --- | --- | --- |
| 轮询  | 客户端反复拉 | 低，但延迟高、空请求多 | 兼容性兜底 |
| **SSE** | **服务器 → 客户端单向** | **就是 HTTP，无需升级协议** | **AI 流式回复、通知推送** |
| WebSocket | 双向  | 需要协议升级、心跳、重连逻辑 | 聊天室、协作编辑 |

AI 对话场景里，客户端只在开头发一次请求，之后全是服务器往回推——**典型的单向流，SSE 是刚好够用的方案**。WebSocket 的双向能力和额外复杂度在这里是浪费。

### 1.4 那为什么大厂 API 不直接用 SSE 标准 EventSource？

浏览器其实内置了 SSE 客户端 `EventSource`，但 AI 场景几乎没人直接用它，原因在下一章展开（剧透：它只能发 GET）。业界主流做法和我模块里一样：**用 `fetch` 发 POST，然后手动解析流**——这也正是前端工程师最值得吃透的一段代码。

<a id="sec2"></a>

## 2. 服务端：把模型的流转发给浏览器

先说结论：服务端做的事情本质上是**"二道贩子"**——把模型网关推过来的流式 chunk，翻译成 SSE 格式推给浏览器。Node 侧用 Express 实现，总共分两步。

### 2.1 最简版：三十行跑通流式接口

这是我模块的第一个版本（`server/routes/chat.js`），保留至今作为教学对照，核心逻辑只有三段：

```js
// server/routes/chat.js（节选）
const { prompt, stream = false } = req.body
// OpenAI SDK：stream:true 时返回"异步可迭代对象"，
// 可以直接 for await——每循环一次拿到一个增量 chunk
const response = await createChatCompletion({ prompt, stream })

if (stream) {
  // SSE 标准响应头
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')

  // for await 每次拿到一个增量 chunk，delta.content 是这一小段文字
  for await (const chunk of response) {
    const content = chunk.choices[0]?.delta?.content || ''
    if (content) {
      // data: 后面加空格，两个换行 = 一条消息结束
      res.write(`data: ${JSON.stringify({ content })}\n\n`)
    }
  }
  res.end()
}
```

三个关键点：

- OpenAI SDK（以及所有兼容网关）在 `stream: true` 时返回的是一个 **异步可迭代对象**，`for await...of` 每次迭代拿到一个 chunk，chunk 里的增量文字在 `choices[0].delta.content` 上；
- 每个 chunk 不一定有内容（有的只带角色信息），用可选链 + 空串兜底；
- `res.write()` 只是写入缓冲，什么时候真正到达浏览器由底层决定——这也是后面 `flushHeaders()` 存在的原因。

### 2.2 生产版要补的细节

真正给聊天页面用的接口（`server/routes/chat-room-agent.js`）在最简版之上补了四类细节。首先是把 SSE 写操作封装成两个函数，避免格式错误：

```js
// server/routes/chat-room-agent.js
// 写一条 SSE 消息：必须是 `data: JSON\n\n` 格式（两个换行=一条消息结束）
function writeSse(res, data) {
  res.write(`data: ${JSON.stringify(data)}\n\n`)
}

// 写结束标记：前端解析到 [DONE] 就知道流正常结束了
function writeDone(res) {
  res.write('data: [DONE]\n\n')
}
```

然后是响应头"四件套"和**立即刷新**：

```js
// server/routes/chat-room-agent.js（节选）
res.status(200)
// ① 声明事件流（带 charset，中文场景必须有）
res.setHeader('Content-Type', 'text/event-stream; charset=utf-8')
// ② 禁缓存；no-transform 禁止代理压缩改写
res.setHeader('Cache-Control', 'no-cache, no-transform')
// ③ 保持长连接，直到服务端主动 res.end()
res.setHeader('Connection', 'keep-alive')
res.setHeader('X-Request-Id', requestId)

// 立即把响应头发出去：让前端尽早进入"接收流"状态
//（不然后面的 res.write 会攒到缓冲区，首字延迟变高）
if (typeof res.flushHeaders === 'function') {
  res.flushHeaders()
}
```

`flushHeaders()` 是很多人会漏掉的一行：不调用它，响应头可能攒在缓冲区里和第一批数据一起走，白白增加首字延迟。

最后是**架构上的关键设计——回调嫁接**。路由层不写业务，业务层（`chatRoomAgentService.js`）只暴露三个回调：`onText`（模型产出一段文字）、`onToolStart` / `onToolEnd`（工具执行前后）。路由层负责把回调"翻译"成 SSE 事件：

```js
// server/routes/chat-room-agent.js（节选）
await runChatRoomAgent({
  input,
  signal: abortController.signal,

  // 模型每产出一段文字，立刻推一条 SSE（打字机效果的来源）
  onText: async (text) => {
    writeSse(res, { type: 'text', content: text })
  },

  // 工具生命周期 → agent_status 事件：
  // 前端在气泡里渲染"正在调用 xx 工具…"状态条
  onToolStart: async ({ name }) => {
    writeSse(res, { type: 'agent_status', phase: 'start', name })
  },
  onToolEnd: async ({ name }) => {
    writeSse(res, { type: 'agent_status', phase: 'end', name })
  },
})
writeDone(res)
```

这个设计还有一个附赠的好处：**非流式模式几乎免费**。业务层根本不知道 HTTP 的存在，非流式只是把 `onText` 攒进一个字符串最后 `res.json()` 一次性返回——同一套 Agent 循环天然兼容两种输出形态，curl / Postman 调试时特别方便。

### 2.3 四个我亲自踩过的坑

**坑 1：SSE 响应头必须在第一条 `data:` 之前设置。**
听起来是废话，但配合 Express 的错误处理中间件就容易翻车：如果第一次 `res.write` 之前抛了错，你还能改状态码返回 500；一旦写过头，一切就晚了——这就是坑 2。

**坑 2：流开始后出错，状态码永远是 200，只能用"错误事件"传错。**
响应头 200 已经发给浏览器了，中途模型挂了怎么办？不能再 `res.status(500)`，只能把错误文案包成一条专用 SSE 事件发给前端，再正常收尾：

```js
// server/routes/chat-room-agent.js（节选）
if (!abortController.signal.aborted) {
  const mapped = mapChatRoomAgentError(error)
  // 状态码改不了了，只能用 error 事件把文案推给前端（前端标红气泡）
  writeSse(res, { type: 'error', content: mapped.message })
  writeDone(res)
}
```

注意细节：`aborted` 为 true 说明是用户自己点了停止，不算错误、不推送。前端也要有对应的处理——收到 `type:'error'` 的事件时把气泡标记成错误态。

**坑 3：监听客户端断开，要监听 `res` 的 close，不是 `req` 的。**
用户点"停止生成"或直接关页面时，服务端应该立刻取消对上游模型的请求（不取消就白烧 token）。直觉写法是 `req.on('close', ...)`，但在 Node 22 实测中这是个假阳性陷阱：**`req` 的 close 在请求体被 `express.json()` 读完之后就会触发**（请求进来约 2ms），和客户端断不断开毫无关系——用它判断"提前断开"会把完全正常的请求也 abort 掉。正确姿势：

```js
// server/routes/chat-room-agent.js（节选）
// res 的 close 在响应结束时触发：正常答完时 writableEnded=true，
// 客户端半路断开时 writableEnded=false——用它区分才可靠
res.on('close', () => {
  if (!res.writableEnded) {
    abortController.abort()
  }
})
```

**坑 4：OpenAI SDK 的配置项是 `baseUrl`，不是 `baseURL`。**
SDK 的构造参数是全小写的 `baseUrl`，写错大小写不会报错，只会静默回落到官方地址——如果你在用中转网关，症状就是"怎么改配置都提示 Key 无效"。我的服务端通过 `OPENAI_BASE_URL` 环境变量统一配置，指向任何 OpenAI 兼容服务（官方、DeepSeek、中转站）都不用改代码。

<a id="sec3"></a>

## 3. 前端：手写一个 SSE 解析器

### 3.1 为什么不用 EventSource

浏览器原生的 `EventSource` 有三个硬伤，在 AI 对话场景全部踩中：

| 能力  | EventSource | 我们需要的 |
| --- | --- | --- |
| 请求方法 | 只能 GET | **POST**（要发 JSON 请求体） |
| 请求体 | 不能带 body | **完整对话历史**（多轮记忆的来源） |
| 自定义头 | 不能设置 | `Authorization` 等 |

所以主流方案是：**`fetch` 发 POST，拿到 `response.body` 这个字节流，自己解析 SSE 格式**。这也是整个模块里我觉得最值得逐行吃透的一段代码（`src/views/aiChatroom/utils/useSSEChat.ts`，节选）：

```ts
// 传输层：发起请求 + 解析 SSE 流，整个模块里唯一碰网络的代码
// 用 fetch 而不是原生 EventSource：EventSource 只支持 GET、
// 不能自定义请求头/请求体，而这里要 POST JSON
const response = await fetch(url, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  // messages：完整对话历史（含本次新消息），服务端优先读它——
  // 这是多轮"记忆"的来源；不传 history 时退化为单条消息
  body: JSON.stringify({
    message,
    messages: history?.length ? history : [{ role: 'user', content: message }],
    stream: true,
  }),
  // abort 信号：外部调 abortController.abort() 时 fetch 抛 AbortError
  signal: abortController?.signal,
})

// 拿到字节流的 reader：每次 read() 返回一小段 Uint8Array，
// 何时有数据、每段多大都由网络决定，需要自己解码拼装
const reader = response.body.getReader()
const decoder = new TextDecoder('utf-8')
let buffer = ''

let finished = false
while (!finished) {
  const { done, value } = await reader.read()
  if (done) {
    finished = true
    break
  }

  // stream:true 允许多字节中文/emoji 被拆在两个 chunk 里，
  // TextDecoder 会记住未完成的字节，等下一段拼上再输出
  const chunk = decoder.decode(value, { stream: true })
  buffer += chunk

  // 按行切开处理，最后一段可能是不完整的行，
  // 留在 buffer 里等下一个 chunk 拼上（流式解析的经典细节）
  const lines = buffer.split('\n')
  buffer = lines.pop() || ''

  for (const line of lines) {
    const trimmedLine = line.trim()
    if (!trimmedLine) continue

    // SSE 协议：有效内容行以 "data:" 开头
    if (trimmedLine.startsWith('data:')) {
      const dataStr = trimmedLine.replace(/^data:\s*/, '').trim()

      // [DONE]：与服务端约定的流结束标记
      if (dataStr === '[DONE]') {
        onComplete()
        isLoading.value = false
        return
      }

      try {
        // 正常情况：一条 JSON 反序列化成一个块
        const data: Partial<SSEData> = JSON.parse(dataStr)
        onChunk({ type: data.type || 'text', ...data } as SSEData)
      } catch {
        // 兜底：JSON 解析失败时按纯文本处理，不让一条坏消息打断整条流
        onChunk({ type: 'text', content: dataStr })
      }
    }
  }
}
```

这段 60 行不到的代码里有三个值得单独拎出来讲的细节，它们是所有"流式解析"的通用套路：

**细节一：`decoder.decode(value, { stream: true })`。**
网络 chunk 是按字节切的，不按字符切。一个中文字占 3 字节、一个 emoji 占 4 字节，完全可能被切在两个 chunk 里——不带 `stream: true` 时，前半截会被解码成乱码。带上之后 `TextDecoder` 会记住未完成的字节序列，等下一段到达后拼起来再输出。

**细节二：`lines.pop()` 半行缓冲。**
SSE 按行解析，但一个 chunk 里最后一行很可能是半截的（比如 `data: {"type":"te`）。所以按 `\n` 切行之后，把最后一段 `pop()` 出来留在 `buffer` 里，等下一个 chunk 拼上再处理。没有这一步，JSON.parse 会随机失败。

**细节三：单条消息解析失败，降级但不中断。**
一条 `data:` 消息 JSON.parse 失败时，按纯文本兜底处理，而不是抛错断流——网络世界的格言是"坏一条消息可以，坏一整条流不行"。

### 3.2 中断：AbortController 与"静默的 AbortError"

"停止生成"按钮的实现只有两行核心代码：

```ts
// src/views/aiChatroom/utils/useSSEChat.ts
const stopChat = () => {
  if (currentAbortController) {
    currentAbortController.abort()
    currentAbortController = null
  }
  isLoading.value = false
}
```

`abort()` 一石三鸟：fetch 抛出 `AbortError`、`reader.read()` 停止、服务端收到连接断开后沿着第 2 章坑 3 的链路取消对上游模型的请求——**中断信号贯穿了整个三层链路**。

一个容易忽略的处理：abort 触发的 `AbortError` 不应该走错误分支。用户主动停止是正常操作，不该弹出红色报错：

```ts
} catch (err) {
  if (err instanceof Error) {
    // 用户主动停止属于正常行为，不算错误，不打扰上层
    if (err.name === 'AbortError') {
      console.log('Request aborted')
    } else {
      error.value = err
      onError(err)
    }
  }
}
```

<a id="sec4"></a>

## 4. 从"流"到"聊天界面"：状态编排层

到这里，"收流"已经解决了。但 `onChunk` 给你的只是一个个碎片 `{"type":"text","content":"你"}`，而用户看到的是一个带打字机动画的聊天气泡——中间隔着一层**状态编排**。我的模块把这层单独抽成了 `useAgentChat.ts`，整体分三层：

```text
chatRoom.vue     UI 编排层：组件接线、页面骨架，不含网络逻辑
    ↓
useAgentChat.ts  状态层：SSE 流 → messages 数组的变化
    ↓
useSSEChat.ts    传输层：唯一碰网络的代码，只认 SSEData
```

分层的判定标准很朴素：**传输层不认识"气泡"（ChatMessage），UI 层不认识"网络"**。改 SSE 协议只动传输层，改界面只动 UI 层。

### 4.1 占位气泡：打字机效果的真相

发送消息时先 push 一条**空的 streaming 气泡**，之后流里收到的所有内容都累积到这条消息上：

```ts
// src/views/aiChatroom/composables/useAgentChat.ts（节选）
// ② 再占位一条"空回复"：气泡先出现，type='streaming' 显示三点动画，
// 之后流里收到的内容全部累积到这条消息上
const assistantId = Date.now() + Math.random()
messages.value.push({
  id: assistantId,
  role: 'assistant',
  content: '',
  blocks: [],
  type: 'streaming',
  timestamp: new Date().toISOString(),
})
```

所谓打字机效果，没有任何 `setInterval` 定时器魔法，就是**"占位气泡 + 每个 chunk 触发一次响应式更新"**——Vue 的响应式系统天然适合流式渲染。`type` 是气泡的状态机：`streaming`（显示三点动画）→ `complete`（显示复制/重新生成按钮）或 `error`（标红）。

### 4.2 相邻 text 增量必须合并成一个块

这是我踩过的最隐蔽的一个坑。最初我每收到一个 chunk 就 push 一个新块，结果markdown 渲染完全错乱——因为每个字都成了独立的 markdown 块，代码块被拦腰截断、列表序号全部重排。修复方式只有几行：

```ts
// src/views/aiChatroom/composables/useAgentChat.ts（节选）
const last = blocks[blocks.length - 1]
// 真流式下文字逐段到达：相邻 text 增量必须合并进同一块。
// 否则每个字都是独立 markdown 块——换行错乱、代码块被拦腰截断
if (data.type === 'text' && last?.type === 'text') {
  last.content = (last.content || '') + (data.content || '')
} else {
  blocks.push({ /* ... */ })
}
```

渲染侧则简单粗暴：每个 text 块**整块 full re-parse**（markdown-it）。你可能会担心性能——实际上现代设备对几百字文本做 markdown 解析是微秒级的，每个 chunk 重解析一次完全无压力。流式 markdown 的"未闭合代码块闪烁"问题也顺带解决了：markdown-it 对不完整内容会容错渲染，下个 chunk 到达后整体重渲染自然补全。

### 4.3 竞态防护：给每个请求发一张"门票"

场景：用户点了"重新生成"，旧请求的 SSE 还在路上。如果不做防护，旧流的迟到数据会写进新对话的气泡里。解法是 requestId"门票"——发请求时领号，回调进来先验票：

```ts
// src/views/aiChatroom/composables/useAgentChat.ts（节选）
// ── 竞态防护三件套 ──
let activeRequestId: number | null = null // 当前有效请求的"门票号"
let activeAbortController: AbortController | null = null
let activeAssistantId: number | null = null // 正在流式输出的气泡 id

const requestId = Date.now() + Math.random()
activeRequestId = requestId

// 每个 chunk 回调进来第一件事：验票
onChunk: (data: SSEData) => {
  // 门票对不上 = 这是已被停止/取代的旧请求，丢弃迟到数据
  if (activeRequestId !== requestId) return
  // ...
}
```

`onComplete` / `onError` 同样先验票再操作。加上重新生成时的处理（找到上一条用户消息、截断其后所有消息、原样重发），一套完整的多轮交互就稳了。

### 4.4 多轮记忆：把历史原样发回去

模型本身没有记忆，所谓"多轮对话"，就是**每次请求都把之前的对话历史原样带上**。组装逻辑：

```ts
// src/views/aiChatroom/composables/useAgentChat.ts（节选）
// 过滤出「有文字的 user 消息」和「已完成且有内容的 assistant 消息」，
// streaming 占位与 error 气泡天然被排除。
// 取最后 20 条：服务端校验有上限，留出安全余量
const history: ApiChatMessage[] = messages.value
  .filter(
    (item) =>
      (item.role === 'user' && item.content.trim()) ||
      (item.role === 'assistant' && item.type === 'complete' && item.content.trim()),
  )
  .map((item) => ({ role: item.role, content: item.content }))
  .slice(-20)
```

两个细节：只发 `role + content` 的最小结构（id、时间戳等 UI 字段不透传给模型）；截取最近 20 条控制上下文长度和 token 成本。

### 4.5 blocks 化渲染：气泡里不只有文字

最后一块拼图：AI 的回复不只是文字——工具调用过程、图片、交互组件都要出现在同一条气泡里。我的设计是把回复建模为**块（block）序列**，三条数据主线：

```text
SSEData（线上格式，一条 data: 消息）
   ↓ useAgentChat 翻译
MessageBlock（气泡内的一块：text / image / component / agent_status）
   ↓ 组成
ChatMessage（完整气泡：id / role / blocks / type 状态机）
```

渲染器 `InteractiveRenderer.vue` 按 `block.type` 分发：

```vue
<!-- src/views/aiChatroom/components/InteractiveRenderer.vue（节选） -->
<template v-for="(block, index) in blocks" :key="index">
  <!-- 文本块：markdown 渲染 -->
  <div v-if="block.type === 'text'" class="text-block"
       v-html="renderedMarkdown(block.content || '')"></div>

  <!-- 组件块：AI 下发的交互组件，白名单 + 懒加载 -->
  <component v-else-if="block.type === 'component' && block.component"
             :is="resolveComponent(block.component)" v-bind="block.props" />

  <!-- 工具状态块：Agent 循环里"正在调用 xx 工具"的过程提示 -->
  <div v-else-if="block.type === 'agent_status'" class="agent-status-block">
    <LoadingOutlined v-if="block.phase === 'start'" spin />
    <CheckCircleOutlined v-else />
    <span>{{ block.phase === 'start' ? `正在调用 ${block.name}…` : `已调用 ${block.name}` }}</span>
  </div>
</template>
```

这个设计的妙处在于**数据驱动**：气泡里"能出现什么"由块类型决定，服务端将来新增一种块（比如视频、表格），前端只需要在渲染器里加一个分支，上层链路一行不改。`component` 块还有一道白名单（`componentMap` 查表，查不到降级为空 div），既防止渲染崩溃，也杜绝了任意组件注入。

<a id="sec5"></a>

## 5. 让 AI 用上你的函数：Function Calling

流式解决的是"说得快"，Function Calling 解决的是"说得对"——你问"现在几点"，模型的训练数据里没有答案，它只能一本正经地编。Function Calling 让模型可以**请求调用你注册的函数**，拿到真实结果后再组织回答。

### 5.1 核心机制：模型只"下订单"，不执行任何东西

这是理解 FC 最重要的心智模型：**模型从头到尾没有执行任何代码**。它做的只是在回复里说一句"我想调用 `get_weather`，参数是 `{"city":"北京"}`"，执行永远发生在你的代码里（前端或后端）。所谓 Function Calling，本质是一套**"工具声明格式 + 消息协议"**的约定。

一个工具的声明分两部分，**同名对应、声明发给模型、实现留在本地**：

```js
// server/modules/chat-room-agent/agentTool.js（节选）
const toolDefinitions = [
  {
    type: 'function',               // OpenAI FC 固定包装层
    function: {
      name: 'get_current_time',     // 必须与 toolHandlers 的键完全一致
      description:
        '获取指定时区的当前时间。当用户询问现在几点、当前日期或当地时间时使用。',
      // parameters 就是 JSON Schema：模型按这个格式"生成"参数
      parameters: {
        type: 'object',
        properties: {
          timezone: {
            type: 'string',
            description: 'IANA 时区名称，例如 Asia/Shanghai、Asia/Tokyo',
          },
        },
        required: ['timezone'],
        additionalProperties: false,
      },
    },
  },
]
```

实践心得：**`description` 的质量直接决定触发准确率**。模型是靠读这段文字来判断"什么时候该用这个工具"的，把典型问法写进去，误触发会显著减少。

### 5.2 两轮循环：一次工具调用的完整时序

把工具发给模型后，一次完整的工具调用是"两轮请求"：

```text
你的代码                                    模型（LLM）
    │  ① messages + tools（工具说明书）        │
    │     tool_choice: 'auto'                 │
    │ ──────────────────────────────────────► │
    │                                         │ 模型：这题我需要调工具
    │  ② assistant 消息，带 tool_calls：      │
    │     { name:'get_current_time',          │
    │       arguments:'{"timezone":           │
    │         "Asia/Shanghai"}' }             │
    │ ◄────────────────────────────────────── │
    │                                         │
    │  ③ 你在本地执行 handler，拿到真实结果     │
    │  ④ messages 追加两条：                   │
    │     - assistant（含 tool_calls 原样回填）│
    │     - tool（含 tool_call_id + 结果）     │
    │                                         │
    │  ⑤ 完整 messages 再次请求（可不再带 tools）│
    │ ──────────────────────────────────────► │
    │  ⑥ assistant: "现在是 2026 年 9 月…"    │
    │ ◄────────────────────────────────────── │
```

三个协议细节最容易踩坑：

1. **`arguments` 是 JSON 字符串，不是对象**。模型返回的 `tool_calls[0].function.arguments` 是 `'{"timezone":"Asia/Shanghai"}'` 这样的一坨字符串，必须自己 `JSON.parse`；
2. **回填 tool 结果前，必须先把带 `tool_calls` 的 assistant 消息 push 进历史**。OpenAI 协议要求 tool 消息必须紧跟发起调用的 assistant 消息，缺了它网关直接 400——这是 FC 接入率最高的报错；
3. **`tool_call_id` 是回执编号**。一次回复可能同时调多个工具（"北京和东京几点？"），每个结果靠 `tool_call_id` 对号入座。

### 5.3 前端教学版：把两轮循环写直白

我的模块里有一版前端直连模型的两轮循环（`src/views/aiChatroom/utils/useSkillsChat.ts`），因为逻辑完全摊平、没有服务端参与，特别适合理解协议本身：

```ts
// src/views/aiChatroom/utils/useSkillsChat.ts（节选）
// ── 第一轮：带工具说明书请求 ──
const firstResponse = await fetch(DEEPSEEK_CHAT_URL, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${apiKey.value}`,
  },
  body: JSON.stringify({
    model,
    messages: conversationForApi,
    tools: toolsForApi,     // 技能声明（由 ChatSkill 转换而来）
    tool_choice: 'auto',    // 模型自己决定调不调
  }),
})
const firstData = await firstResponse.json()
const assistantMsg = firstData.choices[0].message
messages.value.push(assistantMsg)   // ← 坑 2：先回填 assistant 消息

// ── 中间：模型要调工具？在本地执行 ──
if (assistantMsg.tool_calls?.length) {
  for (const toolCall of assistantMsg.tool_calls) {
    let parsedArgs: Record<string, unknown> = {}
    try {
      parsedArgs = JSON.parse(toolCall.function.arguments)  // ← 坑 1：字符串！
    } catch { parsedArgs = {} }
    const result = await executeSkill(options.skills, toolCall.function.name, parsedArgs)
    messages.value.push({
      role: 'tool',
      content: result,
      tool_call_id: toolCall.id,   // ← 坑 3：回执编号
    })
  }

  // ── 第二轮：带着工具结果再请求一次，拿最终回答 ──
  const secondResponse = await fetch(DEEPSEEK_CHAT_URL, {
    method: 'POST',
    headers: { /* ... */ },
    body: JSON.stringify({ model, messages: messages.value }), // 含 tool 结果的完整历史
  })
  const finalMsg = (await secondResponse.json()).choices[0].message
  messages.value.push({ role: 'assistant', content: finalMsg.content ?? '' })
}
```

注意这版代码里工具是**在浏览器里执行的**——教学上清晰，但意味着 API Key 和工具实现全部暴露在前端。生产怎么办？看下一节的后端版。

### 5.4 ChatSkill：让工具声明变成"填表题"

如果每个工具都要手写一遍 OpenAI tools 格式，太啰嗦。我抽象了一个 `ChatSkill` 契约，四件套：

```ts
// src/views/aiChatroom/skills/types.ts
export interface ChatSkill {
  name: string          // 工具名（模型用它下订单）
  description: string   // 说明书（模型靠它决定何时触发）
  category?: string     // 分类（仅 UI 展示用）
  inputSchema: SkillInputSchema  // JSON Schema 参数声明
  handler: (args: Record<string, unknown>) => Promise<string>  // 本地实现
}
```

再配一个转换函数，把 ChatSkill 映射成 OpenAI tools 格式——`inputSchema` 直接落到 `parameters` 位置：

```ts
// src/views/aiChatroom/skills/index.ts
export function skillsToOpenAITools(skills: ChatSkill[]): OpenAIToolDefinition[] {
  return skills.map((skill) => ({
    type: 'function',
    function: {
      name: skill.name,
      description: skill.description,
      parameters: skill.inputSchema,
    },
  }))
}

export async function executeSkill(skills, skillName, args): Promise<string> {
  const skill = skills.find((item) => item.name === skillName)
  if (!skill) return `错误：未找到名为「${skillName}」的技能`
  return await skill.handler(args)
}
```

于是新增一个工具就变成"填表题"，比如这个 mock 天气技能：

```ts
// src/views/aiChatroom/skills/commonSkills.ts（节选）
{
  name: 'get_weather',
  description: '获取指定城市的实时天气信息',
  inputSchema: {
    type: 'object',
    properties: {
      city: { type: 'string', description: '城市名称，如北京、上海' },
    },
    required: ['city'],
  },
  handler: async (args) => {
    const weatherMap: Record<string, string> = {
      北京: '晴，24°C，湿度 40%',
      上海: '阴，28°C，湿度 70%',
    }
    return `${args.city} 天气：${weatherMap[args.city] ?? '晴转多云，22°C'}`
  },
}
```

我的模块里用这个契约注册了 11 个技能：4 个通用演示（天气/计算/翻译/搜索，全 mock）+ 7 个读取真实项目数据的业务技能（大屏目标查询、KPI、告警列表、飞行路线建议、路由导航等）。**业务技能和通用技能走的是同一套契约**——这就是抽象的力量。

### 5.5 生产版：后端 Agent 循环

真正给聊天页面用的版本，工具循环跑在服务端（`server/modules/chat-room-agent/chatRoomAgentService.js`）。相比教学版的两轮，它有三个升级：

**升级一：循环不是固定两轮，而是"直到模型不再要工具"，并设最大轮数防死循环：**

```js
// server/modules/chat-room-agent/chatRoomAgentService.js（节选）
// 核心循环：round 从 0 数到 maxToolRounds-1（默认 4），防止无限循环
for (let round = 0; round < config.agent.maxToolRounds; round += 1) {
  const response = await requestAgentModel({ messages, tools, model, signal })
  // 读流转发文字增量、拼装 tool_calls
  const { assistantMessage, usage } = await consumeAgentStream(response, onText)

  const toolCalls = assistantMessage.tool_calls || []

  // 分支 A：没有 tool_calls = 模型认为可以直接回答了，循环出口
  if (toolCalls.length === 0) {
    return { content: assistantMessage.content || '', rounds: round + 1, usedTools: round > 0 }
  }

  // 分支 B：先回填 assistant.tool_calls 消息（OpenAI 协议要求），
  // 再逐个执行工具、把结果 push 成 tool 消息
  messages.push({
    role: 'assistant',
    content: assistantMessage.content || null,
    tool_calls: toolCalls,
  })
  for (const toolCall of toolCalls) {
    await onToolStart({ name: toolCall.function?.name, callId: toolCall.id })
    const toolResultMessage = await executeToolCall(toolCall)
    messages.push(toolResultMessage)
    await onToolEnd({ name: toolCall.function?.name, callId: toolCall.id })
  }
  // for 循环结束后回到顶部：带着工具结果再次请求模型
}
throw new Error('Agent 工具调用超过最大轮数')
```

比如"帮我对比北京和东京的时间，再换算成纽约时间"这类问题，模型可能连续调用工具两三次，固定两轮的写法就歇菜了。

**升级二：工具执行前过白名单，报错不打断循环：**

```js
// server/modules/chat-room-agent/agentTool.js（节选）
export async function executeAgentTool(name, rawArguments) {
  // ① 白名单查表：toolHandlers 里没有这个名字就直接拒绝，
  //    防止模型"幻觉"出一个不存在的函数名（或被提示注入诱导）
  const handler = toolHandlers[name]
  if (!handler) throw new Error(`未注册的工具：${name}`)

  // ② 模型返回的 arguments 是 JSON 字符串（不是对象），需要手动 parse
  const argumentsObject = JSON.parse(rawArguments || '{}')

  // ③ 交给对应的 handler 执行（handler 内部还会再做一层参数校验）
  return handler(argumentsObject)
}
```

而**工具执行报错**时，不 throw 打断循环，而是把错误也包成一条 tool 消息回填给模型——让模型自己决定怎么向用户解释（换个参数重试，或道歉说明）。这比直接给用户看一坨 stack trace 优雅得多：

```js
// server/modules/chat-room-agent/chatRoomAgentService.js（节选）
} catch (error) {
  // 关键设计：工具报错不 throw 打断循环，而是"把错误也告诉模型"，
  // 让模型自己决定怎么向用户解释（比如换个时区重试或道歉说明）
  return {
    role: 'tool',
    tool_call_id: toolCall.id,
    content: JSON.stringify({ error: error.message }),
  }
}
```

**升级三：流式模式下拼装 tool_calls。** 这是整个 FC 链路最"脏"的活：模型决定调工具时，`tool_calls` 也是**以增量的形式分片到达**的——`id` 和 `name` 只在首个片段出现，`arguments` 字符串被随机切断：

```text
片段1 {index:0, id:'call_x', function:{name:'get_current_time', arguments:'{"time'}}
片段2 {index:0,                    function:{arguments:'zone":"Asia/Shanghai"}'}}
```

`index` 是每个调用的"工位号"（一次可能调多个工具），拼装逻辑：

```js
// server/modules/chat-room-agent/chatRoomAgentService.js（节选）
async function consumeAgentStream(stream, onText) {
  let content = ''
  const toolCalls = []   // toolCalls[index]：按"工位号"对号入座
  let usage = null

  for await (const chunk of stream) {
    const delta = chunk.choices?.[0]?.delta
    if (!delta) continue

    // ① 文字增量：立刻转发给路由层（马上写成一条 SSE → 打字机效果）
    if (delta.content) {
      content += delta.content
      await onText(delta.content)
    }

    // ② 工具调用增量：骨架不存在先建（首个片段带 id 和 name），
    //    后续片段只追加 arguments 字符串片段
    for (const part of delta.tool_calls || []) {
      const { index, id, function: fn } = part
      if (!toolCalls[index]) {
        toolCalls[index] = { id: '', type: 'function', function: { name: '', arguments: '' } }
      }
      if (id) toolCalls[index].id = id
      // 用 += 而不是 = ：个别网关会把 name 也拆成多段
      if (fn?.name) toolCalls[index].function.name += fn.name
      if (fn?.arguments) toolCalls[index].function.arguments += fn.arguments
    }
  }

  // 拼装成与 stream:false 的 message 同构的对象
  return { assistantMessage: { role: 'assistant', content: content || null,
          ...(toolCalls.length ? { tool_calls: toolCalls } : {}) }, usage }
}
```

看懂这段，你就理解了"流式 + Function Calling"叠加态的全部复杂性。而文字增量在这段代码里第一优先级就被 `onText` 转发出去了——**用户不会因为模型在调工具就看不到已生成的文字**，体验和工程在这里汇合。

### 5.6 工具到底该在前端执行还是后端执行？

| 维度  | 前端执行（教学版） | 后端执行（生产版） |
| --- | --- | --- |
| API Key | 暴露在浏览器（致命） | 只存在服务端 `.env` |
| 工具能力 | 只能碰浏览器环境（localStorage、页面跳转） | 可以碰数据库、内网、文件系统 |
| 适合场景 | 教学 demo、纯本地小工具 | 一切正经应用 |
| 中间过程可见性 | 天然可见 | 需要 agent_status 事件专门下发 |

一个真实的反面教材：我教学版的演示页图省事，把一个测试 Key 硬编码在了前端源码里。这类 Key 一旦发布就等于公开（打包产物、浏览器 DevTools 里都看得到），**任何出现在前端的 Key 都应该视为已泄露、立即作废**。生产架构里，前端只跟自己的后端说话，Key 和工具循环全部收在服务端。

<a id="sec6"></a>

## 6. 落地成应用：demo 和工程之间隔着什么

### 6.1 接口的演进：同一个问题的三次回答

我的服务端现在并存三个对话接口，恰好记录了这条链路的演进：

| 接口  | 定位  | 关键能力 |
| --- | --- | --- |
| `POST /api/chat` | 第一版：单轮直通 | 最简 SSE 转发，无历史、无工具、无结束标记 |
| `POST /api/openai-example/chat` | 第二版：规范示例 | 补上 `[DONE]`、charset、abort 接线、requestId |
| `POST /api/chat-room-agent` | 生产版：Agent | 多轮历史 + 工具循环 + agent_status 事件 + 错误事件 |

保留旧版本不是冗余——**对照着重读三版代码，能看到每个设计决策是为什么出现的**，这比直接看最终形态收获大得多。

### 6.2 后端分层与配置集中

服务端按"上层只调下层"分了四层：

```text
server.js    启动入口（listen、端口占用处理）
app.js       应用工厂（中间件、CORS 白名单、路由挂载、统一错误兜底）
routes/      路由层：HTTP 进出 + SSE 翻译，不写业务
modules/     业务层：调模型、跑 Agent 循环
config.js    配置中心：唯一有权读 process.env 的文件
```

配置集中是一条硬规矩：业务代码不允许直接读 `process.env`，统一从 `config.js` 拿。`.env` 里的关键配置项：

```text
PORT=3000
OPENAI_API_KEY=...        # 模型密钥，只存在服务端
OPENAI_BASE_URL=...       # 兼容网关地址（官方 / DeepSeek / 中转站）
OPENAI_MODEL=...          # 默认模型
CORS_ORIGIN=http://localhost:8080   # 前端来源白名单
AGENT_MAX_TOOL_ROUNDS=4   # Agent 循环最大轮数
```

CORS 用的是白名单函数式校验（`app.js` 里 `cors()` 的 `origin` 回调）：无 Origin 的 curl/Postman 放行、命中白名单放行、其余拒绝——比 `origin: '*'` 安全，比写死单个 origin 灵活。

### 6.3 一个真实业务技能长什么样

通用技能（天气/计算）是 mock，业务技能是动真格的。以"查询大屏目标状态"为例，它的 handler 直接读取项目里 TW 大屏的真实数据源，模型问到时返回的是**此刻的真实数据**；另一个 `search_demo_route` 技能的输入数据来自对路由配置的扁平化目录——用户说"打开说明 /twScreen 那个页面"，模型查目录、生成可点击的路由卡片（component 块）直接渲染在气泡里。

这就是 Function Calling 落地的完整闭环：**自然语言 → 模型选工具 → 真实数据 → 结构化块 → 交互组件**。

### 6.4 安全清单

最后把散落在各章的安全点收拢成一张清单，每一条都对应真实代码里的防御：

1. **API Key 只放服务端 `.env`**（前端直连模型仅限教学演示）；
2. **markdown 渲染必须净化**：`markdown-it` 开了 `html:true` + `v-html` 渲染模型输出 = 渲染不可信内容，生产必须过 DOMPurify 防 XSS；
3. **工具执行白名单查表**：模型"幻觉"出的函数名直接拒绝，防提示注入；
4. **工具入参先校验再使用**：永远不要信任模型生成的参数（我的 `get_current_time` handler 第一行就是校验 `timezone` 是非空字符串）；
5. **CORS 白名单**而非全放开；
6. **服务端消息校验**：角色白名单（`system/user/assistant/tool`）、条数上限、单条长度上限——既是安全也是省钱（防超长上下文烧 token）。

<a id="sec7"></a>

## 7. 展望：MCP 离我们还有多远

写到这你可能听说过 MCP（Model Context Protocol）并感到焦虑：Function Calling 会不会很快被淘汰？

先把概念对齐。FC 和 MCP 解决的是同一件事的不同侧面：

| 维度  | Function Calling | MCP |
| --- | --- | --- |
| 是什么 | 模型 API 的一个参数协议 | 一套完整的客户端-服务器协议 |
| 工具从哪来 | 每个应用自己声明、写死在代码里 | 服务器统一注册，客户端动态发现 |
| 工具在哪执行 | 你自己的进程里 | 独立的 MCP 服务器进程里 |
| 生态  | 各家 API 格式略有差异 | 统一协议，工具可跨应用复用 |

一个比喻：**FC 是"点菜"，MCP 是"美食广场"**——FC 里每家餐厅（应用）各自印菜单；MCP 把工具做成独立的档口（MCP Server），任何应用（MCP Client）进来都能发现并点单。

我的模块里有两个 "MCP 风格"的演示页，但说实话：**它们不是 MCP，是用 FC 模拟 MCP 的思想**——把工具声明 JSON Schema 化、声明与执行分离、可以动态增删技能。真正的 MCP 还有一整套传输层（stdio / SSE transport）、会话管理和发现机制。理解了 FC 再看 MCP，会发现它不过是把"工具说明书"从你的代码里搬到了协议层——**FC 是地基，MCP 是地基上的标准化**。先吃透 FC，MCP 只是多学一层皮。

<a id="sec8"></a>

## 8. 写在最后

回头看，这个模块跑通的东西可以压缩成两条心智模型：

> **流式 = 增量协议 + 状态机。** 服务端把生成过程切成 chunk 逐条推送（`data: JSON\n\n` + `[DONE]`），前端用"占位气泡 + 增量合并 + 响应式更新"把流翻译成界面。所有细节——多字节解码、半行缓冲、竞态门票、错误事件——都是这两个词的注脚。

> **Function Calling = 声明与执行分离 + 两轮循环。** 模型只读说明书（tools）下订单（tool_calls），执行永远在你的代码里（handler），结果以 `role:'tool'` + `tool_call_id` 回填，再要一轮请求拿最终回答。安全的关键就一句话：**白名单 + 不信任模型给的任何参数**。

前端工程师做 AI 应用，新东西其实只有这两条协议；剩下的——分层、状态管理、组件设计、错误处理——都是我们的老本行。这也是我做完这个模块最大的感受：**AI 应用开发是前端的延伸，不是转行**。

### 附：完整文件地图

| 文件  | 职责  |
| --- | --- |
| `src/views/aiChatroom/chatRoom.vue` | UI 编排层：组件接线 + 页面骨架 |
| `src/views/aiChatroom/composables/useAgentChat.ts` | 状态层：流 → 气泡、竞态防护、多轮历史 |
| `src/views/aiChatroom/composables/useChatSession.ts` | 会话层：多对话切换 + localStorage 持久化 |
| `src/views/aiChatroom/utils/useSSEChat.ts` | 传输层：fetch + 手写 SSE 解析（唯一碰网络） |
| `src/views/aiChatroom/utils/useSkillsChat.ts` | 前端版 FC 两轮循环（教学） |
| `src/views/aiChatroom/skills/types.ts` | ChatSkill 契约定义 |
| `src/views/aiChatroom/skills/index.ts` | skillsToOpenAITools / executeSkill 桥接 |
| `src/views/aiChatroom/skills/commonSkills.ts` | 通用技能（天气/计算/翻译/搜索） |
| `src/views/aiChatroom/skills/projectSkills.ts` | 业务技能（大屏数据/路由导航） |
| `src/views/aiChatroom/types/chat.ts` | 三条数据主线：SSEData → MessageBlock → ChatMessage |
| `src/views/aiChatroom/components/InteractiveRenderer.vue` | 块渲染器：text/image/component/agent_status 分发 |
| `server/routes/chat.js` | v1：单轮流式直通（教学保留） |
| `server/routes/chat-room-agent.js` | v3：Agent 接口（SSE 翻译 + 中断传播） |
| `server/modules/chat-room-agent/chatRoomAgentService.js` | Agent 核心循环 + 流式 tool_calls 拼装 |
| `server/modules/chat-room-agent/agentTool.js` | 工具说明书 + 白名单执行 |
| `server/config.js` | 配置中心（唯一读 process.env 的文件） |

### 跑起来

```bash
# 服务端（需要 .env 里配置 OPENAI_API_KEY / OPENAI_BASE_URL）
cd server && npm install && npm start    # localhost:3000

# 前端
npm install && npm run dev               # localhost:8080，进入 /chat 路由
```

如果你也想系统性走一遍这条链路，我在项目里维护了一份按阶段划分的学习路线（`docs/learning/智能问答学习路线.md`），从画数据流图开始，到亲手新增一个技能结束。**最好的入门方式，永远是动手改一版自己的**。