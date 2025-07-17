# 04 - 如何创建自定义节点

本文档是一份面向开发者的指南，通过分析 `n8n-nodes-base` 中 `HttpRequest` 等节点的实现，总结出创建和集成一个全新 n8n 节点的核心步骤和最佳实践。

创建一个自定义节点，本质上是实现 n8n 定义的 `INodeType` 接口，并将其打包以便 n8n 主程序能够发现和加载。

---

## 1. 文件结构

一个标准的节点通常包含以下文件结构。以创建一个名为 `MyNode` 的节点为例：

```
/n8n/packages/nodes-base/nodes/MyNode/
├── MyNode.node.ts         # 节点的主定义文件 (入口)
├── MyNode.node.json       # (可选) 节点的静态元数据
├── mynode.svg             # (可选) 节点的图标 (亮色模式)
└── mynode.dark.svg        # (可选) 节点的图标 (暗色模式)
```

-   **`MyNode.node.ts`**: 这是最重要的文件，包含了节点的全部逻辑。
-   **`*.svg`**: 节点的图标文件，推荐提供亮色和暗色两种模式的 SVG 图标。

---

## 2. 节点主定义文件 (`MyNode.node.ts`)

这个 TypeScript 文件需要导出一个实现了 `INodeType` 接口的类。��个类定义了节点的一切行为。

```typescript
import {
	IExecuteFunctions,
	INodeExecutionData,
	INodeType,
	INodeTypeDescription,
} from 'n8n-workflow';

export class MyNode implements INodeType {
	// 1. 节点的描述信息 (description)
	description: INodeTypeDescription = {
		displayName: 'My Awesome Node',
		name: 'myNode',
		icon: 'file:mynode.svg',
		group: ['transform'],
		version: 1,
		description: 'This is a short description of what my node does.',
		defaults: {
			name: 'My Node',
		},
		inputs: ['main'],
		outputs: ['main'],
		properties: [
			// 在这里定义节点的参数
			{
				displayName: 'My String Parameter',
				name: 'myString',
				type: 'string',
				default: '',
				placeholder: 'Placeholder text',
				description: 'A description for this parameter.',
			},
		],
	};

	// 2. 节点的执行逻辑 (execute)
	async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
		const items = this.getInputData();
		const returnData: INodeExecutionData[] = [];

		for (let itemIndex = 0; itemIndex < items.length; itemIndex++) {
			// 从参数中获取用户输入
			const myString = this.getNodeParameter('myString', itemIndex, '') as string;

			// 节点的业务逻辑
			const transformedData = {
				...items[itemIndex].json,
				newNodeData: `The input was: ${myString}`,
			};

			returnData.push({ json: transformedData });
		}

		return this.prepareOutputData(returnData);
	}
}
```

### 2.1. `description` 字段详解

`description` 对象定义了节点的元数据和UI呈现，是 `INodeTypeDescription` 接口的实现。

-   `displayName`: 显示在节点创建菜单和画布中的名称。
-   `name`: 节点的内部唯一标识符，通常是驼峰式命名。
-   `icon`: 节点的图标路径。
-   `group`: 节点所属的类别，决定它在节点菜单的哪个分组下（如 `transform`, `output`, `trigger`）。
-   `version`: 节点的版本号。
-   `description`: 对节点功能的简短描述。
-   `defaults`: 创建节点实例时的默认属性，如默认的 `name`。
-   `inputs` & `outputs`: 定义节点的输入/输出连接点，最常见的是 `['main']`。
-   `properties`: **核心部分**。这是一个 `INodeProperties` 对象数组，定义了在节点设置面板中显示给用户的所有参数字段。每个参数对象包含：
    -   `displayName`: 参数的显示名称。
    -   `name`: 参数的内部名称。
    -   `type`: 参数的类型 (`string`, `number`, `boolean`, `options`, `collection` 等)。
    -   `default`: 参数的默认值。
    -   其他可选属性如 `placeholder`, `description` 等。

### 2.2. `execute` 方法详解

`execute` 方法是节点的**运行时逻辑**，它会在工作流执行到该节点时被调用。

-   **`this` 上下文**: `execute` 方法的 `this` 被绑定为一个 `IExecuteFunctions` 实例，它提供了与 n8n 执行引擎交互的所有工具函数。
-   **获取输入数据**: `this.getInputData()` 是最常用的函数之一，用于获取上游节点传递过来的所有数据项（items）。
-   **获取参数**: `this.getNodeParameter('parameterName', itemIndex)` 用于获取用户在UI上为某个参数配置的值。你需要为每个输入项（item）单独获取参数，因为参数值可能包含表达式，需要逐项解析。
-   **业务逻辑**: 在 `for` 循环中，你可以对每个输入项执行你的核心业务逻辑。
-   **返回输出数据**: `this.prepareOutputData(returnData)` 是一个工具函数，用于将你的处理结果包装成 n8n 要求的标准输出格式 (`INodeExecutionData[][]`)。

---

## 3. 创建凭证 (`MyCredential.credentials.ts`)

如果你的节点需要与受保护的API交互，就需要定义一个凭证类型。

文件位置：`/n8n/packages/nodes-base/credentials/MyCredential.credentials.ts`

```typescript
import { ICredentialType, INodeProperties } from 'n8n-workflow';

export class MyCredential implements ICredentialType {
	name = 'myCredential';
	displayName = 'My API Credential';
	properties: INodeProperties[] = [
		{
			displayName: 'API Key',
			name: 'apiKey',
			type: 'string',
			typeOptions: { password: true }, // 使其在UI上显示为密码字段
			default: '',
		},
		{
			displayName: 'API Secret',
			name: 'apiSecret',
			type: 'string',
			typeOptions: { password: true },
			default: '',
		},
	];
}
```

-   **`name`**: 凭证的内部唯一标识符。
-   **`displayName`**: 显示在凭证创建界面的名称。
-   **`properties`**: 与节点的 `properties` 类似，定义了创建此凭证时需要用户填写的字段。

在节点中关联凭证，需要在 `description` 中添加 `credentials` 字段：

```typescript
// In MyNode.node.ts
description: INodeTypeDescription = {
    // ... other properties
    credentials: [
        {
            name: 'myCredential', // 引用凭证的 name
            required: true,
        },
    ],
    properties: [
        // ... other properties
    ],
};
```

在 `execute` 方法中，通过 `this.getCredentials('myCredential', itemIndex)` 来获取解密后的凭证数据。

## 4. 注册与发现

n8n 使用一种声明式的机制来自动发现和加载节点。你需要将你的节点和凭证文件路径添加到 `packages/nodes-base/package.json` 的 `n8n` 字段中。

```json
// In packages/nodes-base/package.json
{
  "name": "n8n-nodes-base",
  "n8n": {
    "credentials": [
      "dist/credentials/MyCredential.credentials.js",
      // ... other credentials
    ],
    "nodes": [
      "dist/nodes/MyNode/MyNode.node.js",
      // ... other nodes
    ]
  }
}
```

完成以上步骤并重新构建项目 (`pnpm build`) 后，n8n 启动时就能加载你的自定义节点了。