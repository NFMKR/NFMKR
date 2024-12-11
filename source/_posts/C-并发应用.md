---
title: C-并发应用
date: 2024-12-11 16:33:10
tags:
---
# 项目中的并发处理总结

在现代 Web 应用中，并发处理是确保系统稳定性和数据一致性的关键部分。对于本项目而言，涉及多个高并发场景，如抽卡系统、库存管理和订单处理等。本文将深入总结项目中实际应用的并发处理机制，涵盖使用的工具、实现方法及其在具体业务中的应用实例。

## 1. 并发挑战概述

本项目主要面临以下并发处理挑战：

1. **抽卡系统**: 高并发用户同时进行抽卡操作，可能导致库存数据不一致或超卖。
2. **库存管理**: 多个进程或线程同时更新库存，需确保库存数据的准确性。
3. **订单处理**: 并发生成和更新订单状态，避免重复发货或订单状态混乱。

## 2. 并发处理机制

为了解决上述并发问题，项目采用了以下关键机制：

### 2.1 分布式锁（Redis + Redlock）

**工具**: [Redis](https://redis.io/)、[Redlock](https://github.com/mike-marcacci/node-redlock)

**目的**: 确保在高并发环境下，对共享资源的访问具备原子性，防止数据竞争和不一致。

**实现方法**:

- **Redlock** 是一种分布式锁实现，通过 Redis 提供的原子操作确保锁的可靠获取和释放。
- 在需要保护的关键业务逻辑（如抽卡、库存更新）中，先获取锁，执行业务逻辑，然后释放锁。

### 2.2 Redis 事务

**工具**: [ioredis](https://github.com/luin/ioredis)

**目的**: 在多个 Redis 命令执行过程中保证原子性，防止数据中间状态的干扰。

**实现方法**:

- 使用 Redis 的 `MULTI` / `EXEC` 命令封装多个操作，确保这些操作要么全部成功，要么全部失败。

## 3. 关键文件与实现

### 3.1 分布式锁实现

#### `src/middleware/redisLock.js`

```javascript:src/middleware/redisLock.js
const redisClient = require("../config/redis");
const Redlock = require("redlock");

const redlock = new Redlock([redisClient], {
  driftFactor: 0.01, // 时间漂移因子
  retryCount: 1000,   // 重试次数
  retryDelay: 200,     // 重试延迟（ms）
  retryJitter: 200,    // 重试抖动（ms）
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

- **`acquireLock`**: 获取指定键的锁，确保在操作共享资源时的独占访问。
- **`releaseLock`**: 释放已获取的锁，防止死锁和资源占用。

**使用场景**: 在抽卡和库存更新等需要原子性操作的地方使用此锁机制。

#### `src/middleware/drawCard.js`

```javascript:src/middleware/drawCard.js
const { getCardPoolFromCache } = require("../utils/redisCachePool");
const { acquireLock, releaseLock } = require("../middleware/redisLock");
const redisClient = require("../config/redis");

/**
 * 抽卡逻辑
 * @param {string} userId - 用户ID
 * @param {string} poolId - 卡池ID
 * @param {number} numberOfCards - 抽取的卡牌数量
 * @returns {Array} - 抽取的卡牌列表
 */
async function drawCard(userId, poolId, numberOfCards) {
  // 获取卡池数据
  const cardPool = await getCardPoolFromCache(poolId);
  if (!cardPool) throw new Error("卡池不存在");

  // 初始化卡池概率区间
  initializeCardPool(cardPool);

  // 获取锁，确保并发安全
  const lockKey = `drawCard:${poolId}:${userId}`;
  const lock = await acquireLock(lockKey);

  try {
    const selectedCards = [];
    const multi = redisClient.multi();

    for (let i = 0; i < numberOfCards; i++) {
      const rand = Math.floor(Math.random() * 10000); // 假设概率总和为10000
      const card = selectCardByProbability(cardPool, rand);
      selectedCards.push(card.productId);

      // 更新库存
      multi.decr(`cardStock:${poolId}:${card.productId}`);
    }

    // 执行事务
    const execResult = await multi.exec();
    if (execResult === null) {
      throw new Error("Redis 事务执行失败，可能是由于数据冲突");
    }

    return selectedCards;
  } catch (error) {
    console.error("抽卡失败:", error.message);
    throw error;
  } finally {
    await releaseLock(lock);
  }
}

/**
 * 初始化卡池概率区间
 * @param {Object} cardPool - 卡池数据
 */
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

/**
 * 根据随机数选择卡牌
 * @param {Object} cardPool - 卡池数据
 * @param {number} rand - 随机数
 * @returns {Object} - 选中的卡牌
 */
function selectCardByProbability(cardPool, rand) {
  for (const rarityGroup of cardPool.probability) {
    for (const card of rarityGroup.cards) {
      if (rand >= card.rangeStart && rand < card.rangeEnd) {
        return card;
      }
    }
  }
  throw new Error("未能根据概率选择卡牌");
}

module.exports = { drawCard };
```

**功能说明**:

- **锁机制应用**: 在抽卡过程中，先获取锁，确保同一时间只有一个抽卡操作能够修改库存数据。
- **Redis 事务**: 使用 `MULTI` 和 `EXEC` 命令确保库存更新的原子性。
- **概率抽取**: 根据预设的概率区间随机选择卡牌，确保公平性。

**使用步骤**:

1. **获取卡池数据**: 优先从 Redis 缓存中获取，若不存在则从数据库加载并缓存。
2. **初始化概率区间**: 为每张卡牌计算累积概率，方便后续随机抽取。
3. **获取锁**: 使用 `acquireLock` 方法获取锁，确保并发安全。
4. **执行抽卡逻辑**: 随机选择卡牌并准备更新库存。
5. **执行事务**: 使用 Redis 事务批量更新库存，确保操作的原子性。
6. **释放锁**: 操作完成后，释放锁资源。

### 3.3 请求参数验证与并发控制

#### `src/middleware/Validate.js`

```javascript:src/middleware/Validate.js
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

- **参数验证**: 使用 `express-validator` 确保请求参数的合法性，减少因非法请求导致的并发问题或系统异常。
- **错误处理**: 在参数不合法的情况下，提前返回错误响应，避免后续业务逻辑的无效执行。

**使用场景**: 在抽卡、库存更新和订单处理等高并发操作前，验证请求参数，确保系统稳定性。

## 4. 并发控制的实际应用场景

### 4.1 抽卡系统

**流程**:

1. **用户发起抽卡请求**: 通过 `/draw-card` API 端点。
2. **参数验证**: 使用 `express-validator` 和 `Validate` 中间件验证请求参数。
3. **获取锁**: 使用 `acquireLock` 获取针对用户和卡池的锁。
4. **抽卡逻辑执行**: 通过 `drawCard` 函数处理抽卡，选择卡牌并更新库存。
5. **执行 Redis 事务**: 保证库存更新的原子性，防止超卖。
6. **释放锁**: 操作完成后，释放锁资源。
7. **返回抽卡结果**: 向用户返回抽取到的卡牌信息。

**优势**:

- **数据一致性**: 通过分布式锁和事务，确保库存数据的准确性和一致性。
- **并发安全**: 防止多个用户同时抽卡导致的库存超卖或数据竞争。

### 4.2 库存管理

**流程**:

1. **库存更新请求**: 由管理员或系统自动发起，更新特定卡牌的库存数量。
2. **获取锁**: 针对具体卡牌或卡池获取锁，确保操作的独占性。
3. **执行库存更新**: 通过 Redis 事务或 Mongoose 的原子操作更新库存。
4. **释放锁**: 操作完成后，释放锁资源。

**优势**:

- **防止数据竞争**: 多个进程或线程同时更新库存时，确保每次只有一个操作能够成功。
- **数据准确性**: 通过事务，保证库存的准确递减或递增。

### 4.3 订单处理

**流程**:

1. **生成订单请求**: 用户完成支付后，系统生成订单记录。
2. **参数验证**: 确保订单信息的完整性和合法性。
3. **获取锁**: 针对订单生成或状态更新获取锁，防止重复操作。
4. **执行订单创建**: 使用 Mongoose 模型创建订单记录。
5. **执行库存扣减**: 更新相关库存信息，确保商品数量的正确性。
6. **释放锁**: 操作完成后，释放锁资源。
7. **返回订单结果**: 向用户返回订单确认信息。

**优势**:

- **防止重复发货**: 通过锁机制，确保同一订单不会被多次处理。
- **数据一致性**: 确保订单状态和库存数量同步更新，避免数据不一致。

## 5. 并发控制的技术细节

### 5.1 锁的粒度选择

在不同的业务场景中，锁的粒度不同：

- **用户级锁**: 针对特定用户的操作（如抽卡），锁定用户ID和卡池ID的组合，防止同一用户对同一卡池的并发操作。
- **资源级锁**: 针对具体的资源（如特定卡牌的库存），锁定卡牌ID，确保同一卡牌的库存更新不会被多个操作同时修改。

### 5.2 锁的获取与释放

- **获取锁**: 使用 `acquireLock` 方法尝试获取锁，如果失败则通过 Redlock 的重试机制自动重试，直到成功或达到重试限制。
- **释放锁**: 操作完成后，始终在 `finally` 块中调用 `releaseLock`，确保锁能够被正确释放，即使在操作过程中发生异常。

### 5.3 Redis 事务与原子操作

- **多命令原子性**: 使用 Redis 的 `MULTI` 和 `EXEC` 命令，将多个操作封装为一个事务，确保这些操作要么全部成功，要么全部失败。
- **事务执行结果**: 在执行事务后，检查返回值是否为 `null`，以判断事务是否成功执行，防止数据冲突导致的部分操作失败。

### 5.4 错误处理与容错

- **锁获取失败**: 如果长时间无法获取锁，应合理处理，如提示用户稍后重试或进行限流。
- **事务执行失败**: 若 Redis 事务执行失败，应记录日志，通知运维人员或自动进行重试机制。
- **资源释放保障**: 使用 `finally` 块确保锁能够在任何情况下被释放，防止死锁。

## 6. 并发控制的优势

通过上述并发控制机制，本项目在高并发环境下实现了以下优势：

- **数据一致性**: 确保在并发操作下，数据库和缓存中的数据保持一致，避免数据竞争和冲突。
- **系统稳定性**: 通过合理的锁机制和事务管理，防止系统在高负载下出现数据错误或崩溃。
- **用户体验提升**: 确保用户在高并发情况下仍能获得正确的服务响应，提升整体用户体验。
- **可扩展性**: 分布式锁和 Redis 缓存的使用，使系统能够更好地应对大规模并发请求，具备良好的扩展性。

## 7. 总结

本项目通过结合 Redis 分布式锁（Redlock）、Redis 事务和合理的锁粒度选择，有效应对了高并发场景下的数据一致性和系统稳定性问题。关键业务逻辑如抽卡、库存管理和订单处理均采用了严格的并发控制措施，确保在多用户同时操作时，系统能够稳定运行，数据保持准确。此外，通过监控锁的获取与释放过程，及时处理潜在的并发冲突，进一步提升了系统的可靠性和用户满意度。

这种全面的并发控制策略，不仅增强了系统的健壮性，也为未来的功能扩展和业务增长奠定了坚实的基础。
