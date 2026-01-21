---
title: AI Native ：重塑从 0 到 1 的生产力法则与实战复盘
published: 2026-01-21
description: 拒绝营销词汇，回归工程本质。本文复盘了我如何利用 OpenCode、OpenSpec 和 SPEC Driven 理念，构建（半）全自动化流水线 Vibe Coding 的全过程。
image: https://d988089.webp.li/2026/01/21/20260121171152491.avif
tags: ["AI Native", "WorkFlow", "Spec Driven", "Agent"]
category: Agent
draft: false
---

> 拒绝营销词汇，回归工程本质。本文复盘了我如何利用 OpenCode、OpenSpec 和 SPEC Driven 理念，构建（半）全自动化流水线 Vibe Coding 的全过程。

## 1. 重新定义 AI Native：极致的“去人工化”

在我看来，所谓的 **AI Native（AI 原生）** 不应仅仅是一个被滥用的营销词汇，它应当是一条铁律，一种近乎偏执的工程原则：**凡是能用 AI 解决的环节，绝不消耗哪怕一分钟的人力。**

传统的软件工程是“人主导，工具辅助”，而我所实践的 AI Native，是将人类从“执行者”晋升为“决策者”和“审视者”。在这个新的范式里，人的核心价值不再是敲击键盘的速度，而是判断力与想象力。

:::important[核心观念转变]
在 AI Native 的工作流中，开发者不再是 **Coder**，而是 **Architect** 与 **Reviewer** 的结合体。你的代码产出量不应取决于你的手速，而应取决于你描述问题的清晰度。
:::

## 2. 全生命周期的“AI 饱和式”参与

从一个模糊的念头（Idea）诞生，到最终产品上线运营（DevOps & Marketing），我要求 AI 必须高度参与，甚至主导每一个环节。这不仅仅是效率的提升，更是生产流程的重构。

为了践行这一理念，我以最近正在筹备的项目 **Contextfy**（一个针对垂直领域的 CodeAgent 工具）为例，复盘我如何构建这套以 AI 为核心的流水线。

:::note[📢 资源分享：从 0.5 开始的 AI Agent 学习之旅]
最近在清华学习，打算边学习边总结过去半年做 AI Agent 开发的实践经验，于是开了一份文档记录这个过程，想分享给大家。

**📝 这份文档是什么？**
* **非入门教程**：不是从 0 到 1 的入门教程，而是实战经验与工程实践的总结。
* **Contextfy 实战**：围绕本文提到的实战项目 **Contextfy** 展开，解决 AI Coding 在私有框架/遗留系统中“最后一公里”的问题。
* **Live 开发日志**：从全栈视角出发，每天多次更新，实时记录技术决策、踩坑现场与迭代思路。

**⚠️ 适合人群**：
* 已经有一定 AI Agent 基础，想看看实践经验的朋友。
* 对 Vibe Coding 感兴趣，或已经在用 AI Coding 但想进一步提升能力的开发者。
* *不适合 0 基础（文档里有推荐的系统学习资源）。*

如果你也想要一起编辑、补充内容，可以私信我邀请你成为协作者，欢迎大家一起交流探讨！
👉 **[点击查看 Notion 文档](https://www.notion.so/0-5-AI-Agent-2eceda5b79ad806c8e88e313b5cae2e7?source=copy_link)**
:::

## 3. 实战复盘：我的项目开发工作流

### Phase 1: 立项与调研 —— 一个人活成一支队伍

如果你身处一个正常的研发团队，立项、竞品调研、PRD 撰写通常是产品经理（PM）的职责。但作为一个独立开发者（Indie Hacker），借助 AI，我们可以快速补齐这块短板。

**Contextfy** 的诞生源于我在开发 UGC 游戏内容时的痛点：市面上缺乏针对特定垂直领域的 CodeAgent。在立项阶段，我没有急着写代码，而是构建了一个“共生”的决策流：

1.  **Idea 具象化**：我负责提供核心价值观和灵感。
2.  **AI 市场调研**：我要求 AI 搜索并分析市面上现有的产品，列出优劣势对比。
3.  **功能定型**：通过多轮对话，将模糊的想法打磨成严谨的 PRD（产品需求文档）和 MVP 规划。

:::tip[Prompt Trick: 逆向思维]
在做竞品分析时，不要只问“有哪些产品”，尝试让 AI 扮演“挑剔的资深用户”，列出它在使用现有产品时遇到的 10 个最无法忍受的痛点。这些痛点往往就是你产品的**核心机会点（Unique Selling Point）**。
:::

### Phase 2: 开发 —— SPEC Driven 的降维打击

进入编码环节，我的核心理念是 **SPEC Driven（文档驱动开发）**。我不直接写代码，而是编写高精度的技术规范（Spec），让 AI 严格遵循规范产出代码。

#### 2.1 工具链选择：丰俭由人，适合即最优
工欲善其事，必先利其器。市面上的 **CodeAgent** 和 **AI IDE** 琳琅满目，选择哪一款完全取决于你的预算、使用习惯以及对“控制力”的需求。

我个人习惯使用 **OpenCode** 配合 **OpenSpec**，因为我喜欢 CLI 的纯粹和高度可定制性。但你完全可以根据自己的情况选择合适的 `Agent Runtime`。

* **OpenCode**: 一个强大的 CLI Agent Runtime，虽然不如商业软件出名，但它的可定制性极强。
* **OpenSpec**: 一个专门的规范驱动开发工具，用于定义和验证 AI 生成的代码。

:::note[工具说明]
**你不必非要使用 OpenCode。**
对于大多数开发者来说，[Claude Code](https://code.claude.com/docs/zh-CN/overview) 、[Cursor](https://cursor.com/cn) 或 [Antigravity](https://antigravity.google/) 是更容易上手的选择。如果你使用的是 [Kiro](https://kiro.dev/)，那么恭喜你，Kiro 本身就是 Spec Driven 理念的集大成者，它内置的 `requirements.md` / `design.md` 逻辑与我下面描述的流程异曲同工。
:::

:::note[名词解释: Agent Runtime]
你可以把 Agent Runtime 理解为 AI 的“操作系统”。它负责管理上下文（Memory）、调用工具（Tools）、执行文件操作（File I/O）。无论是 OpenCode 还是 Cursor，本质上都是在充当这个角色。
:::

:::warning[成本陷阱]
**请量力而行！** Spec Driven 开发模式会消耗大量的 Context Token。如果你使用的是 Claude 4.5 Sonnet/Ops 、GPT-5.2-Codex 或 Gemini 3 Pro 这样的顶级模型，一个复杂的重构任务可能会消耗几美元。对于个人开发者，建议搭配 GLM-4.7、MiniMax 2.1 以及 DeepSeek 等高性价比模型，或只在关键架构设计时使用顶级模型。
:::

#### 2.2 架构与任务拆解 (The Strategic Split)
项目初始化时，我会先将立项文档放入文档目录（注意：OpenSpec 通常在 `openspec/`，Kiro 在 `.kiro/`，请根据你的工具自行判断）。

这里有一个关键的策略调整：虽然 **OpenSpec** 能够全自动生成详细 Design 和 Tasks，但在项目初期，它往往显得“太重”且昂贵（Context 消耗巨大），或者因细节不足导致代码不确定。
因此，我的做法是 **“分而治之”**：

1.  **全局视野 (Global Context)**：使用 **Gemini 3 Pro**（或其他超长上下文模型）阅读全局文档，输出一份全局的 `Architecture.md` 和合理的 `Tasks.md`。
2.  **局部执行 (Local Execution)**：在全局规划定型后，再让 OpenSpec 介入具体开发。

#### 2.3 The Loop: Proposal & Apply
进入具体开发后，我的工作流遵循严格的循环：

1.  **初始化**：`/openspec proposal` 初始化项目结构，仅实现最基础的 Hello World。
2.  **循环迭代**：
    * 根据 `Tasks.md`，针对**单个功能模块**执行 `/openspec proposal` 生成局部技术方案。
    * **Review**: 人类介入，审查 Proposal 是否符合架构设计。
    * **Execute**: 执行 `/openspec apply` 让 AI 编写代码。
    * **Loop**: 不断测试、修补，直到功能跑通。后续需求变更，也是通过添加新的 Proposal 来实现。

:::important[Review 的重要性]
永远不要跳过 **Review Proposal** 的环节。这是人类防止 AI 产生“幻觉架构”的最后一道防线。一旦 Apply 了错误的代码，回滚的成本远比阅读文档高。
:::

### Phase 3: 前后端对接 —— 上帝视角的 Monorepo

传统的开发模式是：后端写文档 -> 丢给前端 -> 前端对着文档一顿猜 -> 联调撕逼。

在我的工作流中，我通常采用 Monorepo 策略，将前后端仓库放在同一个 Workspace 中。AI 拥有上帝视角，它能直接阅读后端的 Controller、DTO 代码，自动生成前端的 API Client 和类型定义。
**没有沟通成本，因为“沟通”在 Context 内部已经完成了。**

### Phase 4: 测试与 Debug —— 拒绝手动劳动

* **Mock 数据**：对于前端，我让 AI 根据 TypeScript 接口定义直接生成海量 Mock 数据。
* **E2E 测试**：如果你使用的是 [Antigravity](https://antigravity.google/)，你甚至可以直接编排一个 Workflow，让 AI Agent 自动完成全链路的端到端测试（这是 Antigravity 最强的地方）。
* **后端测试**：让 AI 针对每个 Service 方法编写单元测试或者集成测试，并自动生成 API 文档。

:::caution[警惕测试覆盖率]
虽然 AI 能秒生成 100% 覆盖率的单测，但**覆盖率不等于正确性**。AI 很擅长写“只会通过的测试”（Trivial Tests）。你需要抽查测试逻辑，或者让 AI 故意生成极端的测试案例，确保它真的测到了边界情况（Edge Cases）。
:::

### Phase 5: CR 与 CI/CD —— 自动化守门员

* **Code Review (CR)**：CR 往往能预先修复很多潜在问题。我建议配置 GitHub Action 接入 OpenCode（当然市面上也有很多其他做 CR 的产品，比如 Github Copilot） 或使用 Cursor 等 AI IDE 自带的 CR 功能，在 Commit 之前就拦截低级错误。
* **CI/CD**：这年头谁还自己写 YAML 脚本？直接告诉 AI 你的部署环境（Vercel, Docker, K8s），让它生成完整的 Pipeline。

### Phase 6: 文档工程 —— 知识的沉淀

如果你按照 Spec Driven 开发，当你执行 `/openspec archive` 时，确实已经产生了一堆 Spec 文档。**但请注意，这些文档并不适合人类阅读。**

:::caution[Spec 文档的局限性]
OpenSpec（或 Kiro）产出的文档通常是碎片化的（Fragmented）。
* **过于分散**：它们分散在各个 proposal 归档中，缺乏系统性。
* **内容贫瘠**：往往只有功能描述、粗略架构和 Tasks，缺乏具体的 API 使用指南或最佳实践。
* **难以聚合**：一个组件可能经历了多次 Proposal 迭代，你很难快速拼接出它的完整全貌。
:::

#### 我的选择：Nextra (DIY)

我选择 **[Nextra](https://nextra.site/)** (基于 Next.js 的 MDX 框架) 作为载体，原因有三：

1.  **速度与权限**：Next.js 的服务端渲染让加载极快。更重要的是，我可以自己写 Auth 逻辑（接入飞书、Github Org 登录），确保团队内部文档的安全性。
2.  **MDX 的魔力**：这对前端文档至关重要。文档不应只是静态文本，我可以在 MDX 中直接嵌入 React 组件（例如一个活的 API 调试台，或者组件的实时预览）。
3.  **AI 友好**：MDX 结构清晰，非常适合作为一个标准格式，让 AI 一键 Copy 并作为后续任务的 Context。

#### 其他选择 (SaaS & AI Knowledge Base)
当然，这只是我的个人偏好。如果你的团队不想维护文档站代码，市面上还有很多优秀的方案：
* **Mintlify / GitBook**: 开箱即用的文档 SaaS，界面美观，适合对外发布。
* **AI Native 知识库**: 像 **Greptile** 或 **Mem0** 这样的产品，它们不只是静态文档，而是能直接理解你的代码库，并提供类似 Chat 的查询接口。这可能是未来团队文档的终极形态。

:::tip[文档即代码 (Docs as Code)]
无论你选择什么方案，尽量确保文档是**随着代码一起提交的**。只有与代码同源，才能最大程度避免“文档过期”的诅咒。
:::

### Phase 7: 上线与 MCP

最后，本着“能不动手就不动手”的原则，上线过程同样交给 AI。

如果部署平台支持 **[MCP (Model Context Protocol)](https://modelcontextprotocol.io/)**，我甚至会直接配置好 MCP Server，让 AI 自行调用工具完成部署、域名解析等操作。MCP 正在成为连接 AI 与基础设施的标准协议，掌握它，你的 Agent 就能获得“物理世界”的操作能力。

## 结语：只有掌控过程，才能驾驭结果

很多人担心 AI 写代码容易“失控”，或者 AI 做产品容易“幻觉”。这并非 AI 的能力问题，而是**协作模式**的问题。

通过 **SPEC Driven** 约束输出，通过 **Global Context** 进行全局规划，通过 **Nextra** 沉淀知识。我们不再是代码的搬运工，我们是这套数字化流水线的指挥官。

在 AI 浪潮下，**生产力的上限，取决于你对工具链整合的想象力。**