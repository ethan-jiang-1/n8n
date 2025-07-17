# 06 - 部署方案

本文档分析 n8n 的部署策略，重点介绍其官方推荐的、基于 Docker 的部署方法。

---

## 1. 部署方式概述

n8n 提供了多种部署方式以满足不同用户的需求：

1.  **`npx` 快速体验**:
    -   命令: `npx n8n`
    -   **优点**: 无需安装，开箱即用，非常适合快速体验和临时测试。
    -   **缺点**: 不适合生产环境，因为所有数据（工作流、凭证）都是临时的，进程结束后数据会丢失。

2.  **NPM 全局或项目安装**:
    -   通过 `npm install -g n8n` 或在项目中安装。
    -   **优点**: 相比 `npx`，可以持久化数据。
    -   **缺点**: 需要手动管理 Node.js 环境和依赖，升级和维护相对繁琐。

3.  **Docker (官方推荐)**:
    -   **优点**: **官方推荐的生产环境部署方式**。通过 Docker，n8n 将整个应用及其所有系统依赖（如特定版本的 Node.js、字体、系统库）打包成一个隔离的、一致的运行环境。这极大地简化了部署、管理和升级过程。
    -   **缺点**: 需要用户具备基础的 Docker知识。

---

## 2. Docker 部署深入分析

n8n 的 Docker 部署是其核心交付物之一。`README.md` 中提��的快速启动命令是理解其部署模型的最佳入口。

### 2.1. 快速启动命令解析

```bash
# 1. 创建一个 Docker 数据卷用于持久化存储
docker volume create n8n_data

# 2. 运行 n8n 容器
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

-   `docker volume create n8n_data`:
    -   创建一个名为 `n8n_data` 的 Docker 数据卷。数据卷是 Docker 官方推荐的数据持久化机制，它独立于容器的生命周期。

-   `docker run ...`:
    -   `-p 5678:5678`: 将主机的 `5678` 端口映射到容器内部的 `5678` 端口，使得我们可以通过 `http://localhost:5678` 访问 n8n。
    -   `-v n8n_data:/home/node/.n8n`: 这是**最关键**的部分。它将我们刚刚创建的 `n8n_data` 数据卷挂载到容器内的 `/home/node/.n8n` 目录。n8n 默认将其所有数据（SQLite数据库、配置文件、凭证等）存储在这个目录。通过这种方式，即使用户删除了容器（`--rm` 标志会在容器停止后自动删除），所有数据依然安全地保存在 `n8n_data` 数据卷中，下次启动时可以重新挂载，实现数据持久化。
    -   `docker.n8n.io/n8nio/n8n`: 这是 n8n 官方的 Docker 镜像地址。

### 2.2. Docker 镜像构建流程

通过分析 `scripts/dockerize-n8n.mjs` 和 `docker/images/n8n-base/Dockerfile`，我们可以了解到 n8n 镜像是如何构建的：

1.  **构建应用 (`pnpm build:n8n`)**:
    -   在构建 Docker 镜像之前，必须先在项目本地运行 `pnpm build` 或 `node scripts/build-n8n.mjs`。
    -   这个过程会将所有 `packages` 下的 TypeScript 代码编译成 JavaScript，并将所有必要的前端静态资源、节点代码等打包到一个临时的 `compiled` 目录中。

2.  **Dockerize (`scripts/dockerize-n8n.mjs`)**:
    -   这个脚本调用 `docker build` 命令。
    -   它使用 `docker/images/n8n/Dockerfile`（这个文件会基于 `n8n-base`）作为构建蓝图。
    -   构建上下文（Context）是整个项目的根目录，这意味着 Dockerfile 可以访问项目中的任何文件。

3.  **Dockerfile 分析**:
    -   **多阶段构建 (Multi-stage Build)**: n8n 使用了多阶段构建来优化镜像大小。
        -   **`builder` 阶段**: 在一个临时的 `node:XX-alpine` 镜像中，安装所有系统级的依赖，如 `git`, `openssh`, `graphicsmagick`（用于图像处理节点）以及各种字体。
        -   **最终阶段**: 从一个干净的 `node:XX-alpine` 镜像开始，只从 `builder` 阶段拷贝必要的系统依赖，然后将本地已经构建好的 `compiled` 目录下的 n8n 应用代码 `COPY` 到镜像中。
    -   **结果**: 最终的镜像是���度优化的，它只包含运行 n8n 所必需的系统库和已经编译好的应用代码，而不包含任何开发依赖（如 `typescript`, `eslint`）或构建工具，从而使得镜像尽可能小而安全。

### 2.3. 生产环境配置

在生产环境中，通常会通过**环境变量**来配置 n8n 的行为，而不是修改配置文件。一些常见的环境变量包括：

-   `DB_TYPE`: 指定数据库类型 (e.g., `postgresdb`, `mysqldb`)。
-   `DB_POSTGRESDB_HOST`: PostgreSQL 服务器地址。
-   `DB_POSTGRESDB_USER`: 数据库用户名。
-   `DB_POSTGRESDB_PASSWORD`: 数据库密码。
-   `N8N_ENCRYPTION_KEY`: 用于加密凭证的密钥，**非常重要**。
-   `WEBHOOK_URL`: 公开访问 n8n 的 URL，用于生成正确的 Webhook 地址。

一个更贴近生产的 `docker-compose.yml` 示例可能如下：

```yaml
version: '3'

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=mysecretpassword
      - N8N_ENCRYPTION_KEY=myverysecureencryptionkey
    volumes:
      - n8n_data:/home/node/.n8n

  postgres:
    image: postgres:14
    restart: always
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=mysecretpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  postgres_data:
```