---
layout: post
title: "AI 工程化五件套（三）MCP：给 AI 应用接工具的 USB 接口"
date: 2026-09-10 10:15:00
description: "MCP 协议全景：Server 提供的三种能力（tools/resources/prompts）、stdio 与 HTTP 两种传输、一个可运行的天气查询 Server 实例，以及日常开发中的选型建议。"
categories: [AI 应用开发]
tags: [AI, MCP, Agent, Tool]
---

> 「AI 工程化五件套」系列第三篇。开篇与完整概念对照表见 [Spec 篇](/2026/09/10/ai-coding-spec-driven/)。

## 1. MCP 是什么

MCP（Model Context Protocol）是 Anthropic 2024 年 11 月开源的协议，解决的是一个在 AI 应用开发里反复出现的浪费：

> 每做一个 AI 应用，就要给每个模型重新接一遍数据库、文件系统、搜索……M 个应用 × N 个工具 = M×N 份胶水代码。

MCP 把这个问题变成 M+N：工具方实现一次 **MCP Server**，应用方实现一次 **MCP Client**，中间走统一协议。官方的类比很形象——**"AI 应用的 USB-C 接口"**：外设（数据库、GitHub、Figma…）做成标准插头，即插即用。

```
┌─────────┐   MCP 协议    ┌─────────────┐
│ Host 应用 │ ⟷ MCP Client │ MCP Server A │ → GitHub
│ (Claude,  │              ├─────────────┤
│ Cursor…)  │              │ MCP Server B │ → Postgres
└─────────┘              └─────────────┘
```

2025 年它被 OpenAI、Google 等相继采纳，事实上成了 Agent 工具接入的行业标准。

## 2. 一个 MCP Server 能提供三种东西

| 能力 | 是什么 | 控制权在谁 |
|------|--------|-----------|
| **Tools** | 可执行的函数：查库、发请求、写文件 | 模型决定调用（有副作用，应用审批） |
| **Resources** | 只读数据：文件内容、配置、日志 | 应用决定加载 |
| **Prompts** | 预置的提示词模板 | 用户主动选用（如 `/analyze-db`） |

日常开发里 90% 的场景用的是 **Tools**。比如官方的 MCP Server：`filesystem`（读写文件）、`github`（PR/Issue 操作）、`postgres`（安全 SQL 查询）、`puppeteer`（浏览器自动化）。

## 3. 传输方式：stdio 与 HTTP

- **stdio**：Server 作为子进程启动，走标准输入输出。**本机开发用这个**，零网络配置。
- **Streamable HTTP**：Server 独立部署，HTTP 通信。**远程/团队共享用这个**，可加鉴权。

以 Cursor 为例，配置一个本机 Server 只需要一段 JSON：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "C:\\projects"]
    }
  }
}
```

保存后 AI 就多了"直接读写 C:\projects 下文件"的能力，无需任何代码。

## 4. 写一个最小的 MCP Server

用官方 TypeScript SDK，实现一个天气查询工具（`@modelcontextprotocol/sdk`）：

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "weather", version: "1.0.0" });

server.tool(
  "get_weather",                                   // 工具名
  "查询指定城市的实时天气",                          // 描述：模型靠它决定何时调用
  { city: z.string().describe("城市名，如 北京") }, // 参数 schema
  async ({ city }) => ({
    content: [{ type: "text", text: `${city} 晴，26℃` }], // 模拟数据
  })
);

await server.connect(new StdioServerTransport());
```

不到 20 行。应用侧（Client）不用写任何解析代码——SDK 和协议把"发现工具、校验参数、拿到结果"全部标准化了。对比一下没有 MCP 的时代：每个应用都要自己定义工具的注册格式、参数校验、错误约定，接三个工具就是三套写法。

## 5. 和 Function Calling 什么关系

最容易混淆的一对，一张图说清：

```
用户: "北京今天多少度？"
  │
  ▼
模型决策（Function Calling 机制）：该调 get_weather("北京")   ← 大脑
  │
  ▼
MCP Client 发起调用 → MCP Server 执行 → 返回结果              ← 手臂
  │
  ▼
模型拿到结果，组织回答："北京今天 26℃，晴 ☀️"
```

- **Function Calling 是模型的能力**：看到用户问题，决定"要不要调工具、调哪个、传什么参数"——决策层。
- **MCP 是工程的协议**：工具怎么注册、怎么传输、怎么执行——执行层。

模型完全可以通过 Function Calling 调一个 MCP 管理的工具；也可以反过来，不用 MCP、把函数直接写在应用里（下一篇会讲这种直连写法）。**决策靠 Function Calling，接入靠 MCP**，两者是配合关系而非替代关系。

## 6. 日常开发选型建议

- **个人/团队提效**：优先装现成的官方/社区 Server（github、postgres、playwright…），配置即用。
- **自己的应用要接工具**：内部小工具直接用 Function Calling 直连最快；工具要多方复用、或要用别人做好的 Server 时，上 MCP。
- **注意安全边界**：Tools 有副作用（写库、删文件），生产环境务必开启应用侧的审批（human-in-the-loop），别让模型静默执行危险操作。
