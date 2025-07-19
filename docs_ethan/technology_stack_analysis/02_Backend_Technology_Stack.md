# 02 - 后端技术栈深度分析

本文档深入分析 n8n 后端技术栈的架构设计、核心组件和技术选型，涵盖了从 Node.js 运行时到数据库集成的完整后端生态系统。

---

## 1. 后端架构概览

### 1.1 整体架构模式
n8n 后端采用 **分层架构 + 微服务思想** 的设计模式：

```
n8n 后端架构
├── API 层 (packages/cli)           # REST API + Web 服务器
├── 核心引擎 (packages/core)        # 工作流执行引擎
├── 数据模型 (packages/workflow)    # 工作流数据结构
├── 节点库 (packages/nodes-base)    # 业务逻辑节点
├── 数据层 (@n8n/db)               # 数据库抽象
└── 基础设施层 (@n8n/*)            # 配置、权限、DI 等
```

### 1.2 核心技术栈组件
- **运行时**: Node.js 22.16+
- **Web 框架**: Express.js 5.1.0
- **数据库**: SQLite/PostgreSQL/MySQL/MariaDB
- **ORM**: @n8n/typeorm 0.3.20-12 (TypeORM 定制版)
- **队列系统**: Bull 4.16.4 (Redis)
- **认证**: JWT + OAuth2 + SAML
- **缓存**: Redis + 内存缓存
- **监控**: Sentry + Prometheus

---

## 2. Node.js 运行环境

### 2.1 版本要求与特性

#### Node.js 版本支持
```json
{
  "engines": {
    "node": ">=22.16 <= 24.x"
  }
}
```

#### 关键 Node.js 特性使用
- **ES Modules**: 全面支持 ESM
- **Worker Threads**: 任务执行隔离
- **Async/Await**: 异步编程模式
- **Stream API**: 大文件处理
- **Cluster Mode**: 多进程支持

### 2.2 内存管理配置

#### V8 参数优化
```json
{
  "scripts": {
    "build": "NODE_OPTIONS=\"--max-old-space-size=8192\" vite build"
  }
}
```

#### 内存使用策略
- **Heap Size**: 8GB 最大堆内存
- **Worker Memory**: 1MB 空闲内存限制
- **Garbage Collection**: 增量 GC 优化

---

## 3. Express.js Web 框架

### 3.1 Express 应用架构

#### 应用初始化
```typescript
// packages/cli/src/server.ts
import express from 'express';
import helmet from 'helmet';
import compression from 'compression';

const app = express();

// 安全中间件
app.use(helmet());
app.use(compression());

// 请求解析
app.use(express.json({ limit: '16mb' }));
app.use(express.urlencoded({ extended: true }));

// 静态文件服务
app.use('/', express.static('dist'));
```

#### 路由组织
```typescript
// API 路由结构
app.use('/api/v1/workflows', workflowRoutes);
app.use('/api/v1/executions', executionRoutes);
app.use('/api/v1/credentials', credentialRoutes);
app.use('/api/v1/nodes', nodeRoutes);
app.use('/api/v1/users', userRoutes);
app.use('/webhook', webhookRoutes);
```

### 3.2 中间件系统

#### 认证中间件
```typescript
// JWT 认证中间件
export const authenticateJWT = (req: Request, res: Response, next: NextFunction) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ message: 'Access denied' });
  }
  
  try {
    const decoded = jwt.verify(token, config.jwt.secret);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(403).json({ message: 'Invalid token' });
  }
};
```

#### 错误处理中间件
```typescript
// 全局错误处理
app.use((error: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error('API Error:', error);
  
  if (error instanceof ValidationError) {
    return res.status(400).json({ message: error.message });
  }
  
  res.status(500).json({ message: 'Internal server error' });
});
```

### 3.3 API 设计模式

#### RESTful API 设计
```typescript
// 工作流 CRUD 操作
router.get('/workflows', getWorkflows);           // GET /api/v1/workflows
router.post('/workflows', createWorkflow);        // POST /api/v1/workflows
router.get('/workflows/:id', getWorkflow);        // GET /api/v1/workflows/:id
router.put('/workflows/:id', updateWorkflow);     // PUT /api/v1/workflows/:id
router.delete('/workflows/:id', deleteWorkflow);  // DELETE /api/v1/workflows/:id
```

#### OpenAPI 规范
```yaml
# OpenAPI 文档结构
openapi: 3.0.0
info:
  title: n8n API
  version: 1.103.0
paths:
  /workflows:
    get:
      summary: Get all workflows
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Workflow'
```

---

## 4. 数据库架构与 ORM

### 4.1 数据库支持

#### 支持的数据库类型
- **SQLite**: 默认开发环境数据库
- **PostgreSQL**: 推荐生产环境数据库
- **MySQL/MariaDB**: 企业环境支持

#### 数据库配置
```typescript
// 数据库连接配置
export const databaseConfig = {
  type: process.env.DB_TYPE as 'sqlite' | 'postgres' | 'mysql',
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME || 'n8n',
  username: process.env.DB_USERNAME || 'n8n',
  password: process.env.DB_PASSWORD || '',
  
  // SQLite 特定配置
  database: process.env.DB_SQLITE_PATH || './database.sqlite',
  
  // 连接池配置
  pool: {
    max: 10,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
};
```

### 4.2 TypeORM 集成

#### 实体定义
```typescript
// 工作流实体
@Entity('workflows')
export class WorkflowEntity {
  @PrimaryColumn()
  id: string;
  
  @Column()
  name: string;
  
  @Column('json')
  nodes: INode[];
  
  @Column('json')
  connections: IConnections;
  
  @Column({ default: false })
  active: boolean;
  
  @CreateDateColumn()
  createdAt: Date;
  
  @UpdateDateColumn()
  updatedAt: Date;
  
  @ManyToOne(() => User)
  owner: User;
  
  @OneToMany(() => ExecutionEntity, execution => execution.workflow)
  executions: ExecutionEntity[];
}
```

#### Repository 模式
```typescript
// 工作流仓库
@Injectable()
export class WorkflowRepository {
  constructor(
    @InjectRepository(WorkflowEntity)
    private repository: Repository<WorkflowEntity>
  ) {}
  
  async findAllByOwner(ownerId: string): Promise<WorkflowEntity[]> {
    return this.repository.find({
      where: { owner: { id: ownerId } },
      relations: ['owner']
    });
  }
  
  async createWorkflow(data: CreateWorkflowDto): Promise<WorkflowEntity> {
    const workflow = this.repository.create(data);
    return this.repository.save(workflow);
  }
}
```

### 4.3 数据库迁移

#### 迁移文件结构
```typescript
// 迁移示例
export class CreateWorkflowTable1234567890123 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'workflows',
        columns: [
          {
            name: 'id',
            type: 'varchar',
            isPrimary: true
          },
          {
            name: 'name',
            type: 'varchar',
            length: '255'
          },
          {
            name: 'nodes',
            type: 'text'
          }
        ]
      })
    );
  }
  
  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('workflows');
  }
}
```

---

## 5. 队列系统 (Bull)

### 5.1 Bull 队列配置

#### 队列初始化
```typescript
import Bull from 'bull';

// 主执行队列
export const executionQueue = new Bull('execution', {
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
    password: process.env.REDIS_PASSWORD
  },
  
  defaultJobOptions: {
    removeOnComplete: 100,
    removeOnFail: 50,
    attempts: 3,
    backoff: 'exponential'
  }
});

// Webhook 队列
export const webhookQueue = new Bull('webhook', redisConfig);
```

#### 任务处理器
```typescript
// 工作流执行任务处理
executionQueue.process('workflow-execution', async (job) => {
  const { workflowId, inputData, userId } = job.data;
  
  try {
    const workflow = await getWorkflow(workflowId);
    const execution = new WorkflowRunner();
    
    const result = await execution.run(workflow, inputData);
    
    await saveExecutionResult(result);
    
    return result;
  } catch (error) {
    logger.error('Workflow execution failed:', error);
    throw error;
  }
});
```

### 5.2 任务调度策略

#### 优先级队列
```typescript
// 不同优先级的任务
await executionQueue.add('workflow-execution', data, {
  priority: 10 // 高优先级
});

await executionQueue.add('workflow-execution', data, {
  priority: 5  // 普通优先级
});
```

#### 延迟执行
```typescript
// 定时执行任务
await executionQueue.add('scheduled-workflow', data, {
  delay: 60000, // 1分钟后执行
  repeat: { cron: '0 9 * * *' } // 每天9点执行
});
```

---

## 6. 认证与授权系统

### 6.1 JWT 认证

#### JWT 配置
```typescript
// JWT 配置
export const jwtConfig = {
  secret: process.env.JWT_SECRET || 'n8n-secret',
  expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  issuer: 'n8n',
  audience: 'n8n-users'
};

// Token 生成
export const generateToken = (user: User): string => {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    jwtConfig.secret,
    {
      expiresIn: jwtConfig.expiresIn,
      issuer: jwtConfig.issuer,
      audience: jwtConfig.audience
    }
  );
};
```

### 6.2 OAuth2 集成

#### OAuth2 Provider 支持
```typescript
// Google OAuth2 配置
export const googleOAuth2Config = {
  clientId: process.env.GOOGLE_CLIENT_ID,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET,
  redirectUri: process.env.GOOGLE_REDIRECT_URI,
  scope: ['email', 'profile']
};

// OAuth2 处理器
router.get('/auth/google/callback', async (req, res) => {
  const { code } = req.query;
  
  try {
    const tokens = await exchangeCodeForTokens(code);
    const userInfo = await getUserInfo(tokens.access_token);
    
    let user = await findUserByEmail(userInfo.email);
    if (!user) {
      user = await createUser({
        email: userInfo.email,
        firstName: userInfo.given_name,
        lastName: userInfo.family_name
      });
    }
    
    const jwt = generateToken(user);
    res.redirect(`/login?token=${jwt}`);
  } catch (error) {
    res.redirect('/login?error=oauth_failed');
  }
});
```

### 6.3 RBAC 权限系统

#### 角色定义
```typescript
// 角色枚举
export enum Role {
  ADMIN = 'admin',
  OWNER = 'owner', 
  MEMBER = 'member',
  VIEWER = 'viewer'
}

// 权限检查中间件
export const requireRole = (requiredRole: Role) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user?.role;
    
    if (!hasPermission(userRole, requiredRole)) {
      return res.status(403).json({ message: 'Insufficient permissions' });
    }
    
    next();
  };
};
```

---

## 7. 缓存系统

### 7.1 Redis 缓存

#### Redis 配置
```typescript
import Redis from 'ioredis';

// Redis 客户端
export const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  
  retryDelayOnFailover: 100,
  maxRetriesPerRequest: 3,
  lazyConnect: true
});

// 缓存装饰器
export function Cache(ttl: number = 300) {
  return function (target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const method = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      const cacheKey = `${target.constructor.name}:${propertyName}:${JSON.stringify(args)}`;
      
      const cached = await redis.get(cacheKey);
      if (cached) {
        return JSON.parse(cached);
      }
      
      const result = await method.apply(this, args);
      await redis.setex(cacheKey, ttl, JSON.stringify(result));
      
      return result;
    };
  };
}
```

### 7.2 内存缓存

#### Cache Manager 配置
```typescript
import cacheManager from 'cache-manager';

// 多级缓存
export const cache = cacheManager.multiCaching([
  cacheManager.caching({ store: 'memory', max: 100, ttl: 60 }),
  cacheManager.caching({ store: 'redis', ...redisConfig })
]);

// 缓存服务
@Injectable()
export class CacheService {
  async get<T>(key: string): Promise<T | null> {
    return cache.get(key);
  }
  
  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    return cache.set(key, value, { ttl });
  }
  
  async del(key: string): Promise<void> {
    return cache.del(key);
  }
}
```

---

## 8. 工作流执行引擎

### 8.1 执行引擎架构

#### 核心执行类
```typescript
// packages/core/src/workflow-execute.ts
export class WorkflowExecute {
  private workflow: Workflow;
  private runData: IRunData = {};
  
  constructor(
    additionalData: IWorkflowExecuteAdditionalData,
    mode: WorkflowExecuteMode
  ) {
    this.additionalData = additionalData;
    this.mode = mode;
  }
  
  async run(
    workflow: IWorkflowBase,
    startNodes?: INode[],
    destinationNode?: string
  ): Promise<IRun> {
    
    this.workflow = new Workflow({
      id: workflow.id,
      name: workflow.name,
      nodes: workflow.nodes,
      connections: workflow.connections,
      active: workflow.active,
      nodeTypes: this.nodeTypes,
      staticData: workflow.staticData,
      settings: workflow.settings
    });
    
    const executionData = await this.executeWorkflow(startNodes, destinationNode);
    
    return {
      data: executionData,
      mode: this.mode,
      startedAt: new Date(),
      stoppedAt: new Date()
    };
  }
}
```

### 8.2 节点执行机制

#### 节点执行上下文
```typescript
// 节点执行函数
export async function executeNode(
  workflow: Workflow,
  node: INode,
  inputData: INodeExecutionData[],
  runIndex: number,
  additionalData: IWorkflowExecuteAdditionalData
): Promise<INodeExecutionData[][]> {
  
  const nodeType = workflow.nodeTypes.getByNameAndVersion(node.type, node.typeVersion);
  
  // 构建执行上下文
  const context: IExecuteFunction = {
    getNodeParameter: (parameterName: string) => {
      return getNodeParameter(workflow, runIndex, 'main', node, parameterName);
    },
    
    getInputData: () => inputData,
    
    helpers: {
      request: makeRequest,
      requestOAuth1: requestOAuth1,
      requestOAuth2: requestOAuth2
    }
  };
  
  // 执行节点
  try {
    const result = await nodeType.execute.call(context);
    return result;
  } catch (error) {
    throw new NodeOperationError(node, error);
  }
}
```

---

## 9. 监控与日志系统

### 9.1 Winston 日志

#### 日志配置
```typescript
import winston from 'winston';

// 日志配置
export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error'
    }),
    
    new winston.transports.File({
      filename: 'logs/combined.log'
    })
  ]
});
```

### 9.2 Sentry 错误监控

#### Sentry 集成
```typescript
import * as Sentry from '@sentry/node';

// Sentry 初始化
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  
  beforeSend(event) {
    // 过滤敏感信息
    if (event.extra?.credentials) {
      delete event.extra.credentials;
    }
    return event;
  }
});

// 错误捕获中间件
app.use(Sentry.Handlers.errorHandler());
```

### 9.3 Prometheus 指标

#### 指标收集
```typescript
import promClient from 'prom-client';

// 自定义指标
const workflowExecutions = new promClient.Counter({
  name: 'n8n_workflow_executions_total',
  help: 'Total number of workflow executions',
  labelNames: ['status', 'workflow_id']
});

const executionDuration = new promClient.Histogram({
  name: 'n8n_workflow_execution_duration_seconds',
  help: 'Workflow execution duration',
  labelNames: ['workflow_id']
});

// 指标端点
app.get('/metrics', (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.send(promClient.register.metrics());
});
```

---

## 10. 安全特性

### 10.1 输入验证

#### Zod 验证器
```typescript
import { z } from 'zod';

// 工作流创建验证
const createWorkflowSchema = z.object({
  name: z.string().min(1).max(255),
  nodes: z.array(z.object({
    id: z.string(),
    type: z.string(),
    parameters: z.record(z.any())
  })),
  connections: z.record(z.any()).optional(),
  active: z.boolean().default(false)
});

// 验证中间件
export const validateBody = (schema: z.ZodSchema) => {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      res.status(400).json({ message: 'Validation failed', errors: error.errors });
    }
  };
};
```

### 10.2 SQL 注入防护

#### 参数化查询
```typescript
// 安全的数据库查询
export async function getWorkflowsByOwner(ownerId: string) {
  return this.repository
    .createQueryBuilder('workflow')
    .where('workflow.ownerId = :ownerId', { ownerId })
    .getMany();
}
```

### 10.3 CSRF 防护

#### CSRF Token
```typescript
import csrf from 'csurf';

// CSRF 保护
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict'
  }
});

app.use(csrfProtection);
```

---

## 11. 配置管理

### 11.1 环境配置

#### Convict 配置
```typescript
import convict from 'convict';

// 配置模式
const config = convict({
  env: {
    doc: 'The application environment',
    format: ['production', 'development', 'test'],
    default: 'development',
    env: 'NODE_ENV'
  },
  
  port: {
    doc: 'The port to bind',
    format: 'port',
    default: 5678,
    env: 'N8N_PORT'
  },
  
  database: {
    type: {
      doc: 'Database type',
      format: ['sqlite', 'postgres', 'mysql'],
      default: 'sqlite',
      env: 'DB_TYPE'
    }
  }
});

// 验证配置
config.validate({ allowed: 'strict' });
```

---

## 12. 性能优化

### 12.1 连接池优化

#### 数据库连接池
```typescript
// 连接池配置
const poolConfig = {
  max: parseInt(process.env.DB_POOL_MAX || '10'),
  min: parseInt(process.env.DB_POOL_MIN || '0'),
  acquire: parseInt(process.env.DB_POOL_ACQUIRE || '30000'),
  idle: parseInt(process.env.DB_POOL_IDLE || '10000')
};
```

### 12.2 Worker Threads

#### 隔离执行
```typescript
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

// 工作流执行 Worker
if (isMainThread) {
  // 主线程
  export function executeWorkflowInWorker(workflowData: any) {
    return new Promise((resolve, reject) => {
      const worker = new Worker(__filename, {
        workerData: workflowData
      });
      
      worker.on('message', resolve);
      worker.on('error', reject);
      worker.on('exit', (code) => {
        if (code !== 0) {
          reject(new Error(`Worker stopped with exit code ${code}`));
        }
      });
    });
  }
} else {
  // Worker 线程
  const result = executeWorkflow(workerData);
  parentPort?.postMessage(result);
}
```

---

## 13. 总结

### 13.1 技术栈优势

1. **现代化**: Node.js 22+ 提供最新的 JavaScript 特性
2. **类型安全**: 全面的 TypeScript 支持确保代码质量
3. **可扩展**: 模块化架构支持水平扩展
4. **企业级**: 完善的认证、授权和监控系统
5. **高性能**: 队列系统和缓存优化执行效率

### 13.2 架构亮点

1. **分层设计**: 清晰的职责分离和模块边界
2. **插件化**: 支持自定义节点和扩展
3. **多数据库**: 灵活的数据库选择和迁移
4. **安全性**: 多层次的安全防护机制
5. **监控性**: 全面的日志和指标收集

### 13.3 扩展能力

1. **水平扩展**: 支持多实例部署
2. **功能扩展**: 插件化节点系统
3. **集成扩展**: 丰富的第三方服务集成
4. **自定义扩展**: 企业级定制能力

n8n 的后端技术栈展现了现代 Node.js 应用的最佳实践，为构建可扩展的工作流自动化平台提供了强大的技术基础。