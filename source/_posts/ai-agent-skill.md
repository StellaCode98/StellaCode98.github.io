---
layout: post
title: "AI 工程化五件套（二）Skill：给 AI 一本按需加载的团队操作手册"
date: 2026-09-10 10:00:00
description: "Agent Skill 是什么、和系统提示词/自定义命令/RAG/MCP 的区别在哪、SKILL.md 的渐进式加载机制，以及日常开发中怎么把重复劳动和踩坑记录沉淀成 Skill。"
categories: [AI 应用开发]
tags: [AI, Skill, Agent, Prompt 工程]
---

> 「AI 工程化五件套」系列第二篇。开篇与完整概念对照表见 [Spec 篇](/2026/09/10/ai-coding-spec-driven/)。

## 1. Skill 是什么

Skill（特指 Agent Skill，以 Anthropic 2025 年 10 月推出的规范为代表）是一个**放在文件夹里的操作手册**：一个 `SKILL.md` 加若干模板、脚本、参考资料。AI 平时不知道它的存在，**当任务匹配时才加载正文并按手册执行**。

一个最小的 Skill 长这样（以可视化大屏项目为例）：

```
screen-adaptation/
├── SKILL.md      # 手册正文
└── snippets/
    └── useScreenScale.ts
```

```markdown
<!-- SKILL.md -->
---
name: screen-adaptation
description: 可视化大屏的屏幕适配与图表自适应方案。当用户开发
  数据大屏、页面要适配 4K/超宽屏、ECharts 不跟随缩放时使用。
---

# 大屏适配流程

1. 基准分辨率 1920×1080，外层容器用等比 scale 缩放
   （直接复制 snippets/useScreenScale.ts，不要另写一套）
2. ECharts 容器禁止写死宽高，用 ResizeObserver 监听
   容器尺寸变化后调 chart.resize()
3. 超宽屏（≥3440）单独走左右分栏布局，细节见 docs/ultrawide.md
4. 组件卸载时 dispose 图表、断开 observer——大屏要 7×24 长跑，
   内存泄漏是上线后最大的坑
```

frontmatter 里只有两个必填项：`name` 和 `description`。**description 是整个机制的核心**——它常驻在 AI 的上下文里，相当于目录页；AI 靠它判断"当前任务要不要翻开这本手册"。description 写得含糊，Skill 就永远不会被触发。

## 2. 解决什么问题：同一份知识的三种给法

假设"我们团队的大屏图表封装规范"这个知识要交给 AI，有三种方式：

| 方式 | 做法 | 问题 |
|------|------|------|
| 系统提示词 | 每次对话都塞进去 | 所有任务都背着这份上下文，token 永久占用 |
| 传统 RAG | 切片存向量库 | 离散的 chunk 丢了"流程"的结构，AI 知道细节但不知道顺序 |
| Skill | 存成 SKILL.md | 元信息常驻，正文按需加载；手册可以任意长 |

Skill 的本质是**渐进式加载（progressive disclosure）**：上下文里只放目录（name + description），命中任务才读正文，正文里还能再引用脚本、模板进一步展开。这和操作系统的虚拟内存是同一个思想——不是所有知识都要常驻内存。

## 3. Skill vs 容易混淆的几个东西

**vs 自定义命令（slash command）**：命令是"用户显式触发"（输入 `/deploy`），Skill 是"AI 自己判断要不要用"。命令像点菜，Skill 像一个有眼力见的助理自己翻手册。

**vs MCP**：MCP 给 AI 接**能力**（能查数据库、能发请求、能开浏览器）；Skill 给 AI 接**知识**（知道大屏适配该用 scale 方案、ECharts 什么时候 resize、有什么坑）。一个常见组合：MCP 提供浏览器调试工具，Skill 教它大屏联调时图表空白按什么顺序排查、先核对哪份接口文档——**能力靠 MCP，方法论靠 Skill**。

**vs 系统提示词**：系统提示词是"你是一个高级工程师"这种人设和全局规则；Skill 是具体场景的操作知识。前者改一次全局生效，后者按任务匹配。

## 4. 日常开发怎么用

最实际的起步方式：**把你的口头重复劳动沉淀成 Skill**。判断标准很简单——同样的话你对 AI 说过三遍以上，就该写成 Skill 了。比如：

- `bigscreen-release-checklist`：大屏上线前的检查清单（F11 全屏实测、三种分辨率截图对比、定时器与 observer 清理、跑 24 小时看内存曲线）
- `echarts-chart-spec`：团队 ECharts 封装规范（按需引入、统一主题、resize 与 dispose 的时机）
- `bigscreen-data-debug`：大屏数据联调排查手册（接口字段对不上、WebSocket 断线重连、接口挂了降级到 mock 数据）

最好的素材不是凭空想出来的，而是**踩过的坑**。之前写 AI 聊天模块时（[上一篇](/2026/09/09/ai-from-streaming/)），我记了一整节"SSE 四坑 + FC 三个协议细节"——这类知识不沉淀，下次让 AI 改流式代码就得把每个坑重新口述一遍。直接写成 Skill：

```markdown
---
name: ai-chatroom-debug
description: 排查 AI 聊天模块的流式与工具调用问题。
  当 SSE 不出字、中断不生效、FC 报 400 时使用。
---

# AI 聊天模块（aiChatroom）排查手册

## 流式（SSE）
1. 响应头必须在第一条 data: 之前设置；已经开始写流就只能用
   error 事件传错误，不能再改状态码
2. 判断客户端断开：监听 res 的 close 并检查 writableEnded；
   req 的 close 是假阳性（body 读完就触发）
3. SDK 配置项是 baseUrl（全小写），写错不报错、静默回落官方地址

## Function Calling
4. 回填 tool 结果前，必须先 push 带 tool_calls 的 assistant
   消息，否则网关直接 400（接入率最高的报错）
5. arguments 是 JSON 字符串不是对象：parse 要 try/catch，
   参数校验后再用
```

每一条都对应上一篇真实代码里的防御（出处见[那篇的第 2、5 章](/2026/09/09/ai-from-streaming/)）。注意正文只有十几行——手册不是文档站，**只写"下次还会犯"的事**。

写 SKILL.md 的几条经验：

1. **description 写触发条件，不写内容摘要**。"当用户要上线大屏或做适配改造时使用" 好于 "大屏适配方案介绍"。
2. **正文用指令式语气**，步骤可执行、可验收，别写背景科普。
3. **超过 500 行的手册要分层**：SKILL.md 只放流程主干，细节放子文档按需引用。
4. Skill 放在仓库里跟着代码走版本（如 `.claude/skills/`），新人 clone 下来 AI 就自动继承了团队经验——这是比文档站更有效的知识传承方式。

## 5. 小结

Skill 解决的是"AI 不知道这事我们团队怎么做"。它和 Spec 互补：Spec 定义一个功能的意图，Skill 沉淀跨功能的组织知识。下一篇讲 MCP——Skill 教 AI"怎么想"，MCP 给它"手"。
