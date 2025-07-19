# 04 - 测试框架与质量保证工具分析

本文档深入分析 n8n 项目的测试框架体系、质量保证工具和最佳实践，涵盖从单元测试到端到端测试的完整测试策略。

---

## 1. 测试框架架构概览

### 1.1 测试金字塔
n8n 采用标准的测试金字塔策略：

```
n8n 测试金字塔
├── E2E 测试 (Cypress + Playwright)    # 端到端集成测试
├── 集成测试 (Jest + Supertest)        # API 和组件集成测试  
├── 单元测试 (Jest + Vitest)          # 函数和类单元测试
└── 静态分析 (TypeScript + ESLint)    # 编译时质量检查
```

### 1.2 测试技术栈组件
- **单元测试**: Jest 29.6.2 (后端) + Vitest 3.1.3 (前端)
- **E2E 测试**: Cypress + Playwright
- **API 测试**: Supertest 7.1.1
- **覆盖率**: Jest Coverage + V8 Provider
- **Mock 工具**: Jest Mock + Mock Service Worker
- **测试工具**: @testing-library/vue + @vue/test-utils

---

## 2. Jest 单元测试框架

### 2.1 Jest 配置架构

#### 全局 Jest 配置 (jest.config.js)
```javascript
const { pathsToModuleNameMapper } = require('ts-jest');
const { compilerOptions } = require('get-tsconfig').getTsconfig().config;

const tsJestOptions = {
  isolatedModules: true,
  tsconfig: {
    ...compilerOptions,
    declaration: false,
    sourceMap: true,
  },
};

const config = {
  verbose: true,
  testEnvironment: 'node',
  testRegex: '\\.(test|spec)\\.(js|ts)$',
  testPathIgnorePatterns: ['/dist/', '/node_modules/'],
  
  transform: {
    '^.+\\.ts$': ['ts-jest', tsJestOptions],
  },
  
  setupFilesAfterEnv: ['jest-expect-message'],
  collectCoverage: process.env.COVERAGE_ENABLED === 'true',
  coverageReporters: ['text-summary', 'lcov', 'html-spa'],
  workerIdleMemoryLimit: '1MB',
};
```

#### ESM 依赖处理
```javascript
// ESM 包特殊处理
const esmDependencies = [
  'pdfjs-dist',
  'openid-client', 
  'oauth4webapi',
  'jose'
];

const esmDependenciesPattern = esmDependencies.join('|');
const esmDependenciesRegex = `node_modules/(${esmDependenciesPattern})/.+\\.m?js$`;

config.transform[esmDependenciesRegex] = [
  'babel-jest',
  {
    presets: ['@babel/preset-env'],
    plugins: ['babel-plugin-transform-import-meta'],
  },
];

config.transformIgnorePatterns = [`/node_modules/(?!${esmDependenciesPattern})/`];
```

### 2.2 测试模式与环境

#### 不同数据库的测试配置
```javascript
// 数据库特定测试
const testScripts = {
  "test:sqlite": "N8N_LOG_LEVEL=silent DB_TYPE=sqlite jest",
  "test:postgres": "N8N_LOG_LEVEL=silent DB_TYPE=postgresdb DB_POSTGRESDB_SCHEMA=alt_schema DB_TABLE_PREFIX=test_ jest --no-coverage",
  "test:mariadb": "N8N_LOG_LEVEL=silent DB_TYPE=mariadb DB_TABLE_PREFIX=test_ jest --no-coverage", 
  "test:mysql": "N8N_LOG_LEVEL=silent DB_TYPE=mysqldb DB_TABLE_PREFIX=test_ jest --no-coverage"
};
```

#### CI 环境配置
```javascript
// CI 特定配置
if (process.env.CI === 'true') {
  config.collectCoverageFrom = ['src/**/*.ts'];
  config.reporters = ['default', 'jest-junit'];
  config.coverageReporters = ['cobertura'];
}
```

### 2.3 单元测试实践

#### 工作流测试示例
```typescript
// packages/workflow/test/workflow.test.ts
import { Workflow } from '../src/workflow';
import { INode, IConnections } from '../src/interfaces';

describe('Workflow', () => {
  let workflow: Workflow;
  
  beforeEach(() => {
    const nodes: INode[] = [
      {
        id: 'start',
        name: 'Start',
        type: 'n8n-nodes-base.start',
        typeVersion: 1,
        parameters: {},
        position: [0, 0]
      }
    ];
    
    const connections: IConnections = {};
    
    workflow = new Workflow({
      id: 'test',
      name: 'Test Workflow',
      nodes,
      connections,
      active: false,
      nodeTypes: mockNodeTypes,
      staticData: {}
    });
  });
  
  it('should create workflow instance', () => {
    expect(workflow).toBeInstanceOf(Workflow);
    expect(workflow.name).toBe('Test Workflow');
  });
  
  it('should validate workflow structure', () => {
    const issues = workflow.checkReadyForExecution();
    expect(issues).toEqual(null);
  });
  
  it('should handle node execution order', () => {
    const executionOrder = workflow.getExecutionOrder();
    expect(executionOrder).toContain('start');
  });
});
```

#### Mock 工具使用
```typescript
// 数据库 Mock
jest.mock('@n8n/db', () => ({
  getRepository: jest.fn(() => ({
    find: jest.fn(),
    save: jest.fn(),
    delete: jest.fn()
  }))
}));

// HTTP 请求 Mock
import nock from 'nock';

describe('HTTP Request Node', () => {
  beforeEach(() => {
    nock.cleanAll();
  });
  
  it('should make HTTP GET request', async () => {
    nock('https://api.example.com')
      .get('/users')
      .reply(200, { users: [] });
      
    const result = await executeNode(httpRequestNode, inputData);
    expect(result[0][0].json).toEqual({ users: [] });
  });
});
```

---

## 3. Vitest 前端测试

### 3.1 Vitest 配置

#### Vitest 工作区配置
```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config';

export default defineWorkspace([
  // 前端包配置
  'packages/frontend/*/vitest.config.ts',
  
  // 通用包配置
  {
    test: {
      name: 'workflow',
      root: './packages/workflow'
    }
  }
]);
```

#### 前端包 Vitest 配置
```typescript
// packages/frontend/editor-ui/vitest.config.ts
import { defineConfig } from 'vitest/config';
import { resolve } from 'path';

export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/tests/setup.ts'],
    globals: true,
    
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/tests/',
        '**/*.d.ts'
      ]
    }
  },
  
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
      '~': resolve(__dirname)
    }
  }
});
```

### 3.2 Vue 组件测试

#### 测试环境设置
```typescript
// src/tests/setup.ts
import { config } from '@vue/test-utils';
import { createTestingPinia } from '@pinia/testing';
import { i18n } from '@/plugins/i18n';

// 全局测试配置
config.global.plugins = [
  createTestingPinia({
    createSpy: vi.fn
  }),
  i18n
];

// Mock 全局组件
config.global.components = {
  'router-link': {
    template: '<a><slot /></a>'
  },
  'router-view': {
    template: '<div><slot /></div>'
  }
};
```

#### 组件测试示例
```typescript
// src/components/__tests__/NodeCreator.test.ts
import { mount } from '@vue/test-utils';
import { describe, it, expect, vi } from 'vitest';
import { createTestingPinia } from '@pinia/testing';
import NodeCreator from '../NodeCreator.vue';

describe('NodeCreator', () => {
  const createComponent = (props = {}) => {
    return mount(NodeCreator, {
      props,
      global: {
        plugins: [
          createTestingPinia({
            initialState: {
              nodeTypes: {
                allNodeTypes: mockNodeTypes
              }
            }
          })
        ]
      }
    });
  };
  
  it('should render node categories', () => {
    const wrapper = createComponent();
    expect(wrapper.find('[data-test="node-categories"]').exists()).toBe(true);
  });
  
  it('should filter nodes by search term', async () => {
    const wrapper = createComponent();
    const searchInput = wrapper.find('[data-test="search-input"]');
    
    await searchInput.setValue('HTTP');
    
    const nodeItems = wrapper.findAll('[data-test="node-item"]');
    expect(nodeItems.length).toBeGreaterThan(0);
    expect(nodeItems[0].text()).toContain('HTTP');
  });
  
  it('should emit node selection event', async () => {
    const wrapper = createComponent();
    const nodeItem = wrapper.find('[data-test="node-item-http"]');
    
    await nodeItem.trigger('click');
    
    expect(wrapper.emitted('node-selected')).toBeTruthy();
    expect(wrapper.emitted('node-selected')?.[0]).toEqual([
      expect.objectContaining({ type: 'n8n-nodes-base.httpRequest' })
    ]);
  });
});
```

#### Pinia Store 测试
```typescript
// src/stores/__tests__/workflow.store.test.ts
import { setActivePinia, createPinia } from 'pinia';
import { describe, it, expect, beforeEach } from 'vitest';
import { useWorkflowStore } from '../workflow.store';

describe('Workflow Store', () => {
  beforeEach(() => {
    setActivePinia(createPinia());
  });
  
  it('should initialize with empty workflow', () => {
    const store = useWorkflowStore();
    
    expect(store.workflow).toEqual({});
    expect(store.isExecuting).toBe(false);
  });
  
  it('should update workflow name', () => {
    const store = useWorkflowStore();
    
    store.setWorkflowName('Test Workflow');
    
    expect(store.workflow.name).toBe('Test Workflow');
  });
  
  it('should handle workflow execution', async () => {
    const store = useWorkflowStore();
    store.workflow = mockWorkflow;
    
    const promise = store.executeWorkflow();
    
    expect(store.isExecuting).toBe(true);
    
    await promise;
    
    expect(store.isExecuting).toBe(false);
    expect(store.lastExecution).toBeDefined();
  });
});
```

---

## 4. Cypress E2E 测试

### 4.1 Cypress 配置

#### 主配置文件
```javascript
// cypress.config.js
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:8080',
    viewportWidth: 1280,
    viewportHeight: 720,
    
    video: false,
    screenshotOnRunFailure: true,
    
    setupNodeEvents(on, config) {
      // 任务注册
      on('task', {
        resetDatabase: () => resetTestDatabase(),
        seedDatabase: (data) => seedTestData(data),
        log: (message) => {
          console.log(message);
          return null;
        }
      });
      
      // 插件注册
      require('@cypress/code-coverage/task')(on, config);
      
      return config;
    },
    
    env: {
      codeCoverage: {
        exclude: 'cypress/**/*.*'
      }
    }
  }
});
```

#### 支持文件配置
```typescript
// cypress/support/e2e.ts
import './commands';
import '@cypress/code-coverage/support';

// 全局配置
Cypress.on('uncaught:exception', (err, runnable) => {
  // 防止某些预期的错误导致测试失败
  if (err.message.includes('ResizeObserver loop limit exceeded')) {
    return false;
  }
  return true;
});

// 全局钩子
beforeEach(() => {
  // 清理浏览器状态
  cy.clearCookies();
  cy.clearLocalStorage();
});
```

### 4.2 自定义命令

#### 认证命令
```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(email: string, password: string): Chainable<void>;
      createWorkflow(workflow: Partial<IWorkflow>): Chainable<void>;
      addNodeToWorkflow(nodeType: string): Chainable<void>;
    }
  }
}

Cypress.Commands.add('login', (email: string, password: string) => {
  cy.visit('/signin');
  cy.get('[data-test="email-input"]').type(email);
  cy.get('[data-test="password-input"]').type(password);
  cy.get('[data-test="signin-button"]').click();
  
  // 等待登录完成
  cy.url().should('not.include', '/signin');
  cy.get('[data-test="main-sidebar"]').should('be.visible');
});

Cypress.Commands.add('createWorkflow', (workflow) => {
  cy.get('[data-test="new-workflow-button"]').click();
  
  if (workflow.name) {
    cy.get('[data-test="workflow-name-input"]').clear().type(workflow.name);
  }
  
  cy.get('[data-test="save-workflow-button"]').click();
  cy.get('[data-test="success-toast"]').should('be.visible');
});

Cypress.Commands.add('addNodeToWorkflow', (nodeType) => {
  cy.get('[data-test="add-node-button"]').click();
  cy.get('[data-test="node-creator-search"]').type(nodeType);
  cy.get(`[data-test="node-item-${nodeType}"]`).click();
  
  // 等待节点添加到画布
  cy.get(`[data-test="node-${nodeType}"]`).should('be.visible');
});
```

### 4.3 E2E 测试用例

#### 工作流创建测试
```typescript
// cypress/e2e/workflow-creation.cy.ts
describe('Workflow Creation', () => {
  beforeEach(() => {
    cy.task('resetDatabase');
    cy.login('test@example.com', 'password');
  });
  
  it('should create a new workflow', () => {
    cy.visit('/');
    
    // 创建新工作流
    cy.createWorkflow({ name: 'Test Workflow' });
    
    // 验证工作流已创建
    cy.url().should('include', '/workflow/');
    cy.get('[data-test="workflow-name"]').should('contain', 'Test Workflow');
  });
  
  it('should add nodes to workflow', () => {
    cy.createWorkflow({ name: 'HTTP Workflow' });
    
    // 添加手动触发器
    cy.addNodeToWorkflow('manual-trigger');
    
    // 添加 HTTP 请求节点
    cy.addNodeToWorkflow('http-request');
    
    // 配置 HTTP 节点
    cy.get('[data-test="node-http-request"]').click();
    cy.get('[data-test="parameter-url"]').type('https://api.github.com/users/n8n-io');
    cy.get('[data-test="parameter-method"]').select('GET');
    
    // 连接节点
    cy.get('[data-test="output-handle-manual-trigger"]')
      .drag('[data-test="input-handle-http-request"]');
    
    // 保存工作流
    cy.get('[data-test="save-workflow"]').click();
    cy.get('[data-test="success-message"]').should('be.visible');
  });
  
  it('should execute workflow', () => {
    cy.createWorkflow({ name: 'Execution Test' });
    cy.addNodeToWorkflow('manual-trigger');
    
    // 执行工作流
    cy.get('[data-test="execute-workflow"]').click();
    
    // 等待执行完成
    cy.get('[data-test="execution-status"]', { timeout: 10000 })
      .should('contain', 'Success');
      
    // 检查执行结果
    cy.get('[data-test="execution-data"]').should('be.visible');
  });
});
```

#### 凭证管理测试
```typescript
// cypress/e2e/credentials.cy.ts
describe('Credentials Management', () => {
  beforeEach(() => {
    cy.task('resetDatabase');
    cy.login('admin@example.com', 'password');
  });
  
  it('should create new credentials', () => {
    cy.visit('/credentials');
    
    cy.get('[data-test="new-credential"]').click();
    cy.get('[data-test="credential-type-select"]').select('HTTP Basic Auth');
    
    cy.get('[data-test="credential-name"]').type('Test API Credentials');
    cy.get('[data-test="username-input"]').type('testuser');
    cy.get('[data-test="password-input"]').type('testpass');
    
    cy.get('[data-test="save-credential"]').click();
    
    // 验证凭证已保存
    cy.get('[data-test="credential-list"]')
      .should('contain', 'Test API Credentials');
  });
  
  it('should test credential connection', () => {
    // 创建可测试的凭证
    cy.task('seedDatabase', {
      credentials: [{
        name: 'Test HTTP Credential',
        type: 'httpBasicAuth',
        data: { username: 'test', password: 'test' }
      }]
    });
    
    cy.visit('/credentials');
    cy.get('[data-test="credential-Test HTTP Credential"]').click();
    
    // 测试连接
    cy.get('[data-test="test-connection"]').click();
    cy.get('[data-test="connection-status"]', { timeout: 5000 })
      .should('contain', 'Connection successful');
  });
});
```

---

## 5. Playwright 现代 E2E 测试

### 5.1 Playwright 配置

#### 配置文件
```typescript
// packages/testing/playwright/playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30000,
  expect: { timeout: 5000 },
  
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }]
  ],
  
  use: {
    baseURL: 'http://localhost:8080',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },
  
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 5'] }
    }
  ],
  
  webServer: {
    command: 'pnpm start',
    port: 8080,
    reuseExistingServer: !process.env.CI
  }
});
```

### 5.2 Playwright 测试示例

#### 页面对象模型
```typescript
// tests/pages/workflow-editor.page.ts
import { Page, Locator } from '@playwright/test';

export class WorkflowEditorPage {
  readonly page: Page;
  readonly workflowName: Locator;
  readonly addNodeButton: Locator;
  readonly saveButton: Locator;
  readonly executeButton: Locator;
  
  constructor(page: Page) {
    this.page = page;
    this.workflowName = page.getByTestId('workflow-name');
    this.addNodeButton = page.getByTestId('add-node');
    this.saveButton = page.getByTestId('save-workflow');
    this.executeButton = page.getByTestId('execute-workflow');
  }
  
  async goto() {
    await this.page.goto('/workflow/new');
  }
  
  async setWorkflowName(name: string) {
    await this.workflowName.fill(name);
  }
  
  async addNode(nodeType: string) {
    await this.addNodeButton.click();
    await this.page.getByTestId(`node-${nodeType}`).click();
  }
  
  async saveWorkflow() {
    await this.saveButton.click();
    await this.page.waitForSelector('[data-test="success-toast"]');
  }
  
  async executeWorkflow() {
    await this.executeButton.click();
    return this.page.waitForSelector('[data-test="execution-completed"]');
  }
}
```

#### 测试用例实现
```typescript
// tests/workflow-editor.spec.ts
import { test, expect } from '@playwright/test';
import { WorkflowEditorPage } from './pages/workflow-editor.page';

test.describe('Workflow Editor', () => {
  let workflowEditor: WorkflowEditorPage;
  
  test.beforeEach(async ({ page }) => {
    workflowEditor = new WorkflowEditorPage(page);
    
    // 登录
    await page.goto('/signin');
    await page.getByTestId('email').fill('test@example.com');
    await page.getByTestId('password').fill('password');
    await page.getByTestId('signin-button').click();
    
    await workflowEditor.goto();
  });
  
  test('should create and save workflow', async () => {
    await workflowEditor.setWorkflowName('Playwright Test Workflow');
    await workflowEditor.addNode('manual-trigger');
    await workflowEditor.addNode('set-data');
    
    await workflowEditor.saveWorkflow();
    
    await expect(workflowEditor.workflowName).toHaveValue('Playwright Test Workflow');
  });
  
  test('should execute workflow and show results', async () => {
    await workflowEditor.setWorkflowName('Execution Test');
    await workflowEditor.addNode('manual-trigger');
    
    const executionPromise = workflowEditor.executeWorkflow();
    
    await expect(executionPromise).resolves.toBeTruthy();
    
    // 验证执行结果
    await expect(page.getByTestId('execution-data')).toBeVisible();
  });
});
```

---

## 6. API 集成测试

### 6.1 Supertest API 测试

#### API 测试基础设置
```typescript
// packages/cli/test/integration/api.test.ts
import request from 'supertest';
import express from 'express';
import { setupTestApp } from '../utils/test-setup';

describe('API Endpoints', () => {
  let app: express.Application;
  let authToken: string;
  
  beforeAll(async () => {
    app = await setupTestApp();
    
    // 获取认证 token
    const loginResponse = await request(app)
      .post('/api/v1/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password'
      });
      
    authToken = loginResponse.body.token;
  });
  
  describe('Workflows API', () => {
    it('should create workflow', async () => {
      const workflowData = {
        name: 'Test Workflow',
        nodes: [
          {
            id: 'start',
            type: 'n8n-nodes-base.start',
            parameters: {},
            position: [0, 0]
          }
        ],
        connections: {}
      };
      
      const response = await request(app)
        .post('/api/v1/workflows')
        .set('Authorization', `Bearer ${authToken}`)
        .send(workflowData)
        .expect(201);
        
      expect(response.body).toMatchObject({
        id: expect.any(String),
        name: 'Test Workflow',
        active: false
      });
    });
    
    it('should get workflow by id', async () => {
      // 先创建工作流
      const createResponse = await request(app)
        .post('/api/v1/workflows')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Get Test Workflow' });
        
      const workflowId = createResponse.body.id;
      
      // 获取工作流
      const response = await request(app)
        .get(`/api/v1/workflows/${workflowId}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
        
      expect(response.body.name).toBe('Get Test Workflow');
    });
    
    it('should execute workflow', async () => {
      const workflowData = {
        name: 'Execution Test',
        nodes: [
          {
            id: 'manual',
            type: 'n8n-nodes-base.manualTrigger',
            parameters: {},
            position: [0, 0]
          }
        ],
        connections: {}
      };
      
      const createResponse = await request(app)
        .post('/api/v1/workflows')
        .set('Authorization', `Bearer ${authToken}`)
        .send(workflowData);
        
      const workflowId = createResponse.body.id;
      
      const executeResponse = await request(app)
        .post(`/api/v1/workflows/${workflowId}/execute`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
        
      expect(executeResponse.body).toMatchObject({
        data: expect.any(Object),
        finished: true
      });
    });
  });
});
```

### 6.2 数据库集成测试

#### 数据库测试工具
```typescript
// packages/cli/test/utils/test-db.ts
import { DataSource } from 'typeorm';
import { entities } from '@n8n/db';

export class TestDatabase {
  private dataSource: DataSource;
  
  constructor() {
    this.dataSource = new DataSource({
      type: 'sqlite',
      database: ':memory:',
      entities,
      synchronize: true,
      logging: false
    });
  }
  
  async initialize(): Promise<void> {
    await this.dataSource.initialize();
  }
  
  async destroy(): Promise<void> {
    await this.dataSource.destroy();
  }
  
  async reset(): Promise<void> {
    const entities = this.dataSource.entityMetadatas;
    
    for (const entity of entities) {
      await this.dataSource.query(`DELETE FROM ${entity.tableName}`);
    }
  }
  
  async seed(data: any): Promise<void> {
    // 种子数据插入逻辑
    for (const [entityName, records] of Object.entries(data)) {
      const repository = this.dataSource.getRepository(entityName);
      await repository.save(records);
    }
  }
}
```

---

## 7. 代码覆盖率和质量指标

### 7.1 覆盖率配置

#### Jest 覆盖率配置
```javascript
// 覆盖率阈值设置
const coverageThreshold = {
  global: {
    branches: 80,
    functions: 80,
    lines: 80,
    statements: 80
  },
  
  // 包级别阈值
  './packages/core/': {
    branches: 90,
    functions: 90,
    lines: 90,
    statements: 90
  },
  
  './packages/workflow/': {
    branches: 85,
    functions: 85,
    lines: 85,
    statements: 85
  }
};

// 覆盖率收集配置
const collectCoverageFrom = [
  'src/**/*.ts',
  '!src/**/*.d.ts',
  '!src/**/*.test.ts',
  '!src/**/*.spec.ts',
  '!src/test/**/*'
];
```

#### Vitest 覆盖率配置
```typescript
// vitest 覆盖率配置
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      reportsDirectory: './coverage',
      
      thresholds: {
        lines: 80,
        branches: 80,
        functions: 80,
        statements: 80
      },
      
      exclude: [
        'coverage/**',
        'dist/**',
        'packages/*/test{,s}/**',
        '**/*.d.ts',
        'cypress/**',
        'test{,s}/**',
        'test{,-*}.{js,cjs,mjs,ts,tsx,jsx}',
        '**/*{.,-}test.{js,cjs,mjs,ts,tsx,jsx}'
      ]
    }
  }
});
```

### 7.2 质量指标监控

#### SonarQube 集成
```yaml
# sonar-project.properties
sonar.projectKey=n8n
sonar.organization=n8n-io
sonar.sources=packages/
sonar.exclusions=**/node_modules/**,**/dist/**,**/*.test.ts

# 覆盖率报告
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.typescript.lcov.reportPaths=coverage/lcov.info

# 质量门禁
sonar.qualitygate.wait=true
```

#### 代码质量指标
```typescript
// 质量指标定义
export interface QualityMetrics {
  codeCoverage: {
    lines: number;
    branches: number;
    functions: number;
    statements: number;
  };
  
  complexity: {
    cyclomatic: number;
    cognitive: number;
  };
  
  maintainability: {
    technicalDebt: string;
    reliability: 'A' | 'B' | 'C' | 'D' | 'E';
    security: 'A' | 'B' | 'C' | 'D' | 'E';
  };
  
  duplication: {
    lines: number;
    blocks: number;
    files: number;
  };
}
```

---

## 8. 性能测试

### 8.1 性能测试框架

#### K6 负载测试
```javascript
// tests/performance/workflow-execution.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 10 },  // 预热
    { duration: '5m', target: 50 },  // 负载增加
    { duration: '10m', target: 100 }, // 稳定负载
    { duration: '2m', target: 0 },   // 降级
  ],
  
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% 请求在 500ms 内
    http_req_failed: ['rate<0.02'],   // 错误率小于 2%
  }
};

export default function() {
  // 登录获取 token
  const loginResponse = http.post('http://localhost:5678/api/v1/auth/login', {
    email: 'test@example.com',
    password: 'password'
  });
  
  const token = loginResponse.json('token');
  
  // 执行工作流
  const executeResponse = http.post(
    'http://localhost:5678/api/v1/workflows/test-workflow/execute',
    {},
    {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  check(executeResponse, {
    'execution successful': (r) => r.status === 200,
    'execution completed': (r) => r.json('finished') === true
  });
  
  sleep(1);
}
```

### 8.2 前端性能测试

#### Lighthouse CI 配置
```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:8080/',
        'http://localhost:8080/workflow/new',
        'http://localhost:8080/credentials'
      ],
      startServerCommand: 'pnpm start',
      numberOfRuns: 3
    },
    
    assert: {
      assertions: {
        'categories:performance': ['warn', { minScore: 0.8 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:best-practices': ['warn', { minScore: 0.8 }],
        'categories:seo': ['warn', { minScore: 0.8 }]
      }
    },
    
    upload: {
      target: 'lhci',
      serverBaseUrl: process.env.LHCI_SERVER_URL,
      token: process.env.LHCI_TOKEN
    }
  }
};
```

---

## 9. 测试数据管理

### 9.1 测试数据工厂

#### 数据生成器
```typescript
// test/utils/factories.ts
import { faker } from '@faker-js/faker';

export class WorkflowFactory {
  static create(overrides: Partial<IWorkflow> = {}): IWorkflow {
    return {
      id: faker.string.uuid(),
      name: faker.commerce.productName(),
      active: false,
      nodes: [
        {
          id: 'start',
          name: 'Start',
          type: 'n8n-nodes-base.manualTrigger',
          typeVersion: 1,
          parameters: {},
          position: [0, 0]
        }
      ],
      connections: {},
      createdAt: faker.date.past(),
      updatedAt: faker.date.recent(),
      ...overrides
    };
  }
  
  static createMany(count: number, overrides: Partial<IWorkflow> = {}): IWorkflow[] {
    return Array.from({ length: count }, () => this.create(overrides));
  }
}

export class UserFactory {
  static create(overrides: Partial<IUser> = {}): IUser {
    return {
      id: faker.string.uuid(),
      email: faker.internet.email(),
      firstName: faker.person.firstName(),
      lastName: faker.person.lastName(),
      password: faker.internet.password(),
      role: 'user',
      isActive: true,
      ...overrides
    };
  }
}
```

### 9.2 测试数据库种子

#### 数据库种子脚本
```typescript
// test/utils/seed-data.ts
export class DatabaseSeeder {
  static async seedTestData(db: TestDatabase): Promise<void> {
    // 创建测试用户
    const users = UserFactory.createMany(5);
    await db.seed({ User: users });
    
    // 创建测试工作流
    const workflows = WorkflowFactory.createMany(10, {
      ownerId: users[0].id
    });
    await db.seed({ Workflow: workflows });
    
    // 创建测试凭证
    const credentials = CredentialFactory.createMany(3, {
      ownerId: users[0].id
    });
    await db.seed({ Credential: credentials });
  }
  
  static async seedPerformanceData(db: TestDatabase): Promise<void> {
    // 大量数据用于性能测试
    const users = UserFactory.createMany(100);
    const workflows = WorkflowFactory.createMany(1000);
    
    await db.seed({ 
      User: users,
      Workflow: workflows
    });
  }
}
```

---

## 10. CI/CD 集成

### 10.1 GitHub Actions 测试工作流

#### 测试工作流配置
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [master, develop]
  pull_request:
    branches: [master]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [20, 22]
        database: [sqlite, postgres, mysql]
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Run unit tests
        run: pnpm test:${{ matrix.database }}
        env:
          COVERAGE_ENABLED: true
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
  
  e2e-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Build application
        run: pnpm build
      
      - name: Run Cypress tests
        uses: cypress-io/github-action@v6
        with:
          start: pnpm start
          wait-on: 'http://localhost:8080'
          record: true
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
      
      - name: Run Playwright tests
        run: pnpm --filter=n8n-playwright test
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: test-results
          path: test-results/
```

### 10.2 质量门禁

#### 质量检查配置
```yaml
# .github/workflows/quality.yml
name: Quality Checks

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # SonarQube 需要完整历史
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Run linting
        run: pnpm lint
      
      - name: Run type checking
        run: pnpm typecheck
      
      - name: Run tests with coverage
        run: pnpm test
        env:
          COVERAGE_ENABLED: true
      
      - name: SonarQube Scan
        uses: sonarqube-scanner-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## 11. 总结

### 11.1 测试策略优势

1. **全面覆盖**: 从单元测试到 E2E 测试的完整覆盖
2. **现代工具**: 使用最新的测试工具和框架
3. **并行执行**: 高效的并行测试执行
4. **质量保证**: 严格的覆盖率和质量标准
5. **CI/CD 集成**: 完善的持续集成流程

### 11.2 测试工具特点

1. **Jest**: 成熟稳定的后端测试框架
2. **Vitest**: 现代化的前端测试工具
3. **Cypress**: 开发者友好的 E2E 测试
4. **Playwright**: 跨浏览器的现代 E2E 测试
5. **性能测试**: K6 和 Lighthouse 的性能监控

### 11.3 质量保证体系

1. **多层测试**: 测试金字塔的完整实现
2. **自动化**: 高度自动化的测试流程
3. **监控**: 实时的质量和性能监控
4. **反馈**: 快速的测试反馈机制
5. **改进**: 持续的测试策略优化

n8n 的测试框架体系展现了现代软件开发中测试驱动开发的最佳实践，为项目的稳定性和质量提供了强有力的保障。