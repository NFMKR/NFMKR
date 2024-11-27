---
title: QQ调用微信H5支付
date: 2024-11-27 17:21:29
tags:
---
以下是将微信支付接入 QQ 小程序的详细步骤以及需要做的改动。这些内容将帮助你理解如何将微信支付的 JSAPI 支付转换为 H5 支付，适配 QQ 小程序的支付需求。

---

# 微信支付接入 QQ 小程序指南

在将微信小程序的支付功能迁移到 QQ 小程序时，最大的挑战之一是支付方式的适配。微信支付的 JSAPI 支付方式在微信小程序中是直接支持的，但 QQ 小程序只支持 H5 支付方式。本文将详细讲解如何将微信支付的 JSAPI 支付转换为 H5 支付。

## 1. **理解 H5 支付与 JSAPI 支付的区别**

- **JSAPI 支付**：专门为微信小程序提供的支付方式，允许用户直接在小程序内发起支付请求。JSAPI 支付的流程包括用户通过微信小程序发起支付，后端生成支付参数并返回给前端，由前端调用微信支付接口 `wx.requestPayment` 发起支付。

- **H5 支付**：适用于 Web 环境（包括 QQ 小程序的 WebView）。用户通过浏览器（如微信浏览器或 QQ 浏览器）发起支付，支付过程由后端生成支付链接，用户通过该链接进入微信支付页面进行支付。

## 2. **H5 支付的流程**

H5 支付流程包括以下几个步骤：

1. **用户发起支付请求**：用户点击支付按钮，前端通过 API 向后端发送支付请求，后端生成订单。

2. **后端调用微信支付统一下单接口**：后端调用微信支付的统一下单接口，生成支付订单并获取 `prepay_id` 和 `mweb_url`。

3. **生成支付链接**：将支付信息拼接成支付 URL，供前端跳转到微信支付页面。

4. **前端跳转到支付页面**：前端使用 `window.location.href` 或 `wx.navigateTo`（通过 WebView）将用户跳转到微信支付页面。

5. **支付回调与订单更新**：微信支付完成后，会通过支付回调接口通知支付结果，后端接收通知并更新订单状态。

---

## 3. **修改代码实现 H5 支付**

以下是如何将微信支付从 JSAPI 支付转换为 H5 支付的具体实现步骤。

### 3.1 后端请求微信支付统一下单接口

后端通过调用微信支付的统一下单接口，生成订单，并获取支付的 URL (`mweb_url`)。

#### 后端实现（Node.js 示例）

```javascript
const axios = require('axios');

async function createWxPayOrder(openid, amount, order_no) {
  const wxPayParams = {
    appid: WECHAT_APP_ID, // 微信小程序的 AppID
    mch_id: WECHAT_MERCHANT_ID, // 微信商户号
    nonce_str: generateNonceStr(), // 随机字符串
    body: '商品描述', // 商品描述
    out_trade_no: order_no, // 商户订单号
    total_fee: amount,  // 支付金额，单位：分
    spbill_create_ip: '用户IP地址', // 用户的 IP 地址
    notify_url: 'https://your-backend.com/payment_notify',  // 支付成功回调地址
    trade_type: 'MWEB',  // H5支付类型
    payer: { openid: openid },  // 用户的 OpenID
  };

  const sign = generateSign(wxPayParams, API_SECRET); // 生成签名
  wxPayParams.sign = sign;

  const response = await axios.post('https://api.mch.weixin.qq.com/pay/unifiedorder', wxPayParams);

  // 解析返回的 XML 响应并获取 mweb_url
  const mwebUrl = parseXml(response.data).mweb_url;

  return mwebUrl;
}
```

在该代码中，我们通过微信支付的统一下单接口创建支付订单，并获取返回的 `mweb_url`，这是 H5 支付的链接，用户通过该链接发起支付。

### 3.2 前端跳转到支付页面

前端接收到 `mweb_url` 后，可以将其传递给用户，并通过 WebView 或跳转到微信支付页面。

#### 前端实现（QQ 小程序 WebView 示例）

```javascript
// 在 QQ 小程序中，通过 WebView 打开支付页面
wx.navigateTo({
  url: '/pages/webview/webview?url=' + encodeURIComponent(mwebUrl)
});
```

在 Web 环境中，使用 `window.location.href` 跳转到支付页面：

```javascript
// 使用 window.location.href 跳转到支付页面
window.location.href = mwebUrl;
```

### 3.3 支付回调处理

微信支付完成后，会向你的 `notify_url` 发送支付结果通知。你需要在后端处理该回调，验证支付结果，并更新订单状态。

#### 后端处理支付回调

```javascript
// 后端处理支付回调
const wxpay = require('wechatpay-axios-plugin');

app.post('/payment_notify', async (req, res) => {
  try {
    const result = await wxpay.v3.pay.notifications.verify(req.body);
    if (result) {
      // 支付成功，更新订单状态
      const order = await Order.findOne({ out_trade_no: req.body.out_trade_no });
      order.status = 'PAID';
      await order.save();

      // 向微信支付确认通知已处理
      res.send('<xml><return_code><![CDATA[SUCCESS]]></return_code></xml>');
    } else {
      res.send('<xml><return_code><![CDATA[FAIL]]></return_code></xml>');
    }
  } catch (error) {
    console.error('支付回调验证失败:', error);
    res.send('<xml><return_code><![CDATA[FAIL]]></return_code></xml>');
  }
});
```

在这个步骤中，微信支付会将支付结果通过回调通知发送到你的 `notify_url`，后端需要验证通知的签名，并确认支付是否成功。如果支付成功，更新订单状态并回复给微信支付一个成功的响应。

---

## 4. **注意事项**

1. **支付状态检查**：H5 支付方式下，用户支付完成后，你需要在支付回调中确认支付状态。确保订单的状态被正确更新。回调通知是微信支付支付成功的确认依据。

2. **前端跳转**：QQ 小程序通过 WebView 打开支付页面时，要确保 WebView 设置正确，并且微信支付页面能够正常加载。对于其他 Web 环境，可以使用 `window.location.href` 跳转到支付页面。

3. **签名与证书管理**：确保后端签名与证书的安全性，使用 HTTPS 协议来保障数据传输的安全。

4. **微信支付的支付限制**：H5 支付仅支持部分场景，如用户必须使用微信支付的浏览器来完成支付。确保你的用户使用的浏览器支持微信支付。

---

## 5. **总结**

将微信支付从 JSAPI 支付方式转接到 QQ 小程序的 H5 支付方式，主要涉及以下几个步骤：

1. **后端发起支付**：使用微信支付的统一下单接口，指定 `trade_type: 'MWEB'`，获取 `mweb_url` 支付链接。
2. **前端跳转**：将返回的支付链接通过 WebView 或 `window.location.href` 跳转给用户。
3. **支付回调处理**：在支付完成后，微信会向你的 `notify_url` 发送支付回调，你需要验证支付状态并更新订单状态。

通过上述修改，你可以将微信支付集成到 QQ 小程序中，实现 H5 支付功能。