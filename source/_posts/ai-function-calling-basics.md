---
layout: post
title: "AI 工程化五件套（五）Function Calling：让模型从「会说」到「会做」的机制"
date: 2026-09-10 10:30:02
description: "Function Calling 的两轮循环：模型不执行函数，只产出调用意图；用 OpenAI SDK 实现一个查天气的完整闭环，以及流式、并行调用、强制调用等进阶用法。"
categories: [AI 应用开发]
tags: [AI, Function Calling, LLM, Agent]
---

> 「AI 工程化五件套」系列第五篇（完结）。开篇与完整概念对照表见 [Spec 篇](/2026/09/10/ai-coding-spec-driven/)。我之前写过一篇偏前端工程视角的 [AI 入门：从流式接口到落地一个 AI 应用](/2026/09/09/ai-from-streaming/)，SSE 解析器等细节在那边，本文专注 Function Calling 机制本身。

## 1. Function Calling 是什么

Function Calling（工具调用）是 LLM API 的一项能力：**你把函数清单告诉模型，模型在合适的时机不输出文字，而是输出"我要调哪个函数、参数是什么"的结构化 JSON**。

首先要纠正一个最常见的误解：

> **模型从头到尾不执行任何函数。** 它只输出调用意图（函数名 + 参数的 JSON），真正的执行永远发生在你的应用里。流程是两轮循环：

```
第一轮：用户问题 + 函数清单 → 模型返回 tool_calls（意图）
              │
              ▼
你的应用解析意图，真正执行函数（查库、发请求……）
              │
              ▼
第二轮：把执行结果回传 → 模型组织成自然语言回答用户
```

一句话：**模型是大脑，你的代码是手**。

## 2. 完整代码：一个查天气的闭环

OpenAI SDK（任何兼容网关如 DeepSeek 同样适用）：

```typescript
import OpenAI from "openai";
const client = new OpenAI();

// ① 你的业务函数——普普通通的代码，模型不知道它的存在
function getWeather(city: string) {
  return { city, weather: "晴", temp: 26 };
}

const tools = [
  {
    type: "function" as const,
    function: {
      name: "getWeather",
      description: "查询指定城市的实时天气",   // 模型靠它决定何时调用
      parameters: {
        type: "object",
        properties: { city: { type: "string", description: "城市名" } },
        required: ["city"],
      },
    },
  },
];

// ② 第一轮：把问题 + 工具清单发给模型
const messages: any[] = [{ role: "user", content: "北京今天多少度？" }];
const res = await client.chat.completions.create({
  model: "gpt-4o",
  messages,
  tools,
});

const call = res.choices[0].message.tool_calls?.[0];
if (call) {
  // ③ 模型没有回答，而是返回了调用意图
  const args = JSON.parse(call.function.arguments); // { city: "北京" }
  const result = getWeather(args.city);             // ← 执行的是你的代码

  // ④ 第二轮：意图 + 结果回传，让模型组织回答
  messages.push(res.choices[0].message);            // 模型的 tool_calls 消息
  messages.push({
    role: "tool",
    tool_call_id: call.id,
    content: JSON.stringify(result),                // { city: "北京", weather: "晴", temp: 26 }
  });
  const final = await client.chat.completions.create({
    model: "gpt-4o", messages, tools,
  });
  console.log(final.choices[0].message.content);    // "北京今天 26℃，晴 ☀️"
}
```

注意 ④ 里回传了两条消息：模型那条 `tool_calls`（意图）和一条 `role: "tool"`（结果），**缺第一条模型就不知道自己在等哪个结果**——这是新手最常踩的坑。

## 3. 几个关键细节

**description 是给模型看的接口文档。** 写得越准，调用时机和参数就越对。"查询指定城市的实时天气" 会比 "天气查询" 命中率高得多。参数的 `description` 同理。

**参数是模型生成的，必须校验。** `JSON.parse` 可能失败，`city` 可能是模特自由发挥的"北京市北京市"。生产代码里 parse 要 try/catch，参数要过一遍 zod 之类的校验，非法时把错误信息以 `role: "tool"` 回传——模型看到错误会自我修正。

**并行调用**：一句"北京和上海呢？"模型可能返回多个 `tool_calls`，逐个执行、逐个回传即可。

**强制走工具**：设置 `tool_choice: "required"`，适合"必须查库才能答"的场景；默认的 `"auto"` 则由模型自己判断。

## 4. 从两轮到循环：不靠框架写出 Agent 的雏形

上一节的闭环是"最多调一次工具"的直线流程。把两轮循环套进循环体，让模型拿到工具结果后还能继续要下一个工具，就得到了 Agent 的雏形——不需要任何框架，核心只有十几行：

```typescript
const MAX_ROUNDS = 4;                  // 循环上限：防无限调工具烧 token
for (let round = 0; round < MAX_ROUNDS; round++) {
  const res = await callModel(messages);
  const calls = res.choices[0].message.tool_calls;
  if (!calls) break;                   // 模型不再调工具 → 最终回答
  messages.push(res.choices[0].message);
  for (const c of calls) {
    messages.push({
      role: "tool",
      tool_call_id: c.id,
      content: JSON.stringify(await dispatch(c)), // 按函数名路由执行
    });
  }
}
```

所谓 Agent，本质就是 **LLM + 工具 + 循环**——模型在每一轮里自主决定下一步做什么，直到它认为可以直接回答。这个雏形我没有借助任何 Agent 框架：[上一篇](/2026/09/09/ai-from-streaming/)里我服务端的 `chatRoomAgentService.js` 就是它的生产版，在雏形之外多加了三道保险——`maxToolRounds` 循环上限、工具白名单查表、工具报错不打断循环而是包成 tool 消息回填给模型（细节见那篇 5.5 节）。换句话说，**没学过 Agent 框架，照样能写出能用的 Agent**；框架做的事，无非是把这个循环连同状态管理、重试一起工程化——先把裸循环吃透，以后真要用框架，也只是认一层语法糖。

## 5. 系列收官：五件套的全景图

回到开篇那张表，现在可以画成一张协作图：

```
需求 ──Spec──→ 规格文档 ──→ AI 生成代码          （开发流程层）
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
           Skill            RAG            Function Calling
        （团队方法论）    （私有知识）      （行动机制）
              └───────────────┼───────────────┘
                              ▼
                            MCP                （工具接入标准）
```

- **Spec** 管"写代码之前"：把意图写清楚，AI 才不跑偏——[第一篇](/2026/09/10/ai-coding-spec-driven/)
- **Skill** 管"怎么做"：团队经验按需注入——[第二篇](/2026/09/10/ai-agent-skill/)
- **RAG** 管"知道什么"：私有知识先检索再生成——[第四篇](/2026/09/10/ai-rag-retrieval/)
- **Function Calling** 管"动手"：从说话到调用——本篇
- **MCP** 管"工具从哪来"：一次接入处处可用——[第三篇](/2026/09/10/ai-mcp-protocol/)

没有哪一个能替代其他：Spec 是流程，Skill/RAG 是知识，Function Calling 是机制，MCP 是协议。日常开发里它们也确实总是一起出现——你每天都在用的 Cursor / Claude Code，就是这五件的集大成者。
