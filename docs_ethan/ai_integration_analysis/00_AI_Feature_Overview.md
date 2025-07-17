# 00 - AI 功能概览

本文档旨在提供 n8n 中 AI 及 LangChain 集成功能的顶层视图，解释其目标和核心设计理念。

---

## 1. 核心理念：将 LLM 应用模块化

n8n 的 AI 功能，其核心设计理念是将构建复杂 AI 应用（尤其是基于 LangChain 的应用）的过程，从纯代码编写转变为**可视化的、基于节点的工作流编排**。

传统的 LangChain 开发需要在代码中实例化和链接各种类（LLMs, Prompts, Chains, Agents, Tools, Retrievers）。n8n 将这些核心的 LangChain 组件，一一映射为可视化的**节点**。

这种方法带来了几个核心优势：

-   **降低门槛**: 用户无需编写大量样板代码，只需通过拖拽和配置节点，即可构建出强大的 AI 应用，如 RAG（检索增强生成）系统、自定义 AI 代理等。
-   **可视化与可调试性**: 整个 AI 应用的逻辑流和数据流一目了然。用户可以清晰地看到数据如何从文档加载器流向文本分割器，再到嵌入模型，最终存入向量数据库。每个步骤的输入和输出都可以在 n8n 的 UI 中直接查看，极大地简化了调试���程。
-   **与传统自动化的无缝集成**: 这是 n8n AI 功能最强大的地方。一个由 LLM 驱动的 AI Agent，可以轻易地将“发送 Slack 消息”或“更新 Salesforce 记录”等传统 n8n 节点作为其**工具（Tool）**来使用。这打通了 AI 决策与实际业务执行之间的壁垒。

## 2. 技术基石：LangChain.js

通过分析 `@n8n/n8n-nodes-langchain` 包的依赖，可以明确看出 n8n 的 AI 功能是深度构建在 **LangChain.js** 生态系统之上的。

-   它直接依赖 `langchain` 主包以及 `@langchain/openai`, `@langchain/cohere`, `@langchain/community` 等多个核心组件。
-   n8n 中的大部分 AI 节点，其内部实现都是对 LangChain.js 中相应类的封装和调用。

## 3. 功能模块划分

n8n 的 AI 节点库遵循了 LangChain 的模块划分，提供了一整套用于构建端到端 AI 应用的“积木块”：

-   **模型 I/O (LLMs & Chat Models)**: 用于与各种大语言模型服务（如 OpenAI, Anthropic, Cohere, Ollama）进行交互的节点。
-   **数据连接 (Document Loaders & Vector Stores)**: 用于从各种来源加载数据，并与向量数据库（如 Pinecone, Qdrant, Weaviate）进行交互的节点。
-   **数据转换 (Text Splitters & Embeddings)**: 用于将长文本分割成小块，并调用嵌入模型将其转换为向量的节��。
-   **记忆 (Memory)**: 为 AI 提供短期或长期记忆的节点，使得 AI 可以在多轮对话中保持上下文。
-   **链 (Chains)**: 将多个组件（通常是 LLM 和 Prompt）串联起来，实现特定任务（如文本摘要、问答）的预设模板。
-   **代理 (Agents)**: AI 应用的核心大脑。Agent 节点能够调用 LLM 进行思考和决策，并根据决策选择和使用一个或多个**工具（Tools）**来完成任务。