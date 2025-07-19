# n8n 本地开发环境完整指南

欢迎使用 n8n 本地开发环境指南！本指南将帮助您从零开始搭建完整的 n8n 开发环境，包括环境配置、开发模式、调试技巧和最佳实践。

## 📋 指南概览

本指南包含 5 个核心文档，涵盖了 n8n 本地开发的各个方面：

### [01 - 环境要求和初始设置](01_环境要求和初始设置.md)
**适用人群**: 新手开发者、首次配置环境
- ✅ 系统要求和硬件建议
- ✅ Node.js 和 pnpm 安装配置
- ✅ Corepack 设置和使用
- ✅ 构建工具安装 (各平台)
- ✅ 项目克隆和依赖管理
- ✅ 初始构建和验证

### [02 - 开发服务器启动指南](02_开发服务器启动指南.md)
**适用人群**: 所有开发者
- 🚀 全量开发模式 (`pnpm dev`)
- 🎯 选择性开发模式 (`pnpm dev:be`, `pnpm dev:fe`, `pnpm dev:ai`)
- 🔥 热重载配置 (`N8N_DEV_RELOAD=true`)
- 💾 多数据库支持配置
- ⚡ 性能优化建议
- 🛠️ 手动分包开发策略

### [03 - 调试配置指南](03_调试配置指南.md)
**适用人群**: 需要深度调试的开发者
- 🐛 VSCode 调试配置详解
- 🔍 后端 Node.js 调试技巧
- 🎨 前端 Vue.js + Vite 调试
- 🧪 Jest 单元测试调试
- 🔗 E2E 测试调试方法
- 📊 性能分析和监控工具

### [04 - 开发工作流和最佳实践](04_开发工作流和最佳实践.md)
**适用人群**: 团队开发者、代码质量关注者
- 🔄 标准开发周期流程
- 📝 Git 工作流和提交规范
- ✨ 代码质量保证 (格式化、检查、类型检查)
- 🧪 完整测试策略 (单元、集成、E2E)
- 👥 代码审查指南
- 📈 持续集成最佳实践

### [05 - 故障排除和性能优化](05_故障排除和性能优化.md)
**适用人群**: 遇到问题的开发者、性能优化需求者
- 🚨 常见启动问题解决方案
- 💾 数据库相关问题处理
- 🎨 前端开发问题排查
- ⚡ 性能优化策略
- 📊 系统监控和分析
- 🔧 自动化故障检测

---

## 🚀 快速开始

如果您是首次使用 n8n 进行开发，建议按以下顺序阅读：

### 1️⃣ 环境准备阶段
```bash
# 按照文档 01 进行环境配置
# 1. 安装 Node.js 22.16+
# 2. 配置 pnpm 和 corepack  
# 3. 安装构建工具
# 4. 克隆项目并安装依赖
```

### 2️⃣ 开发启动阶段
```bash
# 按照文档 02 选择合适的开发模式
pnpm dev        # 全栈开发
pnpm dev:be     # 仅后端开发
pnpm dev:fe     # 仅前端开发
```

### 3️⃣ 调试配置阶段
```bash
# 按照文档 03 配置调试环境
# 1. 设置 VSCode 调试
# 2. 配置断点和日志
# 3. 测试调试功能
```

### 4️⃣ 开发实践阶段
```bash
# 按照文档 04 建立开发习惯
# 1. 遵循 Git 工作流
# 2. 执行代码质量检查
# 3. 编写和运行测试
```

### 5️⃣ 问题解决阶段
```bash
# 参考文档 05 解决遇到的问题
# 1. 诊断性能问题
# 2. 排查启动故障
# 3. 优化开发环境
```

---

## 📊 开发模式对比

| 特性 | `pnpm dev` | `pnpm dev:be` | `pnpm dev:fe` | `pnpm dev:ai` |
|------|------------|---------------|---------------|---------------|
| **启动时间** | 🐌 长 (2-5分钟) | 🚀 中 (1-3分钟) | 🚀 中 (1-3分钟) | ⚡ 短 (30秒-1分钟) |
| **内存使用** | 📈 高 (2-4GB) | 📊 中 (1-2GB) | 📊 中 (1-2GB) | 📉 低 (500MB-1GB) |
| **适用场景** | 全栈开发 | 后端专项开发 | 前端专项开发 | AI 节点开发 |
| **热重载** | ✅ 全支持 | ✅ 后端支持 | ✅ 前端支持 | ⚠️ 有限支持 |

---

## 🛠️ 必备工具清单

### 基础环境
- [ ] Node.js 22.16+ 
- [ ] pnpm 10.2.1+
- [ ] Git 最新版本
- [ ] 构建工具 (gcc, make, python3)

### 开发工具
- [ ] Visual Studio Code (推荐)
- [ ] Vue Devtools (浏览器扩展)
- [ ] 终端工具 (Terminal, iTerm2, Windows Terminal)

### 可选工具
- [ ] Docker (容器化开发)
- [ ] PostgreSQL/MySQL (外部数据库)
- [ ] Redis (缓存和队列)

---

## 🏗️ 项目架构速览

```
n8n/
├── packages/
│   ├── cli/                    # 🖥️  主服务器和 CLI
│   ├── core/                   # ⚙️  工作流执行引擎
│   ├── editor-ui/              # 🎨 Vue.js 前端编辑器
│   ├── nodes-base/             # 🧩 基础节点集合
│   ├── workflow/               # 📋 工作流数据结构
│   └── @n8n/
│       ├── design-system/      # 🎛️  UI 组件库
│       ├── n8n-nodes-langchain/ # 🤖 AI/LangChain 节点
│       └── ...                 # 其他共享包
├── cypress/                    # 🧪 E2E 测试
├── docs/                       # 📚 官方文档
└── docs_ethan/                 # 📖 本开发指南
    └── local_development_guide/
```

---

## 📋 开发环境检查清单

### 环境配置检查
- [ ] Node.js 版本 >= 22.16
- [ ] pnpm 版本 >= 10.2.1
- [ ] corepack 已启用
- [ ] 构建工具已安装
- [ ] Git 已配置用户信息

### 项目设置检查
- [ ] 项目已成功克隆
- [ ] 远程仓库已配置 (origin + upstream)
- [ ] 依赖已安装 (`pnpm install`)
- [ ] 项目已构建 (`pnpm build`)
- [ ] 可以启动开发服务器 (`pnpm dev`)

### 开发工具检查
- [ ] VSCode 调试配置正常
- [ ] 代码格式化工具可用
- [ ] ESLint 检查通过
- [ ] TypeScript 编译通过
- [ ] 测试可以正常运行

---

## 🚨 常见问题快速解决

### ❓ 无法启动开发服务器
```bash
# 1. 检查端口占用
lsof -i :5678
kill -9 <PID>

# 2. 清理并重新安装
pnpm clean && pnpm install && pnpm build

# 3. 使用不同端口
N8N_PORT=8080 pnpm dev
```

### ❓ 内存不足错误
```bash
# 1. 增加 Node.js 内存限制
NODE_OPTIONS="--max-old-space-size=4096" pnpm dev

# 2. 使用选择性开发模式
pnpm dev:be  # 或 pnpm dev:fe
```

### ❓ 热重载不工作
```bash
# 1. 确保启用热重载
N8N_DEV_RELOAD=true pnpm dev

# 2. 检查文件权限
ls -la packages/*/dist/
```

### ❓ 调试断点不生效
```bash
# 1. 重新构建项目
pnpm build

# 2. 检查源码映射配置
# 确保 VSCode launch.json 中 sourceMaps: true
```

---

## 📚 进阶学习资源

### 官方文档
- [n8n 开发者文档](https://docs.n8n.io/integrations/)
- [n8n GitHub 仓库](https://github.com/n8n-io/n8n)
- [n8n 社区论坛](https://community.n8n.io/)

### 技术栈学习
- [Vue.js 3 官方文档](https://v3.vuejs.org/)
- [Vite 构建工具](https://vitejs.dev/)
- [TypeScript 手册](https://www.typescriptlang.org/docs/)
- [Express.js 指南](https://expressjs.com/)

### 相关项目分析
- [技术栈分析文档](../technology_stack_analysis/)
- [架构设计分析](../architecture_analysis/)
- [项目结构分析](../project_structure_analysis/)

---

## 🤝 贡献和反馈

如果您在使用本指南过程中遇到问题或有改进建议：

1. **报告问题**: 在项目中创建 Issue 描述问题
2. **改进建议**: 提交 Pull Request 改进文档
3. **经验分享**: 在团队内分享使用心得

---

## 📄 许可证

本开发指南遵循与 n8n 项目相同的许可证。

---

**祝您开发愉快！** 🎉

如有任何问题，请参考相应章节的详细说明或寻求技术支持。