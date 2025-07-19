# 01 - 工作流存储格式

本文档将深入分析 n8n 工作流的底层文件格式，解析其核心 JSON 结构，为版本控制和自动化处理提供基础。

## n8n 工作流 JSON 结构概览

n8n 工作流以 JSON 格式存储，包含完整的工作流定义、节点配置、连接关系和元数据。这种结构化的格式为 Workflow as Code 提供了天然的优势。

### 顶层结构

```json
{
  "id": "35",
  "name": "工作流名称",
  "active": false,
  "createdAt": "2021-02-17T16:01:29.116Z",
  "updatedAt": "2021-02-17T18:38:58.265Z",
  "nodes": [...],
  "connections": {...},
  "settings": {...},
  "staticData": null,
  "meta": null,
  "pinData": null,
  "versionId": null,
  "triggerCount": 0,
  "tags": []
}
```

## 核心字段详解

### 1. 基础元数据

| 字段 | 类型 | 描述 | 版本控制考虑 |
|------|------|------|-------------|
| `id` | string | 工作流唯一标识符 | 环境间可能不同，需要映射策略 |
| `name` | string | 工作流显示名称 | 应保持一致，便于识别 |
| `active` | boolean | 工作流激活状态 | 环境相关，需要外部化配置 |
| `createdAt` | string | 创建时间戳 | 版本控制时可忽略 |
| `updatedAt` | string | 更新时间戳 | 版本控制时可忽略 |
| `versionId` | string | 版本标识符 | n8n 内部版本管理 |
| `triggerCount` | number | 触发器数量 | 自动计算字段 |
| `tags` | array | 工作流标签 | 用于分类和管理 |

### 2. 节点定义 (nodes)

节点数组包含工作流中的所有操作单元：

```json
{
  "parameters": {
    "channel": "random",
    "text": "Hello World",
    "attachments": []
  },
  "name": "Slack",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 1,
  "position": [420, 630],
  "id": "1f43049e-8dd6-425f-bb8a-e06edce9418b",
  "credentials": {
    "slackApi": {
      "id": "18",
      "name": "Slack Token"
    }
  },
  "alwaysOutputData": true
}
```

#### 节点字段说明

| 字段 | 类型 | 描述 | 版本控制注意事项 |
|------|------|------|-----------------|
| `id` | string | 节点唯一标识符 | UUID 格式，环境间保持一致 |
| `name` | string | 节点显示名称 | 工作流内唯一，便于引用 |
| `type` | string | 节点类型标识符 | 决定节点功能，版本敏感 |
| `typeVersion` | number | 节点类型版本 | 影响兼容性，需要管理 |
| `position` | array | 编辑器中的位置 | 可选，影响可视化布局 |
| `parameters` | object | 节点配置参数 | 核心配置，需要仔细管理 |
| `credentials` | object | 凭证引用 | 敏感信息，需要外部化 |

### 3. 连接关系 (connections)

定义节点间的数据流连接：

```json
{
  "Start": {
    "main": [
      [
        {
          "node": "Slack",
          "type": "main",
          "index": 0
        },
        {
          "node": "Slack13",
          "type": "main",
          "index": 0
        }
      ]
    ]
  }
}
```

#### 连接结构说明

- **键**: 源节点名称
- **值**: 连接数组，支持多输出端口
- **连接对象**:
  - `node`: 目标节点名称
  - `type`: 连接类型 (通常为 "main")
  - `index`: 目标节点的输入端口索引

### 4. 工作流设置 (settings)

```json
{
  "timezone": "Asia/Shanghai",
  "saveDataExecution": "all",
  "saveDataSuccess": "all",
  "saveDataError": "all",
  "saveManualExecutions": true,
  "callerPolicy": "workflowsFromSameOwner",
  "errorWorkflow": "error-handler-workflow-id"
}
```

#### 设置字段说明

| 字段 | 描述 | 环境考虑 |
|------|------|----------|
| `timezone` | 时区设置 | 可能因环境而异 |
| `saveDataExecution` | 执行数据保存策略 | 环境相关配置 |
| `callerPolicy` | 调用权限策略 | 安全相关设置 |
| `errorWorkflow` | 错误处理工作流 | 需要 ID 映射 |

### 5. 静态数据和固定数据

- **staticData**: 工作流级别的持久化数据
- **pinData**: 节点的固定测试数据
- **meta**: 元数据信息

## 版本控制最佳实践

### 1. 字段标准化

#### 需要标准化的字段
```json
{
  "createdAt": null,
  "updatedAt": null,
  "versionId": null,
  "triggerCount": 0
}
```

#### 环境相关字段
```json
{
  "active": false,  // 通过环境变量控制
  "id": "{{WORKFLOW_ID}}"  // 使用模板变量
}
```

### 2. 凭证外部化

原始格式：
```json
{
  "credentials": {
    "slackApi": {
      "id": "18",
      "name": "Slack Token"
    }
  }
}
```

外部化后：
```json
{
  "credentials": {
    "slackApi": {
      "id": "{{SLACK_CREDENTIAL_ID}}",
      "name": "{{SLACK_CREDENTIAL_NAME}}"
    }
  }
}
```

### 3. 参数模板化

敏感参数外部化：
```json
{
  "parameters": {
    "channel": "{{SLACK_CHANNEL}}",
    "text": "{{MESSAGE_TEMPLATE}}",
    "webhook_url": "{{WEBHOOK_URL}}"
  }
}
```

## 工作流导入导出

### CLI 命令

```bash
# 导出工作流
n8n export:workflow --id=35 --output=./workflows/

# 导入工作流
n8n import:workflow --input=./workflows/workflow-35.json

# 批量导出
n8n export:workflow --all --output=./workflows/

# 带凭证导出 (小心使用)
n8n export:credentials --output=./credentials/
```

### API 接口

```bash
# 获取工作流
curl -X GET "http://localhost:5678/api/v1/workflows/35" \
  -H "Authorization: Bearer YOUR_TOKEN"

# 创建工作流
curl -X POST "http://localhost:5678/api/v1/workflows" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d @workflow.json

# 更新工作流
curl -X PUT "http://localhost:5678/api/v1/workflows/35" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d @workflow.json
```

## 文件组织结构

### 推荐的仓库结构

```
workflows/
├── src/                          # 源工作流文件
│   ├── data-processing/
│   │   ├── user-sync.json
│   │   └── order-processing.json
│   ├── notifications/
│   │   ├── slack-alerts.json
│   │   └── email-reports.json
│   └── integrations/
│       ├── crm-sync.json
│       └── payment-webhook.json
├── templates/                    # 模板文件
│   ├── workflow.template.json
│   └── node-templates/
├── configs/                      # 环境配置
│   ├── development.env
│   ├── staging.env
│   └── production.env
├── tests/                        # 测试文件
│   ├── unit/
│   ├── integration/
│   └── fixtures/
└── scripts/                      # 自动化脚本
    ├── deploy.sh
    ├── validate.js
    └── transform.js
```

### 命名规范

#### 工作流文件命名
- 使用 kebab-case: `user-data-sync.json`
- 包含功能描述: `slack-notification-alerts.json`
- 版本标识: `payment-webhook-v2.json`

#### 节点命名
- 描述性名称: `Fetch User Data`, `Send Slack Alert`
- 避免默认名称: 不要使用 `HTTP Request`, `Slack`
- 保持一致性: 同类节点使用相似命名模式

## 数据验证和清理

### JSON Schema 验证

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["name", "nodes", "connections"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 128
    },
    "nodes": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "required": ["id", "name", "type", "typeVersion"],
        "properties": {
          "id": {
            "type": "string",
            "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"
          }
        }
      }
    }
  }
}
```

### 清理脚本示例

```javascript
// clean-workflow.js
function cleanWorkflowForVersionControl(workflow) {
  return {
    ...workflow,
    createdAt: null,
    updatedAt: null,
    versionId: null,
    triggerCount: 0,
    // 移除位置信息 (可选)
    nodes: workflow.nodes.map(node => {
      const { position, ...cleanNode } = node;
      return cleanNode;
    })
  };
}
```

## 安全考虑

### 敏感信息处理

1. **凭证 ID 映射**
   - 使用环境变量替换凭证 ID
   - 建立凭证名称到 ID 的映射表

2. **参数加密**
   - 识别敏感参数字段
   - 使用占位符替换实际值

3. **访问控制**
   - 限制工作流文件的访问权限
   - 使用 Git 钩子检查敏感信息

### 示例安全脚本

```bash
#!/bin/bash
# check-secrets.sh
# 检查工作流文件中的敏感信息

SECRETS_PATTERN="(password|token|key|secret|credential)"

find ./workflows -name "*.json" -exec grep -l -i "$SECRETS_PATTERN" {} \;

if [ $? -eq 0 ]; then
  echo "警告: 发现可能包含敏感信息的文件"
  exit 1
fi
```

## 下一步

理解了工作流的存储格式后，下一步将探讨如何将这些 JSON 文件有效地纳入 Git 版本控制系统，包括分支策略、合并冲突处理和环境配置管理。
