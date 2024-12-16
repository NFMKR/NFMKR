---
title: C-模型定义
date: 2024-11-10 10:41:04
categories: 
  - 笔记
---
# MongoDB 数据模型设计与实现笔记

## 1. 基础模型设计模式

### 1.1 基础模型结构
```javascript
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

const BaseSchema = new Schema(
  {
    // 基础字段
    title: { type: String, required: true },
    description: { type: String },
    status: { 
      type: String,
      enum: ["ACTIVE", "INACTIVE", "DELETED"],
      default: "ACTIVE"
    },
    // 关联字段
    relatedId: { 
      type: Schema.Types.ObjectId, 
      ref: "RelatedModel" 
    },
  },
  { 
    timestamps: true,  // 自动管理创建和更新时间
    versionKey: false  // 不使用版本号
  }
);

module.exports = mongoose.model("BaseModel", BaseSchema);
```

### 1.2 索引设计
```javascript
// 单字段索引
Schema.index({ fieldName: 1 });  // 1 升序，-1 降序

// 复合索引
Schema.index({ field1: 1, field2: -1 });

// 唯一索引
Schema.index({ uniqueField: 1 }, { unique: true });
```

## 2. 常用模型类型示例

### 2.1 活动模型
```javascript
const ActivitySchema = new Schema({
  title: { type: String, required: true },
  startDate: { type: Date, required: true },
  endDate: { type: Date, required: true },
  status: {
    type: String,
    enum: ["UPCOMING", "ACTIVE", "ENDED"],
    default: "UPCOMING"
  },
  type: { type: String, required: true },
  priority: { type: Number, default: 0 },
  config: { type: Map, of: Schema.Types.Mixed }
});
```

### 2.2 用户模型
```javascript
const UserSchema = new Schema({
  username: { 
    type: String, 
    required: true,
    unique: true 
  },
  email: { 
    type: String,
    match: /^[\w-]+(\.[\w-]+)*@([\w-]+\.)+[a-zA-Z]{2,7}$/ 
  },
  password: { type: String, required: true },
  role: {
    type: String,
    enum: ["USER", "ADMIN"],
    default: "USER"
  },
  profile: {
    nickname: String,
    avatar: String,
    bio: String
  },
  settings: { type: Map, of: Schema.Types.Mixed }
});
```

### 2.3 商品模型
```javascript
const ProductSchema = new Schema({
  name: { type: String, required: true },
  price: { 
    type: Number,
    required: true,
    min: 0 
  },
  stock: { 
    type: Number,
    required: true,
    min: 0 
  },
  category: { type: String, required: true },
  attributes: [{
    name: String,
    value: Schema.Types.Mixed
  }],
  images: [String]
});
```

## 3. 高级特性应用

### 3.1 虚拟字段
```javascript
const Schema = new mongoose.Schema({
  firstName: String,
  lastName: String
});

// 定义虚拟字段
Schema.virtual('fullName')
  .get(function() {
    return `${this.firstName} ${this.lastName}`;
  })
  .set(function(v) {
    const parts = v.split(' ');
    this.firstName = parts[0];
    this.lastName = parts[1];
  });
```

### 3.2 中间件（钩子）
```javascript
// 前置钩子
Schema.pre('save', async function(next) {
  // 保存前的处理逻辑
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// 后置钩子
Schema.post('remove', async function(doc) {
  // 删除后的清理逻辑
  await cleanup(doc);
});
```

### 3.3 实例方法
```javascript
Schema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

Schema.methods.generateToken = function() {
  return jwt.sign({ id: this._id }, process.env.JWT_SECRET);
};
```

## 4. 关联关系处理

### 4.1 引用关联
```javascript
const OrderSchema = new Schema({
  user: { 
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  products: [{
    product: { 
      type: Schema.Types.ObjectId,
      ref: 'Product'
    },
    quantity: Number
  }]
});

// 使用 populate 加载关联数据
const order = await Order
  .findById(orderId)
  .populate('user')
  .populate('products.product');
```

### 4.2 嵌入式文档
```javascript
const CommentSchema = new Schema({
  content: String,
  createdAt: { type: Date, default: Date.now }
});

const PostSchema = new Schema({
  title: String,
  content: String,
  comments: [CommentSchema]
});
```

## 5. 数据验证与处理

### 5.1 自定义验证器
```javascript
const Schema = new Schema({
  email: {
    type: String,
    required: true,
    validate: {
      validator: function(v) {
        return /^[\w-]+(\.[\w-]+)*@([\w-]+\.)+[a-zA-Z]{2,7}$/.test(v);
      },
      message: props => `${props.value} 不是有效的邮箱地址!`
    }
  }
});
```

### 5.2 默认值处理
```javascript
const Schema = new Schema({
  status: {
    type: String,
    default: 'PENDING'
  },
  createdAt: {
    type: Date,
    default: Date.now
  },
  tags: {
    type: [String],
    default: function() {
      return ['general'];
    }
  }
});
```

## 6. 最佳实践

### 6.1 模型组织结构
```
models/
  ├── base/
  │   └── BaseModel.js
  ├── user/
  │   ├── User.js
  │   └── Profile.js
  ├── product/
  │   ├── Product.js
  │   └── Category.js
  └── order/
      ├── Order.js
      └── Payment.js
```

### 6.2 模型导出模式
```javascript
// models/index.js
module.exports = {
  User: require('./user/User'),
  Product: require('./product/Product'),
  Order: require('./order/Order')
};

// 使用
const { User, Product, Order } = require('./models');
```

### 6.3 错误处理
```javascript
Schema.post('save', function(error, doc, next) {
  if (error.name === 'MongoError' && error.code === 11000) {
    next(new Error('重复键错误'));
  } else {
    next(error);
  }
});
```