---
layout: post
title: "AI 工程化五件套（一）Spec：把需求写成规格，AI 才不会自由发挥"
date: 2026-09-10 10:00:00
description: "Spec-Driven Development 入门：为什么 AI 编码时代要先写规格再写代码，一条 requirements → design → tasks → implement 的流水线怎么搭，附 EARS 格式示例。"
categories: [AI 应用开发]
tags: [AI, Spec, AI 编码]
---

> 这是「AI 工程化五件套」系列第一篇。这五个词经常一起出现，但各自解决的是完全不同的问题：
>
> | 概念 | 一句话定位 | 解决什么 |
> |------|-----------|---------|
> | **Spec** | 先写规格再写代码 | AI 不知道"要做什么" |
> | **Skill** | 按需加载的操作手册 | AI 不知道"这事我们团队怎么做" |
> | **MCP** | 工具接入的统一协议 | 每接一个工具就写一遍胶水代码 |
> | **RAG** | 先检索再生成 | AI 不知道"你的私有知识" |
> | **Function Calling** | 模型决定调哪个函数 | AI 只会说话，不会动手 |
>
> 系列其他篇：[Skill](/2026/09/10/ai-agent-skill/) · [MCP](/2026/09/10/ai-mcp-protocol/) · [RAG](/2026/09/10/ai-rag-retrieval/) · [Function Calling](/2026/09/10/ai-function-calling-basics/)

## 1. Spec 是什么

Spec = Specification（规格）。**Spec-Driven Development（规格驱动开发）** 的核心思想只有一句话：

> 开发流程中最有价值的产物不再是代码，而是规格文档；代码让 AI 写，人负责把规格写对。

过去写规格文档是给同事看的，写多细全凭自觉。现在规格的直接读者是 AI——它是一个"体力无限但缺乏上下文的初级工程师"：你不告诉它的，它就自由发挥。自由发挥在 demo 里没事，在日常迭代里就是返工。

2025 年这条路线集中爆发：GitHub 的 **Spec Kit**、AWS 的 **Kiro**、Claude Code 的各类 spec workflow 插件，流程大同小异，都是四步：

```
specify（写需求） → plan（定设计） → tasks（拆任务） → implement（生成代码）
```

## 2. 一条流水线长什么样

以"给后台系统加个密码登录"为例，三个产物各司其职：

**requirements.md —— 只写"做什么"，不写"怎么做"**

用 EARS（Easy Approach to Requirements Syntax）格式，每条需求都是 WHEN/IF + SHALL/THEN：

```markdown
## FR-1 密码登录

**WHEN** 用户输入正确的邮箱和已注册密码
**THEN** 系统签发有效期 2 小时的 JWT，并跳转首页

**WHEN** 同一账号密码连续错误 5 次
**THEN** 锁定该账号 15 分钟，并提示剩余等待时间

## FR-2 密码规则

**IF** 注册密码不满足"12 位以上，含大小写和数字"
**THEN** 前端与后端都拒绝提交，错误信息一致
```

这种格式的好处：每条需求都是可验证的断言，AI 生成完代码后可以逐条对照验收，而不是"看起来实现了"。

**design.md —— 技术决策，写清楚为什么**

```markdown
## 认证方案：JWT 双令牌
- access token 2h + refresh token 7d，理由：后台系统安全要求高于体验
- 不用 session：服务端要水平扩展
- 密码存储：bcrypt，成本因子 12
```

**tasks.md —— 拆成 AI 能一次吃下的任务**

```markdown
- [ ] T1: 建 users 表迁移文件（email 唯一索引）
- [ ] T2: 实现 /api/login 接口，bcrypt 校验 + JWT 签发
- [ ] T3: 登录失败计数与锁定逻辑（Redis，key: login:fail:{email}）
- [ ] T4: 前端登录表单 + 校验规则与后端对齐
```

任务粒度的经验值：**一个任务一次对话内能完成并自测**。粒度太大 AI 会偷工减料，太小又丢失上下文。

## 3. 和传统开发文档有什么区别

| | 传统 PRD / 设计文档 | Spec |
|---|---|---|
| 读者 | 人 | 人 + AI（机器友好优先） |
| 位置 | Confluence 里吃灰 | 仓库里，跟着代码走版本 |
| 变更 | 改了文档没人知道 | 改 spec → diff 可评审 → AI 按新 spec 重构 |
| 验收 | 靠测试同学的经验 | 需求本身是断言，可逐条核对 |

关键区别是最后一点：**spec 成了可维护的资产**。需求变了，先改 spec、提 PR 评审，再让 AI 按 diff 改代码——评审的是意图而不是实现，这对人是更轻松的工作。

## 4. 什么时候值得写 Spec

- ✅ 值得：新模块、多人协作的功能、有明确验收标准的需求（上面那条流水线 20 分钟就能跑完）
- ❌ 不值得：改个样式、修个明确 bug——直接把上下文扔给 AI 反而快

一个实用判断：**如果你跟 AI 的第一句话需要写超过 200 字来描述需求，那就该写成 spec 文件了**——反正这些字每次对话都要重复输入，存成文件还能迭代。

## 5. 与其他四件的关系

Spec 管"开发流程"（需求→代码），另外四个管"运行时能力"（AI 干活时缺什么）：Skill 补团队流程知识、MCP 补工具、RAG 补数据、Function Calling 是动手机制。Spec 里的 design.md 经常会直接写明："本项目接入 xx MCP server"、"领域知识用 RAG 检索"——规格先行，能力按需组合。
