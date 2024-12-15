---
title: C-中间件
date: 2024-12-11 16:33:33
tags:
category: 
  - 知识
  - 工具
  - 项目
  - 问题
---

### 6. Express 中间件配置

虽然严格来说这些文件不是“配置文件”，但它们在应用程序的工作流程中起到了重要的配置作用。以下是主要的中间件文件及其作用：

#### 6.1 `src/middleware/auth.js`

**路径**: `src/middleware/auth.js`

**作用**: 用户认证中间件，验证请求中的 JWT Token，确保请求的合法性和安全性。

**内容**:

```javascript
// middleware/auth.js
const User = require("@/models/user/User");
const { verifyToken } = require("../utils/jwt");

const auth = async (req, res, next) => {
  try {
    const authHeader = req.headers["authorization"];

    // 检查 Authorization 头是否存在且以 'Bearer ' 开头
    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return res.status(401).json({
        code: 401,
        error: "未授权的访问：缺少 Token",
      });
    }

    // 提取 Token
    const token = authHeader.split(" ")[1];
    if (!token) {
      return res.status(401).json({
        code: 401,
        error: "未授权的访问：缺少 Token",
      });
    }

    // 验证 Token
    const decoded = verifyToken(token);
    if (!decoded) {
      return res.status(401).json({
        code: 401,
        error: "未授权的访问：无效的 Token",
      });
    }

    // 查找用户
    const user = await User.findById(decoded._id);
    if (!user) {
      return res.status(401).json({ message: "未授权的访问：用户不存在" });
    }

    // 添加用户信息到请求对象
    req.user = user;

    next();
  } catch {
    return res.status(401).json({
      code: 401,
      error: "Token 无效或已过期",
    });
  }
};

module.exports = auth;
```

**功能说明**:

- **Authorization Header 检查**: 确认请求头中包含有效的 Bearer Token。
- **Token 验证**: 使用 `verifyToken` 方法解码和验证 JWT Token 的有效性。
- **用户存在性检查**: 根据 Token 中的用户 ID 查找数据库中的用户，确保用户存在。
- **用户信息挂载**: 将用户信息添加到 `req.user`，供后续中间件和路由使用。

**使用步骤**:

1. **引入中间件**:

   在需要认证的路由文件中，引入并使用该中间件。

   ```javascript
   const auth = require("../middleware/auth");

   app.get("/protected-route", auth, (req, res) => {
     res.json({ message: "受保护的路由", user: req.user });
   });
   ```

2. **实现 JWT 工具**:

   确保 `src/utils/jwt.js` 中有 `verifyToken` 方法，用于解码和验证 JWT Token。

   ```javascript
   // utils/jwt.js
   const jwt = require("jsonwebtoken");
   require("dotenv").config({ path: `.env.${process.env.NODE_ENV}` });

   const verifyToken = (token) => {
     try {
       return jwt.verify(token, process.env.JWT_SECRET);
     } catch (err) {
       return null;
     }
   };

   module.exports = { verifyToken };
   ```

3. **配置 JWT Secret**:

   在 `.env.development` 和 `.env.production` 中添加 `JWT_SECRET`。

   ```dotenv
   JWT_SECRET=your_jwt_secret_key
   ```

#### 6.2 `src/middleware/authFun.js`

**路径**: `src/middleware/authFun.js`

**作用**: 辅助的认证功能，提供更灵活的认证逻辑，如多种客户端类型的认证代码的验证。这通常用于支持不同的客户端（Web、移动端等）的认证需求。

**内容**:

```javascript
// middleware/authFun.js
const User = require("@/models/user/User");
const { verifyToken } = require("../utils/jwt");

const authFun = async (req) => {
  try {
    const authHeader = req.headers["authorization"];

    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return false;
    }

    const token = authHeader.split(" ")[1];
    if (!token) {
      return false;
    }

    const decoded = verifyToken(token);
    if (!decoded) {
      return false;
    }

    const user = await User.findById(decoded._id);

    if (user && user.verificationCode[decoded.clientType] === decoded.verificationCode) {
      return user; // 成功验证，返回用户信息
    }
  } catch (error) {
    // console.warn("Token 验证失败", error);
  }

  return false;
};

module.exports = { authFun, auth };
```

**功能说明**:

- **authFun**: 提供一个函数来验证 Token，并根据特定条件（如 `clientType` 和 `verificationCode`）返回用户信息或 `false`。
- **灵活认证**: 允许在控制器或其他逻辑中灵活地调用认证功能，而不仅限于标准的 Express 认证中间件。

**使用步骤**:

1. **引入 `authFun`**:

   在需要使用该功能的地方引入 `authFun`，如在控制器或路由处理函数中。

   ```javascript
   const { authFun } = require("../middleware/authFun");

   app.get("/another-protected-route", async (req, res) => {
     const user = await authFun(req);
     if (!user) {
       return res.status(401).json({ message: "未授权的访问" });
     }
     res.json({ message: "受保护的路由", user });
   });
   ```

2. **确保 `User` 模型中有 `verificationCode` 属性**:

   根据代码逻辑，`user.verificationCode` 应该是一个包含不同 `clientType` 的验证码对象。

   ```javascript
   // src/models/user/User.js
   const mongoose = require("mongoose");

   const userSchema = new mongoose.Schema({
     username: { type: String, required: true, unique: true },
     password: { type: String, required: true },
     verificationCode: {
       web: { type: String },
       mobile: { type: String },
       // 其他客户端类型...
     },
     // 其他字段...
   });

   module.exports = mongoose.model("User", userSchema);
   ```

#### 6.3 `src/middleware/authManager.js`

**路径**: `src/middleware/authManager.js`

**作用**: 管理员认证中间件，专门用于验证管理员身份，确保只有具备管理员权限的用户才能访问某些路由。

**内容**:

```javascript
// src/middleware/authManager.js
const Manager = require("@/models/manager/Manager");
const { verifyToken } = require("../utils/jwt");
const redisClient = require("@/config/redis");

const authManager = async (req, res, next) => {
  try {
    const authHeader = req.headers["authorization"];

    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return res.status(401).json({
        code: 401,
        error: "未授权的访问：缺少 Token",
      });
    }

    const token = authHeader.split(" ")[1];
    if (!token) {
      return res.status(401).json({
        code: 401,
        error: "未授权的访问：缺少 Token",
      });
    }

    const decoded = verifyToken(token);
    if (!decoded) {
      return res.status(401).json({
        code: 401,
        error: "未授权的访问：无效的 Token",
      });
    }

    const manager = await Manager.findById(decoded._id);
    if (!manager) {
      return res.status(401).json({
        code: 401,
        error: "未授权的访问：管理员不存在",
      });
    }

    // 检查管理员角色
    if (manager.role !== "admin") {
      return res.status(403).json({
        code: 403,
        error: "禁止的访问：无合适权限",
      });
    }

    req.manager = manager;
    next();
  } catch (error) {
    console.error("Error in authManager middleware:", error);
    return res.status(401).json({
      code: 401,
      error: "Token 无效或已过期",
    });
  }
};

const findUserByPhone = async (phoneNumber) => {
  try {
    const user = await User.findOne({ phoneNumber });
    if (user) {
      return user;
    }
    return false;
  } catch (error) {
    console.error("Error finding user:", error);
    return false;
  }
};

module.exports = {
  authManager,
  findUserByPhone,
};
```

**功能说明**:

- **authManager**: 验证请求中的 JWT Token，确保用户是已登录的管理员，并且拥有相应的权限。
- **findUserByPhone**: 辅助函数，通过手机号查找用户，可用于管理员管理用户的功能。

**使用步骤**:

1. **引入中间件**:

   在需要管理员权限的路由中，引入并使用 `authManager` 中间件。

   ```javascript
   const { authManager } = require("../middleware/authManager");

   app.post("/admin-dashboard", authManager, (req, res) => {
     res.json({ message: "欢迎，管理员", manager: req.manager });
   });
   ```

2. **配置管理员模型**:

   确保 `src/models/manager/Manager.js` 定义了管理员的模型，包含角色字段。

   ```javascript
   const mongoose = require("mongoose");
   const Schema = mongoose.Schema;

   const managerSchema = new Schema({
     username: { type: String, required: true, unique: true },
     password: { type: String, required: true },
     role: { type: String, enum: ["user", "admin"], default: "admin" },
     // 其他字段...
   }, { timestamps: true });

   module.exports = mongoose.model("Manager", managerSchema);
   ```

3. **确保 JWT 中包含管理员信息**:

   在生成管理员的 Token 时，包含其角色信息，以便中间件进行验证。

   ```javascript
   // 生成管理员 Token
   const { generateToken } = require("../utils/jwt");

   const token = generateToken({ _id: manager._id, role: manager.role });
   ```

#### 6.4 `src/middleware/deliveryController.js`

**路径**: `src/middleware/deliveryController.js`

**作用**: 处理用户卡牌的发货逻辑，包括筛选用户卡牌、选择要发货的卡牌、更新库存等。

**内容**:

```javascript
// deliveryController.js
const mongoose = require("mongoose");
const UserCard = require("@/models/user/UserCard"); // 用户卡牌模型
const DeliveryOrder = require("@/models/user/DeliveryOrder"); // 发货订单模型

// 筛选用户卡牌的函数
async function filterUserCards(userId) {
  return await UserCard.findOne({ userId }).populate("cardList.cards");
}

// 选择卡牌进行发货的函数
async function selectDeliveryCard(userId, cardId) {
  const userCard = await UserCard.findOne({
    userId,
    "cardList.cards._id": cardId, // 使用 _id 进行匹配
  });
  // ...逻辑省略...
}

// 处理发货的控制器函数
async function handleDelivery(req, res) {
  try {
    const { userId, cardId } = req.body;

    // 筛选用户卡牌
    const userCards = await filterUserCards(userId);
    if (!userCards) {
      return res.status(404).json({ message: "用户卡牌不存在" });
    }

    // 选择卡牌进行发货
    const selectedCard = await selectDeliveryCard(userId, cardId);
    if (!selectedCard) {
      return res.status(404).json({ message: "指定的卡牌不存在" });
    }

    // 创建发货订单
    const deliveryOrder = new DeliveryOrder({
      userId,
      cardId,
      status: "待发货",
      // 其他字段...
    });

    await deliveryOrder.save();

    res.json({ message: "发货订单已创建", order: deliveryOrder });
  } catch (error) {
    console.error("发货时出错：", error);
    res.status(500).json({ error: "服务器内部错误" });
  }
}

// 导出控制器函数
module.exports = {
  handleDelivery,
};
```

**功能说明**:

- **filterUserCards**: 根据用户 ID 筛选并返回用户拥有的卡牌库，包含卡牌详细信息。
- **selectDeliveryCard**: 根据用户 ID 和卡牌 ID 选择需要发货的卡牌，并进行相关库存更新。
- **handleDelivery**: 主要的发货处理函数，调用上述辅助函数，创建发货订单，并返回响应。

**使用步骤**:

1. **引入控制器**:

   在路由文件中，引入并使用 `handleDelivery` 函数来处理发货请求。

   ```javascript
   const { handleDelivery } = require("../middleware/deliveryController");
   const authManager = require("../middleware/authManager").authManager;

   app.post("/deliver-card", authManager, handleDelivery);
   ```

2. **配置模型**:

   确保 `UserCard` 和 `DeliveryOrder` 模型存在，并具备必要的字段来管理卡牌库存和发货订单信息。

   ```javascript
   // src/models/user/UserCard.js
   const mongoose = require("mongoose");
   const Schema = mongoose.Schema;

   const userCardSchema = new Schema({
     userId: { type: Schema.Types.ObjectId, ref: "User", required: true },
     cardList: [
       {
         cards: { type: Schema.Types.ObjectId, ref: "Card", required: true },
         quantity: { type: Number, default: 0 },
       }
     ],
     // 其他字段...
   });

   module.exports = mongoose.model("UserCard", userCardSchema);
   ```

   ```javascript
   // src/models/user/DeliveryOrder.js
   const mongoose = require("mongoose");
   const Schema = mongoose.Schema;

   const deliveryOrderSchema = new Schema({
     userId: { type: Schema.Types.ObjectId, ref: "User", required: true },
     cardId: { type: Schema.Types.ObjectId, ref: "Card", required: true },
     status: { type: String, enum: ["待发货", "已发货", "已完成"], default: "待发货" },
     // 其他字段...
   }, { timestamps: true });

   module.exports = mongoose.model("DeliveryOrder", deliveryOrderSchema);
   ```

#### 6.5 `src/middleware/drawCard.js`

**路径**: `src/middleware/drawCard.js`

**作用**: 处理抽卡功能，包括初始化卡池、计算抽卡概率、并发控制等。确保抽卡操作的公平性和数据一致性。

**内容**:

```javascript
// drawCard.js
const { getCardPoolFromCache } = require("../utils/redisCachePool");
const { acquireLock, releaseLock } = require("../middleware/redisLock");
const redisClient = require("../config/redis");

// 初始化卡池，计算每张卡的累积区间
function initializeCardPool(cardPool) {
  cardPool.probability.forEach((rarityGroup) => {
    let cumulativeCount = 0;
    rarityGroup.cards.forEach((card) => {
      card.rangeStart = cumulativeCount;
      cumulativeCount += card.count;
      card.rangeEnd = cumulativeCount;
    });
  });
}

// 抽卡逻辑
async function drawCard(userId, poolId, numberOfCards) {
  // 获取卡池
  const cardPool = await getCardPoolFromCache(poolId);
  if (!cardPool) throw new Error("卡池不存在");

  // 计算抽卡概率
  initializeCardPool(cardPool);

  // 获取锁，确保并发安全
  const lock = await acquireLock(`drawCard:${poolId}:${userId}`);

  try {
    const selectedCards = [];
    const multi = redisClient.multi();

    for (let i = 0; i < numberOfCards; i++) {
      // 生成随机数
      const rand = Math.floor(Math.random() * 10000);

      // 根据概率选择卡牌
      for (const rarityGroup of cardPool.probability) {
        for (const card of rarityGroup.cards) {
          if (rand >= card.rangeStart && rand < card.rangeEnd) {
            selectedCards.push(card.productId);
            break;
          }
        }
      }

      // 减少库存（假设有库存管理逻辑）
      multi.decr(`cardStock:${poolId}:${card.productId}`);
    }

    // 执行事务
    const execResult = await multi.exec();
    if (execResult === null) {
      throw new Error("Redis 事务执行失败，可能是由于数据冲突");
    }

    await releaseLock(lock);

    return selectedCards;
  } catch (error) {
    if (lock) await releaseLock(lock);
    console.error(error.message);
  }
}

module.exports = { drawCard };
```

**功能说明**:

- **initializeCardPool**: 为每个稀有度组计算每张卡牌的累积概率区间，用于后续的随机抽取。
- **drawCard**: 核心的抽卡逻辑，包括获取卡池数据、加锁处理并发抽卡、随机选择卡牌、更新库存等。
- **分布式锁**: 使用 Redis 锁机制，确保在高并发场景下抽卡操作的原子性和数据一致性。

**使用步骤**:

1. **引入抽卡中间件**:

   在需要进行抽卡操作的路由中，引入并使用 `drawCard`。

   ```javascript
   const { drawCard } = require("../middleware/drawCard");
   const auth = require("../middleware/auth");

   app.post("/draw-card", auth, async (req, res) => {
     try {
       const userId = req.user._id;
       const poolId = req.body.poolId;
       const numberOfCards = req.body.number;

       const cards = await drawCard(userId, poolId, numberOfCards);
       res.json({ cards });
     } catch (error) {
       res.status(500).json({ error: error.message });
     }
   });
   ```

2. **实现 Redis 缓存卡池**:

   确保 `getCardPoolFromCache` 在 `src/utils/redisCachePool.js` 中实现，负责从 Redis 缓存中获取卡池数据。

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

3. **实现锁机制**:

   使用 `redisLock.js` 实现分布式锁，确保并发抽卡不会导致数据不一致。

   ```javascript
   // src/middleware/redisLock.js
   const Redlock = require("redlock");
   const redisClient = require("../config/redis");

   const redlock = new Redlock([redisClient], {
     driftFactor: 0.01, // 时间漂移因子
     retryCount: 1000, // 重试次数
     retryDelay: 200, // 重试延迟（ms）
     retryJitter: 200, // 重试抖动（ms）
   });

   // 获取锁
   async function acquireLock(resource, ttl = 1000) {
     try {
       const lock = await redlock.acquire([resource], ttl);
       return lock;
     } catch (error) {
       throw new Error("获取锁失败: " + error.message);
     }
   }

   // 释放锁
   async function releaseLock(lock) {
     try {
       await lock.release();
     } catch (error) {
       console.error("释放锁失败:", error.message);
     }
   }

   module.exports = { acquireLock, releaseLock };
   ```

4. **使用 Redlock 进行锁管理**:

   在 `drawCard.js` 中使用 `acquireLock` 和 `releaseLock` 进行锁管理，确保抽卡操作的原子性。

5. **维护卡池库存**:

   在 Redis 中维护卡池库存，以便在抽卡时动态更新库存数量。

   ```bash
   # 设置卡池库存
   redis-cli SET cardStock:poolId:productId 100
   ```

### 6.6 `src/middleware/redisLock.js`

**路径**: `src/middleware/redisLock.js`

**作用**: 实现分布式锁机制，确保在高并发情况下对共享资源的安全访问。使用 `redlock` 库管理分布式锁。

**内容**:

```javascript
// src/middleware/redisLock.js
const redisClient = require("../config/redis");
const Redlock = require("redlock");

const redlock = new Redlock([redisClient], {
  driftFactor: 0.01, // 时间漂移因子
  retryCount: 1000, // 重试次数
  retryDelay: 200, // 重试延迟（ms）
  retryJitter: 200, // 重试抖动（ms）
});

/**
 * 获取锁
 * @param {string} key - 锁的键
 * @param {number} ttl - 锁的有效期（毫秒）
 * @returns {Lock} - 成功获取的锁对象
 */
async function acquireLock(key, ttl = 1000) {
  try {
    const lock = await redlock.acquire([key], ttl);
    return lock;
  } catch (error) {
    throw new Error("获取锁失败: " + error.message);
  }
}

/**
 * 释放锁
 * @param {Lock} lock - 要释放的锁对象
 */
async function releaseLock(lock) {
  if (lock) {
    try {
      await lock.release();
    } catch (error) {
      console.error("释放锁失败:", error.message);
    }
  }
}

module.exports = { acquireLock, releaseLock };
```

**功能说明**:

- **Redlock 实例**: 初始化 `redlock` 实例，连接到 Redis 客户端，配置锁的漂移因子、重试次数、重试延迟和抖动。
- **`acquireLock` 方法**: 获取指定键的锁，如果获取失败则抛出错误。
- **`releaseLock` 方法**: 释放已获取的锁，捕获并记录释放过程中可能发生的错误。

**使用步骤**:

1. **安装依赖**:

   确保项目中已安装 `redlock` 和 `ioredis`。

   ```bash
   npm install redlock ioredis
   ```

2. **引入锁中间件**:

   在需要控制并发访问的地方，引入并使用 `acquireLock` 和 `releaseLock`。

   ```javascript
   const { acquireLock, releaseLock } = require("../middleware/redisLock");

   async function someCriticalOperation() {
     const lock = await acquireLock("resource-key");
     try {
       // 执行需要锁保护的操作
     } finally {
       await releaseLock(lock);
     }
   }
   ```

3. **结合业务逻辑**:

   将锁机制嵌入到业务逻辑中，如抽卡、库存管理等，以防止数据竞争和不一致。

### 6.7 `src/middleware/Validate.js`

**路径**: `src/middleware/Validate.js`

**作用**: 请求参数验证中间件，使用 `express-validator` 检查请求中的参数是否合法，如果存在验证错误则返回错误响应。

**内容**:

```javascript
// src/middleware/Validate.js
const { validationResult } = require("express-validator");

module.exports = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({
      code: 400,
      errors: {
        error: errors.array(),
      },
    });
  }
  next();
};
```

**功能说明**:

- **参数验证**: 利用 `express-validator` 验证请求参数是否符合预期。
- **错误处理**: 如果存在验证错误，返回 400 状态码和错误信息，否则继续执行下一个中间件或路由处理器。

**使用步骤**:

1. **安装依赖**:

   确保项目中已安装 `express-validator`。

   ```bash
   npm install express-validator
   ```

2. **在路由中使用**:

   在需要验证请求参数的路由中，使用 `express-validator` 的验证链和 `Validate` 中间件。

   ```javascript
   const { body } = require("express-validator");
   const Validate = require("../middleware/Validate");

   app.post(
     "/register",
     [
       body("username").isString().notEmpty(),
       body("password").isString().isLength({ min: 6 }),
       // 其他验证规则...
     ],
     Validate,
     (req, res) => {
       // 处理注册逻辑
     }
   );
   ```

3. **处理验证结果**:

   如果请求参数不符合验证规则，`Validate` 中间件会自动返回错误响应，路由处理函数不会被执行。