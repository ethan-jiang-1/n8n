# 05 - 开发环境与工作流

本文档提供搭建 n8n 本地开发环境的详细指南，并解释其核心开发命令和工作流。

n8n 是一个基于 pnpm 和 Turborepo 的大型 Monorepo 项目。理解其构建和脚本系统是高效开发的前提。

---

## 1. 环境要求

根据根目录 `package.json` 的 `engines` 字段，你需要以下环境：

-   **Node.js**: `>=22.16`
-   **pnpm**: `>=10.2.1`

建议使用 `nvm` 或 `fnm` 等工具来管理 Node.js 版本，以确保版本匹配。

---

## 2. 安装依赖

在项目根目录下，执行以下命令来安装所有模块的依赖：

```bash
pnpm install
```

`pnpm` 会利用其工作区（Workspace）特性，根据 `pnpm-workspace.yaml` 的配置，高效地安装和链接所有 `packages/*` 下的模块依赖。

---

## 3. 核心开发命令

n8n 的所有核心开发任务都通过 `pnpm` 脚本执行，这些脚本大多由 `Turborepo` (`turbo`) 来编排和加速。

### 3.1. 启动开发模式 (`dev`)

这是最常用的命令，用于同时启动前端和后端的开发服务器，并监视文件变化以实现热重载。

```bash
pnpm dev
```

-   **工作原理**:
    -   该命令实际上执行的是 `turbo run dev --parallel ...`。
    -   `turbo` 会查找所有工作区模块中 `package.json` 里定义的 `dev` 脚本，并**并行**运行它们。
    -   例如，它会同时启动 `packages/cli` 的后端服务（通常使用 `nodemon` 或 `tsc-watch`）和 `packages/frontend/editor-ui` 的 Vite 开发服务器。
-   **访问**:
    -   前端编辑器 UI: `http://localhost:8080`
    -   后端 API: `http://localhost:5678`

### 3.2. 构建项目 (`build`)

用于编译所有模块的 TypeScript/Vue 代码，并生成可运行的 JavaScript 文件。

```bash
pnpm build
```

-   **工作原理**:
    -   执行 `turbo run build`。
    -   `turbo` 会智能地分析模块间的依赖关系（在 `turbo.json` 的 `tasks.build.dependsOn` 中定义），并以正确的顺序构建它们。
    -   它会利用缓存，如果某个模块的代码没有发生变化，`turbo` 会直接使用上次的构建产物，极大地提升了构建速度。
    -   构建产物通常位于各个模块的 `dist/` 目录下。

### 3.3. 运行测试 (`test`)

用于执行所有模块的单元测试和集成测试。

```bash
pnpm test
```

-   **工作原理**:
    -   执行 `turbo run test`。
    -   `turbo` 会并行运行所有模块中定义的 `test` 脚本。
    -   n8n 使用 `jest` 和 `vitest` 作为其主要的测试框架。
    -   同样，`turbo` 的缓存机制也适用于测试，只有发生变化的代码相关的测试才会被重新运行。

### 3.4. 代码格式化与检查 (`lint` & `format`)

用于确保代码风格的统一和质量。

-   **格式化代码**:
    ```bash
    pnpm format
    ```
    此命令会使用 `biome` 和 `prettier` 自动格式化所有代码。

-   **检查格式与风格**:
    ```bash
    pnpm lint
    ```
    此命令会使用 `eslint` 和 `biome` 检查代码是否存在语法错误或不符合风格规范的地方。在提交代码前运行此命令是一个好习惯。

---

## 4. 开发工作流示例：添加新功能

假设你需要为一个现有节点添加一个新功能，一个典型的开发流程如下：

1.  **启动开发环境**:
    ```bash
    pnpm dev
    ```
2.  **修改节点代码**:
    -   在 `packages/nodes-base/nodes/YourNode/` 目录下找到对应的 `*.node.ts` 文件。
    -   修改 `description` 来添加新的参数，或修改 `execute` 方法来改变其行为。
3.  **实时查看效果**:
    -   由于 `pnpm dev` 正在运行，你对后端代码（节点逻辑）的修改会触发 `tsc-watch` 重新编译，并由 `nodemon` 重启后端服务。
    -   你对前端代码的修改会由 Vite 开发服务器实时反馈到浏览器。
    -   你可以在 `http://localhost:8080` 上打开你的工作流，刷新页面后就能看到节点UI的变化，并可以手动执行来测试新的逻辑。
4.  **添加单元测试**:
    -   在节点目录下的 `test` 文件夹中，为你的新功能添加或修改单元测试。
    -   可以单独运行特定模块的测试以加快速度，例如：
        ```bash
        # 仅运行 nodes-base 包的测试
        pnpm --filter n8n-nodes-base test
        ```
5.  **提交代码**:
    -   在提交前，确保通过了代码检查：
        ```bash
        pnpm lint
        pnpm format:check
        ```
    -   提交你的代码。项目配置了 `lefthook`，会在 `pre-commit` 阶段自动运行检查，确保提交的代码质量。