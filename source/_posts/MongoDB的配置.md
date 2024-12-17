---
title: MongoDB的配置
date: 2024-12-17 13:52:29
tags:
categories:
  - 文章
---
# MongoDB 配置步骤

## 1. 基础配置文件

创建 `src/config/mongodb.js`:

```javascript:src/config/mongodb.js
const mongoose = require("mongoose");
const dotenv = require("dotenv");

class MongoDBConfig {
  constructor() {
    this.loadEnvConfig();
    this.initOptions();
  }

  // 加载环境变量
  loadEnvConfig() {
    dotenv.config({
      path: `.env.${process.env.NODE_ENV}`
    });
    this.mongoURL = process.env.MONGO_URL;
  }

  // 初始化连接选项
  initOptions() {
    this.options = {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      maxPoolSize: 10,               // 连接池大小
      serverSelectionTimeoutMS: 5000,// 服务器选择超时
      socketTimeoutMS: 45000,        // Socket 超时
      family: 4,                     // 使用 IPv4
      keepAlive: true,              // 保持连接活跃
      autoIndex: true               // 自动创建索引
    };
  }

  // 连接数据库
  async connect() {
    try {
      await mongoose.connect(this.mongoURL, this.options);
      console.info("MongoDB 连接成功:", this.mongoURL);
      this.setupListeners();
    } catch (error) {
      console.error("MongoDB 连接失败:", error);
      process.exit(1);
    }
  }

  // 设置事件监听器
  setupListeners() {
    mongoose.connection.on("error", (err) => {
      console.error("MongoDB 错误:", err);
    });

    mongoose.connection.on("disconnected", () => {
      console.warn("MongoDB 连接断开");
    });

    process.on("SIGINT", this.gracefulShutdown.bind(this));
  }

  // 优雅关闭
  async gracefulShutdown() {
    try {
      await mongoose.connection.close();
      console.log("MongoDB 连接已关闭");
      process.exit(0);
    } catch (error) {
      console.error("关闭 MongoDB 连接时出错:", error);
      process.exit(1);
    }
  }
}

// 导出单例实例
module.exports = new MongoDBConfig();
```

## 2. 环境变量配置

### 2.1 开发环境 `.env.development`
```env
MONGO_URL=mongodb://localhost:27017/your_database_dev
```

### 2.2 生产环境 `.env.production`
```env
MONGO_URL=mongodb://username:password@host:port/your_database_prod
```

## 3. 项目入口配置

在 `src/app.js` 或 `src/index.js` 中:

```javascript:src/app.js
const express = require("express");
const mongoDBConfig = require("./config/mongodb");

const app = express();

// 连接数据库
async function startServer() {
  try {
    // 确保数据库连接成功
    await mongoDBConfig.connect();
    
    // 启动服务器
    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => {
      console.log(`服务器运行在端口 ${PORT}`);
    });
  } catch (error) {
    console.error("启动服务器失败:", error);
    process.exit(1);
  }
}

startServer();
```

## 4. 依赖安装

```bash
# 安装必要的依赖
npm install mongoose dotenv
```

## 5. 监控和日志

添加数据库监控:

```javascript:src/config/mongodb.js
// 在 MongoDBConfig 类中添加
setupMonitoring() {
  mongoose.connection.on('connected', () => {
    console.log('Mongoose 已连接');
  });

  mongoose.connection.on('error', (err) => {
    console.error('Mongoose 连接错误:', err);
  });

  mongoose.connection.on('disconnected', () => {
    console.log('Mongoose 已断开连接');
  });

  // 监控查询执行时间
  mongoose.set('debug', (collectionName, method, query, doc) => {
    console.log(`${collectionName}.${method}`, JSON.stringify(query), doc);
  });
}
```

## 6. 错误处理中间件

```javascript:src/middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  console.error('数据库操作错误:', err);
  
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      code: 400,
      error: '数据验证失败',
      details: err.errors
    });
  }
  
  if (err.name === 'MongoServerError' && err.code === 11000) {
    return res.status(409).json({
      code: 409,
      error: '数据重复'
    });
  }
  
  res.status(500).json({
    code: 500,
    error: '服务器内部错误'
  });
};

module.exports = errorHandler;
```

## 7. 配置检查清单

- [ ] 安装必要依赖
- [ ] 设置环境变量
- [ ] 创建数据库配置文件
- [ ] 定义数据模型
- [ ] 添加索引优化
- [ ] 配置错误处理
- [ ] 设置监控和日志
- [ ] 实现优雅关闭
- [ ] 测试数据库连接
- [ ] 确保安全配置
- [ ] 环境变量管理
- [ ] 连接池配置
- [ ] 错误处理
- [ ] 监控和日志
- [ ] 优雅关闭
- [ ] 索引优化
---
# mongoose详解


### Mongoose — MongoDB ODM

#### 1. **概述**
- **Mongoose** 是一个 MongoDB 的 ODM（Object Data Modeling）库，允许你通过定义模式（Schema）来创建 MongoDB 中的数据模型。
- Mongoose 为 MongoDB 提供了许多高级功能，如数据验证、虚拟字段、查询构建等，简化了操作 MongoDB 文档的过程。

#### 2. **安装 Mongoose**
```bash
npm install mongoose
```

#### 3. **连接到 MongoDB**
```javascript
const mongoose = require('mongoose');

// 连接到 MongoDB 数据库
mongoose.connect('mongodb://localhost:27017/test', { useNewUrlParser: true, useUnifiedTopology: true });

// 连接成功回调
mongoose.connection.on('connected', () => {
  console.log('MongoDB connected');
});

// 连接错误回调
mongoose.connection.on('error', (err) => {
  console.error('MongoDB connection error:', err);
});
```

#### 4. **定义模式（Schema）**
Mongoose 通过 `Schema` 定义 MongoDB 文档的结构：
```javascript
const { Schema, model } = mongoose;

// 定义一个模式（Schema）
const userSchema = new Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
});

// 创建模型（Model）
const User = model('User', userSchema);
```

#### 5. **基本操作**
- **创建文档**:
  ```javascript
  const user = new User({ name: 'John Doe', email: 'john.doe@example.com' });
  user.save()
    .then(doc => {
      console.log('User saved:', doc);
    })
    .catch(err => {
      console.error('Error saving user:', err);
    });
  ```

- **查询文档**:
  ```javascript
  User.find()
    .then(users => {
      console.log('All users:', users);
    })
    .catch(err => {
      console.error('Error fetching users:', err);
    });
  ```

- **查找单个文档**:
  ```javascript
  User.findOne({ email: 'john.doe@example.com' })
    .then(user => {
      console.log('User found:', user);
    })
    .catch(err => {
      console.error('Error finding user:', err);
    });
  ```

- **更新文档**:
  ```javascript
  User.findByIdAndUpdate(id, { name: 'Jane Doe' })
    .then(() => {
      console.log('User updated');
    })
    .catch(err => {
      console.error('Error updating user:', err);
    });
  ```

- **删除文档**:
  ```javascript
  User.findByIdAndDelete(id)
    .then(() => {
      console.log('User deleted');
    })
    .catch(err => {
      console.error('Error deleting user:', err);
    });
  ```

#### 6. **数据验证**
Mongoose 支持在模式中定义字段的验证规则：
```javascript
const userSchema = new Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
});

// 保存数据时会进行验证
const user = new User({ name: 'John', email: 'john@example.com' });
user.save()
  .then(doc => {
    console.log('User saved:', doc);
  })
  .catch(err => {
    console.error('Error:', err);
  });
```

#### 7. **虚拟字段**
Mongoose 允许定义虚拟字段，这些字段并不存储在数据库中，而是动态计算得出的：
```javascript
userSchema.virtual('fullName').get(function() {
  return this.firstName + ' ' + this.lastName;
});
```

#### 8. **常用查询方法**
- **`find()`**: 查找多个文档。
- **`findOne()`**: 查找一个文档。
- **`findById()`**: 根据 `_id` 查找文档。
- **`findByIdAndUpdate()`**: 根据 `_id` 查找并更新文档。
- **`findByIdAndDelete()`**: 根据 `_id` 查找并删除文档。

#### 9. **查询构建**
Mongoose 提供了链式调用来构建复杂的查询：
```javascript
User.find({ age: { $gte: 18 } })
  .where('status').equals('active')
  .sort({ name: 1 })
  .limit(10)
  .exec((err, users) => {
    console.log(users);
  });
```

### 总结
- **Mongoose** 是 MongoDB 的 ODM 库，使用 Schema 来定义 MongoDB 文档的结构，并支持数据验证、查询构建和虚拟字段等功能。