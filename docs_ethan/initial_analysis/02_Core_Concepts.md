# 02 - 核心概念

本文档基于 `n8n-workflow` 和 `n8n-core` 包的源代码，对 n8n 系统中的核心领域概念进行定义和解释。理解这些概念是深入了解 n8n 工作原理的基础。

---

### 1. 工作流 (Workflow)

- **源码定义**: `packages/workflow/src/workflow.ts` (class `Workflow`), `packages/workflow/src/interfaces.ts` (interface `IWorkflow`)
- **核心职责**: 工作流是 n8n 的核心编排单元，它是一个包含了业务流程所有逻辑的蓝图。
- **关键属性**:
    - `id`: 工作流的唯一标识符。
    - `name`: 用户定义的工作流名称。
    - `nodes`: 一个包含工作流中所有**节点（Node）**对象（`INode`）的集合。
    - `connections`: 定义了节点之间连接关系的对象。
    - `active`: 一个布尔值，表示该工作流当前是否处于激活状态。激活的工作流才能被触发器（如 Webhook、Cron）自动执行。
    - `settings`: 工作流级别的配置，例如时区（`timezone`）、错误处理工作流（`errorWorkflow`）等。
    - `staticData`: 用于存储与工作流相关的持久化数据，例如由某个节点注册的 Webhook 的ID��

---

### 2. 节点 (Node)

- **源码定义**: `packages/workflow/src/interfaces.ts` (interface `INode`)
- **核心职责**: 节点是工作流中的**基本执行单元**，代表着一个具体的操作步骤。
- **关键属性**:
    - `name`: 节点在当前工作流中的唯一名称（例如 "HTTP Request1", "Set"）。
    - `type`: 节点的类型（例如 "n8n-nodes-base.httpRequest"），它决定了该节点的功能。
    - `typeVersion`: 节点类型的版本号，用于处理节点的功能迭代。
    - `parameters`: 一个对象，存储了用户在该节点UI上配置的所有参数（例如 HTTP Request 节点的 URL、请求方法等）。
    - `credentials`: 存储了该节点所使用的凭证信息。
    - `position`: 节点在前端画布上的 `[x, y]` 坐标。
    - `disabled`: 一个布尔值，如果为 `true`，该节点在工作流执行时会被跳过。

---

### 3. 连接 (Connection)

- **源码定义**: `packages/workflow/src/interfaces.ts` (interface `IConnection`, `IConnections`)
- **核心职责**: 连接定义了数据在不同节点之间的**流向**。
- **数据结构**:
    - 一个工作流的 `connections` 属性是一个复杂对象，它以**源节点（Source Node）**的名称作为主键。
    - 每个源节点下，又以**输出类型（Output Type）**（默认为 `main`）��为键。
    - 最终指向一个 `IConnection` 对象数组，该对象包含：
        - `node`: 目标节点（Destination Node）的名称。
        - `type`: 目标节点上的输入类型（默认为 `main`）。
        - `index`: 目标节点上的输入索引（一个节点可以有多个相同类型的输入）。

---

### 4. 凭证 (Credential)

- **源码定义**: `packages/workflow/src/interfaces.ts` (interface `ICredentialType`, `ICredentialsEncrypted`)
- **核心职责**: 凭证用于安全地存储和管理访问第三方服务所需的认证信息（如 API Keys, OAuth2 Tokens）。
- **关键特征**:
    - **类型化**: 每种凭证都有一个明确的类型（例如 `googleApi`, `aws`），由 `ICredentialType` 定义。
    - **加密存储**: 凭证的敏感数据（如 token, secret）在数据库中是以**加密**形式存储的。
    - **节点关联**: 节点通过其 `credentials` 属性与一个或多个凭证相关联。
    - **自动刷新**: 对于支持 OAuth2 的凭证，n8n 的核心逻辑会自动处理 Access Token 的刷新。

---

### 5. 执行 (Execution)

- **源码定义**: `packages/core/src/execution-engine/workflow-execute.ts`, `packages/workflow/src/interfaces.ts` (interface `IRun`, `ITaskData`)
- **核心职责**: 执行代表了一次完整的工作流运行实例。
- **关键数据结���**:
    - **`IRun`**: 代表一次完整的端到端执行。
        - `status`: 执行的状态（`running`, `success`, `error`, `canceled`）。
        - `startedAt` / `stoppedAt`: 执行的开始和结束时间。
        - `data` (`IRunData`): 包含了此次执行中**每个节点**的详细运行数据。
    - **`IRunData`**: 一个以节点名称为键的对象，值为一个 `ITaskData` 数组（因为一个节点在一次执行中可能运行多次，例如在循环中）。
    - **`ITaskData`**: 代表一个节点的一次具体运行。
        - `data`: 包含了该节点此次运行的**输出数据**。
        - `error`: 如果运行出错，则包含错误信息。
        - `executionTime`: 本次运行的耗时。
        - `executionStatus`: 本次运行的状态 (`success`, `error`)。