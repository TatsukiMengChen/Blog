---
title: 解构 Claude Code：构建代码原生 AI 代理的技术蓝图
published: 2025-10-08
description: 深度剖析 Claude Code 的设计哲学与核心架构，从零开始构建一个生产级的代码原生 AI 代理。本文涵盖规划器-执行器模型、针对代码优化的 RAG、LangGraph 状态机以及 Docker 安全沙箱等关键技术。
image: https://d988089.webp.li/2025/10/08/20251008154247786.avif
tags: ["Agentic AI", "Agent", "LLM", "Claude Code", "RAG", "LangGraph"]
category: Agent
draft: false
---

> 深度剖析 Claude Code 的设计哲学与核心架构，从零开始构建一个生产级的代码原生 AI 代理。本文涵盖规划器-执行器模型、针对代码优化的 RAG、LangGraph 状态机以及 Docker 安全沙箱等关键技术。

:::important[免责声明]
本文由 AI 协作于国庆假期耗时约 8 天，整理了大量资料并总结内容进行编写，本人仅做了初步的内容验证，请您自行分辨其中内容的真伪。
:::

## 第一部分：代码原生代理的剖析：哲学与架构

本部分将阐述我们代理的基本原则，这些原则直接借鉴自 Claude Code 的设计哲学。我们将介绍“迷你版 Claude”的高层架构，并说明为何选择规划器-执行器模型作为处理复杂编码任务的最适范式。

### 1.1 Claude Code 的哲学：一个低层级、无偏见的强大工具

Anthropic 的 Claude Code 背后的设计哲学，代表了与高度结构化、有主见的 AI 助手的一次重大分野。其核心原则是“有意保持低层级和无偏见，提供近乎原始的模型访问，而不强加特定的工作流程”[1](https://www.anthropic.com/engineering/claude-code-best-practices)。这种方法将 Claude Code 定位为一个灵活、可定制、可编写脚本的强大工具，而非一个规定特定开发流程的僵化助手。其目标是赋能开发者，增强他们现有的工作流程，而不是取而代之。这种哲学体现在一种常被描述为“令人愉悦”的用户体验中，它在自主性与控制之间提供了精心校准的平衡。用户反馈称，该代理拥有足够的自主性来执行复杂有趣的任务，但又不会引发那种在更具侵略性的全自主系统中可能出现的“令人不安的失控感”[2](https://minusx.ai/blog/decoding-claude-code/), [3](https://prismic.io/blog/claude-code), [4](https://rafaelquintanilha.com/is-claude-code-worth-the-hype-or-just-expensive-vibe-coding/)。

这种设计的一个关键方面是它与开发者主要工作空间——终端的原生集成[3](https://prismic.io/blog/claude-code)。通过在命令行中操作，Claude Code 最大限度地减少了上下文切换，成为开发者环境的无缝扩展，就像代码编辑器或版本控制系统一样。这种深度集成使其能够与整个项目结构互动，自动化繁琐的任务，如解决 linting 问题或合并冲突，甚至能从外部来源（如网络文档或 Figma 中的设计文件）提取信息[5](https://docs.claude.com/en/docs/claude-code/overview#:~:text=Claude%20Code%20maintains%20awareness%20of%20your%20entire%20project%20structure%2C%20can%20find)。

这种在强大自主性与精确用户引导性之间的平衡，并非偶然，而是一项深思熟虑的工程决策。大型语言模型（LLM）固有的不确定性和“黑箱”特性，使得调试复杂的代理系统异常困难。正如逆向工程分析所指出的，尽管 AI 开发的趋势是复杂的“多代理系统”，但这种复杂性会使调试“难度增加10倍”[2](https://minusx.ai/blog/decoding-claude-code/)。一个未能按预期执行任务的 LLM，几乎不会提供任何堆栈跟踪或可复现的错误。当多个代理互动时，这个问题会变得更加复杂，使得诊断失败的根本原因几乎不可能。因此，Claude Code 的架构优先考虑了简单性和可观察性。它倾向于采用单一、有状态的控制循环，而非复杂的交互代理网络[2](https://minusx.ai/blog/decoding-claude-code/)。这一“简单性原则”是源于构建 LLM 实际挑战的核心工程信条。它承认，要构建一个可靠的工具，必须保持理解和调试其行为的能力。因此，我们的“迷你版 Claude”将采纳这一哲学，避免不必要的复杂性，转而采用一个强大、可观察且有状态的架构，并始终置于开发者的掌控之下。

### 1.2 高层架构：“迷你版 Claude”的三大支柱

为了构建一个体现灵活性、强大功能和可控性原则的代理，我们的“迷你版 Claude”将基于一个由三个核心、相互关联的系统组成的模块化架构。这种设计超越了依赖单一、庞大的 LLM 调用来解决问题的单体方法。相反，它认识到一个复杂的编码代理需要专门的组件，每个组件都为软件开发任务的特定子问题进行优化。

我们代理架构的三大支柱是：

1. **具备代码库感知的 RAG（检索增强生成）：** 这是代理的“长期记忆”和上下文理解系统。它是一个专门的 RAG 管道，旨在消化整个软件项目，并为代理提供关于代码库的深度、上下文知识。为了有效，该系统必须超越简单的文本检索，理解源代码的句法和语义结构，使代理能够推理函数依赖、类结构和架构模式[6](https://medium.com/google-cloud/build-a-rag-system-for-your-codebase-in-5-easy-steps-a3506c10599b)。
2. **规划器-执行器控制循环：** 这是代理的“大脑”或中枢神经系统。它是一个健壮的、有状态的控制循环，管理任务的整个生命周期。它负责将高层级的用户请求分解为一系列较小的、可执行的步骤（规划），然后使用一组定义的工具来执行这些步骤（执行）。我们将使用 LangGraph 框架将其实现为一个状态机，这非常适合构建我们设计哲学所要求的可观察、单进程的控制流[7](https://www.datacamp.com/tutorial/langgraph-agents), [8](https://www.langchain.com/langgraph)。
3. **Docker化执行沙箱：** 这是代理的“双手”及其关键的安全机制。它为执行 AI 生成的代码和 shell 命令提供了一个安全、隔离的环境。通过容器化执行，我们防止代理对主机系统构成任何风险，使其能够安全地编译代码、运行测试或使用命令行工具，而不会产生意外副作用的危险[9](https://anukriti-ranjan.medium.com/building-a-sandboxed-environment-for-ai-generated-code-execution-e1351301268a)。

这种模块化设计确保每个组件都可以独立开发、测试和优化，从而形成一个更健壮、更有能力的最终系统。它直接反映了这样一种理解：构建一个成功的代理是一项系统工程，而不仅仅是提示工程。

### 1.3 为何选择规划器-执行器？超越简单的 ReAct 循环

控制循环架构的选择对代理的能力至关重要。一个常见且简单的架构是 ReAct（推理-行动）模型，它在一个“生成思考、选择工具、执行、观察结果”的紧密循环中运行[10](https://blog.langchain.com/planning-agents/)。虽然对于简单的单步任务有效，但当面对软件开发中固有的复杂、多步、长周期的任务时，ReAct 模型显示出显著的局限性[10](https://blog.langchain.com/planning-agents/), [11](https://arxiv.org/html/2503.09572v2)。其主要缺点是缺乏远见——它一次只规划一个步骤，这可能导致效率低下或路径次优——以及其高昂的运营成本，因为它每次行动都需要调用一个强大（且通常昂贵）的 LLM。

为了克服这些局限，我们的“迷你版 Claude”将基于更先进的**规划器-执行器**架构[11](https://arxiv.org/html/2503.09572v2), [12](https://www.emergentmind.com/topics/planner-executor-architecture-4c9e0097-fe2b-4870-b41c-9519c49a07c8), [13](https://www.promptlayer.com/glossary/plan-and-execute-agents)。该范式明确地将战略规划的高层任务与执行的低层任务分离开来。其过程如下：

1. **规划：** 一个由强大 LLM 驱动的中央**规划器**模块接收用户的高层目标。它分析整个任务，并将其分解为一个全面的、多步骤的计划。这个计划不仅仅是下一个动作，而是一个为实现最终目标而设计的完整步骤序列。
2. **执行：** 一个或多个**执行器**模块接收此计划。然后，它们使用预定义的工具集按顺序执行计划中的每一步。关键在于，执行器通常可以是更简单的代理，甚至是直接的函数调用，它们在每次行动后无需咨询主规划器 LLM。
3. **重新规划：** 在执行一系列步骤后，结果会反馈给规划器。规划器随后可以评估进展，验证目标是否已达成，或者在初始策略不足或遇到意外障碍时生成一个新的、修订过的计划[11](https://arxiv.org/html/2503.09572v2)。

这种架构的优势是巨大的，并直接解决了 ReAct 模型的弱点。它提高了效率和执行速度，因为对主要推理 LLM 的调用次数减少了[10](https://blog.langchain.com/planning-agents/), [11](https://arxiv.org/html/2503.09572v2), [14](https://blog.langchain.com/planning-agents/)。出于同样的原因，它也更具成本效益。最重要的是，它带来了更好的性能和更高的任务完成率，因为初始规划阶段迫使代理对整个问题进行整体性推理，从而产生更连贯、更有效的策略[11](https://arxiv.org/html/2503.09572v2)。这一架构选择直接支持了对代理编码最有效的复杂、多阶段工作流，例如为 Claude Code 用户推荐的“探索、规划、编码、提交”模式[1](https://www.anthropic.com/engineering/claude-code-best-practices)。

## 第二部分：代理的心智：控制循环与高级提示工程

本部分详细介绍了代理“大脑”的实现。我们将首先逆向工程驱动 Claude Code 行为的复杂提示技术，然后使用 LangGraph 构建有状态的控制循环来管理代理的生命周期。

### 2.1 解构 Claude Code 系统提示

一个由 LLM 驱动的代理的效能，深受其系统提示的质量和结构的影响。对 Claude Code 的逆向工程分析显示，其提示并非简单的单行指令，而是庞大、精细且高度工程化的产物[2](https://minusx.ai/blog/decoding-claude-code/)。仅系统提示就估计约有 2800 个令牌，而其可用工具的描述则占据了惊人的 9400 个令牌。这种详尽程度表明，提示同时充当了配置文件、行为指南和操作启发式规则集。基础 LLM 的无偏见特性，需要一个高度有主见和结构化的提示来有效引导其行为。复杂性被有意地从代理的硬编码逻辑转移到了其更灵活的自然语言配置中。

要构建一个有效的“迷你版 Claude”，我们必须将其系统提示视为一个主要的工程产物，应进行版本控制、测试和优化。基于对 Claude Code 提示的解构，我们的系统提示应包含以下关键组件：

- **角色与人格定义：** 此部分设定代理的整体行为。它应定义其语调（例如，“你是一位专业且严谨的 AI 软件工程师”）、风格（例如，“你的代码应整洁、注释良好，并遵循行业最佳实践”）以及主动性水平（例如，“如果用户的请求含糊不清，请在继续之前提出澄清问题”）[2](https://minusx.ai/blog/decoding-claude-code/)。

- **任务管理与显式算法：** 这是最关键的部分之一。提示不应让 LLM 自由决定如何处理问题，而应明确概述其必须遵循的算法。对于一个规划器-执行器代理，这可以是：“1. 彻底分析用户的请求和提供的上下文。2. 制定一个详细的、分步的计划以实现目标。3. 按顺序执行计划的每一步。4. 执行后，审查结果并确定目标是否已达成，或是否需要重新规划。” 这构建了代理的推理过程，使其行为更可预测、更可靠[2](https://minusx.ai/blog/decoding-claude-code/)。

- **启发式规则与带 XML 标签的示例：** LLM 从示例中学习的效果非常好。提示应利用这一点，提供良好和不良行为的具体示例，并用特殊的 XML 标签括起来。例如，为了指导工具的使用：

  ```xml
  <good-example>
  用户请求：“查找项目中的所有测试文件。”
  工具调用：Glob(pattern="**/*_test.py")
  </good-example>
  <bad-example>
  用户请求：“查找项目中的所有测试文件。”
  工具调用：Bash(command="find. -name '*_test.py'")
  </bad-example>
  ```

  这种在 Claude Code 中观察到的技术，有助于将最佳实践编码化，并引导模型避开效率较低或更易出错的方法[2](https://minusx.ai/blog/decoding-claude-code/)。

- **引导性与提醒：** 为了强制执行关键约束并防止常见的失败模式，提示应使用强烈的强调标记和提醒标签。诸如 `IMPORTANT`、`VERY IMPORTANT`、`NEVER` 和 `ALWAYS` 等短语可用于吸引模型对关键规则的注意（例如，`VERY IMPORTANT: 你必须在进行任何代码更改后验证测试是否通过`）[2](https://minusx.ai/blog/decoding-claude-code/)。此外，`<system-reminder>` 标签可用于重申模型在长对话中可能忘记的规则，例如“你有一个 `write_file` 工具；除非是最终答案的一部分，否则不要直接向用户输出代码”[2](https://minusx.ai/blog/decoding-claude-code/)。

- **动态上下文信息：** 提示应动态填充有关代理环境的实时信息。这包括当前日期和时间、操作系统和平台、当前工作目录，甚至是 `git log -n 5` 的输出来提供近期更改的上下文。这将代理置于其当前的操作现实中[2](https://minusx.ai/blog/decoding-claude-code/)。

### 2.2 实现 `claude.md` 模式以实现持久化上下文

虽然系统提示提供了一套全局指令，但软件开发任务是高度依赖上下文的。适用于一个代码仓库的规则（例如，“使用制表符进行缩进”）可能在另一个代码仓库中是错误的。为了解决这个问题，Claude Code 采用了一种优雅的机制来进行项目特定的配置：`claude.md` 文件[1](https://www.anthropic.com/engineering/claude-code-best-practices), [2](https://minusx.ai/blog/decoding-claude-code/)。这是一个特殊的 markdown 文件，如果存在于项目目录中，其内容会在会话开始时自动被读取并附加到用户提示的前面。这实际上允许开发者根据每个项目来“编程”代理的上下文和偏好。

在我们的“迷你版 Claude”中实现这种模式是一个直接但功能强大的增强。应用程序的入口点应包含一个函数，该函数在当前工作目录及其所有父目录中搜索指定的上下文文件（例如，`.mini_claude_rules.md`），直到代码仓库的根目录。这允许分层配置，其中根级文件可以为整个 monorepo 定义规则，而子目录的文件可以为特定服务添加或覆盖规则。

该文件的内容充当代理的“备忘单”，为其提供关于项目的基本、非显而易见的信息。这是用户级“提示编程”的一个关键机制，使代理能够适应新的代码库，而无需对其核心代码进行任何更改。应鼓励开发者将这些文件检入版本控制，以便整个团队都能从提供给代理的共享上下文中受益。在 `.mini_claude_rules.md` 文件中包含的有价值信息的示例如下：

- **构建和测试命令：** “要运行此项目的测试套件，请使用命令 `npm run test:unit`。要运行 linter，请使用 `npm run lint`。”
- **架构指引：** “用户认证的核心业务逻辑位于 `src/services/auth.py`。主要的数据库模型定义在 `src/models/` 中。”
- **编码风格和约定：** “此项目使用 Black 代码格式化工具。始终使用双引号表示字符串。所有新组件必须是使用 React Hooks 的函数式组件。”
- **代码仓库礼仪：** “所有分支应使用 `feature/TICKET-123-description` 的格式命名。提交信息必须遵循 Conventional Commits 规范。”
- **已知问题或变通方法：** “运行本地开发服务器时，你可能会看到关于 `some-dependency` 的良性警告；这可以安全地忽略。” [1](https://www.anthropic.com/engineering/claude-code-best-practices)

### 2.3 使用 LangGraph 构建状态机

通过高级提示定义了代理的指令框架后，我们现在转向其操作核心：控制循环。如前所述，我们将使用规划器-执行器架构来管理复杂任务。实现这一目标的理想现代框架是 LangGraph，这是一个构建在 LangChain 之上的库，允许开发者将代理工作流定义为有状态的图[7](https://www.datacamp.com/tutorial/langgraph-agents), [8](https://www.langchain.com/langgraph)。这种方法与我们的“简单性原则”完美匹配，因为它允许我们将整个代理逻辑构建为单一、可观察的状态机，而不是一个复杂的交互代理网络。

使用 LangGraph 构建的第一步是定义中央 `State` 对象。这是一个 Python `TypedDict`，代表了我们代理在任何时间点的全部状态。它将被传递给图中的每个节点，并且每个节点都可以修改它。对于我们的规划器-执行器代理，状态将跟踪管理任务从开始到结束所需的所有关键信息。根据该架构的标准模式，我们的状态将定义如下 [15](https://langchain-ai.github.io/langgraph/tutorials/plan-and-execute/plan-and-execute/)：

```python
import operator
from typing import Annotated, List, Tuple
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage

class AgentState(TypedDict):
    # 初始用户请求
    input: str
    
    # 规划器生成的多步计划
    plan: List[str]
    
    # 已执行步骤及其结果的历史记录，用于重新规划
    past_steps: Annotated[List[Tuple[str, str]], operator.add]
    
    # 返回给用户的最终响应
    response: str
    
    # 我们还可以为执行器子代理添加消息历史记录
    messages: Annotated[List[BaseMessage], operator.add]
```

带有 `operator.add` 的 `Annotated` 类型是 LangGraph 的一个特性，它确保当一个节点为 `past_steps` 或 `messages` 返回一个值时，该值会被附加到现有列表中，而不是覆盖它，从而允许状态随时间累积[8](https://www.langchain.com/langgraph)。

定义了状态之后，我们现在可以概述图的节点和边。流程将精确地反映规划器-执行器模式，并将在第四部分中详细实现：

1. **`planner` 节点：** 图的入口点。它从状态中接收 `input` 并生成初始 `plan`。
2. **`executor` 节点：** 此节点获取当前的 `plan`，使用其工具执行下一步，并将结果附加到 `past_steps`。
3. **`re-planner` 节点：** 执行后，此节点检查 `past_steps` 并决定任务是否完成，或者 `plan` 是否需要更新。
4. **条件边：** 一个路由函数，检查 `re-planner` 的输出。如果已生成最终的 `response`，它将图转换到特殊的 `END` 状态。否则，它会循环回到 `executor` 节点以继续执行计划。

这种图结构为我们代理的核心逻辑提供了一个健壮、灵活且可调试的基础。

## 第三部分：实现代码库感知：构建特定于代码的 RAG 管道

本部分深入探讨如何创建代理理解软件项目的能力。我们将从头开始构建一个检索增强生成（RAG）管道，重点关注专门为源代码优化的技术，以便为代理提供准确、相关的上下文。

### 3.1 针对代码的天真 RAG 的问题

检索增强生成是一种强大的技术，用于将 LLM 植根于外部知识，有效地为它们提供一本“开卷书”以在回答问题前进行查阅[16](https://github.com/resources/articles/ai/software-development-with-retrieval-augmentation-generation-rag)。对于编码代理来说，这本“书”就是目标代码库。然而，将为自然语言文本设计的标准 RAG 技术应用于源代码是注定要失败的。

最常见的分块策略，如固定大小分块（每 N 个字符分割文本）或递归字符分块（按段落、句子、单词分割），从根本上不适合代码的高度结构化特性[17](https://medium.com/@jouryjc0409/ast-enables-code-rag-models-to-overcome-traditional-chunking-limitations-b0bc1e61bdab), [18](https://www.ibm.com/think/tutorials/chunking-strategies-for-rag-with-langchain-watsonx-ai)。这些方法对编程语言的句法结构一无所知。固定大小的分块器可能会将一个函数定义一分为二，将函数签名与其主体分开。它可能会破坏一个类定义、一个循环，甚至一个简单的条件语句。当这些支离破碎、句法上无意义的片段被检索并作为上下文提供给 LLM 时，它们提供了一个被破坏和令人困惑的代码库视图。这会导致幻觉、不正确的代码生成，以及代理普遍无法有效推理项目架构[17](https://medium.com/@jouryjc0409/ast-enables-code-rag-models-to-overcome-traditional-chunking-limitations-b0bc1e61bdab), [19](https://arxiv.org/html/2506.15655v1)。要构建一个真正具备代码库感知的代理，我们必须超越这些天真的方法，采用一种尊重源代码结构完整性的方法。

### 3.2 使用抽象语法树（AST）进行智能代码分块

解决代码分块问题的方案在于使用一种本身就能理解代码结构的表示方法：**抽象语法树（AST）**。AST 是由解析器生成的源代码的层次化、树状表示。树中的每个节点代表代码中的一个构造，如函数定义、类声明、导入语句或循环[17](https://medium.com/@jouryjc0409/ast-enables-code-rag-models-to-overcome-traditional-chunking-limitations-b0bc1e61bdab), [20](https://www.unite.ai/code-embedding-a-comprehensive-guide/)。通过操作 AST 而不是原始文本，我们可以设计出一种保留代码单元语义和句法完整性的分块策略。

我们的实现将基于 `cAST`（通过抽象语法树进行分块）方法论，该方法论已被证明在代码 RAG 管道中非常有效[17](https://medium.com/@jouryjc0409/ast-enables-code-rag-models-to-overcome-traditional-chunking-limitations-b0bc1e61bdab), [19](https://arxiv.org/html/2506.15655v1)。

#### AST 解析工具

为了将各种语言的代码解析成 AST，我们将使用 `tree-sitter`。`tree-sitter` 是一个增量解析库，它速度快、健壮，并通过一个可插拔的语法系统支持大量编程语言[21](https://medium.com/@shreshthg30/a-beginners-guide-to-tree-sitter-6698f2696b48), [22](https://tree-sitter.github.io/tree-sitter/using-parsers/)。我们将使用其官方 Python 绑定将其集成到我们的 RAG 管道中。

#### “先分割后合并”的 AST 分块算法

我们分块逻辑的核心将是一个递归的“先分割后合并”算法，该算法遍历 AST。目标是创建尽可能大且信息密集的块，同时不超过预定义的大小限制，并尊重 AST 定义的句法边界[17](https://medium.com/@jouryjc0409/ast-enables-code-rag-models-to-overcome-traditional-chunking-limitations-b0bc1e61bdab), [19](https://arxiv.org/html/2506.15655v1)。

该算法的步骤如下：

1. **解析：** 首先使用 `tree-sitter` 将源代码文件解析成一个完整的 AST。
2. **自顶向下遍历（分割）：** 算法从 AST 的根节点开始，自顶向下遍历。对于每个节点（例如，一个类定义），它检查整个节点的文本内容是否在我们的最大块大小之内。
3. **递归分解：** 如果一个节点太大而无法放入单个块中，算法不会简单地将其分割。相反，它会递归地下降到该节点的子节点（例如，类中的各个方法），并对它们应用相同的逻辑。这确保我们总是尝试将尽可能大的完整句法单元保持在一起。
4. **贪婪兄弟节点合并（合并）：** 在递归分割过程之后，我们可能会剩下许多小的、相邻的节点（例如，多个导入语句或一系列简单的函数）。为了最大化信息密度，算法随后执行一个贪婪的合并步骤，将相邻的兄弟节点组合成一个块，只要它们的组合大小不超过限制。

#### 块大小度量

此过程中的一个关键细节是我们如何衡量块的大小。按行数衡量是不可靠的，因为两个行数相同的代码段由于格式和空白的不同，其实际内容量可能大相径庭。一个更健壮和一致的度量标准是**非空白字符数**。这确保我们的块大小预算反映了代码内容的实际密度，使其在不同的文件、语言和编码风格之间具有可比性[17](https://medium.com/@jouryjc0409/ast-enables-code-rag-models-to-overcome-traditional-chunking-limitations-b0bc1e61bdab), [19](https://arxiv.org/html/2506.15655v1)。

#### 使用 `astchunk` 的实际实现

虽然可以使用 `tree-sitter` 库从头开始实现此算法，但有几个开源库提供了预构建的解决方案。`astchunk` 库是实现 `cAST` 方法论的一个典型例子[23](https://github.com/yilinjz/astchunk)。以下 Python 代码演示了如何使用它来智能地分块源代码文件：

```python
from astchunk import ASTChunkBuilder

# 示例 Python 源代码
source_code = """
import os
import sys

class DataProcessor:
    def __init__(self, data):
        self.data = data

    def process(self):
        # 一个非常复杂的处理步骤
        processed_data = [item * 2 for item in self.data]
        return processed_data

def main():
    processor = DataProcessor()
    result = processor.process()
    print(f"Result: {result}")

if __name__ == "__main__":
    main()
"""

# 初始化基于 AST 的分块构建器
# 我们设置一个小的块大小来演示分割逻辑
configs = {
    "max_chunk_size": 150,  # 最大非空白字符数
    "language": "python",
    "metadata_template": "default"
}
chunk_builder = ASTChunkBuilder(**configs)

# 生成语法感知的块
chunks = chunk_builder.chunkify(source_code)

# 显示结果块
for i, chunk in enumerate(chunks):
    print(f"--- Chunk {i+1} ---")
    print(chunk['content'])
    print(f"Metadata: {chunk['metadata']}")
    print("-" * 20)
```

运行此代码可能会产生三个不同的块：一个用于导入语句，一个用于 `DataProcessor` 类，一个用于 `main` 函数及其执行块，这演示了句法结构是如何被保留的。

### 3.3 代码优化的嵌入和向量检索

一旦代码库被智能地分块，RAG 管道的下一步就是将这些块转换为数值表示（嵌入），以便进行索引以实现高效的相似性搜索。正如天真的分块对代码无效一样，通用的文本嵌入模型也是次优的。主要在自然语言文本上训练的模型不能充分捕捉源代码中存在的独特语义关系[20](https://www.unite.ai/code-embedding-a-comprehensive-guide/), [24](https://modal.com/blog/6-best-code-embedding-models-compared)。例如，在通用文本模型中，令牌 `request` 可能在语义上接近 `plea` 或 `inquiry`。在代码感知模型中，它应该更接近 `response`、`API` 或 `HTTP`。

因此，使用一个专门在大量源代码语料库上训练的嵌入模型至关重要。这些模型学习的向量表示能够理解算法相似性、库使用模式和语言语法等概念[24](https://modal.com/blog/6-best-code-embedding-models-compared)。

#### 选择代码嵌入模型

代码嵌入模型领域正在迅速发展。在为我们的“迷你版 Claude”选择模型时，应考虑几个因素：在代码检索基准上的性能、上下文长度、嵌入维度和可访问性（即，专有 API 与开源、可自托管的模型）。下表根据最近的分析，比较了当今几种领先的模型[24](https://modal.com/blog/6-best-code-embedding-models-compared)。

| 模型名称                          | 上下文长度  | 输出维度    | 关键特性                                          | 访问方式              |
| --------------------------------- | ----------- | ----------- | ------------------------------------------------- | --------------------- |
| **VoyageCode-2**                  | 32,768 令牌 | 256 至 2048 | 专为代码设计；在超过300种语言的数万亿令牌上训练。 | Voyage AI API         |
| **OpenAI text-embedding-3-large** | 8,191 令牌  | 3072        | 顶级的通用模型，在代码任务上表现非常出色。        | OpenAI API            |
| **Jina Code Embeddings V2**       | 8,192 令牌  | 1.37亿参数  | 开源，推理速度快，为代码相似性和搜索任务优化。    | Hugging Face / 自托管 |
| **Nomic Embed Code**              | 2,048 令牌  | 70亿参数    | 完全开源的模型、数据和训练代码；出色的检索性能。  | Hugging Face / 自托管 |

对于寻求最大控制和隐私的开发者来说，像 Jina Code V2 或 Nomic Embed Code 这样的开源模型将是绝佳选择。对于那些优先考虑尖端性能和易用性的人来说，来自 Voyage AI 或 OpenAI 的基于 API 的模型是强有力的竞争者。

#### 索引和检索实现

最后一步是实现数据注入和检索工作流 [16](https://github.com/resources/articles/ai/software-development-with-retrieval-augmentation-generation-rag)：

1. **数据注入脚本：** 将创建一个 Python 脚本来协调索引过程。该脚本将：
   - 遍历用户提供的目标项目目录。
   - 对于每个支持的源文件（例如，`.py`、`.js`、`.java`），读取其内容。
   - 使用我们的 AST 分块器将文件分割成语义完整的块。
   - 对于每个块，调用所选的嵌入模型（通过 API 或本地实例）以生成其向量嵌入。
   - 将块的文本内容及其对应的向量嵌入存储在向量数据库中。对于本地开发，像 FAISS 或 Chroma 这样的库是绝佳选择。
2. **检索函数：** 将定义一个函数作为我们 RAG 工具的核心。该函数将：
   - 接受一个自然语言查询（例如，用户的问题或代理计划中的一个步骤）。
   - 使用相同的代码嵌入模型为该查询生成一个嵌入。
   - 查询向量数据库，以根据余弦相似度找到 top-k 个最相似的代码块。
   - 将这些检索到的块的文本内容连接成一个单一的字符串。
   - 这个上下文信息字符串随后被直接注入到发送给代理主 LLM 的提示中，为其提供执行任务所需的具体、相关知识。

## 第四部分：引擎室：实现规划器-执行器和工具集

本部分在第二部分建立的 LangGraph 框架内构建代理的核心操作逻辑。我们将实现规划器、执行器和重新规划器节点，并为代理定义一套基础工具，以便与环境交互。

### 4.1 实现图节点

我们代理的逻辑被定义为一个状态图，其中每个节点都是一个可以修改代理状态的函数或可运行对象。这些节点的实现将紧密遵循官方 LangGraph 规划与执行教程中展示的健壮模式[15](https://langchain-ai.github.io/langgraph/tutorials/plan-and-execute/plan-and-execute/), [25](https://www.youtube.com/watch?v=vpD9kf5Xwo0&vl=en)。

#### 规划器节点

规划器是代理的战略核心。其作用是接收用户的高层目标，并将其分解为一个具体的、分步的计划。为确保输出结构化且可靠，我们将使用 LLM 的函数调用或结构化输出功能，强制其返回符合预定义 Pydantic 模式的计划。

首先，我们定义 `Plan` 模式：

```python
from pydantic import BaseModel, Field
from typing import List

class Plan(BaseModel):
    """一个结构化的、分步的计划，以实现用户的目标。"""
    steps: List[str] = Field(
        description="为解决任务而按正确顺序列出的离散、可操作的步骤列表。"
    )
```

接下来，我们创建规划器本身，它将一个提示模板与一个配置为使用 `Plan` 模式进行输出的 LLM 结合起来：

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

planner_prompt = ChatPromptTemplate.from_template(
    """对于给定的目标，创建一个简单的、分步的计划。
这个计划应由独立的、可执行的任务组成，如果正确执行，将导致预期的结果。
不要包含任何不必要的步骤。最后一步的结果应为最终答案。

目标: {objective}
"""
)

# 使用一个强大的模型进行规划
planner_llm = ChatOpenAI(model="gpt-4o", temperature=0)
planner = planner_prompt | planner_llm.with_structured_output(Plan)
```

最后，我们图的 `planner_node` 函数只需用状态中的用户输入调用此链：

```python
async def planner_node(state: AgentState):
    plan_result = await planner.ainvoke({"objective": state["input"]})
    return {"plan": plan_result.steps}
```

#### 执行器节点

执行器的任务是获取状态中的当前计划，执行*下一个*步骤，并记录结果。一个非常有效的模式是将执行器本身实现为一个更简单的“微代理”。这创建了一个层次化的控制结构：主图充当项目经理，而执行器是派去处理一个特定任务的专业工人。这种模块化使得整个系统更容易理解和调试。管理者不需要知道例如“重构一个函数”所需的复杂工具调用序列；它只需要知道该步骤是成功还是失败[2](https://minusx.ai/blog/decoding-claude-code/), [11](https://arxiv.org/html/2503.09572v2), [15](https://langchain-ai.github.io/langgraph/tutorials/plan-and-execute/plan-and-execute/)。

我们将使用 LangGraph 预构建的 `create_react_agent` 来创建这个执行器微代理。它将配备下一小节中定义的工具。

```python
from langgraph.prebuilt import create_react_agent

# 工具将在 4.2 节中定义
# 如果需要，executor_llm 可以是一个更小、更快的模型
executor_agent = create_react_agent(executor_llm, tools)

async def executor_node(state: AgentState):
    # 获取要执行的下一步
    step = state["plan"]
    
    # 使用 ReAct 代理执行该步骤
    result = await executor_agent.ainvoke({"messages": [("user", step)]})
    
    # ReAct 代理的结果通常在最后一条消息中
    result_text = result['messages'][-1].content
    
    # 用已完成的步骤及其结果更新状态
    return {
        "past_steps": [(step, result_text)],
        "plan": state["plan"][1:] # 移除已完成的步骤
    }
```

#### 重新规划器节点

每执行一个步骤后，都会调用重新规划器节点以提供一个关键的反馈循环。它评估到目前为止取得的进展，并决定下一步的行动方案。它可以决定任务已完成并制定最终响应，或者更新计划以继续工作。这使得代理能够在步骤失败或初始计划不足时进行自我纠正。

与规划器类似，重新规划器将使用一个结构化输出模式 `Act`，它可以包含最终的 `Response` 或一个新的 `Plan`。

```python
from typing import Union

class Response(BaseModel):
    """给用户的最终响应。"""
    response: str

class Act(BaseModel):
    """下一步要采取的行动。"""
    action: Union[Response, Plan] = Field(
        description="要执行的行动。使用 'Response' 回复用户，或使用 'Plan' 继续执行新计划。"
    )

replanner_prompt = ChatPromptTemplate.from_template(
    """你是一位专家项目经理。你的角色是评估任务的进展并决定下一步的行动。
    
原始目标: {input}
原始计划: {original_plan}
已采取的步骤和结果: {past_steps}

根据进展，用剩余的步骤更新计划。如果目标已完成，请向用户提供最终响应。
只包括仍然需要完成的步骤。
"""
)

replanner_llm = ChatOpenAI(model="gpt-4o", temperature=0)
replanner = replanner_prompt | replanner_llm.with_structured_output(Act)

async def replanner_node(state: AgentState):
    output = await replanner.ainvoke(state)
    if isinstance(output.action, Response):
        return {"response": output.action.response}
    else:
        return {"plan": output.action.steps}
```

#### 条件边

图逻辑的最后一部分是条件边，它在重新规划器行动后指导流程。这个简单的函数检查状态，如果最终响应可用，则将代理路由到 `END`，否则返回到 `executor_node` 以继续执行新计划。

```python
from langgraph.graph import END

def should_continue(state: AgentState):
    if state.get("response"):
        return END
    else:
        return "executor_node"
```

### 4.2 设计代理的工具箱

工具是代理与世界交互的接口，使其能够执行超越简单生成文本的行动。受 Claude Code 中观察到的分层工具设计（低、中、高级）的启发，我们将定义一套最小但功能强大的基础工具集[2](https://minusx.ai/blog/decoding-claude-code/)。每个工具都实现为一个 Python 函数，并用 LangChain 的 `@tool` 装饰器进行装饰，该装饰器会自动为 LLM 生成一个模式和描述。

#### 文件 I/O 工具

这些工具允许代理在项目目录中读取和写入文件。

```python
from langchain_core.tools import tool
import os

@tool
def read_file(path: str) -> str:
    """读取指定路径下文件的全部内容。"""
    try:
        with open(path, 'r', encoding='utf-8') as f:
            return f.read()
    except Exception as e:
        return f"读取文件时出错: {e}"

@tool
def write_file(path: str, content: str) -> str:
    """将给定内容写入指定路径的文件，如果文件存在则覆盖。"""
    try:
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, 'w', encoding='utf-8') as f:
            f.write(content)
        return f"成功写入 {path}"
    except Exception as e:
        return f"写入文件时出错: {e}"

# 一个更高级的工具，使用 LLM 进行原地编辑
@tool
def edit_file(path: str, instructions: str) -> str:
    """读取一个文件，根据自然语言指令应用编辑，然后写回。"""
    try:
        original_content = read_file.invoke(path)
        if "Error" in original_content:
            return original_content
            
        edit_prompt = ChatPromptTemplate.from_template(
            "根据提供的代码应用以下编辑指令。\n"
            "只输出完整的、修改后的代码。不要添加任何评论或 markdown 格式。\n\n"
            "指令: {instructions}\n\n"
            "原始代码:\n```\n{code}\n```"
        )
        
        editor_llm = ChatOpenAI(model="claude-3-5-sonnet-20240620", temperature=0)
        editor_chain = edit_prompt | editor_llm
        
        edited_content = editor_chain.invoke({
            "instructions": instructions,
            "code": original_content
        }).content
        
        return write_file.invoke(path, edited_content)
    except Exception as e:
        return f"编辑文件时出错: {e}"
```

#### 文件系统工具

这些工具提供基本的文件系统导航功能。

```python
from typing import List

@tool
def list_files(path: str = '.') -> List[str]:
    """列出指定路径下的所有文件和目录。"""
    try:
        return os.listdir(path)
    except Exception as e:
        return f"列出文件时出错: {e}"
```

#### 执行工具

这是最强大且可能最危险的工具，允许代理执行任意的 shell 命令。其实现对于运行测试或构建脚本等功能至关重要。**此工具的安全实现将是第五部分的唯一重点。** 目前，我们定义其接口，以便可以将其包含在提供给执行器代理的工具集中。

```python
@tool
def execute_bash(command: str) -> str:
    """在一个安全的、沙箱化的环境中执行一个 shell 命令，并返回其 stdout 和 stderr。"""
    # 完整的、安全的实现将在第五部分详细介绍。
    # 这只是接口的占位符。
    return "此工具将在沙箱化部分实现。"

# 执行器代理的最终工具列表
tools = [read_file, write_file, edit_file, list_files, execute_bash]
```

## 第五部分：安全网：使用 Docker 沙箱进行安全代码执行

本部分解决了代码生成代理最关键的安全问题：运行不受信任的、AI 生成的代码。我们将提供一个完整、实用的指南，用于构建和集成使用 Docker 的沙箱化执行环境。

### 5.1 沙箱化的必要性

授予 AI 代理在主机上执行任意代码或 shell 命令的能力存在巨大的安全风险。没有适当的隔离，可能会出现许多危险情况 [9](https://anukriti-ranjan.medium.com/building-a-sandboxed-environment-for-ai-generated-code-execution-e1351301268a)：

- **恶意代码注入：** 代理可能被提示生成并执行删除文件、窃取敏感数据或安装恶意软件的代码。
- **资源耗尽：** 生成的代码中的一个 bug 可能导致无限循环或内存泄漏，消耗所有可用的 CPU 和内存，并可能导致主机系统崩溃。
- **意外的系统访问：** 代理可能执行读取敏感系统文件（例如，`/etc/passwd`）、访问私有网络资源或修改系统配置的命令。

试图通过简单的 Python `exec` 函数包装器来减轻这些风险是不足够且危险的。唯一健壮的、行业标准的解决方案是在**沙箱化环境**中执行所有不受信任的代码。容器化技术，特别是 Docker，提供了必要的进程级隔离、资源控制和依赖管理，以创建一个安全的执行环境[9](https://anukriti-ranjan.medium.com/building-a-sandboxed-environment-for-ai-generated-code-execution-e1351301268a), [26](https://github.com/substratusai/sandboxai)。

### 5.2 构建 Docker 执行环境

我们的沙箱将是一个从专用 `Dockerfile` 构建的 Docker 容器。该文件定义了一个一致、可复现的代码执行环境。它将基于一个轻量级的 Python 镜像，并预装 AI 生成的代码可能需要的常用库，从而避免了危险的运行时包安装需求。

以下是我们的执行环境的一个最小但功能齐全的 `Dockerfile`：

```dockerfile
# 使用一个轻量级的、官方的 Python 基础镜像
FROM python:3.11-slim

# 在容器内设置一个工作目录
WORKDIR /workspace

# 安装用于数据科学和网络请求的常用 Python 库
# 这避免了代理在运行时使用 pip install 的需要
RUN pip install --no-cache-dir pandas numpy requests beautifulsoup4

# 创建一个非 root 用户以增加安全性
RUN useradd --create-home --shell /bin/bash appuser
USER appuser

# 容器将启动并等待命令
CMD ["tail", "-f", "/dev/null"]
```

这个 `Dockerfile` 建立了一些关键的安全性和可用性特性：

- 它使用一个 `slim` Python 镜像以减小体积。
- 它预装了一组受信任的、常用的库。
- 它创建了一个非 root 用户 `appuser` 来运行代码，遵循最小权限原则。
- 它将 `/workspace` 设置为工作目录，我们将把用户的项目挂载到该目录中。
- 默认的 `CMD` 使容器无限期运行，以便我们可以在其中执行命令。

要构建此镜像，用户将在包含 `Dockerfile` 的目录中运行 `docker build -t mini-claude-sandbox.`。

### 5.3 使用 Docker Python SDK 进行程序化控制

构建了沙箱镜像后，我们现在可以实现 4.2 节中定义的 `execute_bash` 工具。此实现将使用 `docker-py` Python SDK 来以编程方式管理容器并在其中执行命令，捕获输出以返回给代理[27](https://docker-py.readthedocs.io/), [28](https://leftasexercise.com/2018/07/09/controlling-docker-container-with-python/)。

该工具的逻辑如下：

1. **初始化 Docker 客户端：** 创建一个客户端对象以与 Docker 守护进程交互。
2. **查找或启动沙箱容器：** 检查名为 `mini-claude-sandbox-instance` 的容器是否已在运行。如果没有，则从我们的 `mini-claude-sandbox` 镜像启动一个新的。启动容器时，将用户的当前项目目录挂载到容器的 `/workspace` 目录中至关重要。这使代理能够以受控的方式访问项目文件。
3. **执行命令：** 使用 `exec_run` 方法在正在运行的容器内执行所需的命令。此方法是安全的，因为命令在容器的隔离环境中运行。
4. **捕获并返回输出：** `exec_run` 方法方便地返回命令的退出代码及其合并的 stdout 和 stderr 输出。该工具将解码此输出并将其作为字符串返回给代理，为重新规划步骤提供必要的反馈。

以下是 `execute_bash` 工具的完整 Python 实现：

```python
from langchain_core.tools import tool
import docker
import os

# 为容器名称定义一个常量
SANDBOX_CONTAINER_NAME = "mini-claude-sandbox-instance"
SANDBOX_IMAGE_NAME = "mini-claude-sandbox:latest"

@tool
def execute_bash(command: str) -> str:
    """
    在一个安全的、沙箱化的 Docker 环境中执行一个 shell 命令
    并返回其 stdout 和 stderr。用户的当前工作
    目录被挂载为沙箱中的 /workspace。
    """
    try:
        client = docker.from_env()

        # 检查沙箱容器是否正在运行
        try:
            container = client.containers.get(SANDBOX_CONTAINER_NAME)
            if container.status != "running":
                container.start()
        except docker.errors.NotFound:
            # 未找到容器，因此创建并启动它
            print(f"正在启动新的沙箱容器: {SANDBOX_CONTAINER_NAME}")
            container = client.containers.run(
                SANDBOX_IMAGE_NAME,
                name=SANDBOX_CONTAINER_NAME,
                detach=True,
                volumes={os.getcwd(): {'bind': '/workspace', 'mode': 'rw'}},
                working_dir="/workspace",
                tty=True, # 保持容器存活
            )

        # 以非 root 用户身份在容器内执行命令
        exit_code, output = container.exec_run(
            command,
            user="appuser"
        )
        
        decoded_output = output.decode('utf-8')

        if exit_code == 0:
            return f"命令成功执行:\n{decoded_output}"
        else:
            return f"命令执行失败，退出代码 {exit_code}:\n{decoded_output}"

    except Exception as e:
        return f"尝试在 Docker 中执行命令时发生错误: {e}"
```

此实现为代理提供了一种健壮且安全的方式来与 shell 环境交互，完成了我们代理核心能力的最后一部分。

## 第六部分：组装与操作：让“迷你版 Claude”活起来

本最终实现部分将前面各节构建的所有组件——RAG 管道、LangGraph 状态机和沙箱化工具集——集成到一个单一、连贯的应用程序中。我们将提供完整的、可运行的脚本，并通过一个实际示例来演示其操作。

### 6.1 完整的“迷你版 Claude”代理脚本

以下脚本将我们之前定义的所有组件整合在一起。它定义了主应用程序逻辑，包括设置 RAG 管道、编译 LangGraph 以及通过命令行使用用户输入运行代理。

```python
# main.py - 完整的“迷你版 Claude”代理

import argparse
import asyncio
from typing import List, Tuple, Annotated
from typing_extensions import TypedDict
import operator

from langchain_core.messages import BaseMessage
from langgraph.graph import StateGraph, END, START

# --- 假设所有先前定义的组件都在单独的文件中并已导入 ---
# from rag_pipeline import setup_rag_retriever
# from agent_nodes import planner_node, executor_node, replanner_node, should_continue
# from tools import tools # 此列表包括安全的 execute_bash

# --- 1. 定义代理状态 ---
class AgentState(TypedDict):
    input: str
    plan: List[str]
    past_steps: Annotated[List[Tuple[str, str]], operator.add]
    response: str
    messages: Annotated[List[BaseMessage], operator.add]

# --- 2. 构建图 ---
def build_agent_graph():
    workflow = StateGraph(AgentState)

    # 添加节点
    workflow.add_node("planner_node", planner_node)
    workflow.add_node("executor_node", executor_node)
    workflow.add_node("replanner_node", replanner_node)

    # 定义边
    workflow.add_edge(START, "planner_node")
    workflow.add_edge("planner_node", "executor_node")
    workflow.add_edge("executor_node", "replanner_node")
    
    # 添加用于循环或结束的条件边
    workflow.add_conditional_edges(
        "replanner_node",
        should_continue,
        {
            "executor_node": "executor_node",
            END: END
        }
    )

    # 将图编译成可运行对象
    return workflow.compile()

# --- 3. 主应用程序逻辑 ---
async def main():
    parser = argparse.ArgumentParser(description="迷你版 Claude：一个代码原生 AI 代理")
    parser.add_argument("task", type=str, help="代理要执行的编码任务。")
    parser.add_argument("--project_path", type=str, default=".", help="项目目录的路径。")
    args = parser.parse_args()

    print("--- 初始化迷你版 Claude ---")
    
    # 在实际应用中，你将在此处设置 RAG 管道
    # 对于此示例，我们将直接传递任务
    # print(f"正在索引项目于: {args.project_path}...")
    # retriever = setup_rag_retriever(args.project_path)
    
    # 构建并编译代理图
    agent_app = build_agent_graph()

    print(f"--- 正在执行任务: {args.task} ---")
    
    # 运行代理
    initial_state = {"input": args.task, "past_steps": [], "messages": []}
    
    async for event in agent_app.astream(initial_state):
        for key, value in event.items():
            print(f"--- 事件: 节点 '{key}' ---")
            print(value)
            print("-" * 30)

    final_state = list(event.values())
    print("\n--- 任务完成 ---")
    print(f"最终响应: {final_state['response']}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 6.2 端到端演练：一个实际的编码任务

为了看到“迷你版 Claude”的实际运行情况，让我们追踪它在一个真实的软件开发任务上的执行过程。

**任务：** “此项目中的 `requirements.txt` 文件已过时。请将 `pandas` 库更新到最新版本，然后运行测试套件以确保升级不会破坏任何东西。测试使用 `pytest` 命令运行。”

**执行追踪：**

1. **初始状态：** 用户运行 `python main.py "The requirements.txt file..."`。初始状态为 `{"input": "The requirements.txt...", "past_steps": []}`。
2. **规划器节点输出：** 图从 `planner_node` 开始。规划器 LLM 接收目标并生成一个计划。
   - **新状态：** `{"plan": ["1. 读取 requirements.txt 的内容。", "2. 识别 pandas 的当前版本。", "3. 将 pandas 的版本更新为最新版本。", "4. 安装更新后的依赖项。", "5. 运行测试套件以验证更改。"]}`
3. **执行器节点（迭代 1）：** 图移动到 `executor_node`。它执行第一步，“读取 requirements.txt 的内容”，并调用 `read_file` 工具。
   - **工具调用：** `read_file(path='requirements.txt')`
   - **工具输出：** (文件的内容)
   - **新状态：** `{"past_steps": [("1. 读取...", (文件内容))], "plan": ["2. 识别...",... ]}`
4. **重新规划器节点（迭代 1）：** 重新规划器看到第一步成功，但计划尚未完成。它将现有计划传递下去。
   - **新状态：** (计划保持不变)
   - **路由：** `should_continue` 边路由回 `executor_node`。
5. **执行器节点（迭代 2-5）：** 代理继续执行计划：
   - **第 2 & 3 步：** 它使用 `edit_file` 工具更新 `pandas` 版本。
   - **第 4 步：** 它使用 `execute_bash` 工具，命令为 `pip install -r requirements.txt`。该命令在安全的 Docker 沙箱内运行。
   - **第 5 步：** 它使用 `execute_bash` 工具，命令为 `pytest`。假设测试通过。
6. **重新规划器节点（最终）：** 重新规划器现在看到完整的 `past_steps` 历史，包括成功的测试运行。它确定目标已完成。
   - **LLM 输出：** 重新规划器 LLM 生成一个最终的 `Response`。
   - **新状态：** `{"response": "我已成功将 requirements.txt 中的 pandas 库更新到最新版本，安装了新的依赖项，并确认所有测试都通过了。"}`
7. **END：** `should_continue` 边看到 `response` 字段现在已填充，并将图转换到 `END` 状态。应用程序打印最终响应并退出。

### 6.3 交互的最佳实践：向 Claude Code 用户学习

代理是一个强大的协作者，但当用户采用某些交互模式时，其效能会最大化。这些最佳实践，源自高级 Claude Code 用户的经验，将代理从一个简单的工具转变为一个真正的结对程序员[1](https://www.anthropic.com/engineering/claude-code-best-practices), [29](https://harper.blog/2025/05/08/basic-claude-code/)。

#### 拥抱测试驱动开发（TDD）

TDD 是与编码代理协同工作的一种异常强大的工作流。过程不是直接要求代理实现一个功能，而是反过来的：

1. **编写失败的测试：** 首先，指示代理：“为一个新函数 `calculate_premium` 编写一套测试，该函数接受一个用户对象并返回一个价格。测试应覆盖标准用户、管理员用户和有有效订阅的用户的案例。”
2. **确认失败：** 告诉代理运行测试并确认它们失败（因为函数尚不存在）。
3. **实现以通过测试：** 现在，指示代理：“编写 `calculate_premium` 函数的实现，使所有新测试都通过。不要修改测试。”

这种工作流非常有效，因为它为代理提供了一个清晰、客观且机器可验证的“完成”定义。测试套件充当了所需功能的精确规范，减少了模糊性，并带来了更准确的结果[1](https://www.anthropic.com/engineering/claude-code-best-practices), [29](https://harper.blog/2025/05/08/basic-claude-code/)。

#### 使用迭代优化和路线修正

应将代理视为协作者，而非无懈可击的神谕。最有效的用户会积极引导代理的过程。

- **审查计划：** 始终检查代理生成的初始计划。如果它看起来有缺陷或效率低下，请在执行开始前立即提供反馈以进行纠正。
- **中断和重定向：** 如果在执行过程中，代理似乎走错了路，应用程序应提供一种中断它的机制（模仿 Claude Code 中的‘Escape’键功能）[1](https://www.anthropic.com/engineering/claude-code-best-practices)。然后用户可以提供纠正性反馈，重新规划器可以根据此人工输入生成一个新的、更好的计划。代理是一个强大的工具，但开发者必须始终是架构师。

## 第七部分：结论与未来展望

本文为解构像 Claude Code 这样复杂编码代理的核心原则，并重建一个简化但功能齐全的版本“迷你版 Claude”，提供了一份全面的技术蓝图。通过将其架构分解为三大关键支柱——具备代码库感知的 RAG、规划器-执行器控制循环和 Docker 化执行沙箱——我们为构建强大的开发者工具铺设了一条实用的路线图。

### 7.1 关键架构原则总结

构建“迷你版 Claude”的历程揭示了为软件开发创建有效 AI 代理的几个基本原则：

- **架构简单性本身就是一种特性：** 在一个由 LLM 的不确定性主导的领域，复杂性是一种负担。一个简单、可观察的单进程状态机比一个由多个交互代理组成的复杂网络更健壮、更易于调试。
- **提示工程是一项核心开发活动：** 对于复杂的代理，系统提示不仅仅是一条指令，而是一个详细的软件产物。它必须以与代理代码同等的严谨性进行工程设计，包含明确的算法、启发式规则和安全约束。
- **上下文必须具备语法感知能力：** 要想对代码进行推理，代理必须首先理解它。标准的 RAG 技术之所以失败，是因为它们对语法一无所知。基于 AST 的分块方法对于为 LLM 提供一个连贯、语义上有意义的代码库视图至关重要。
- **分层控制管理复杂性：** 规划器-执行器模型为处理长周期任务提供了一个健壮的框架。通过将高层战略规划与低层工具执行分离开来，它创建了一个模块化且高效的控制循环，能够自我纠正和适应。
- **执行必须沙箱化：** 执行代码的能力伴随着安全执行的绝对责任。一个安全的、容器化的沙箱不是一个可选功能，而是任何能够编写和运行代码的代理的不可协商的要求。

### 7.2 高级功能路线图

我们设计的“迷你版 Claude”代理是一个强大的基础，但这仅仅是个开始。以下路线图概述了未来发展的几个关键领域，灵感来自完整 Claude Code 产品的高级功能，以进一步增强其能力和实用性。

- **检查点与状态持久化：** 对于长时间运行的任务，一个关键特性是能够保存和恢复代理的状态。LangGraph 内置了对检查点机制的支持（例如，`InMemorySaver`，或更健壮的数据库支持的保存器）[8](https://www.langchain.com/langgraph)。实现这一点将允许代理被停止和重新启动，而不会丢失其在复杂计划上的进度，这是生产级代理中一个备受期待的功能[30](https://www.anthropic.com/news/claude-sonnet-4-5)。
- **多模态能力：** 现代软件开发不仅限于文本。开发者会处理 UI 模型、架构图和错误截图。代理的下一次演进将是集成一个多模态视觉模型。这将启用新的工作流，例如向代理提供一个 UI 截图并指示它“编写实现此设计的 React 组件”，这是 Claude Code 所擅长的能力[1](https://www.anthropic.com/engineering/claude-code-best-practices)。
- **IDE 集成：** 虽然一个终端原生的工具很强大，但更深度地集成到开发者的 IDE 中，例如一个 VS Code 扩展，可以解锁更无缝的用户体验[30](https://www.anthropic.com/news/claude-sonnet-4-5)。一个扩展可以提供代理建议更改的可视化差异对比，允许用户选择代码片段作为上下文，并直接从编辑器中触发代理操作，从而进一步减少摩擦。
- **高级工具与模型上下文协议（MCP）：** 我们定义的简单工具集可以显著扩展。构建高级工具的一个强大范式是模型上下文协议（MCP），这是一种代理与外部服务交互的标准化方式[5](https://docs.claude.com/en/docs/claude-code/overview#:~:text=Claude%20Code%20maintains%20awareness%20of%20your%20entire%20project%20structure%2C%20can%20find), [31](https://medium.com/@elisowski/mcp-explained-the-new-standard-connecting-ai-to-everything-79c5a1c98288)。通过构建或使用 MCP 服务器，代理可以获得直接与 Figma 等服务交互以读取设计规范、与 Google Drive 交互以访问项目文档，甚至通过 Playwright MCP 服务器与正在运行的 Web 应用程序交互以对其自己的代码进行端到端测试和视觉验证的能力[30](https://www.anthropic.com/news/claude-sonnet-4-5), [31](https://medium.com/@elisowski/mcp-explained-the-new-standard-connecting-ai-to-everything-79c5a1c98288)。另一个强大的应用是利用 Context7 MCP 服务器，它可以在代理生成代码时动态地注入最新的库文档和代码示例，从而解决 AI 依赖过时训练数据的问题[32](https://upstash.com/blog/context7-mcp)。此外，一个最新的突破性进展是 Chrome 开发者工具 MCP 服务器的出现，它允许代理直接在 Chrome 浏览器中调试和验证网页，诊断网络和控制台错误，并分析性能，让代理真正具备了“看见”其代码运行效果的能力[33](https://developer.chrome.com/blog/chrome-devtools-mcp?hl=zh-cn)。这指向了一个未来，代理不仅参与编写代码，而且参与软件开发的整个设计、实现和验证生命周期。

## 参考资料

1. https://www.anthropic.com/engineering/claude-code-best-practices
2. https://minusx.ai/blog/decoding-claude-code/
3. https://prismic.io/blog/claude-code
4. https://rafaelquintanilha.com/is-claude-code-worth-the-hype-or-just-expensive-vibe-coding/
5. https://docs.claude.com/en/docs/claude-code/overview#:~:text=Claude%20Code%20maintains%20awareness%20of,conflicts%2C%20and%20write%20release%20notes
6. https://www.reddit.com/r/AI_Agents/comments/1jexngk/optimizing_ai_agents_with_opensouce/
7. https://python.langchain.com/docs/tutorials/agents/
8. https://www.datacamp.com/tutorial/langgraph-agents
9. https://anukriti-ranjan.medium.com/building-a-sandboxed-environment-for-ai-generated-code-execution-e1351301268a
10. https://blog.langchain.com/planning-agents/
11. https://www.promptlayer.com/glossary/plan-and-execute-agents
12. https://www.emergentmind.com/topics/planner-executor-architecture-4c9e0097-fe2b-4870-b41c-9519c49a07c8
13. https://arxiv.org/html/2503.09572v2
14. https://www.youtube.com/watch?v=uRya4zRrRx4
15. https://langchain-ai.github.io/langgraph/tutorials/plan-and-execute/plan-and-execute/
16. https://www.ibm.com/think/tutorials/chunking-strategies-for-rag-with-langchain-watsonx-ai
17. https://medium.com/@jouryjc0409/ast-enables-code-rag-models-to-overcome-traditional-chunking-limitations-b0bc1e61bdab
18. https://community.databricks.com/t5/technical-blog/the-ultimate-guide-to-chunking-strategies-for-rag-applications/ba-p/113089
19. https://arxiv.org/html/2506.15655v1
20. https://www.unite.ai/code-embedding-a-comprehensive-guide/
21. https://medium.com/@shreshthg30/a-beginners-guide-to-tree-sitter-6698f2696b48
22. https://tree-sitter.github.io/tree-sitter/using-parsers/
23. https://github.com/yilinjz/astchunk
24. https://modal.com/blog/6-best-code-embedding-models-compared
25. https://www.youtube.com/watch?v=vpD9kf5Xwo0
26. https://e2b.dev/
27. https://docker-py.readthedocs.io/en/stable/containers.html
28. https://leftasexercise.com/2018/07/09/controlling-docker-container-with-python/
29. https://harper.blog/2025/05/08/basic-claude-code/
30. https://www.anthropic.com/news/claude-sonnet-4-5
31. https://medium.com/@elisowski/mcp-explained-the-new-standard-connecting-ai-to-everything-79c5a1c98288
32. https://upstash.com/blog/context7-mcp
33. https://developer.chrome.com/blog/chrome-devtools-mcp?hl=zh-cn