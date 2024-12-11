---
title: C-数据库
date: 2024-12-11 16:34:03
tags:
---
### 4. 数据库与缓存配置

#### 4.1 `src/config/mongodb.js`

**路径**: `src/config/mongodb.js`

**作用**: 配置并连接到 MongoDB 数据库，管理数据库连接的建立和错误处理。使用 Mongoose 作为 ODM（对象文档映射）工具。

**内容**:

```javascript
const mongoose = require("mongoose");
const dotenv = require("dotenv");

dotenv.config();

require("dotenv").config({
  path: `.env.${process.env.NODE_ENV}`, // 根据 NODE_ENV 选择加载对应的 .env 文件
});
const mongoURL = process.env.MONGO_URL;

// 连接到 MongoDB
mongoose
  .connect(mongoURL, {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  })
  .then(() => {
    console.info("成功连接到 MongoDB", mongoURL);
  })
  .catch((err) => {
    console.warn("连接到 MongoDB 失败:", err);
    process.exit(1); // 连接失败时退出应用
  });

module.exports = mongoose;
```

**功能说明**:

- **加载环境变量**: 使用 `dotenv` 根据 `NODE_ENV` 加载对应的 `.env` 文件，获取 `MONGO_URL`。
- **连接 MongoDB**: 使用 `mongoose.connect` 方法连接到 MongoDB 数据库。
- **连接成功/失败处理**: 成功时打印连接信息，失败时打印错误并退出应用。
- **导出 Mongoose 实例**: 供项目中其他模块使用 Mongoose 进行数据库操作。

**使用步骤**:

1. **安装依赖**:

   确保项目中已安装 `mongoose` 和 `dotenv`。

   ```bash
   npm install mongoose dotenv
   ```

2. **配置环境变量**:

   在 `.env.development` 和 `.env.production` 中定义 `MONGO_URL`，指向相应环境的 MongoDB 实例。

   ```dotenv
   MONGO_URL=mongodb://localhost:27017/myapp-dev
   ```

3. **引入数据库配置**:

   在应用的入口文件（如 `src/index.js`）中引入 `mongodb.js`，以确保数据库在应用启动时连接。

   ```javascript
   // src/index.js
   const mongoose = require("./config/mongodb");
   ```

4. **定义和使用 Mongoose 模型**:

   在 `src/models/` 目录下定义 Mongoose 模型，并在业务逻辑中使用它们。

   ```javascript
   // src/models/user/User.js
   const mongoose = require("mongoose");

   const userSchema = new mongoose.Schema({
     username: { type: String, required: true, unique: true },
     password: { type: String, required: true },
     // 其他字段...
   });

   module.exports = mongoose.model("User", userSchema);
   ```

#### 4.2 `src/config/redis.js`

**路径**: `src/config/redis.js`

**作用**: 配置并连接到 Redis，用于缓存和分布式锁等功能。使用 `ioredis` 作为 Redis 客户端，并处理连接和关闭。

**内容**:

```javascript
const Redis = require("ioredis");
const dotenv = require("dotenv");

// 加载对应的环境配置文件
dotenv.config({ path: `.env.${process.env.NODE_ENV}` });

const redisURL = process.env.REDIS_URL;

const db = process.env.NODE_ENV === "development" ? 1 : 0;

// 创建 ioredis 客户端
const redisClient = new Redis(redisURL, {
  db,
  // 其他配置，如密码、超时等
});

// 处理应用程序退出时的 Redis 关闭
process.on("SIGINT", async () => {
  try {
    await redisClient.quit();
    console.info("Redis 客户端已关闭");
    process.exit(0);
  } catch (err) {
    console.error("关闭 Redis 客户端时发生错误:", err);
    process.exit(1);
  }
});

module.exports = redisClient;
```

**功能说明**:

- **加载环境变量**: 使用 `dotenv` 根据 `NODE_ENV` 加载对应的 `.env` 文件，获取 `REDIS_URL`。
- **创建 Redis 客户端**: 使用 `ioredis` 连接到指定的 Redis 实例，选择数据库 `1`（开发环境）或 `0`（其他环境）。
- **处理关闭事件**: 监听 `SIGINT` 信号（如 Ctrl+C），优雅地关闭 Redis 连接，确保资源释放。
- **导出 Redis 客户端实例**: 供项目中其他模块使用 Redis 进行缓存、锁机制等操作。

**使用步骤**:

1. **安装依赖**:

   确保项目中已安装 `ioredis` 和 `dotenv`。

   ```bash
   npm install ioredis dotenv
   ```

2. **配置环境变量**:

   在 `.env.development` 和 `.env.production` 中定义 `REDIS_URL`，指向相应环境的 Redis 实例。

   ```dotenv
   REDIS_URL=redis://localhost:6379
   ```

3. **引入 Redis 配置**:

   在需要使用 Redis 的地方，导入 `redisClient`。

   ```javascript
   // src/utils/redisCachePool.js
   const redisClient = require("../config/redis");

   async function getCardPoolFromCache(poolId) {
     const poolData = await redisClient.get(`cardPool:${poolId}`);
     return poolData ? JSON.parse(poolData) : null;
   }

   async function setCardPoolToCache(poolId, poolData) {
     await redisClient.set(`cardPool:${poolId}`, JSON.stringify(poolData));
   }

   module.exports = { getCardPoolFromCache, setCardPoolToCache };
   ```

4. **使用 Redis**:

   在业务逻辑中使用 `redisClient` 进行缓存操作或其他 Redis 功能。

   ```javascript
   const redisClient = require("./config/redis");

   // 设置缓存
   await redisClient.set("key", "value");

   // 获取缓存
   const value = await redisClient.get("key");