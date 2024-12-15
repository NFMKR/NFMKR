---
title: C-模型定义
date: 2024-12-10 10:41:04
category: 
  - 知识
  - 工具
  - 项目
  - 问题
---
## 4. 数据库模型配置

项目中使用 Mongoose 来定义和管理 MongoDB 的数据模型。以下是主要的模型文件及其作用：

### 4.1 `src/models/activity/ActivityCover.js`

**路径**: `src/models/activity/ActivityCover.js`

**作用**: 活动封面模型，关联活动详情和卡包，管理活动的基本信息和关联关系。

**内容**:

```javascript
// src/models/activity/ActivityCover.js
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

// 活动封面模型
const ActivityCoverSchema = new Schema(
  {
    title: { type: String, required: true }, // 活动标题
    description: { type: String, required: false }, // 活动描述
    startDate: { type: Date, required: false }, // 活动开始时间
    endDate: { type: Date, required: false }, // 活动结束时间
    activityTime: { type: String, required: false }, // 长期，具体时间
    status: {
      type: String,
      enum: ["进行中", "未开始", "已结束"], // 活动状态
      default: "未开始",
    },
    activityType: { type: String }, // 活动类型
    label: { type: String }, // 活动标签（限时、优惠、新上）
    weight: { type: Number, default: 0 }, // 活动权重，影响活动排序
    CardAtlasId: { type: Schema.Types.ObjectId, ref: "CardAtlas", required: false }, // 对应的图鉴ID，关联到图鉴模型
    cardPackId: { type: Schema.Types.ObjectId, ref: "CardPack", required: false }, // 对应的卡包ID，关联到卡包模型
    detailedActivityId: { type: Schema.Types.ObjectId, ref: "DetailedActivity", required: true }, // 详细活动进入ID，关联到详细活动
    reservedId: { type: String },
    extraField: { type: String },
  },
  { timestamps: true } // 自动添加创建时间和更新时间
);

module.exports = mongoose.model("ActivityCover", ActivityCoverSchema);
```

**使用步骤**:

1. **定义模型**:

   使用 Mongoose 定义活动封面的数据结构，指定字段类型和关联关系。

2. **引入模型**:

   在业务逻辑中引入并使用该模型进行数据库操作。

   ```javascript
   const ActivityCover = require("../models/activity/ActivityCover");

   // 创建新的活动封面
   const newActivityCover = new ActivityCover({
     title: "新春活动",
     description: "春节专属活动",
     detailedActivityId: someDetailedActivityId,
     // 其他字段...
   });

   await newActivityCover.save();
   ```

### 4.2 `src/models/activity/DetailedActivity.js`

**路径**: `src/models/activity/DetailedActivity.js`

**作用**: 详细活动模型，存储活动的具体信息，如标题、描述、时间、背景图片等。与 `ActivityCover` 模型关联，提供更详细的活动内容。

**内容**:

```javascript
// src/models/activity/DetailedActivity.js
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

// 详细活动模型
const DetailedActivitySchema = new Schema(
  {
    title: { type: String, required: true }, // 活动标题
    firstTitle: { type: String, default: null }, // 一级标题
    secondTitle: { type: String, default: null }, // 二级标题
    description: { type: String, default: null }, // 活动描述
    secondDescription: { type: String, default: null }, // 二级描述
    startDate: { type: Date, default: null }, // 活动开始时间
    endDate: { type: Date, default: null }, // 活动结束时间
    backgroundImageUrl: { type: String, default: null }, // 背景图片
    activityImageUrl: { type: String, default: null }, // 活动图片
    // 其他字段...
  },
  { timestamps: true } // 自动添加创建时间和更新时间
);

module.exports = mongoose.model("DetailedActivity", DetailedActivitySchema);
```

**使用步骤**:

1. **定义模型**:

   使用 Mongoose 定义详细活动的数据结构，指定字段类型和默认值。

2. **引入模型**:

   在业务逻辑中引入并使用该模型进行数据库操作。

   ```javascript
   const DetailedActivity = require("../models/activity/DetailedActivity");

   // 创建新的详细活动
   const newDetailedActivity = new DetailedActivity({
     title: "春节欢庆",
     firstTitle: "欢庆新春",
     description: "春节期间的各种活动",
     backgroundImageUrl: "https://example.com/image.png",
     // 其他字段...
   });

   await newDetailedActivity.save();
   ```

### 4.3 `src/models/activity/LuckyWheel.js`

**路径**: `src/models/activity/LuckyWheel.js`

**作用**: 幸运大转盘模型，定义转盘的奖品、背景图片、概率分布等信息。用于实现抽奖活动。

**内容**:

```javascript
// src/models/activity/LuckyWheel.js
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

const PrizesSchema = new Schema(
  {
    text: { type: String, required: true },
    productId: { type: Schema.Types.ObjectId, ref: "Product", required: false },
    // 其他奖品相关字段...
  },
  { _id: false }
);

const buttonSchema = new Schema({
  text: { type: String, required: true },
  image: { type: String, required: true },
}, { _id: false });

const LuckyWheelSchema = new Schema(
  {
    title: { type: String, required: true },
    backgroundImage: { type: String, required: false },
    productId: { type: Schema.Types.ObjectId, ref: "Product", required: true },
    prizes: [PrizesSchema],
    button: buttonSchema,
    probability: { type: [Number], required: true }, // 概率分布，数组长度应与奖品数量一致
    status: {
      type: String,
      enum: ["NORMAL", "CLOSE", "EXPIRED"], // 活动状态
      default: "NORMAL",
    },
  },
  { timestamps: true }
);

// 确保每个 productId 是唯一的
LuckyWheelSchema.index({ productId: 1 }, { unique: true });

module.exports = mongoose.model("LuckyWheel", LuckyWheelSchema);
```

**使用步骤**:

1. **定义模型**:

   使用 Mongoose 定义幸运大转盘的数据结构，指定奖品、按钮、概率等信息。

2. **引入模型**:

   在业务逻辑中引入并使用该模型进行数据库操作。

   ```javascript
   const LuckyWheel = require("../models/activity/LuckyWheel");

   // 创建新的幸运大转盘
   const newLuckyWheel = new LuckyWheel({
     title: "春节幸运转盘",
     backgroundImage: "https://example.com/background.png",
     productId: someProductId,
     prizes: [
       { text: "一等奖", productId: firstPrizeProductId },
       { text: "二等奖", productId: secondPrizeProductId },
       // 其他奖品...
     ],
     button: {
       text: "开始",
       image: "https://example.com/button.png",
     },
     probability: [910, 909, 909, 909, 909, 909, 909, 909, 909, 909, 909], // 确保总和为 10000
     status: "NORMAL",
   });

   await newLuckyWheel.save();
   ```

### 4.4 `src/models/card/Card.js`, `CardAtlas.js`, `CardInventory.js`, `CardPool.js`

**路径**:

- `src/models/card/Card.js`
- `src/models/card/CardAtlas.js`
- `src/models/card/CardInventory.js`
- `src/models/card/CardPool.js`

**作用**: 定义与卡牌相关的数据模型，如卡牌详细信息、图鉴、库存、卡池等，支持卡牌系统的管理和操作。

**内容**:

```javascript
// src/models/card/Card.js
const mongoose = require("mongoose");

// 定义卡牌模型
const cardSchema = new mongoose.Schema(
  {
    title: { type: String, required: true },
    rarity: { type: String, required: true, default: "common" },
    cardCraft: { type: String, required: false }, // 卡牌工艺
    cardSerialNumber: { type: String, required: false }, // 卡牌序列号
    cardTotalNumber: { type: String, required: false },
    pointWorth: { type: Number, required: true, default: 0 },
    cardId: { type: String, required: false },
    description: { type: String, required: true, default: "卡牌描述" },
    imageUrl: { type: String, required: true },
    atlasId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "CardAtlas",
      index: true,
      required: true,
    }, // 关联图鉴ID
  },
  {
    timestamps: true,
  }
);

module.exports = mongoose.model("Card", cardSchema);
```

```javascript
// src/models/card/CardAtlas.js
const mongoose = require("mongoose");

// 定义图鉴模型，并启用虚拟属性
const cardAtlasSchema = new mongoose.Schema(
  {
    title: { type: String, required: true },
    imageUrl: { type: String, required: true },
    group: { type: String, required: true },
    sortOrder: { type: Number, default: 0 }, // 添加 sortOrder 属性，默认为 0
    raritys: [String],
  },
  {
    toJSON: { virtuals: true }, // 启用 toJSON 虚拟属性
    toObject: { virtuals: true }, // 启用 toObject 虚拟属性
  }
);

// 添加虚拟属性 groups
cardAtlasSchema.virtual("groups").get(async function () {
  const groups = await this.model("CardAtlas").distinct("group");
  return groups; // 返回唯一的 group 值
});

module.exports = mongoose.model("CardAtlas", cardAtlasSchema);
```

```javascript
// src/models/card/CardInventory.js
const mongoose = require("mongoose");
const { Schema } = require("@/config/mongodb");

const cardInventorySchema = new Schema({
  poolId: {
    type: mongoose.Schema.Types.ObjectId,
    required: true, // 卡池ID是必填的
    ref: "CardPool", // 关联到CardPool模型
  },
  title: {
    type: String,
    required: true, // 标题是必填的
  },
  count: {
    type: Number,
    required: true, // 总数是必填的
  },
  countsByRarity: {
    type: Map,
    of: Number, // 各种稀有度的数量
    required: true, // 各稀有度数量是必填的
  },
});

const CardInventory = mongoose.model("CardInventory", cardInventorySchema);

module.exports = CardInventory;
```

```javascript
// src/models/card/CardPool.js
const mongoose = require("mongoose");
const Schema = mongoose.Schema;

// 定义卡池模型
const cardPoolSchema = new Schema(
  {
    poolId: { type: Schema.Types.ObjectId, ref: "CardInventory", required: true },
    cardsByRarity: {
      type: Map,
      of: [Schema.Types.ObjectId], // 每个稀有度对应的卡牌ID列表
      required: true,
    },
    rate: {
      type: Map,
      of: Number, // 每个稀有度对应的抽取权重
      required: true,
    },
  },
  { timestamps: true }
);

module.exports = mongoose.model("CardPool", cardPoolSchema);
```

**使用步骤**:

1. **定义模型**:

   使用 Mongoose 定义卡牌、图鉴、库存和卡池的模型，指定字段类型和关联关系。

2. **引入模型**:

   在业务逻辑中引入并使用这些模型进行数据库操作。

   ```javascript
   const Card = require("../models/card/Card");
   const CardAtlas = require("../models/card/CardAtlas");
   const CardInventory = require("../models/card/CardInventory");
   const CardPool = require("../models/card/CardPool");

   // 创建新的卡牌
   const newCard = new Card({
     title: "闪电卡",
     rarity: "SR",
     description: "强力的闪电卡",
     imageUrl: "https://example.com/lightning.png",
     atlasId: someAtlasId,
   });

   await newCard.save();
   ```

3. **维护卡池和库存**:

   使用 `CardInventory` 和 `CardPool` 管理卡池的库存和卡牌分布。

   ```javascript
   // 创建卡库
   const newInventory = new CardInventory({
     poolId: somePoolId,
     title: "春季卡池",
     count: 1000,
     countsByRarity: {
       SR: 200,
       R: 500,
       N: 300,
     },
   });

   await newInventory.save();

   // 创建卡池
   const newCardPool = new CardPool({
     poolId: somePoolId,
     cardsByRarity: {
       SR: [cardId1, cardId2],
       R: [cardId3, cardId4],
       N: [cardId5, cardId6],
     },
     rate: {
       SR: 10,
       R: 50,
       N: 40,
     },
   });

   await newCardPool.save();
   ```

### 4.5 其他模型文件

类似地，项目中还包含其他模型文件，如 `BindAccount.js`、`BindCode.js`、`Platform.js`、`User.js`、`UserCard.js`、`DeliveryOrder.js`、`Manager.js` 等，这些文件定义了与用户账号绑定、平台信息、用户卡牌和订单等相关的数据结构。

每个模型文件的作用均类似于上述卡牌模型，定义了数据的字段、类型和关联关系，并导出 Mongoose 模型以供使用。