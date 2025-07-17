# 01 - LangChain 节点架构

本文档将深入分析 `@n8n/n8n-nodes-langchain` 包，拆解不同类型 AI 节点的内部结构、参数以及它们之间独特的连接方式。

---

## 1. 统一的节点结构

与标准的 n8n 节点类似，每个 LangChain 节点也是一个实现了 `INodeType` 接口的类。其核心同样由 `description` 对象和 `execute` 方法组成。

然而，LangChain 节点在 `description` 的 `inputs` 和 `outputs` 定义上，引入了一套新的**自定义连接类型**，以反映 LangChain 组件之间的逻辑关系。

## 2. 自定义连接类型 (AI Node Connections)

除了标准的 `main` 输入/输出，AI 节点之间通过一系列以 `ai_` 为前缀的自定义连接类型进行交互。这些连接类型定义在 `packages/workflow/src/interfaces.ts` 的 `NodeConnectionTypes` 对象中。

常见的 AI 连接类型包括：

-   `ai_agent`: 代表一个 AI Agent 实例。
-   `ai_chain`: 代表一个 Chain 实例。
-   `ai_llm`: 代表一个 LLM 或 ChatModel 实例。
-   `ai_tool`: 代表一个可供 Agent 使用的工具。
-   `ai_memory`: 代表一个记忆模块。
-   `ai_retriever`: 代表一个检索器。
-   `ai_document`: 代表一份或多份文档。
-   `ai_embedding`: 代表一个嵌入模型实例。
-   `ai_vectorStore`: 代表一个向量数据库实例。

**这种设计是 n8n AI 功能的核心**。它允许 `execute` 方法的返回值不再是传统的数据（JSON），而是一个 **LangChain 类的实例**。当一个节点（如 "OpenAI" LLM 节点）通过 `ai_llm` 输出连接到另一个节点（如 "LLM Chain" 节点）时，它传递的不是文本数据，而是一个配置好的 `ChatOpenAI` 对象实例。下游节点可以直接使用这个对象实例，而无需关心其内部实现细节。

## 3. 节点分类与交互示例

### 3.1. LLM 节点 (e.g., `LmChatOpenAi.node.ts`)

-   **职责**: 配置并实例化一个具体的语言模型。
-   **输入**: 通常没有 AI 类型的输入。
-   **输出**: `outputs: ['ai_llm']`。
-   **`execute` 逻辑**:
    1.  通过 `this.getCredentials()` 获取 OpenAI 的 API Key。
    2.  通过 `this.getNodeParameter()` 获取用户配置的模型名称、温度等参数。
    3.  `return [[new ChatOpenAI({ ...parameters ... })]]`。返回一个 LangChain 的 `ChatOpenAI` 类的实例。

### 3.2. Chain 节点 (e.g., `ChainLlm.node.ts`)

-   **职责**: 将一个 LLM 和一个 Prompt 模板组合成一个 Chain。
-   **输入**: `inputs: ['main', 'ai_llm']`。它需要一个 `ai_llm` 类型的输入来接收上游的 LLM 实例，同时需要一个 `main` 输入来接收运行时的数据（用于填充 Prompt）。
-   **输出**: `outputs: ['main', 'ai_chain']`。它既可以像普通节点一样输出处理后的数据，也可以输出一个 Chain 实例供下游的 Agent 使用。
-   **`execute` 逻辑**:
    1.  通过 `this.getInputData(NodeConnectionTypes.AiLlm)` 获取上游传入的 `ChatOpenAI` 实例。
    2.  通过 `this.getNodeParameter()` 获取用户编写的 Prompt 模板。
    3.  创建一个 `LLMChain` 实例：`new LLMChain({ llm: llmInstance, prompt: promptTemplate })`。
    4.  如果 `main` 输入有数据，则调用 `chain.call(mainInputData)` 来执行并返回结果。
    5.  同时，将创建的 `LLMChain` 实例通过 `ai_chain` 输出传递下去。

### 3.3. Agent 节点 (e.g., `Agent.node.ts`)

-   **职责**: AI 应用的大脑，负责决策和工具调用。
-   **输入**: `inputs: ['main', 'ai_llm', 'ai_tool', 'ai_memory']`。它需要连接一个 LLM、一个或多个工具，以及一个可选的记忆模块。
-   **输出**: `outputs: ['main']`。Agent 的最终执行结果会通过 `main` 输出。
-   **`execute` 逻辑**:
    1.  分别通过 `getInputData` 获取上游传入的 LLM 实例、工具实例数组和记忆实例���
    2.  根据用户配置的 Agent 类型（如 `OpenAIFunctionsAgent`），初始化一个 Agent Executor。
    3.  调用 `agentExecutor.call({ input: ... })` 来启动 Agent。
    4.  Agent Executor 内部会循环调用 LLM 进行思考，选择合适的工具并执行，直到任务完成。
    5.  返回最终的执行结果。

通过这种巧妙的、基于实例传递的连接机制，n8n 成功地将 LangChain 的复杂性封装在了节点背后，为用户提供了一个直观、强大且高度可扩展的 AI 应用构建平台。