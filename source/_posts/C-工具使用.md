---
title: C-工具使用
date: 2024-12-11 16:33:54
tags:
---
## 7. 工具与实用资源配置

### 7.1 `src/utils/jwt.js`

**路径**: `src/utils/jwt.js`

**作用**: 提供 JWT 生成和验证的工具函数，用于用户认证模块。

**内容**:

```javascript
// src/utils/jwt.js
const jwt = require("jsonwebtoken");
const dotenv = require("dotenv");

dotenv.config({
  path: `.env.${process.env.NODE_ENV}`,
});

const generateToken = (payload) => {
  return jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: "1h" });
};

const verifyToken = (token) => {
  try {
    return jwt.verify(token, process.env.JWT_SECRET);
  } catch (err) {
    return null;
  }
};

module.exports = { generateToken, verifyToken };
```

**功能说明**:

- **生成 Token**: 使用 `jwt.sign` 生成包含用户信息的 JWT Token，并设置有效期（如 1 小时）。
- **验证 Token**: 使用 `jwt.verify` 验证 Token 的有效性和合法性，返回解码后的数据或 `null`。

**使用步骤**:

1. **引入工具函数**:

   在需要生成或验证 Token 的地方，引入 `generateToken` 和 `verifyToken` 函数。

   ```javascript
   const { generateToken, verifyToken } = require("../utils/jwt");
   ```

2. **生成 Token**:

   当用户成功登录或注册后，生成并返回 JWT Token。

   ```javascript
   app.post("/login", async (req, res) => {
     const { username, password } = req.body;
     const user = await User.findOne({ username, password });

     if (user) {
       const token = generateToken({ _id: user._id, role: user.role });
       res.json({ token });
     } else {
       res.status(401).json({ message: "用户名或密码错误" });
     }
   });
   ```

3. **验证 Token**:

   在认证中间件中，使用 `verifyToken` 函数验证请求中的 Token 是否有效。

   ```javascript
   const { verifyToken } = require("../utils/jwt");

   const auth = (req, res, next) => {
     const token = req.headers["authorization"]?.split(" ")[1];
     const decoded = verifyToken(token);

     if (decoded) {
       req.user = decoded;
       next();
     } else {
       res.status(401).json({ message: "无效或过期的 Token" });
     }
   };
   ```

### 7.2 `src/utils/redisCachePool.js`

**路径**: `src/utils/redisCachePool.js`

**作用**: 管理卡池的数据缓存，通过 Redis 提高数据访问效率，减少数据库压力。

**内容**:

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

**功能说明**:

- **获取卡池**: 从 Redis 缓存中获取指定卡池 ID 的数据，如果缓存中不存在，则返回 `null`。
- **设置卡池**: 将卡池数据以 JSON 格式存储到 Redis 缓存中，便于快速访问。

**使用步骤**:

1. **引入工具函数**:

   在需要缓存卡池数据的地方，引入 `getCardPoolFromCache` 和 `setCardPoolToCache` 函数。

   ```javascript
   const { getCardPoolFromCache, setCardPoolToCache } = require("../utils/redisCachePool");
   ```

2. **使用缓存功能**:

   在相关业务逻辑中，优先从缓存中获取卡池数据，如果不存在则从数据库中读取，并设置到缓存中。

   ```javascript
   async function getCardPool(poolId) {
     let pool = await getCardPoolFromCache(poolId);
     if (!pool) {
       pool = await CardPool.findById(poolId).populate("cardsByRarity");
       if (pool) {
         await setCardPoolToCache(poolId, pool);
       }
     }
     return pool;
   }
   ```
