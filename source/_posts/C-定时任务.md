---
title: C-定时任务
date: 2024-11-12 11:21:40
tags:
categories: 
  - 笔记
---
# express 定时任务开发笔记

## 1. 基础架构

### 1.1 核心依赖
```javascript
const cron = require("node-cron");          // 定时任务调度
const Redlock = require("redlock");         // 分布式锁
const redisClient = require("./redis");     // Redis客户端
const mongoose = require("mongoose");        // MongoDB
```

### 1.2 Cron 表达式格式
```
* * * * * *
│ │ │ │ │ │
│ │ │ │ │ └── 星期几 (0-7, 0和7都表示周日)
│ │ │ │ └──── 月份 (1-12)
│ │ │ └────── 日期 (1-31)
│ │ └──────── 小时 (0-23)
│ └────────── 分钟 (0-59)
└──────────── 秒 (0-59, 可选)
```

## 2. 分布式锁配置

```javascript
const redlock = new Redlock([redisClient], {
  driftFactor: 0.01,  // 时钟漂移因子
  retryCount: 3,      // 获取锁失败时的重试次数
  retryDelay: 200,    // 重试间隔（毫秒）
  retryJitter: 200,   // 随机延迟，避免同时重试
});

// 错误处理
redlock.on("clientError", (err) => {
  console.error("Redis锁错误:", err);
});
```

## 3. 定时任务控制器基础结构

```javascript
class TaskController {
  constructor(options) {
    this.options = options;
  }

  // 任务执行方法示例
  async executeTask() {
    try {
      // 1. 获取分布式锁
      const lock = await redlock.lock("taskLock", 30000);

      try {
        // 2. 执行任务逻辑
        await this.taskLogic();
      } finally {
        // 3. 释放锁
        await lock.unlock();
      }
    } catch (error) {
      console.error("任务执行错误:", error);
    }
  }

  // 具体任务逻辑
  async taskLogic() {
    // 实现具体任务逻辑
  }
}
```

## 4. 数据库操作模板

### 4.1 查询操作
```javascript
async function findRecords() {
  try {
    const records = await Model.find({
      status: { $ne: "EXPIRED" },
      endTime: { $lte: new Date() }
    }).select("field1 field2");
    
    return records;
  } catch (error) {
    console.error("查询错误:", error);
    throw error;
  }
}
```

### 4.2 批量更新
```javascript
async function batchUpdate(conditions, updateData) {
  try {
    const result = await Model.updateMany(
      conditions,
      { $set: updateData }
    );
    
    console.log(`更新了 ${result.modifiedCount} 条记录`);
    return result;
  } catch (error) {
    console.error("更新错误:", error);
    throw error;
  }
}
```

## 5. 定时任务调度器

```javascript
function scheduleTask(cronExpression, task) {
  cron.schedule(
    cronExpression,
    async () => {
      const lockKey = "lock:task";
      const lockTTL = 30000; // 锁的有效期（毫秒）

      try {
        // 获取锁
        const lock = await redlock.lock(lockKey, lockTTL);

        try {
          // 执行任务
          await task();
        } finally {
          // 释放锁
          await lock.unlock();
        }
      } catch (error) {
        console.error("任务执行失败:", error);
      }
    },
    {
      scheduled: true,
      timezone: "Asia/Shanghai"
    }
  );
}
```

## 6. 实用工具函数

### 6.1 日志工具
```javascript
const logger = {
  info: (message) => {
    console.log(`[${new Date().toISOString()}] INFO: ${message}`);
  },
  error: (message, error) => {
    console.error(`[${new Date().toISOString()}] ERROR: ${message}`, error);
  }
};
```

### 6.2 时间处理工具
```javascript
const dateUtils = {
  formatDate: (date) => {
    return date.toISOString().split('T')[0];
  },
  
  addDays: (date, days) => {
    const result = new Date(date);
    result.setDate(result.getDate() + days);
    return result;
  }
};
```

## 7. 任务示例

### 7.1 定期检查过期记录
```javascript
class ExpirationCheckTask {
  async execute() {
    try {
      // 1. 查找过期记录
      const expiredRecords = await Model.find({
        expiryDate: { $lte: new Date() },
        status: "ACTIVE"
      });

      if (expiredRecords.length === 0) return;

      // 2. 批量更新状态
      await Model.updateMany(
        { _id: { $in: expiredRecords.map(r => r._id) } },
        { $set: { status: "EXPIRED" } }
      );

      logger.info(`已处理 ${expiredRecords.length} 条过期记录`);
    } catch (error) {
      logger.error("处理过期记录失败", error);
    }
  }
}
```

### 7.2 定期数据同步
```javascript
class DataSyncTask {
  async execute() {
    const session = await mongoose.startSession();
    
    try {
      await session.withTransaction(async () => {
        // 1. 获取需要同步的数据
        const data = await SourceModel.find({ needSync: true });
        
        // 2. 处理数据
        const processedData = data.map(item => ({
          // 数据转换逻辑
        }));
        
        // 3. 保存到目标集合
        await TargetModel.insertMany(processedData);
        
        // 4. 更新同步状态
        await SourceModel.updateMany(
          { _id: { $in: data.map(d => d._id) } },
          { $set: { needSync: false, lastSyncTime: new Date() } }
        );
      });
      
      logger.info("数据同步完成");
    } catch (error) {
      logger.error("数据同步失败", error);
    } finally {
      session.endSession();
    }
  }
}
```

## 8. 配置管理

```javascript
const config = {
  tasks: {
    expirationCheck: {
      cronExpression: "0 0 * * *",    // 每天零点执行
      lockTimeout: 30000,             // 锁超时时间（毫秒）
      retryAttempts: 3                // 重试次数
    },
    dataSync: {
      cronExpression: "*/30 * * * *", // 每30分钟执行
      batchSize: 100,                 // 批处理大小
      timeout: 25000                  // 任务超时时间
    }
  },
  redis: {
    lockPrefix: "lock:",
    keyPrefix: "app:"
  }
};
```

## 9. 总结

1. **错误处理**
   - 使用 try-catch 包装异步操作
   - 记录详细的错误日志
   - 实现错误重试机制

2. **性能优化**
   - 使用批量操作替代单条操作
   - 合理使用索引
   - 实现数据分页处理

3. **可维护性**
   - 模块化设计
   - 统一的日志格式
   - 配置与代码分离

4. **安全性**
   - 使用分布式锁避免任务重复执行
   - 设置合理的超时时间
   - 保护敏感数据

5. **可扩展性**
   - 支持动态添加新任务
   - 配置驱动的任务管理
   - 支持任务优先级