---
title: C-数据库与缓存
date: 2024-11-21 16:34:03
categories: 
  - 笔记
---
## 1. MongoDB 配置

### 1.1 基础配置
```javascript
const mongoose = require('mongoose');
const dotenv = require('dotenv');

class DatabaseConfig {
  constructor() {
    this.loadEnvConfig();
    this.options = {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      autoIndex: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    };
  }

  loadEnvConfig() {
    dotenv.config({
      path: `.env.${process.env.NODE_ENV}`
    });
    this.mongoUrl = process.env.MONGO_URL;
  }

  async connect() {
    try {
      await mongoose.connect(this.mongoUrl, this.options);
      console.log('MongoDB connected successfully');
      this.setupListeners();
    } catch (error) {
      console.error('MongoDB connection error:', error);
      process.exit(1);
    }
  }

  setupListeners() {
    mongoose.connection.on('error', err => {
      console.error('MongoDB error:', err);
    });

    mongoose.connection.on('disconnected', () => {
      console.warn('MongoDB disconnected');
    });

    process.on('SIGINT', this.gracefulShutdown.bind(this));
  }

  async gracefulShutdown() {
    try {
      await mongoose.connection.close();
      console.log('MongoDB connection closed');
      process.exit(0);
    } catch (error) {
      console.error('Error during MongoDB shutdown:', error);
      process.exit(1);
    }
  }
}

module.exports = new DatabaseConfig();
```

### 1.2 连接池配置
```javascript
const mongooseOptions = {
  poolSize: 10,                    // 连接池大小
  bufferMaxEntries: 0,            // 禁用缓冲
  connectTimeoutMS: 10000,        // 连接超时时间
  socketTimeoutMS: 45000,         // Socket 超时时间
  family: 4,                      // 使用 IPv4
  keepAlive: true,               // 保持连接活跃
  keepAliveInitialDelay: 300000  // 保活初始延迟
};
```

## 2. Redis 配置

### 2.1 Redis 客户端配置
```javascript
const Redis = require('ioredis');

class RedisConfig {
  constructor() {
    this.loadEnvConfig();
    this.options = {
      db: this.getDbNumber(),
      retryStrategy: this.retryStrategy,
      maxRetriesPerRequest: null,
      enableReadyCheck: true,
      reconnectOnError: this.reconnectOnError
    };
  }

  loadEnvConfig() {
    this.redisUrl = process.env.REDIS_URL;
  }

  getDbNumber() {
    return process.env.NODE_ENV === 'development' ? 1 : 0;
  }

  retryStrategy(times) {
    const delay = Math.min(times * 50, 2000);
    return delay;
  }

  reconnectOnError(err) {
    const targetError = 'READONLY';
    if (err.message.includes(targetError)) {
      return true;
    }
    return false;
  }

  createClient() {
    const client = new Redis(this.redisUrl, this.options);
    this.setupListeners(client);
    return client;
  }

  setupListeners(client) {
    client.on('connect', () => {
      console.log('Redis connected');
    });

    client.on('error', (err) => {
      console.error('Redis error:', err);
    });

    client.on('ready', () => {
      console.log('Redis ready');
    });

    process.on('SIGINT', () => {
      this.gracefulShutdown(client);
    });
  }

  async gracefulShutdown(client) {
    try {
      await client.quit();
      console.log('Redis connection closed');
      process.exit(0);
    } catch (error) {
      console.error('Error during Redis shutdown:', error);
      process.exit(1);
    }
  }
}

module.exports = new RedisConfig();
```

### 2.2 Redis 缓存管理器
```javascript
class RedisCacheManager {
  constructor(redisClient) {
    this.client = redisClient;
    this.defaultTTL = 3600; // 1小时
  }

  async get(key) {
    try {
      const data = await this.client.get(key);
      return data ? JSON.parse(data) : null;
    } catch (error) {
      console.error(`Error getting cache for key ${key}:`, error);
      return null;
    }
  }

  async set(key, value, ttl = this.defaultTTL) {
    try {
      await this.client.set(
        key,
        JSON.stringify(value),
        'EX',
        ttl
      );
      return true;
    } catch (error) {
      console.error(`Error setting cache for key ${key}:`, error);
      return false;
    }
  }

  async delete(key) {
    try {
      await this.client.del(key);
      return true;
    } catch (error) {
      console.error(`Error deleting cache for key ${key}:`, error);
      return false;
    }
  }

  async clear(pattern = '*') {
    try {
      const keys = await this.client.keys(pattern);
      if (keys.length > 0) {
        await this.client.del(keys);
      }
      return true;
    } catch (error) {
      console.error('Error clearing cache:', error);
      return false;
    }
  }
}

module.exports = RedisCacheManager;
```

## 3. 环境配置管理

### 3.1 环境变量配置
```javascript
// config/env.js
const dotenv = require('dotenv');
const path = require('path');

class EnvConfig {
  constructor() {
    this.env = process.env.NODE_ENV || 'development';
    this.loadEnvFile();
    this.validateEnv();
  }

  loadEnvFile() {
    const envPath = path.resolve(process.cwd(), `.env.${this.env}`);
    dotenv.config({ path: envPath });
  }

  validateEnv() {
    const requiredEnvVars = [
      'MONGO_URL',
      'REDIS_URL',
      'JWT_SECRET'
    ];

    requiredEnvVars.forEach(varName => {
      if (!process.env[varName]) {
        throw new Error(`Missing required environment variable: ${varName}`);
      }
    });
  }

  get config() {
    return {
      mongodb: {
        url: process.env.MONGO_URL,
        options: {
          useNewUrlParser: true,
          useUnifiedTopology: true
        }
      },
      redis: {
        url: process.env.REDIS_URL,
        db: this.env === 'development' ? 1 : 0
      },
      jwt: {
        secret: process.env.JWT_SECRET,
        expiresIn: '1d'
      }
    };
  }
}

module.exports = new EnvConfig();
```

## 4. 使用示例

### 4.1 数据库操作示例
```javascript
const db = require('./config/mongodb');
const User = require('./models/User');

async function createUser(userData) {
  try {
    await db.connect();
    const user = new User(userData);
    await user.save();
    return user;
  } catch (error) {
    console.error('Error creating user:', error);
    throw error;
  }
}
```

### 4.2 缓存操作示例
```javascript
const RedisConfig = require('./config/redis');
const CacheManager = require('./utils/RedisCacheManager');

const redisClient = RedisConfig.createClient();
const cacheManager = new CacheManager(redisClient);

async function getCachedData(key) {
  try {
    let data = await cacheManager.get(key);
    if (!data) {
      data = await fetchDataFromDB();
      await cacheManager.set(key, data);
    }
    return data;
  } catch (error) {
    console.error('Error getting cached data:', error);
    throw error;
  }
}
```