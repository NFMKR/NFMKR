---
title: C-工具使用
date: 2024-11-11 16:33:54
categories: 
  - 笔记
---
## 1. JWT 工具类实现

### 1.1 基础 JWT 工具类
```javascript
const jwt = require("jsonwebtoken");
const dotenv = require("dotenv");

// 加载环境变量
dotenv.config({
  path: `.env.${process.env.NODE_ENV}`,
});

class JwtUtil {
  static generateToken(payload, expiresIn = "1h") {
    return jwt.sign(payload, process.env.JWT_SECRET, { expiresIn });
  }

  static verifyToken(token) {
    try {
      return jwt.verify(token, process.env.JWT_SECRET);
    } catch (err) {
      return null;
    }
  }
}

module.exports = JwtUtil;
```

### 1.2 JWT 中间件实现
```javascript
const JwtUtil = require("../utils/jwt");

const authMiddleware = (req, res, next) => {
  const token = req.headers["authorization"]?.split(" ")[1];
  const decoded = JwtUtil.verifyToken(token);

  if (decoded) {
    req.user = decoded;
    next();
  } else {
    res.status(401).json({ message: "Invalid or expired token" });
  }
};

module.exports = authMiddleware;
```

## 2. 缓存工具类实现

### 2.1 Redis 缓存管理器
```javascript
class CacheManager {
  constructor(redisClient) {
    this.client = redisClient;
    this.defaultTTL = 3600; // 默认过期时间（秒）
  }

  async get(key) {
    const data = await this.client.get(key);
    return data ? JSON.parse(data) : null;
  }

  async set(key, value, ttl = this.defaultTTL) {
    await this.client.set(
      key,
      JSON.stringify(value),
      'EX',
      ttl
    );
  }

  async delete(key) {
    await this.client.del(key);
  }

  async exists(key) {
    return await this.client.exists(key);
  }
}

module.exports = CacheManager;
```

### 2.2 缓存键管理器
```javascript
class CacheKeyManager {
  static USER_PREFIX = "user:";
  static PRODUCT_PREFIX = "product:";
  static LIST_PREFIX = "list:";

  static getUserKey(userId) {
    return `${this.USER_PREFIX}${userId}`;
  }

  static getProductKey(productId) {
    return `${this.PRODUCT_PREFIX}${productId}`;
  }

  static getListKey(type, params = {}) {
    const queryString = new URLSearchParams(params).toString();
    return `${this.LIST_PREFIX}${type}:${queryString}`;
  }
}

module.exports = CacheKeyManager;
```

## 3. 数据验证工具

### 3.1 通用验证器
```javascript
const Joi = require("joi");

class Validator {
  static validate(schema, data) {
    const { error, value } = schema.validate(data, {
      abortEarly: false,
      stripUnknown: true
    });

    if (error) {
      const errors = error.details.map(detail => ({
        field: detail.path.join('.'),
        message: detail.message
      }));
      throw new ValidationError(errors);
    }

    return value;
  }

  static schemas = {
    user: Joi.object({
      username: Joi.string().min(3).max(30).required(),
      email: Joi.string().email().required(),
      password: Joi.string().min(6).required()
    }),

    product: Joi.object({
      name: Joi.string().required(),
      price: Joi.number().min(0).required(),
      description: Joi.string()
    })
  };
}

class ValidationError extends Error {
  constructor(errors) {
    super("Validation Error");
    this.errors = errors;
    this.status = 400;
  }
}

module.exports = { Validator, ValidationError };
```

## 4. 日志工具类

### 4.1 日志管理器
```javascript
const winston = require("winston");

class Logger {
  constructor() {
    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || "info",
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      ),
      transports: [
        new winston.transports.File({ 
          filename: "error.log", 
          level: "error" 
        }),
        new winston.transports.File({ 
          filename: "combined.log" 
        })
      ]
    });

    if (process.env.NODE_ENV !== "production") {
      this.logger.add(new winston.transports.Console({
        format: winston.format.simple()
      }));
    }
  }

  info(message, meta = {}) {
    this.logger.info(message, meta);
  }

  error(message, error = null) {
    this.logger.error(message, {
      error: error?.stack || error
    });
  }

  warn(message, meta = {}) {
    this.logger.warn(message, meta);
  }
}

module.exports = new Logger();
```

## 5. 工具函数集合

### 5.1 通用工具函数
```javascript
const utils = {
  // 生成随机字符串
  generateRandomString(length = 8) {
    return crypto
      .randomBytes(Math.ceil(length / 2))
      .toString("hex")
      .slice(0, length);
  },

  // 深度克隆对象
  deepClone(obj) {
    return JSON.parse(JSON.stringify(obj));
  },

  // 延迟函数
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  },

  // 格式化日期
  formatDate(date, format = "YYYY-MM-DD HH:mm:ss") {
    return moment(date).format(format);
  },

  // 分页助手
  getPagination(page = 1, limit = 10) {
    const skip = (page - 1) * limit;
    return { skip, limit };
  }
};

module.exports = utils;
```

## 6. 错误处理工具

### 6.1 自定义错误类
```javascript
class AppError extends Error {
  constructor(message, status = 500, code = "INTERNAL_ERROR") {
    super(message);
    this.status = status;
    this.code = code;
  }

  static badRequest(message) {
    return new AppError(message, 400, "BAD_REQUEST");
  }

  static unauthorized(message = "Unauthorized") {
    return new AppError(message, 401, "UNAUTHORIZED");
  }

  static forbidden(message = "Forbidden") {
    return new AppError(message, 403, "FORBIDDEN");
  }

  static notFound(message = "Resource not found") {
    return new AppError(message, 404, "NOT_FOUND");
  }
}

module.exports = AppError;
```

### 6.2 错误处理中间件
```javascript
const errorHandler = (err, req, res, next) => {
  const status = err.status || 500;
  const message = err.message || "Internal Server Error";
  const code = err.code || "INTERNAL_ERROR";

  // 记录错误日志
  logger.error(message, err);

  res.status(status).json({
    success: false,
    error: {
      code,
      message,
      ...(process.env.NODE_ENV === "development" && {
        stack: err.stack
      })
    }
  });
};

module.exports = errorHandler;
```