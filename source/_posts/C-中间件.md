---
title: C-中间件
date: 2024-11-05 16:33:33
tags:
categories: 
  - 笔记
---
## 1. 认证中间件

### 1.1 基础认证中间件
```javascript
const jwt = require('jsonwebtoken');

class AuthMiddleware {
  static async authenticate(req, res, next) {
    try {
      const token = req.headers.authorization?.split(' ')[1];
      if (!token) {
        return res.status(401).json({ 
          message: 'No token provided' 
        });
      }

      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req.user = decoded;
      next();
    } catch (error) {
      res.status(401).json({ 
        message: 'Invalid token' 
      });
    }
  }

  static roles(...allowedRoles) {
    return (req, res, next) => {
      if (!req.user || !allowedRoles.includes(req.user.role)) {
        return res.status(403).json({ 
          message: 'Access forbidden' 
        });
      }
      next();
    };
  }
}

module.exports = AuthMiddleware;
```


### 1.2 角色验证中间件
```javascript
const RoleMiddleware = {
  isAdmin: (req, res, next) => {
    if (req.user?.role !== 'admin') {
      return res.status(403).json({
        message: 'Admin access required'
      });
    }
    next();
  },

  isManager: (req, res, next) => {
    if (!['admin', 'manager'].includes(req.user?.role)) {
      return res.status(403).json({
        message: 'Manager access required'
      });
    }
    next();
  }
};

module.exports = RoleMiddleware;
```


## 2. 参数验证中间件

### 2.1 通用验证中间件
```javascript
const { validationResult } = require('express-validator');

class ValidationMiddleware {
  static validate(validations) {
    return async (req, res, next) => {
      await Promise.all(validations.map(validation => validation.run(req)));

      const errors = validationResult(req);
      if (errors.isEmpty()) {
        return next();
      }

      res.status(400).json({
        errors: errors.array().map(err => ({
          field: err.param,
          message: err.msg
        }))
      });
    };
  }

  static schemas = {
    user: [
      body('email').isEmail(),
      body('password').isLength({ min: 6 }),
      body('name').notEmpty()
    ],
    product: [
      body('name').notEmpty(),
      body('price').isNumeric(),
      body('description').optional()
    ]
  };
}

module.exports = ValidationMiddleware;
```


## 3. 分布式锁中间件

### 3.1 Redis 分布式锁
```javascript
const Redlock = require('redlock');
const Redis = require('ioredis');

class DistributedLock {
  constructor() {
    this.client = new Redis(process.env.REDIS_URL);
    this.redlock = new Redlock([this.client], {
      driftFactor: 0.01,
      retryCount: 10,
      retryDelay: 200,
      retryJitter: 200
    });
  }

  async acquire(resource, ttl = 1000) {
    try {
      return await this.redlock.acquire([resource], ttl);
    } catch (error) {
      throw new Error(`Lock acquisition failed: ${error.message}`);
    }
  }

  async release(lock) {
    try {
      await lock.release();
    } catch (error) {
      console.error(`Lock release failed: ${error.message}`);
    }
  }

  async withLock(resource, ttl, callback) {
    const lock = await this.acquire(resource, ttl);
    try {
      return await callback();
    } finally {
      await this.release(lock);
    }
  }
}

module.exports = new DistributedLock();
```


## 4. 错误处理中间件

### 4.1 全局错误处理
```javascript
class ErrorMiddleware {
  static handle(err, req, res, next) {
    console.error(err.stack);

    const status = err.status || 500;
    const message = err.message || 'Internal Server Error';

    res.status(status).json({
      error: {
        message,
        status,
        ...(process.env.NODE_ENV === 'development' && {
          stack: err.stack
        })
      }
    });
  }

  static notFound(req, res, next) {
    const error = new Error('Not Found');
    error.status = 404;
    next(error);
  }

  static async wrapper(handler) {
    return async (req, res, next) => {
      try {
        await handler(req, res, next);
      } catch (error) {
        next(error);
      }
    };
  }
}

module.exports = ErrorMiddleware;
```


## 5. 请求限制中间件

### 5.1 速率限制
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

class RateLimitMiddleware {
  constructor() {
    this.client = new Redis(process.env.REDIS_URL);
    this.store = new RedisStore({
      client: this.client,
      prefix: 'rate-limit:'
    });
  }

  createLimiter(options = {}) {
    return rateLimit({
      store: this.store,
      windowMs: 15 * 60 * 1000, // 15分钟
      max: 100, // 限制每个IP 100次请求
      message: 'Too many requests',
      ...options
    });
  }

  static apiLimiter = new RateLimitMiddleware().createLimiter();
  
  static authLimiter = new RateLimitMiddleware().createLimiter({
    windowMs: 60 * 60 * 1000, // 1小时
    max: 5, // 限制每个IP 5次登录尝试
    message: 'Too many login attempts'
  });
}

module.exports = RateLimitMiddleware;
```


## 6. 日志中间件

### 6.1 请求日志
```javascript
const winston = require('winston');
const morgan = require('morgan');

class LoggerMiddleware {
  constructor() {
    this.logger = winston.createLogger({
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.File({ 
          filename: 'error.log', 
          level: 'error' 
        }),
        new winston.transports.File({ 
          filename: 'combined.log' 
        })
      ]
    });

    if (process.env.NODE_ENV !== 'production') {
      this.logger.add(new winston.transports.Console({
        format: winston.format.simple()
      }));
    }
  }

  requestLogger() {
    return morgan('combined', {
      stream: {
        write: (message) => this.logger.info(message.trim())
      }
    });
  }

  errorLogger() {
    return (err, req, res, next) => {
      this.logger.error({
        message: err.message,
        stack: err.stack,
        method: req.method,
        path: req.path
      });
      next(err);
    };
  }
}

module.exports = new LoggerMiddleware();
```