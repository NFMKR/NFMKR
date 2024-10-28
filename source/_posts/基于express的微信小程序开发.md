---
title: 基于express的微信小程序开发
date: 2024-10-24 15:13:45
tags:
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

### 抽卡卡包封面api

### 抽卡机api

### 抽卡卡池概率实现

抽到的卡要关联卡池/我的背包，卡包

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

### 图鉴和卡牌的关联

## 用户页面api的开发

### 登录api

中间件的auth的认证，生成token，用于其它api需要用户判断的地方

### 支付api

1. 微信支付相关资料的获取
商户号（mch_id）
商户密钥（API Key）
APPID（小程序的 APPID）
APP Secret（小程序的密钥）
2. 小程序支付的工作流程概览
用户点击支付，小程序请求后端生成订单信息。
后端调用微信支付统一下单 API，生成预支付交易单 prepay_id。
小程序根据 prepay_id 调用微信支付组件。
微信支付系统处理支付，并回调通知后端支付结果。
3. 微信接口的写入
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

### 发货订单api

通过我的背包的卡牌中的唯一id进行发货，生成相应的发货订单，发货订单里面的状态可以进行自我调换。

### 抽卡订单api

## mongodb的使用

### MongoDB的连接登录使用等认知

### MongooseShell代码的使用

### 数据库集合中各种快捷操作和使用

## 代码与服务器的上传

### 服务器的使用

### git和gitlab
