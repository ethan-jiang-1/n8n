# 03 - 工作流执行生命周期

本文档深入 `packages/core/src/execution-engine/workflow-execute.ts` 的源码，追踪一个 n8n 工作流从被触发到所有节点执行完毕的完整过程。其核心是 `WorkflowExecute` 类。

## 1. 启动与初始化 (`run` 方法)

工作流的执行始于 `WorkflowExecute` 实例的 `run` 方法。

1.  **确定起始节点**:
    -   方法首先调用 `workflow.getStartNode()` 来确定从哪个节点开始执行。通常情况下，这是一个触发器节点（如 `Cron`, `Webhook`）或 `Start` 节点。
    -   如果没有找到起始节点，执行会直接抛出错误。

2.  **构建初始执行上下文**:
    -   一个名为 `runExecutionData` 的核心对象被创建，它将贯穿整个执行过程，用于追踪状态。
    -   其中最重要的部分是 `executionData.nodeExecutionStack`，这是一个**执行栈（LIFO 队列）**。
    -   起始节点和它的初始输入数据（对于手动执行，通常是一个空的 `json: {}` 对象）被包装成一个 `IExecuteData` 对象，并被 `push` 进 `nodeExecutionStack`。

3.  **调用核心处理函数**:
    -   最后，`run` 方法调用 `this.processRunExecutionData(workflow)`，将控制权交给主执行循环。

## 2. 主执行循环 (`processRunExecutionData` 方法)

这是驱动整个工作流前进的引擎，其核心是一个 `while` 循环。

-   **循环条件**: `while (this.runExecutionData.executionData.nodeExecutionStack.length !== 0)`
    -   只要执行栈不为空，循环就会一直持续。

-   **循环体核心步骤**:
    1.  **出栈**: `executionData = nodeExecutionStack.shift()`
        -   从执行栈的**头部**取出一个待执行的节点任务 (`IExecuteData`)。这包含了要执行的节点对象（`node`）和它所需要的输入数据（`data`）。
    2.  **执行节点**: `runNodeData = await this.runNode(...)`
        -   调用 `runNode` 方法来实际执行这个节点（详见下一节）。
        -   `runNode` 会返回节点的输出数据 `nodeSuccessData`。
    3.  **处理结果**:
        -   节点的执行结果（包括输出数据、耗时、状态等）被包装成 `ITaskData` 对象，并存入 `runExecutionData.resultData.runData` 中，以节点名称作为 key 进行归档。
        -   如果节点执行成功但没有返回任何数据 (`nodeSuccessData` 为 `null`)，当前执行分支会终止，循环会 `continue` 到下一次，处理栈中的下一个任务。
    4.  **调度后续节���**:
        -   如果节点成功并返回了数据，代码会查找当前节点的所有**输出连接**。
        -   对于每一个连接到的下游节点，调用 `addNodeToBeExecuted(...)` 方法，将其加入到待执行队列中。

## 3. 单节点执行 (`runNode` 方法)

这个方法负责执行单个节点的逻辑。

1.  **获取节点类型**: 通过 `workflow.nodeTypes.getByNameAndVersion()` 获取节点的类型定义，其中包含了节点的 `execute` 方法。
2.  **准备执行上下文**: 创建一个 `ExecuteContext` 实例。这个上下文对象（`this`）会被传入节点的 `execute` 方法中，为节点提供了访问各种工具函数（如 `getCredentials`, `getNodeParameter`, `httpRequest`）的能力。
3.  **调用 execute**: `data = await nodeType.execute.call(context)`
    -   这是最核心的一步，实际调用了节点自身的 `execute` 方法来执行其业务逻辑。
4.  **返回输出**: `execute` 方法的返回值（即节点的输出数据）最终被 `runNode` 方法返回。

## 4. 节点调度与多输入处理 (`addNodeToBeExecuted` 方法)

当一个节点执行完毕后，此方法负责决定它的下游节点是否应该被执行。

1.  **检查多输入**:
    -   方法首先检查下游节点是否拥有多个输入。
    -   **如果只有一个输入**：直接将该下游��点和上游节点的输出数据打包成 `IExecuteData`，`push` 到 `nodeExecutionStack` 的**尾部**，等待后续执行。
    -   **如果有多个输入**：情况变得复杂。n8n 需要等待该节点的所有输入都就绪后才能执行它。

2.  **等待多输入就绪**:
    -   当第一个上游节点完成时，会为这个多输入节点在 `waitingExecution` 对象中创建一个“等待区”。
    -   第一个上游节点的输出数据会被存入这个等待区的对应输入槽位。
    -   当后续的上游节点陆续执行完毕，它们各自的输出数据也会被存入相应的槽位。
    -   每次存入数据后，都会检查该节点的所有输入槽位是否都已有数据。
    -   **一旦所有输入都就绪**，`addNodeToBeExecuted` 方法会将这个多输入节点从 `waitingExecution` 中取出，连同其所有合并好的输入数据，一同 `push` 到 `nodeExecutionStack` 的尾部。

## 5. 错误处理与重试

-   **错误捕获**: 在主执行循环的 `for` 循环（用于重试）中，有一个 `try...catch` 块包围着 `runNode` 的调用。
-   **重试机制**:
    -   如果节点配置了 `retryOnFail`，`for` 循环会根据配置的 `maxTries` 和 `waitBetweenTries` 进行多次尝试。
-   **失败处理**:
    -   如果重试后依然失败，并且节点没有配置 `continueOnFail`，错误信息会被记录到 `runExecutionData.resultData.error` 中，然后 `break` 退出主执行循环，整个工作流执行失败。
    -   如果配置了 `continueOnFail`，则会将节点的输入数据作为其输出，继续执行后续流程。

---

**总结**: n8n 的执行引擎是一个基于**栈（LIFO队列）的事件循环**。它从一个起点开始，不断地从栈中取出任务、执行任务、然后将新的任务（下游节点）推入栈中，直到栈被清空。通过一个 `waitingExecution` 机制，它优雅地处理了多输入节点的同步问题。