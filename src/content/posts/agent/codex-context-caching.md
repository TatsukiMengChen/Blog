---
title: Context Engineering：从 OpenAI Codex 架构看 AI Agent 的性能优化之道
published: 2026-01-25
description: 基于 OpenAI Codex 的技术实践，本文探讨了 AI Agent 开发中往往被忽视的性能关键——Context Caching。通过五大工程法则，解析如何在昂贵的 KV Cache 和有限的上下文窗口间寻求平衡，实现低延迟的生产级架构。
image: https://d988089.webp.li/2026/01/25/20260125165051037.avif
tags: ["上下文工程", "性能优化", "KV Cache", "OpenAI", "Agent"]
category: Agent
draft: false
---

> 基于 OpenAI Codex 的技术实践，本文探讨了 AI Agent 开发中往往被忽视的性能关键——Context Caching。通过五大工程法则，解析如何在昂贵的 KV Cache 和有限的上下文窗口间寻求平衡，实现低延迟的生产级架构。

最近，我深入阅读了 OpenAI 发布的关于 Codex CLI 架构的技术文章 [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/)。这篇文章虽然主要在讲 Codex Agent 的工作流，但其字里行间透露出的关于 **Context（上下文）构建与管理** 的细节，让我对 AI Agent 的工程化有了全新的认识。

很多人认为 Agent 开发就是写好 Prompt，但 OpenAI 告诉我们：**Context Engineering（上下文工程）本质上是系统架构设计。**

对于生产级 Agent 而言，**Prompt Caching（前缀缓存）** 是降低延迟（Latency）和成本的关键。然而，如果不理解其底层原理，我们很容易写出导致缓存失效的代码。在这篇文章中，我们将深入探讨如何通过精密的 Context 结构设计，在昂贵的计算资源（KV Cache）和有限的注意力窗口（Context Window）之间寻找平衡。

## 核心原理：为什么 Cache Hit 如此重要？

在深入法则之前，我们需要纠正一个常见的认知误区。

Prompt Caching 的核心机制是 **前缀匹配（Prefix Matching）**。缓存是按顺序存储的，一旦中间某个 Token 发生变化，其后所有内容的 KV Cache（Key-Value Cache）都会失效，必须重新计算。

:::note[技术深潜：Compute Bound vs. Memory Bound]
我们常说 Cache Hit 能“省下算力”，这里的技术细节非常关键：

1.  **Cache Miss (全量计算)**：GPU 需要对输入的所有 Token 进行 **Prefill（前处理）**。这是一个**计算密集型 (Compute Bound)** 的过程，涉及大量的矩阵乘法运算。
2.  **Cache Hit (增量计算)**：如果前缀匹配，我们跳过了昂贵的 Prefill 阶段。
3.  **Attention 开销**：虽然跳过了 Prefill，但新生成的 Token 依然需要通过 Attention 机制“关注”所有的历史 Token。这并非零开销，但它主要变成了 **内存带宽密集型 (Memory Bound)** 的操作。

**结论：** Cache Hit 主要大幅降低了 **首字延迟 (Time to First Token, TTFT)**，让 Agent 的响应感觉更加敏捷。
:::

基于 Codex 的实践，我总结了以下五条 Context Engineering 的 **“黄金法则”**。

## 第一部分：追求极致 Cache Hit 的五大法则

### 法则一：静态在前，动态在后 (The Static-First Rule)

**核心原则：** Context 的结构设计必须像“千层饼”一样分层。**越是静态、不可变的内容，越要放在最前面。**

#### ❌ 常见误区：把时间戳放在第一行
很多开发者觉得在 System Prompt 第一行告知模型当前时间是很严谨的：
```text
[System Message]
Current Time: 2026-01-25 10:00:01  <-- 这一秒在变，导致此行之后的所有缓存每秒都在失效！
Role: You are a helpful assistant...
```

:::warning[后果]
Prompt Caching 的核心是**前缀匹配（Prefix Matching）**。缓存是按顺序存储的，一旦开头的 Token 变了，其后所有内容的 KV Cache（Key-Value Cache）都会失效。

这就像你去图书馆借书，虽然人没变，但每次你都换了一张不同号码的身份证。图书管理员系统无法识别你是“老用户”，必须把你当作全新用户重新录入资料，导致首字延迟（TTFT）大幅增加。
:::

#### ✅ 优化方案：动态信息沉底
将时间戳、Request ID 等高度动态的信息，移到 Context 的**最底部**，或者放在 User Message 中，让 System Prompt 的头部保持绝对静止。

### 法则二：追加优于修改 —— “合同”与“补充协议” (Append Over Modify)

这是最具启发性的一点。核心在于权衡：**用少量的 Context 长度增长，换取昂贵的 Prefill 计算节省。**

#### 场景分析
当 Agent 运行中途，环境发生变化（例如用户将权限从“只读”改为“可写”）。

#### ❌ 错误做法：修改原始定义
* *修改前：* `System: [Sandbox: Read-Only]` ... (后接 5000 token 历史)
* *修改后：* `System: [Sandbox: Write-Allowed]` ... (后接 5000 token 历史)

**技术后果：** 修改开头意味着破坏前缀。GPU 必须**重新计算**这 5000 个 Token 的 KV 向量（Prefill 阶段），这涉及大量的矩阵乘法运算，带来显著延迟。

#### ✅ 正确做法：追加补充协议
* *保持开头：* `System: [Sandbox: Read-Only]` (保持不变，骗过缓存机制)
* *...中间的对话历史...*
* *追加结尾：* `System: [Update: User has approved Write Access.]`

:::tip[原理解析]
这避免了对历史 Token 进行昂贵的 **Prefill（前处理）** 计算。 

虽然新生成的 Token 依然需要去“关注（Attention）”所有的历史信息（这依然消耗显存带宽），但我们跳过了最耗时的**历史 Token 的特征提取与投影计算**。
- **收益：** 大幅降低 Time to First Token (TTFT)。
- **代价：** Context 长度缓慢增加，直到触发窗口限制（见法则五）。
:::

:::tip[比喻]
就像签署商业合同，不要因为条款变更就撕毁合同重签（Cache Miss，很贵），而是在合同最后钉上一页 **“补充协议”**（Cache Hit，很便宜）。模型完全有能力理解“后文覆盖前文”的逻辑。
:::

### 法则三：确定性是工程的底线 (Determinism is Key)

**核心原则：** 缓存匹配通常是 **字节级（Byte-level）** 的精确匹配。对于人类来说语义相同的列表，如果顺序不同，对于缓存机制来说就是完全不同的序列。

#### ❌ 错误案例：随机的工具列表
* 请求 A: `Tools: [Shell, Search]`
* 请求 B: `Tools: [Search, Shell]`

这会导致莫名其妙的 Cache Miss。

#### ✅ 优化方案：强制确定性排序
1.  **工具列表排序：** 始终按字母顺序（A-Z）排列工具定义。
2.  **JSON 序列化：** 确保 JSON 对象在转为字符串时，Key 的顺序是固定的（许多语言的 Map 是无序的，需要专门处理）。

### 法则四：将不确定性隔离与“打补丁” (Isolation & Patching)

**核心原则：** 将不稳定的外部定义隔离，对环境状态的变更采用“打补丁”策略。

#### 场景 A：外部工具 (Isolation)
Codex 将工具分为“内置工具”（如 Shell，定义稳定）和“外部工具”（如 MCP 协议插件）。如果引入了外部工具，其 Schema 可能随时变化。  
**策略：** 将这些不稳定的内容与核心 System Prompt 隔离。一旦外部配置变更，尽量通过追加说明而非重构上下文来处理。

#### 场景 B：环境变更 (Patching)
用户执行了 `cd /var/www`，导致当前工作目录（CWD）改变。
* **❌ 错误：** 回去修改 System Prompt 中的 `<cwd>` 标签。
* **✅ 正确：** 在对话流中插入一条事件消息：`User executed cd. CWD is now /var/www`.

:::note[工程权衡 Trade-off]
这种“打补丁”的方式依赖于模型能够理解“后文覆盖前文”的逻辑。现代强模型通常能处理得很好，但对于极长上下文，模型可能会出现“遗忘”或“幻觉”。这是一种为了性能而在准确性边缘试探的策略。
:::

### 法则五：正视窗口限制与压缩 (The Compaction Reality)

**核心原则：** “追加优于修改”会导致 Context 像滚雪球一样变大，必须在达到阈值（Context Window）前进行智能压缩。

当 Token 数量达到阈值（如 `auto_compact_limit`）时，必须进行**压缩（Compaction）**。

1.  **非简单截断：** 不能只保留最后几句，否则会丢失任务目标。
2.  **隐式摘要与重置：** 高级的做法是调用压缩端点，将长历史压缩为摘要。
    * *进阶技巧：* 这个摘要不仅包含文字，甚至可以包含 **隐状态（Latent State）** 或 `encrypted_content`，以保留模型对任务的“潜意识”理解。
3.  **接受代价：** 压缩发生的瞬间，Cache 会被迫刷新（Miss），但这是为了防止 Context 溢出并维持长期运行所必须支付的代价。

## 第二部分：实战思考 —— 从“日志折叠”到“价值过滤”

理解了上述法则后，我们来解决一个 Agent 开发中极度棘手的真实场景。

### 场景描述：复杂的 Read-Edit Loop
在处理大型代码库时，Agent 的行为往往伴随着大量的试错和分块读取：
1.  **探索：** Agent 试图修复一个 Bug，先读取了 `main.py` 的 1-100 行。
2.  **深入：** 没找到，又读取了 101-300 行。
3.  **发现：** 最终发现只有 **251-300 行** 是相关的核心逻辑，而前面的 1-250 行全是无关的配置代码。

此时，Context 中堆积了数千 Token 的代码片段。

### 演进思考：如何优雅地“压缩”？

根据 **法则五（正视窗口限制）**，我们直觉的解决方案是：**定期压缩 Context，将繁杂的 Tool Log 替换为摘要。**

#### 第一阶段：朴素的摘要机制 (The Naive Approach)
我们可以设计一个机制，将上述操作总结为：
> "Read `main.py` from line 1 to 300."

并提供一个 `expand` 工具，允许模型在需要时查看详情。

**但在实战中，我发现这个方案有一个致命缺陷：**
当 Agent 后续需要“展开”这段记忆时，系统会机械地把 1-300 行代码**全部**塞回 Context。这意味着，我们刚刚费力扔出去的 **1-250 行噪音（Noise）**，又被模型自己捡回来了。这不仅浪费 Token，更会用无关代码干扰模型的注意力。

#### 第二阶段：基于“价值过滤”的高精度折叠 (Value Filtering)

为了解决这个问题，我意识到：**压缩不仅仅是变短，更是一次信息清洗。**
我们需要在生成 Summary 的阶段，就将“沙子”和“金子”分离开。

**优化后的 Context 结构：**

当子任务完成时，我们要求 LLM 总结操作，并明确区分 **“已读但无用（Discarded）”** 和 **“已读且有用（Retained）”** 的部分。

```markdown
[System Event: Context Compressed]
- Processed `main.py` (Lines 1-300):
  - Lines 1-250: Discarded (Irrelevant boilerplate).
  - Lines 251-300: **Retained**. Contains `init_db_connection` logic. (ID: ref_m1_valid)
**NOTE**
Context is compressed. Use `expand_context(ids=['ref_m1_valid'])` to view ONLY the relevant code snippets.
:::
```

**设计亮点：**

1.  **高召回率 (High Recall)：** 明确记录 1-250 行“被丢弃了”。这告诉模型“那部分我看过了，没用”，防止模型因不知道读过而重新去读。
2.  **高精确度 (High Precision)：** `ID: ref_m1_valid` 并不指向 1-300 行的原始记录，而是指向 **经过裁剪的 251-300 行切片**。

### 配套工具设计

为了支撑这个机制，我们需要配套的工具定义。

#### 1. 批量懒加载工具 (Batch Expansion Tool)
为了减少交互轮次，工具应当支持批量操作。

```typescript
{
  name: "expand_context",
  description: "Expand the RELEVANT details of compressed logs.",
  parameters: {
    ids: {
      type: "array",
      items: { type: "string" },
      description: "List of IDs to expand. Note: This will only expand the useful code snippets retained during compression."
    }
  }
}
```

#### 2. 展开策略：只返回“黄金切片” (应用法则二与四)

当模型调用 `expand_context(ids=["ref_m1_valid"])` 时，系统只会返回之前被标记为“有用”的那 50 行代码。

且为了遵守 **法则二（追加优于修改）**，我们不修改历史摘要，而是将内容追加到对话末尾：

```markdown
... (前面是压缩后的摘要，作为静态地基，保持不动) ...

[Current Turn]
User: I need to fix the db connection bug found in main.py.
Agent: Call tool `expand_context(ids=["ref_m1_valid"])`
System: [Tool Output] Expanding relevant snippets:
1. `main.py` (ref_m1_valid, lines 251-300): 
   "def init_db_connection():
      # ... Only the 50 relevant lines are shown here ...
   "
```

通过这种方式，我们既保留了完整的**探索路径**（我知道我读过哪），又实现了**上下文的极致精简**（我只展开有用的）。

## 结语

Context Engineering 不仅仅是关于 Prompt 怎么写，更是一种 **数据结构的设计哲学**。

通过 Codex 的文章和这个实战案例，我们可以看到：优秀的 Agent 架构，应该像数据库索引一样：
1.  利用 **静态前缀** 最大化缓存命中。
2.  利用 **追加更新** 最小化重算开销。
3.  利用 **智能压缩** 在有限的窗口内保留高价值信息（如读取进度）。
4.  利用 **懒加载** 机制按需获取细节。

只有理解了这些底层的算力逻辑，你的 Agent 才能在生产环境中跑得既快又稳。

> **Reference:**
>
> * [Unrolling the Codex agent loop | OpenAI](https://openai.com/index/unrolling-the-codex-agent-loop/)