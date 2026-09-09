# 博客内容规划：从 0 到 1

> 本文档是内容侧的「宪法」：定位、栏目、选题、节奏、工作流都定在这里，写作时随时回来对照。
> 站点配置相关的改动记录在 `_config.yml` / `_config.butterfly.yml`。

## 一、博客定位

- **一句话定位**：一个 5 年前端的进阶笔记——写原理、写实战、写踩坑，不写入门水文。
- **目标读者**：1~5 年的前端同学（包括面试场景）、以及半年后的自己。
- **差异化**：每篇都回答「为什么」，带可运行的代码验证，尽量有真实项目案例或源码依据；别人能搜到的名词解释不写，写「我踩过坑之后的理解」。
- **写作信条**：写不清楚 = 没搞懂。一篇讲透一个小主题，好过一篇糊十个。

## 二、栏目体系（6 个分类，不轻易增改）

| 分类（categories） | 覆盖方向 | tag 规范（举例） |
| --- | --- | --- |
| 前端基础 | HTML / CSS / JavaScript / TypeScript | `html` `css` `javascript` `typescript` `浏览器原理` `es6+` |
| Vue 进阶 | Vue2 / Vue3 / 生态 | `vue2` `vue3` `composition-api` `响应式` `pinia` `源码` |
| 小程序 | 原理 / 性能 / 跨端 | `微信小程序` `双线程` `taro` `uni-app` |
| 性能优化 | 指标 / 加载 / 渲染 / 监控 | `core-web-vitals` `懒加载` `长列表` `前端监控` |
| 前端工程化 | 构建 / 规范 / CI/CD / 微前端 / monorepo | `vite` `webpack` `pnpm` `qiankun` `ci-cd` |
| AI 与前端 | LLM 应用 / AI 编程 / RAG / Agent | `llm` `rag` `agent` `sse` `ai编程` |

规则：

1. 分类做「栏目」，tag 做「技术点」；一篇只属于一个分类，tag 可以多个。
2. 基础类用二级分类，如 `[前端基础, JavaScript]`；其余用一级。
3. 想到新 idea 时，先丢进 `source/_drafts/` 当素材（一行标题也算），攒够了再展开成文。

## 三、三阶段路线图

### 阶段一 · 冷启动（第 1~2 月，目标 8~10 篇）

从手里**最有素材**的开始写：最近做过的事、踩过的坑、面试被问住的题。
`source/_drafts/` 里已经放好 9 个方向各一篇的首篇骨架，直接往里填内容即可。
这个阶段的目标不是爆款，是**建立「写完一篇」的手感**。

### 阶段二 · 体系化（第 3~6 月，目标累计 20+ 篇）

按下方选题库把方向补成「系列」，优先级：**JS 原理系列 > Vue3 系列 > 性能优化系列**（这三类搜索流量和面试复用度最高）。
每个系列 4~6 篇，成系列后可在 Butterfly 里置顶目录页。

### 阶段三 · 树招牌（6 个月后）

选 1~2 个最擅长的方向写深度长文（如「从 0 实现一个迷你 Vue」「前端监控体系落地全记录」），形成个人标签；同步分发到掘金/知乎引流，博客做内容的「源站」。

## 四、选题库

> 标 ⭐ 的是系列首篇候选；`（实战）`表示适合结合自己项目写案例。

### HTML
- 被低估的 HTML：语义化、可访问性与那些你没用的标签 ⭐
- 现代浏览器渲染管线：从 HTML 到像素
- Web Components：原生组件方案还香吗

### CSS
- 现代布局选型指南：Flex / Grid / 容器查询 ⭐
- 层叠上下文与 BFC：那些年「莫名其妙」的样式问题
- CSS 动画性能：合成层、will-change 与掉帧排查
- 移动端适配方案演进：rem / vw / clamp
- 新特性实战：:has()、嵌套、Cascade Layers

### JavaScript
- 把事件循环讲透：宏任务、微任务与渲染时机 ⭐
- 内存管理与泄漏排查实战（实战）
- Promise / async-await 手写与设计剖析
- 原型链与 class：继承的演进史
- 深入浅出：手写防抖节流不如想清楚什么时候不用
- ES 模块与 Tree Shaking 的底层逻辑

### TypeScript
- TypeScript 进阶地图：从「会用」到「会设计类型」 ⭐
- 泛型与条件类型实战：写一个类型安全的请求封装
- tsconfig 关键配置逐项讲清
- TS 大型项目落地：类型收窄、unknown 与 any 治理（实战）

### Vue2 / Vue3
- Vue2 与 Vue3 响应式原理对比：defineProperty → Proxy ⭐
- diff 算法与 key 的正确用法
- Composition API 的设计思想：为什么不再用 Options
- Vue3 编译优化：PatchFlags、静态提升与 Block Tree
- Pinia vs Vuex：状态管理的取舍
- Vue2 项目迁移 Vue3 全流程实录（实战）

### 小程序
- 双线程架构解析：为什么 setData 是性能瓶颈 ⭐
- 小程序性能优化清单：分包、骨架屏、长列表（实战）
- 跨端方案对比：Taro / uni-app 怎么选
- 小程序登录态与支付的前端侧全流程

### 性能优化
- 前端性能优化全景图：指标、手段、监控体系 ⭐
- Core Web Vitals：LCP / INP / CLS 的优化清单
- 加载优化：资源体积、懒加载、预加载与 HTTP 缓存（实战）
- 长列表渲染：虚拟滚动方案对比与实现
- 从 0 搭建前端监控体系：采集、上报、告警（实战）

### 前端工程化
- Vite 为什么快：esbuild、按需编译原理与落地 ⭐
- Vite vs Webpack：迁移成本与收益评估（实战）
- pnpm + monorepo：多包仓库实践
- 代码规范全家桶：ESLint / Prettier / husky / CI 卡点
- 微前端落地：qiankun / Module Federation 选型（实战）
- 从 0 搭建一个组件库并发布 npm（实战）

### AI 与前端
- 前端工程师的 AI 入门：从流式接口到落地一个 AI 应用 ⭐
- SSE 流式渲染：打字机效果的前端实现细节
- Function Calling 与 Agent：LLM 应用的前端架构
- RAG 入门：给大模型接上你自己的知识库
- AI 编程工作流：Cursor / Copilot 的正确打开方式
- 浏览器端推理：Transformer.js / WebLLM 实践

## 五、文章结构模板

标题公式：**具体技术点 + 疑问或结果**（「为什么 X」「如何 X」「从 X 到 Y」「X 踩坑实录」）。

正文骨架：

1. **背景/问题**：什么场景下遇到的，为什么值得讲（1~2 段）
2. **原理拆解**：配图（excalidraw / mermaid），配源码或规范出处
3. **代码验证**：可运行的最小 demo，关键步骤加注释
4. **总结**：一张表或一张图收束；「一句话记住」
5. **参考**：规范、源码、优秀文章链接

front-matter 模板（`hexo new` 已内置）：

```yaml
---
title: 把事件循环讲透：宏任务、微任务与渲染时机
date: 2026-09-09 10:00:00
categories:
  - [前端基础, JavaScript]
tags:
  - JavaScript
  - 浏览器原理
---
```

## 六、本仓库写作工作流

```bash
npx hexo new draft "my-first-post"   # 新建草稿（source/_drafts/）
npx hexo server --draft              # 本地预览：http://localhost:4000
npx hexo publish my-first-post       # 定稿：移动到 source/_posts/
npm run clean && npm run build       # 构建到 public/
npm run deploy                       # 推送到 GitHub Pages（main 分支）
```

注意事项：

- **文件名用英文 slug**（permalink 含 `:title`，中文文件名会生成编码后的长 URL）；标题在 front-matter 里写中文。
- 源码分支是 `source`，部署目标是 `main`；建议每次发布后顺手 commit 源文件。
- 配图放 `source/img/`，文章里用 `/img/xxx.png` 引用。
- 发布前 checklist：本地 `--draft` 预览过、分类和 tag 填了、代码块语言标了、首图/摘要（`description`）写了。

## 七、更新节奏

- **周更 1 篇短文 > 月更 1 篇长文**；先完成，再完美。
- 每周日固定「选题 30 分钟」：把本周工作中的 idea 归入选题库或 `_drafts`。
- 允许鸽，不允许断更超过 3 周；卡住就先写一篇短的。
