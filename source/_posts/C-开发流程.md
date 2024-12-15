---
title: C-开发流程
date: 2024-11-20 16:29:38
categories: 
  - 笔记
tags:
---
## 项目搭建过程

1. **克隆仓库**

   ```bash
   git clone <repository-url>
   ```

2. **安装依赖**

   安装 **Node.js** 和 **npm**

   ```bash
   npm install
   ```

3. **配置环境变量**

   根据开发或生产环境，复制对应的 `.env` 文件并填写必要的配置。

   ```bash
   cp .env.development.example .env.development
   cp .env.production.example .env.production
   ```

   编辑 `.env.development` 或 `.env.production`，填写如 `MONGO_URL`、`REDIS_URL`、`PORT` 等必要的环境变量。

4. **设置 Git 钩子**

   Husky 已配置 `pre-commit` 钩子，在每次提交前会自动运行 `lint-staged`。

   ```bash
   npx husky install
   ```

5. **启动 MongoDB 和 Redis 服务**

   确保本地或远程的 MongoDB 和 Redis 服务已启动，并且连接配置正确。

6. **运行项目**

   开发环境：

   ```bash
   npm run dev
   ```

   生产环境：

   ```bash
   npm start
   ```

7. **证书配置**

   将 `src/cert/` 目录下的证书文件替换为自己的证书，确保微信支付等功能正常运行。

8. **访问 API**

   服务器启动后，可以通过配置的端口访问各个 API 端点，如 `/A`、`/B/C` 等。

## 主要技术栈

- **Node.js**: 服务器端运行环境。
- **Express**: Web 框架，处理路由和中间件。
- **MongoDB**: NoSQL 数据库，使用 Mongoose 进行数据建模。
- **Redis**: 内存数据库，用于缓存和分布式锁。
- **Husky**: Git 钩子管理，提高代码质量。
- **Prettier**: 代码格式化工具，确保统一的代码风格。
- **JWT**: JSON Web Token，用于认证和授权。
- **微信支付集成**: 处理支付相关的功能。

## 功能模块

1. **用户管理**
   - 用户注册、登录。
   - 用户信息管理。

2. **系统功能实现**
  - 首页、图鉴页、活动页、用户页。

3. **支付系统**
   - 集成微信支付，实现支付和回调处理。
   - 订单管理和支付状态跟踪。


## 总结

通过使用中间件、模型和路由的分离，确保了代码的可维护性和扩展性。并结合 Redis 的缓存和锁机制，有效处理了高并发场景下的数据一致性问题。项目还注重代码质量，通过 Husky 和 Prettier 等工具，保证了代码的规范性。
