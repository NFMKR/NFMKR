---
title: C-JsApi支付
date: 2024-09-27 16:47:54
tags:
categories: 
  - 笔记
---
# 微信小程序微信支付接入步骤详解

微信支付是微信生态系统中重要的支付功能，广泛应用于微信小程序中。本文将详细介绍如何接入微信支付，涵盖从商户号注册、API 配置、到前后端流程的完整实现。通过该步骤，你将能够独立完成微信支付功能的集成。

---

## 1. 商户号注册与配置

### 1.1 注册微信支付商户号
要接入微信支付，首先需要在微信商户平台注册账号。访问 [微信商户平台](https://pay.weixin.qq.com) 完成商户号注册，以下是需要注意的几项配置：

- **商户号（mch_id）**: 每个商户有唯一的商户号。
- **API 密钥（API key）**: 用于签名的密钥，保证交易安全。需要在商户平台配置。
- **API 证书**: 用于与微信支付服务进行安全通信，包含商户的公钥和私钥。
- **小程序的 AppID**: 小程序在微信公众平台申请的唯一标识。
  
### 1.2 获取相关配置信息
在微信支付商户平台获取以下信息：
- **商户号（mch_id）**: 你的商户唯一标识。
- **API 密钥（API key）**: 配置用于签名的密钥。
- **商户证书**: 商户用于与微信支付进行安全通信的证书。
- **小程序 AppID 与 AppSecret**: 用于微信登录、支付的应用标识和密钥。

---

## 2. 安装 SDK 或工具

根据你的后端技术栈，选择适合的微信支付 SDK。微信支付官方提供了多种语言的 SDK，包括：
- **Java SDK**
- **PHP SDK**
- **Node.js SDK** (例如：`wechatpay-axios-plugin`)

### 2.1 安装 Node.js SDK

如果使用 Node.js，可以安装 `wechatpay-axios-plugin`，这是一个基于 Axios 的微信支付插件，支持微信支付 API V3 的调用。

```bash
npm install wechatpay-axios-plugin axios
```

然后，配置证书和密钥。

---

## 3. 后端实现

### 3.1 获取用户的 OpenID

微信支付要求用户的 OpenID 进行身份确认，获取用户的 OpenID 是接入微信支付的关键步骤。用户通过小程序登录后，前端调用 `wx.login` 获取一个临时的 `code`，然后将该 `code` 发送给后端，通过微信的 `jscode2session` 接口获取 OpenID 和 `session_key`。

#### 3.1.1 前端调用微信登录接口

```javascript
// 前端：获取用户的 OpenID
wx.login({
  success(res) {
    if (res.code) {
      // 将 code 发送给后端，后端通过微信接口获取 openid
      wx.request({
        url: 'https://your-backend.com/api/get_openid',
        data: { code: res.code },
        success(response) {
          const openid = response.data.openid;
        }
      });
    }
  }
});
```

#### 3.1.2 后端获取 OpenID

后端使用微信的 `jscode2session` 接口，将 `code` 发送给微信，获取 OpenID。

```javascript
// 后端：获取 OpenID
const axios = require('axios');
const { WECHAT_APP_ID, WECHAT_APP_SECRET } = process.env;

async function getOpenID(code) {
  const url = `https://api.weixin.qq.com/sns/jscode2session?appid=${WECHAT_APP_ID}&secret=${WECHAT_APP_SECRET}&js_code=${code}&grant_type=authorization_code`;

  try {
    const response = await axios.get(url);
    return response.data; // 返回 openid 和 session_key
  } catch (error) {
    throw new Error('获取 OpenID 失败');
  }
}
```

---

### 3.2 创建微信支付订单

用户获取到 `openid` 后，后端根据商品信息（如商品 ID 和价格）生成订单，并向微信支付发起请求。支付订单会返回一个 `prepay_id`，该 ID 用于生成支付参数。

#### 3.2.1 生成订单号

为了保证订单的唯一性，一般使用自定义的订单号生成规则，如结合时间戳和随机数。

```javascript
const { v4: uuidv4 } = require('uuid');

function generateCustomOrderNo(prefix) {
  return prefix + Date.now() + uuidv4();
}
```

#### 3.2.2 创建支付订单

使用微信支付的 `transactions.jsapi` 接口创建订单，生成支付所需的 `prepay_id`。

```javascript
const wxpay = new Wechatpay({
  mchid: WECHAT_MERCHANT_ID,
  serial: WECHAT_SERIAL_NO,
  privateKey: PRIVATE_KEY,
  certs: CERTS, // 动态加载证书
});

// 构建支付参数
const orderParams = {
  appid: WECHAT_APP_ID,
  mchid: WECHAT_MERCHANT_ID,
  description: '商品描述',
  out_trade_no: order_no, // 商户订单号
  amount: {
    total: amount_in_cents, // 支付金额，单位：分
    currency: 'CNY',
  },
  notify_url: 'https://your-backend.com/api/payment_notify', // 支付成功回调地址
  payer: {
    openid: openid, // 用户的 OpenID
  },
};

// 向微信支付 API 发起请求，生成预支付订单
const result = await wxpay.v3.pay.transactions.jsapi.post(orderParams);
```

---

### 3.3 生成支付签名

微信支付要求订单的支付参数必须进行签名，确保支付请求的数据完整性和安全性。使用商户的私钥进行签名。

```javascript
const { Rsa, Formatter } = require('wechatpay-axios-plugin');

// 支付参数生成
const newParams = {
  appId: WECHAT_APP_ID,
  timeStamp: `${Formatter.timestamp()}`,
  nonceStr: Formatter.nonce(),
  package: `prepay_id=${result.data.prepay_id}`,
  signType: 'RSA',
  orderId: order_no,
};

// 使用商户私钥生成签名
newParams.paySign = Rsa.sign(
  Formatter.joinedByLineFeed(
    newParams.appId,
    newParams.timeStamp,
    newParams.nonceStr,
    newParams.package
  ),
  PRIVATE_KEY
);
```

将生成的支付参数返回给前端，用于发起支付请求。

```javascript
res.json({
  code: 200,
  message: '订单创建成功',
  data: newParams,
});
```

---

## 4. 前端支付流程

### 4.1 调用微信支付

前端通过微信小程序的 `wx.requestPayment` 发起支付请求，传递后端返回的支付参数。

```javascript
wx.requestPayment({
  timeStamp: paymentParams.timeStamp,
  nonceStr: paymentParams.nonceStr,
  package: paymentParams.package,
  signType: 'RSA',
  paySign: paymentParams.paySign,
  success(res) {
    console.log('支付成功', res);
  },
  fail(err) {
    console.log('支付失败', err);
  }
});
```

### 4.2 支付成功与失败处理

- **成功**: 用户支付成功后，显示支付成功页面，并根据订单号查询订单状态。
- **失败**: 如果支付失败，用户可以重新发起支付，或者提示支付失败。

---

## 5. 支付成功回调

微信支付会在支付完成后，向你设置的 `notify_url` 发送支付结果通知。你需要验证通知并更新订单状态。

### 5.1 支付结果回调处理

微信支付发送的回调通知包含支付结果信息（如 `transaction_id` 和 `out_trade_no`）。后端需要验证签名，并根据支付结果更新订单状态。

```javascript
app.post('/api/payment_notify', async (req, res) => {
  try {
    const result = await wxpay.v3.pay.notifications.verify(req.body);
    if (result) {
      // 验证成功，更新订单状态
      const order = await Order.findOne({ out_trade_no: req.body.out_trade_no });
      order.status = 'PAID';
      await order.save();
      res.send('<xml><return_code><![CDATA[SUCCESS]]></return_code></xml>');
    }
  } catch (error) {
    console.error('支付回调验证失败:', error);
    res.send('<xml><return_code><![CDATA[FAIL]]></return_code></xml>');
  }
});
```
### 5.2 回调讲解

1. 首先，后端接收微信支付回调通知，并进行签名验证。
2. 如果验证成功，则更新订单状态为已支付。
3. 回调给的地址其实就是一个接口，收到微信回调的信息后将信息按照接口内容进行相应的处理。
4. 如果要将一些字段映射到数据库中，先将微信回调的数据进行定义解析，然后将解析后的数据映射到数据库
---

## 6. 调试与上线

- **调试环境**: 微信支付提供沙箱环境，用于测试支付功能。确保开发环境和正式环境的配置一致。
- **安全性**: 确保商户密钥、证书的安全存储，并使用 HTTPS 协议加密通信。
- **时间同步**: 微信支付接口对时间有严格要求，确保服务器时间与微信服务器时间同步。

---

## 总结

微信小程序微信支付接入的步骤包括：
1. **注册商户号并配置 API 信息**: 注册

并获取商户号、API 密钥、证书等。
2. **前端获取 OpenID**: 通过 `wx.login` 获取用户 `code`，后端获取 OpenID。
3. **后端生成支付订单**: 使用微信支付 API 创建订单，并获取 `prepay_id`。
4. **支付签名与返回**: 对支付参数进行签名，并返回前端。
5. **前端发起支付**: 使用 `wx.requestPayment` 发起支付请求。
6. **支付回调与订单更新**: 微信支付支付成功后，通过回调接口通知支付结果，更新订单状态。

通过这些步骤，你可以顺利将微信支付集成到小程序中，实现金融级的支付功能。