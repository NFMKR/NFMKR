---
title: C-Express小程序开发笔记
date: 2024-10-24 15:13:45
tags: 将看不懂的代码，放进来分析总结解释理解
category: 
  - 知识
  - 工具
  - 项目
  - 问题
---
## 目录

### express框架的搭建，环境的配置部署，编写风格的统一

1. express的框架结构搭建
2. 安装部署相关环境 

### 抽卡页api的开发

1. 横幅api
2. 抽卡卡包封面api
3. 抽卡机api
4. 抽卡概率方法

### 图鉴页api的开发

1. 图鉴卡包api
2. 卡牌api

### 用户页面api的开发

1. 登录api
2. 支付api
3. 我的背包api
4. 地址api
5. 用户信息api
6. 发货订单api
7. 抽卡订单api

### mongodb的使用

### 代码与服务器的上传

1. 服务器的使用
2. git和gitlab

## express框架的搭建，环境的配置部署，编写风格的统一

### express的框架结构搭建

1. 安装 Node.js ```node -v npm -v```
2. 创建项目文件夹
3. 初始化项目 npm init -y
4. 安装 Express ```npm install express```
5. 创建项目结构
(将路由，模型，中间件，启动服务器，初始化express应用都再这个里面)
```
mkdir src
cd src
touch index.js
```
6. 编写基本的 Express 服务器
```
const express = require('express');
const app = express();
const port = 3000;

//中间件
app.use(express.json());

//路由
app.get('/', (req, res) => {
  res.send("hello world");
});

//启动服务器
app.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```
7. 设置路由
```
const express = require('express');
const router = express.Router();

//示例路由
router.get('/cards', (req, res) => {
  //处理获取卡牌的逻辑
  res.send(''获取卡牌)；
})；

//导出路由
module.exports = router;
```
在src/index.js 中引入路由：
const route = require('./router');
app.user('/api', routes);
8. 启动服务器
node src/index.js(npm run dev是先设置好的，设置成保存代码文件后自动重新启动)

9. 代码保存后自动启动服务器的实现

### 安装部署相关环境

1. nodejs的安装LTS版
2. MongoDB安装community版本
3. 开发工具vscode和微信开发者工具
4. 安装依赖：express：后端框架，mongoose：用于连接和操作mongodb数据库，dotenv：管理环境变量，body-parser：解析请求体数据，nodemon：开发时自动重启服务器, 创建.env文件配置数据库

### 项目部署和测试与调试

1. 本地服务器部署：使用 PM2 或直接使用 node 启动。
2. 云服务器部署：如阿里云、腾讯云、Vercel。
``` 
npm install pm2 -g
pm2 start index.js

```
3.  本地测试:Postman
4.  接口编写apifox

### 上线注意事项

1. HTTPS：微信小程序只能请求 HTTPS 地址。
2. 域名备案：如果使用国内服务器，确保你的域名已经备案。
3. 日志与监控：部署 PM2 后使用 pm2 logs 查看运行日志。

### 编写风格的统一

Airbnb JavaScript Style Guide：广泛使用、严格的风格。
Standard JS：简洁而实用的风格。
Prettier：自动格式化代码。

## 抽卡页api的开发

### 横幅的api

1. 模型Schema，
2. 导入mongoose，const mongoose = require("@/config/mongodb");
3. 定义schema
4. 定义横幅模型：图片，详细，自动创建时间
5. 导出模型
6. api书写

### 抽卡卡包封面api



### 抽卡机api

### 抽卡卡池概率实现

抽到的卡要关联卡池/我的背包，卡包
1. 模型定义：
```
const poolSchema = new mongoose.Schema({
  name: String,
  cards : [{type: mongoose.Schema.Types.ObjectId, ref: 'Card'}],
  totalCards: Number
  });

  const Pool = mongoose.model('Pool', poolSchema);
```
2. 创建卡牌和卡池
3. 抽卡逻辑实现
```
function drawCard(pool){
  const cards = pool.cards;
  const probabilities = cards.map(card => card.probability);
  
  cnst cumulativeProbabilities = probabilities.reduce((acc, cur) => acc + cur, 0);
  
  const randomValue = Math.random() * cumulativeProbabilities;
  
  for (let i = 0; i < cards.length; i++) {
    if (randomValue <= probabilities[i]) {
      return cards[i];
    }
  }
  return null;
}
```
4. 抽卡接口
```
const express = require('express');
const router = express.Router();

router.get('/draw', async (req, res) => {
  const {poolId} = req.body;
  const pool = await Pool.findById(poolId).populate('cards');
  
  if (!pool) {
    return res.status(404).json({error: 'Pool not found'});
  }
  const drawCard = drawCard(pool);
  
  res.json(drawCard);
});

module.exports = router;
```


### 商品api

### 刷新实时数据模块中添加缓存功能

1. 实现功能的逻辑
功能：提高概率信息的获取效率，通过缓存机制减少重复的数据库查询。
逻辑流程：
接收请求，提取 Id 值。
检查缓存中是否存在该 Id 的数据且数据有效。
如果有效，直接返回缓存数据；否则进行数据库查询。
从数据库获取相关的卡片机和产品信息，计算合并的概率数据。
将新的概率数据存入缓存并返回。
2. 编写逻辑
初始化一个 cache 对象，用于存储不同 Id 的概率数据和请求时间。
在每次请求时，检查请求的 Id 是否存在于缓存中，且请求时间距离上次请求是否在10秒之内。
如果缓存有效，返回缓存的概率数据。
如果缓存无效，执行后续的数据库查询，获取新的概率数据并合并。
将新的概率数据和当前请求时间存入缓存，以备下次请求使用。
3. 总结知识点和实现原理
缓存机制：通过对象存储请求数据，避免重复的数据库查询，降低响应时间和系统负担。
时间戳管理：使用时间戳来判断缓存的有效性，这是一种常见的缓存策略，能够控制数据的时效性。
异步操作：在 Node.js 中，利用异步操作（如 await）来处理数据库查询，保证代码的非阻塞性。
错误处理：在数据库操作中加入错误处理逻辑，确保系统的稳定性，能够返回适当的错误信息。
实现原理
缓存的实现原理基于“时间窗口”的概念。设定一个有效期（如10秒），在这个时间内，对于相同的请求返回缓存数据，可以显著提高响应速度。通过适当的错误处理和数据验证，确保系统在面对请求时的稳定性与可靠性。
4. 代码实现
```
const cache = {};
if (cache[id] && Date.now() - cache[id].timestamp < 10000) {
  return res.status(200).json({
    code: 200,
    message:"查询缓存",
    data: cache[id].data，
  })；
}
try{
cache[id] = {
  data : nowdata,
  lastfetchTime: currentTime,
};
return res.status(200).json({
  code: 200,
  message:"成功获取目前概率",
  data:nowdate,
});
}catch(error){
  console.error(error);
  res.status(500).json({
    code: 500,
    message: error.message || "服务器错误",
  });
}
```

## 图鉴页api的开发

### 图鉴卡包api

判断卡牌的张数，并进行记录

### 卡牌api

### 卡池api

### 图鉴和卡牌的关联

## 管理员模型Manager

管理员的数据模型，包括了管理员的基本信息和一些基本的业务逻辑（如自动更新更新时间）。这个模型可以用于管理管理员账户的数据库操作。

## 用户页面api的开发

### 用户模型接口

1. Schema 是 Mongoose 中用于定义数据结构的构造函数
2. function:函数，在模型中可以写生成随机昵称的函数。先定义函数function，然后定义常量随机的内容，再定义一个变量let的随机名字，使用for循环来遍历，const randomIndex = Math.floor(Math.random() * characters.length);：这是循环体内的第一行代码。Math.random()函数生成一个0到1之间的随机数（不包括1），然后乘以characters.length（characters数组的长度）。Math.floor()函数用于取这个乘积的整数部分（向下取整）。这样，randomIndex就是一个0到characters.length - 1之间的整数，用于随机选择characters数组中的一个索引。
3. 用户函数模型，名字，生日，电话号，头像，密码等其中名字等要设置为``` trim: true,```trim是一个模式属性，它指定在保存文档之前应该从字符串字段的两端移除空格
4. 接口导入redis缓存，auth处理身份验证，validate验证请求数据的有效性，user模型，generatecode工具函数用于生成验证码，jwt生成JSON Web Tokens（JWT），express，express-validator请求数据的验证和清理，dotenv从.env文件中加载环境变量的Node.js库，sms工具函数sendSms，用于发送短信，mongoose。
5. 手机号验证
```
  const validateUserPhone = [
    body("phone)
  ]
```
6. 登录logon，使用Validate
7. 发送验证码认证
8. 修改用户信息
9. 用户卡包


### 登录api

中间件的auth的认证，生成token，用于其它api需要用户判断的地方
#### 发送验证码路由

#### 登录路由

1. 引入验证中间件Validate，验证数据是否有效合适。
2. post路由，当用户发送登录请求到/login路径时，会触发这个函数。当用户发送登录请求到/login路径时，会触发这个函数。
3. 从req中提取出登录信息，
4. 进行数据库信息进行验证
5. 生成token
6. 删除redis中的验证码，防止重复使用。

#### auth中间件

1. 目的:是验证请求是否包含有效的令牌（Token），如果有效，就允许请求继续向下执行；如果无效，就返回错误信息。
2. User：这是用户模型，用于数据库中查找用户信息。
3. verifyToken：这是一个函数，用于验证JWT（JSON Web Tokens）的有效性。

#### authFun中间件

1. 这个中间件用于保护路由，确保只有持有有效 Token 的用户才能访问。如果 Token 无效或用户不存在，它会返回 401 未授权状态码。如果验证成功，它会将用户信息添加到请求对象中，并继续执行后续的中间件或路由处理函数。
#### authManager中间件

1. 这个中间件用于保护需要管理员权限的路由，确保只有持有有效 Token 的管理员才能访问。它还使用 Redis 来缓存 Token 的有效性，以减少对数据库的查询次数，提高性能。如果 Token 无效或管理员不存在，它会返回 401 未授权状态码。如果验证成功，它会更新管理员的最后登录时间，并将管理员信息添加到请求对象中，然后继续执行后续的中间件或路由处理函数。


### 支付api

1. 微信支付相关资料的获取
商户号（mch_id）
商户密钥（API Key）
APPID（小程序的 APPID）
APP Secret（小程序的密钥）
1. 小程序支付的工作流程概览
用户点击支付，小程序请求后端生成订单信息。
后端调用微信支付统一下单 API，生成预支付交易单 prepay_id。
小程序根据 prepay_id 调用微信支付组件。
微信支付系统处理支付，并回调通知后端支付结果。
1. 微信接口的写入
anxios：用于HTTP请求
crypto：签名校验
一个下单支付接口
```
const axios = require('axios');
const crypto = require('crypto');

// 商户和小程序信息
const APPID = '你的小程序APPID';
const MCH_ID = '你的商户号';
const API_KEY = '你的API密钥';
const NOTIFY_URL = 'https://你的服务器地址/wechat/notify'; // 支付结果通知地址

// 工具函数：生成签名
function createSign(params) {
  const str = Object.keys(params)
    .sort()
    .map(key => `${key}=${params[key]}`)
    .join('&') + `&key=${API_KEY}`;
  
  return crypto.createHash('md5').update(str).digest('hex').toUpperCase();
}

// 微信统一下单接口
async function createUnifiedOrder(req, res) {
  const { openid, totalFee, outTradeNo } = req.body;

  const params = {
    appid: APPID,
    mch_id: MCH_ID,
    nonce_str: Math.random().toString(36).substr(2, 15),
    body: '订单描述',
    out_trade_no: outTradeNo,
    total_fee: totalFee, // 单位为分
    spbill_create_ip: req.ip,
    notify_url: NOTIFY_URL,
    trade_type: 'JSAPI',
    openid: openid
  };

  // 生成签名并添加到参数
  params.sign = createSign(params);

  try {
    const { data } = await axios.post(
      'https://api.mch.weixin.qq.com/pay/unifiedorder',
      `<xml>
        ${Object.entries(params).map(([key, value]) => `<${key}>${value}</${key}>`).join('')}
      </xml>`,
      { headers: { 'Content-Type': 'application/xml' } }
    );

    const prepayIdMatch = data.match(/<prepay_id><!\[CDATA\[(.*)\]\]><\/prepay_id>/);
    if (prepayIdMatch) {
      res.json({ prepayId: prepayIdMatch[1] });
    } else {
      res.status(500).send('微信下单失败');
    }
  } catch (error) {
    console.error('微信下单接口错误:', error);
    res.status(500).send('微信下单失败');
  }
}

module.exports = createUnifiedOrder;

```
一个支付回调接口
```
const express = require('express');
const bodyParser = require('body-parser');
const crypto = require('crypto');

const router = express.Router();
router.use(bodyParser.text({ type: 'text/xml' }));

// 支付结果通知接口
router.post('/wechat/notify', (req, res) => {
  const xml = req.body;

  // 解析 XML（可使用 xml2js 解析）
  if (xml.includes('<return_code><![CDATA[SUCCESS]]></return_code>')) {
    console.log('支付成功');
    // 在这里更新订单状态
    res.send('<xml><return_code><![CDATA[SUCCESS]]></return_code></xml>');
  } else {
    console.error('支付失败');
    res.send('<xml><return_code><![CDATA[FAIL]]></return_code></xml>');
  }
});

module.exports = router;

```
### 我的背包api

### 我的卡包api

### 地址api

获取，写入，删除，修改，设置为默认地址，同时他们都需要用户认证。

### 用户信息api

### appconfig更新信息的定义

### 自动发送邮件的定义

### 发货订单api

通过我的背包的卡牌中的唯一id进行发货，生成相应的发货订单，发货订单里面的状态可以进行自我调换。

### 抽卡订单api

### 积分订单api

积分商品的兑换，创建，获取。积分订单的创建，获取。

## 入口文件的研究

1. 引入依赖
express：Web 应用框架，用于创建服务器和路由。
path：Node.js 核心模块，用于处理文件路径。
dotenv：用于加载环境变量的模块。
fs：Node.js 核心模块，用于文件系统操作。
2. 配置环境变量
3. 设置别名
4. CORS 中间件
5. 用户代理中间件
6. 路由配置：
引入并使用多个路由模块，这些模块定义了应用程序的不同路由和处理函数。
使用 app.use() 方法将路由模块挂载到服务器上。
7. 中间件配置
8. 启动服务器

## mongodb的使用

### MongoDB的连接登录使用等认知

### MongooseShell代码的使用

### 数据库集合中各种快捷操作和使用

## 代码与服务器的上传

### 服务器的使用

### git和gitlab

## 项目总结

### 框架理解

1. .husky/：Husky是一个Git hooks工具，用于在提交代码之前运行脚本。这个 Husky 脚本的作用是在每次 Git 提交之前，通过 lint-staged 检查所有暂存的文件，确保它们符合项目定义的代码质量标准。如果检查失败，提交将会被阻止，直到代码符合标准。这是一种常见的实践，用于在团队开发中保持代码质量。
2. .vscode/:Visual Studio Code的配置文件，比如代码格式化、linting规则等。
3. basic/：放置一些文件信息
4. node_modules/：这个文件夹是Node.js项目的依赖库文件夹，由npm install命令生成，包含了项目所有依赖的第三方库。
5. src/：这个文件夹通常包含了项目的源代码。
6. types/：TypeScript的类型定义文件。
7. .editorconfig：这是一个配置文件，用于定义代码编辑器的通用配置，如缩进、编码等。
8. .env：这是一个环境变量文件，用于存储项目的配置信息，如数据库连接字符串、API密钥等。
9. .env.development：这是开发环境的环境变量文件，可能包含了开发环境特有的配置。
10. .gitignore：这是一个Git配置文件，用于指定哪些文件或文件夹不应该被Git版本控制。
11. .prettierrc：这是Prettier的配置文件，用于定义代码格式化的规则。
12. basic.json：项目里的草稿本，不计入项目。
13. checkcert.sh：这是一个Shell脚本文件，可能用于检查SSL证书的有效性。
14. ecosystem.config.js：这是PM2的配置文件，PM2是一个Node.js的进程管理器，用于管理和保持应用的持续运行。
15. .eslint.config.js：这是ESLint的配置文件，用于定义JavaScript代码的linting规则。
16. .jest.config.js：这是Jest的配置文件，Jest是一个JavaScript测试框架。
17. nodemon.json：这是Nodemon的配置文件，Nodemon是一个开发工具，用于在开发过程中自动重启Node.js应用。
18. package-lock.json：这是一个由npm生成的文件，用于锁定项目依赖的确切版本。
19. package.json：这是Node.js项目的配置文件，包含了项目的元数据、依赖、脚本等信息。
20. README.md：这是项目的自述文件，通常包含了项目的介绍、安装和使用说明等。
21. tsconfig.json：这是TypeScript的配置文件，用于定义TypeScript编译器的编译选项。

### src文件夹

1. __tests__：用于存放测试代码。
2. cert：包含了SSL证书文件，用于配置HTTPS服务，确保数据传输的安全性。
3. config：用于存放配置文件，数据库配置和redis配置。
4. middleware：这个文件夹用于存放中间件函数。中间件是在请求处理流程中执行特定任务的函数，如身份验证、日志记录、请求解析等。
5. models：这个文件夹用于存放数据模型，如果是使用ORM（对象关系映射）的话，这里会定义与数据库表对应的模型。
6. router：这个文件夹用于存放路由定义。路由是定义应用如何处理不同URL请求的代码，通常与控制器或处理函数相关联。
7. utils：这个文件夹用于存放工具函数或辅助函数，这些函数可以在应用的多个地方被复用，如日期处理、字符串操作等。
8. index.js：这个文件通常是应用的入口点。它可能用于初始化应用、配置服务器、连接数据库、启动路由等。在Express应用中，这个文件通常会创建一个Express实例并开始监听端口。