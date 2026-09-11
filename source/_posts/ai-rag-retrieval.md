---
layout: post
title: "AI 工程化五件套（四）RAG：先检索再生成，让 AI 用上你的私有知识"
date: 2026-09-11 13:20:00
description: "RAG 的完整链路：文档切分 → 向量化 → 相似度检索 → 带上下文生成；不用任何框架、纯手写搭一个能查内部文档的问答，以及日常落地中最影响效果的三个环节。"
categories: [AI 应用开发]
tags: [AI, RAG, LLM]
---

> 「AI 工程化五件套」系列第四篇。开篇与完整概念对照表见 [Spec 篇](/2026/09/10/ai-coding-spec-driven/)。

## 1. RAG 是什么

RAG = Retrieval-Augmented Generation（检索增强生成）。解决一个问题：

> 模型的知识来自训练数据，训练截止之后的事、你公司内部的事，它一概不知——不知道就会**一本正经地胡编**（幻觉）。

RAG 的思路直白：**回答之前先查资料**。把"开卷考试"替代"闭卷考试"：

```
用户提问 ──→ ① 检索：从你的知识库里找出最相关的几段
         ──→ ② 生成：把问题 + 检索到的段落一起给模型，"根据以下资料回答"
```

典型场景：内部文档问答、客服机器人、代码库问答、法律/医疗等需要引用出处的场景。相比微调模型，RAG **成本低、知识可随时更新、能给出处**，是私有知识接入的第一选择。

## 2. 完整链路：四步

**离线阶段（建库）**：文档 → 切分 → 向量化 → 存入向量库

**在线阶段（问答）**：问题 → 向量化 → 相似度检索 → 拼上下文 → 生成

展开每一步：

**① 切分（Chunking）**：把长文档切成段。常用 500 字一块、重叠 50 字——重叠是为了避免关键句正好被切断。

**② 向量化（Embedding）**：用 embedding 模型把每段文字变成一个高维向量（如 1024 维），**语义相近的文本，向量距离近**。"怎么退货"和"退款流程"向量很近，虽然一个字都不相同——这就是超越关键词搜索的地方。

**③ 检索**：用户问题也向量化，在向量库里找最近的 K 个块（余弦相似度）。进阶做法是混合检索：向量 + 关键词（BM25）各查一遍再融合，兼顾语义和精确匹配（如错误码、函数名）。

**④ 生成**：把检索结果拼进提示词。

```
你是客服助手，仅根据以下资料回答，资料里没有就说不知道。

【资料】
{retrieved_chunks}

【问题】
{question}
```

"仅根据资料回答"这句约束 + 给出处，是压制幻觉的关键手段。

## 3. 手写一个最小 RAG：不用任何框架

RAG 的最小实现只需要你已经会的东西：**一个 embedding 接口 + 一个相似度函数 + 普通的 chat 接口**——后两个在[上一篇](/2026/09/09/ai-from-streaming/)里都手写过。先裸写一遍，才知道框架到底帮你做了什么：

```typescript
const BASE_URL = "https://api.openai.com/v1"; // 任何 OpenAI 兼容网关均可

// embedding 接口：文本数组进、向量数组出——RAG 唯一要用到的新接口
async function embed(texts: string[]): Promise<number[][]> {
  const res = await fetch(`${BASE_URL}/embeddings`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${API_KEY}` },
    body: JSON.stringify({ model: "text-embedding-3-small", input: texts }),
  });
  return (await res.json()).data.map((d) => d.embedding);
}

// 余弦相似度：向量夹角越小 → 语义越近。十几行，初中数学
function cosine(a: number[], b: number[]) {
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i]; na += a[i] ** 2; nb += b[i] ** 2;
  }
  return dot / (Math.sqrt(na) * Math.sqrt(nb));
}

// ── 离线：建库（此刻"向量库"就是一个平行数组）──
const doc = await fs.readFile("onboarding.md", "utf-8");
const chunks: string[] = [];
for (let i = 0; i < doc.length; i += 450) chunks.push(doc.slice(i, i + 500)); // 500 块长，50 重叠

const vectors = await embed(chunks); // chunks[i] ↔ vectors[i] 一一对应

// ── 在线：检索 + 生成 ──
const question = "新员工报销流程是什么？";
const [qVec] = await embed([question]);

const top3 = vectors
  .map((v, i) => ({ chunk: chunks[i], score: cosine(qVec, v) })) // 逐块打分
  .sort((a, b) => b.score - a.score)                              // 按相似度排序
  .slice(0, 3);                                                   // 取前三

// 拼上下文，走普通 chat 接口——就是上一篇用过的那个
const messages = [
  { role: "system", content: "仅根据以下资料回答，没有就说不知道。\n\n" +
    top3.map((t, i) => `【${i + 1}】${t.chunk}`).join("\n\n") },
  { role: "user", content: question },
];
```

没有任何新框架——所谓"向量检索"，裸写出来就是**遍历打分、排序、取前三**。跑通之后再回头看那些名词，就祛魅了：

- **向量数据库（Chroma、Milvus…）**：核心就是把上面这个平行数组**持久化**，再加近似最近邻（ANN）索引——数据量大了不挨个算，用近似换速度。几百个块的规模，一个数组就够，不必引库。
- **LangChain 这类 RAG 框架**：把 ①②③④ 串起来、把每一步做成可替换组件。知道了底下是什么，框架就只是语法糖，要用再用。

顺带一提：把 `top3` 拼进上一篇聊天模块的 system 消息，那个 AI 助手立刻就长出了"私有知识"——RAG 和你已经写过的代码之间，只隔着这几十行。

## 4. 落地时最影响效果的三个环节

按踩坑频率排序：

1. **切分策略 > 模型选择**。多数"RAG 答不准"是切坏了：表格被拦腰切断、代码块和解释分了家。按文档结构（标题/段落）切，优于按固定字数切。
2. **检索质量 > 生成模型**。检索不相关，再强的模型也只能瞎答或拒答。宁可 top-5 里有两块不相干，也别漏掉关键块；预算允许就加 rerank。
3. **评测要建在前面**。准备 20~50 个"问题-标准答案-出处"三元组，每次调整切分/检索参数就跑一遍。没有评测集，调优就是盲调。

## 5. 和其他四件的关系

RAG 补的是**知识**（数据侧），Skill 补的也是知识，但形态不同：RAG 适合**海量、离散、持续更新**的知识（几万份文档），Skill 适合**少量、结构化、流程性**的知识（几十个操作手册）。一个判断口诀：**知识能写成"步骤"就是 Skill，只能"检索"的就是 RAG**。而检索到的资料要变成行动，就轮到 Function Calling 和 MCP 了——见下一篇。
