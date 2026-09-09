---
title: 前端工程师的 AI 入门：从流式接口到落地一个 AI 应用
date: 2026-09-09 10:00:00
categories:
  - [AI 与前端]
tags:
  - llm
  - sse
  - ai编程
description: AI 方向不是算法工程师的专利。前端离 AI 应用最近的一公里：流式渲染、会话状态、上下文管理，这篇给出完整入门路径。
---

> 本篇目标：作为 AI 分类开篇，用「前端视角」把 LLM 应用的骨架搭起来，并以一个可部署的小应用（如网页版 AI 助手）作为贯穿案例。

## 一、前端在 AI 应用里的位置

- LLM 应用的技术栈拆解：模型 / 推理服务 / 编排 / **前端交互层**
- 前端的核心战场：流式体验、上下文管理、工具调用 UI、降级与容错

## 二、第一个接口：流式输出（SSE）

- SSE vs WebSocket vs 轮询：为什么 Chat 类应用清一色用 SSE
- fetch + ReadableStream 手写流式解析（处理 chunk 边界、event/data 协议）
- 打字机效果的三个实现档位：定时器 / requestAnimationFrame / 直接追加，各自体验差异

## 三、会话状态设计

- 消息列表的数据结构（role/content/工具调用记录）
- 上下文窗口管理：截断策略、摘要压缩（埋 RAG 篇伏笔）
- 多轮会话的 UI 状态机：loading / streaming / error / abort

## 四、贯穿案例：网页版 AI 助手（实战占位）

- 功能清单：流式对话 + 历史记录 + 中断生成 + 错误重试
- 技术选型：Vue3 + SSE，直接跑在浏览器 or 走轻量代理（ apiKey 安全问题）
- 分步实现计划，最终开源到 GitHub 并附 Demo 链接

## 五、AI 编程工作流（个人经验）

- Cursor / Copilot 的使用分层：补全 → 对话改码 → Agent 跑任务
- 什么时候信 AI、什么时候必须人审：一份个人 checklist

## 六、总结

- 一句话记住：**前端的 AI 机会在「最后一公里」的体验层，先把流式与状态管理做到极致**

## 参考

- OpenAI / Anthropic 流式接口文档
- MDN: Server-Sent Events
- Vercel AI SDK（交互层参考实现）
