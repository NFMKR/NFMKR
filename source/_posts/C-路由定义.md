---
title: C-路由定义
date: 2024-11-11 16:32:57
categories: 
  - 笔记
---
## 1. 路由基础结构

### 1.1 基础路由
```javascript
const express = require('express');
const router = express.Router();
const controller = require('../controllers/someController');
const { authenticate } = require('../middleware/auth');
const validate = require('../middleware/validate');

// 基础 CRUD 路由
router.get('/', controller.getAll);
router.get('/:id', controller.getById);
router.post('/', authenticate, validate.createSchema, controller.create);
router.put('/:id', authenticate, validate.updateSchema, controller.update);
router.delete('/:id', authenticate, controller.delete);

module.exports = router;
```

### 1.2 路由分组结构
```
src/
└── routes/
    ├── index.js           # 路由聚合
    ├── auth.routes.js     # 认证相关路由
    ├── user.routes.js     # 用户相关路由
    └── api/              # API 版本控制
        ├── v1/
        └── v2/
```

## 2. 路由中间件实现

### 2.1 认证中间件
```javascript
const jwt = require('jsonwebtoken');

const authenticate = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ message: 'Invalid token' });
  }
};

module.exports = { authenticate };
```

### 2.2 验证中间件
```javascript
const { validationResult } = require('express-validator');

const validate = (validations) => {
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
};

module.exports = validate;
```

## 3. 路由控制器模式

### 3.1 控制器基类
```javascript
class BaseController {
  constructor(model) {
    this.model = model;
  }

  // 获取所有记录
  getAll = async (req, res) => {
    try {
      const { page = 1, limit = 10, sort = '-createdAt' } = req.query;
      const options = {
        page: parseInt(page),
        limit: parseInt(limit),
        sort
      };

      const data = await this.model.paginate({}, options);
      res.json(data);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  };

  // 获取单个记录
  getById = async (req, res) => {
    try {
      const doc = await this.model.findById(req.params.id);
      if (!doc) {
        return res.status(404).json({ message: 'Not found' });
      }
      res.json(doc);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  };

  // 创建记录
  create = async (req, res) => {
    try {
      const doc = new this.model(req.body);
      const savedDoc = await doc.save();
      res.status(201).json(savedDoc);
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  };

  // 更新记录
  update = async (req, res) => {
    try {
      const doc = await this.model.findByIdAndUpdate(
        req.params.id,
        req.body,
        { new: true, runValidators: true }
      );
      if (!doc) {
        return res.status(404).json({ message: 'Not found' });
      }
      res.json(doc);
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  };

  // 删除记录
  delete = async (req, res) => {
    try {
      const doc = await this.model.findByIdAndDelete(req.params.id);
      if (!doc) {
        return res.status(404).json({ message: 'Not found' });
      }
      res.json({ message: 'Successfully deleted' });
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  };
}

module.exports = BaseController;
```

### 3.2 具体控制器实现
```javascript
const BaseController = require('./BaseController');
const UserModel = require('../models/User');

class UserController extends BaseController {
  constructor() {
    super(UserModel);
  }

  // 自定义方法
  async updateProfile(req, res) {
    try {
      const { id } = req.params;
      const { name, email } = req.body;

      const user = await this.model.findByIdAndUpdate(
        id,
        { name, email },
        { new: true }
      );

      if (!user) {
        return res.status(404).json({ message: 'User not found' });
      }

      res.json(user);
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  }
}

module.exports = new UserController();
```

## 4. 路由配置最佳实践

### 4.1 API 版本控制
```javascript
// src/routes/api/v1/index.js
const express = require('express');
const router = express.Router();

router.use('/users', require('./users'));
router.use('/products', require('./products'));

module.exports = router;

// src/app.js
app.use('/api/v1', require('./routes/api/v1'));
```

### 4.2 错误处理中间件
```javascript
const errorHandler = (err, req, res, next) => {
  console.error(err.stack);

  const status = err.status || 500;
  const message = err.message || 'Something went wrong!';

  res.status(status).json({
    status: 'error',
    message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  });
};

// 注册错误处理中间件
app.use(errorHandler);
```

## 5. 路由安全实践

### 5.1 请求限制
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分钟
  max: 100 // 限制每个IP 15分钟内最多100次请求
});

// 应用到所有路由
app.use(limiter);

// 或应用到特定路由
app.use('/api/', limiter);
```

### 5.2 安全头部配置
```javascript
const helmet = require('helmet');

app.use(helmet()); // 添加各种 HTTP 头部安全配置
```

## 6. 路由文档生成

### 6.1 Swagger 配置
```javascript
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'API Documentation',
      version: '1.0.0',
    },
  },
  apis: ['./src/routes/*.js'], // 路由文件路径
};

const specs = swaggerJsdoc(options);
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(specs));
```

### 6.2 路由注释示例
```javascript
/**
 * @swagger
 * /api/users:
 *   get:
 *     summary: 获取用户列表
 *     tags: [Users]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *         description: 页码
 *     responses:
 *       200:
 *         description: 成功返回用户列表
 */
router.get('/users', controller.getUsers);
```