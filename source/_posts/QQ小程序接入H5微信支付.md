---
title: qq小程序接入微信支付
date: 2024-12-02 10:04:47
tags:
---
#### 官方文档链接

1. [H5接入文档](https://q.qq.com/wiki/develop/miniprogram/server/virtual-payment/wx_pay.html#%E6%8E%A5%E5%85%A5%E6%AD%A5%E9%AA%A4)
2. [接口调用凭证getAccessToken](https://q.qq.com/wiki/develop/game/server/open-port/getAccessToken.html)
3. [H5下单API](https://pay.weixin.qq.com/wiki/doc/apiv3/apis/chapter3_3_1.shtml)
4. [H5支付开发指引](https://pay.weixin.qq.com/wiki/doc/apiv3/open/pay/chapter2_6_2.shtml)

# **QQ小程序接入微信支付（H5支付）教程**

### 目录
**1. 从零到完成的接入步骤**
**2. jsapi转到h5支付的步骤**

## **一、从0到完成的接入步骤**

#### 1. **开通微信支付**
   - 在 **微信支付商户平台** 开通具备 H5 支付能力的微信支付商户号。
   - 开通路径：
     - 登录微信支付商户平台
     - 进入 **产品中心** -> **我的产品** -> **H5支付**
     - 申请开通 **H5支付**，然后设置好商户号。

#### 2. **绑定微信支付商户号**
   - 将微信支付商户号与微信 AppID 绑定。
   - 开通路径：
     - 进入微信支付商户平台 -> **产品中心** -> **AppID账号管理**
     - 绑定你的 **微信 AppID**，确保该商户号与微信 AppID 关联。

#### 3. **在 QQ 小程序后台开通微信支付**
   - 登录 **QQ 小程序开发者管理端**，进入 **支付接入** -> **微信支付**。
   - 在 **QQ 小程序开发者管理端**中绑定微信支付商户号。
   - 设置 **支付结果通知回调地址**，该地址接收微信支付结果的通知。

#### 4. **获取微信支付的 H5 支付跳转链接**
   - 使用 QQ 小程序提供的 **代理下单 API** 获取微信支付跳转链接。
   - 需要调用 **QQ 小程序后台的统一下单接口**，并在请求中传递相关参数（如 `appid`, `access_token`, `notify_url` 等）来获取支付链接。

#### 5. **调用 `qq.requestWxPayment` 来发起支付**
   - 在 QQ 小程序中，你不能直接使用微信支付的原生接口呼起支付，而是需要通过调用 `qq.requestWxPayment` 来启动微信支付。
   - `qq.requestWxPayment` 需要传入由后台下单接口返回的 **H5 支付跳转链接**。

#### 6. **接入的支付API版本选择**
   - **V3版**（包含直连模式、服务商模式、合单支付API）：
     - **直连模式**：适用于商户号直接与微信支付连接。
     - URL：`https://api.q.qq.com/wxpay/v3/pay/transactions/h5?appid=YourQQAppID&access_token=YourAccessToken&real_notify_url=UrlEncodedNotifyUrl`

#### 7. **请求参数说明**
   - **appid**：QQ 小程序的 appid。
   - **access_token**：通过调用 QQ 小程序后台接口获取的 access_token。
   - **real_notify_url**：微信支付通知回调地址。如果不填写，默认为在 QQ 小程序后台设置的支付回调地址。
   - **notify_url**（V2版）或 **notify_url**（V3版）：支付回调的地址，直接接收微信支付的异步通知。

#### 8. **微信支付的支付回调地址**
   - **V2 版**：`https://api.q.qq.com/wxpay/notify`
   - **V3 版**：`https://api.q.qq.com/wxpay/v3/notify/MCH_ID/OUT_TRADE_NO`
     - `MCH_ID` 和 `OUT_TRADE_NO` 分别为下单时的商户号和订单号。
     - 直连模式填普通商户号和订单号。

#### 9. **签名问题**
   - 在 **V3版支付** 中，签名时应使用 **微信支付的原版URL**，不应包含 `appid`, `access_token`, `real_notify_url` 等参数。
   - 其中有一种说法：不需要自己进行签名，qq的透明转发会自动进行签名。

###### 9.1 **透明转发签名问题**

   1. 在微信支付回调中，微信会自动进行签名验证，不需要自己进行签名。
   2. 透明转发（Transparent Forwarding）是指QQ小程序后台在处理与微信支付的对接时，对于某些接口（如统一下单和支付回调），会自动将这些请求转发到微信支付后台。开发者不需要直接与微信支付的 API 进行交互，QQ小程序会代为处理请求和响应。这意味着开发者不需要手动处理请求的签名和验证，QQ小程序会自动为你签名并发起请求。
   3. 透明转发适用于以下接口：
统一下单接口：通过这个接口，开发者发起支付请求，QQ小程序后台会代为转发请求并返回相应结果（如 prepay_id 等）。
支付回调接口：当微信支付处理完支付时，会向开发者的服务器发送回调通知，QQ小程序后台会将此通知转发给开发者。
   3. 透明转发的工作原理
统一下单：
开发者只需要发起一个包含必要参数的请求到QQ小程序后台（如产品ID、金额、订单号等），QQ小程序会自动处理签名、请求发送、获取 prepay_id 等。
返回的 prepay_id 可以用于生成前端支付参数。
支付回调：
支付完成后，微信支付会通过回调通知开发者。QQ小程序会将支付回调结果（支付成功或失败）转发到开发者指定的回调 URL。

#### 10. **错误码及解决方法**
   - **9030**：商户号未绑定。解决方法：在 QQ 小程序开发者管理端绑定商户号。
   - **9031**：notify_url 设置错误。解决方法：按照文档说明正确设置 `notify_url`。
   - **9032**：没有开通微信支付。解决方法：去 QQ 小程序开发者管理端开通微信支付。
   - **9034**：请求 body 解析失败。解决方法：检查请求 body 内容，确保格式正确。

#### 11. **发起微信支付**
   - 通过 `qq.requestWxPayment` 发起支付请求，传入返回的 **H5 支付链接**。当支付完成后，支付结果会通过配置的回调地址发送给你的服务器。

#### 12. **支付结果回调处理**
   - 服务器根据微信支付回调的内容，验证签名，处理支付结果（如更新订单状态）。
   - 确保支付回调地址和服务器逻辑能够正确处理支付状态的变化（如支付成功、支付失败等）。
---

在 H5支付中，获取到跳转链接后，后续的流程是自动化的，基本上可以按照以下几个步骤进行描述：

#### 13. **用户点击支付链接**
   - 当用户在你的小程序中发起支付请求时，后端会生成一个支付链接，这个链接通常是一个指向微信支付页面的URL。你可以把这个链接展示给用户，或者通过重定向、弹窗等方式引导用户点击进入支付页面。

#### 14. **用户跳转到微信支付页面**
   - 用户点击支付链接后，会自动跳转到微信支付的H5支付页面。这个页面是由微信支付平台渲染和处理的，包括支付金额、订单信息、支付方式选择等内容。
   - 在这个过程中，用户无需手动输入支付信息，微信支付会自动获取用户的账户信息（如openid）并显示支付界面。用户可以选择支付方式（如微信支付、银行卡等）并完成支付。

#### 15. **用户完成支付**
   - 用户在微信支付页面完成支付后，微信会根据支付结果自动跳转回你的回调地址（即`notify_url`）。这时，支付流程就会结束，微信会通过后台回调通知你的服务器支付是否成功。

#### 16. **支付结果回调（`notify_url`）**
   - 支付成功后，微信支付会向你在支付请求时指定的回调 URL（即 `notify_url`）发送支付结果的通知。
   - 回调消息中会包含订单的支付状态信息（如 `trade_state`），你可以根据这个信息判断支付是否成功。

#### 17. **后端处理支付结果**
   - 当你的服务器收到支付回调通知后，可以进行验证（比如签名验证）并确认支付状态。
   - 如果支付成功，后端通常会更新订单状态为“已支付”，并可能触发一些业务流程（如发货、账户充值等）。

#### 18. **返回支付结果给前端**
   - 在支付完成后，你可以通过前端的接口或者再次引导用户跳转到特定页面，告诉用户支付结果。
   - 如果支付成功，可以跳转到订单详情页、支付成功页面，或者给用户一个“支付成功”的提示。
   - 如果支付失败，则可以跳转到支付失败页面，或者显示失败信息，并引导用户重新支付。

#### 典型流程总结：
1. **生成支付链接**：后端生成支付请求并返回给前端一个支付链接。
2. **用户跳转支付页面**：用户点击链接后跳转到微信支付页面。
3. **用户完成支付**：用户选择支付方式并完成支付。
4. **支付结果通知**：微信支付通过回调通知你的后端支付结果。
5. **处理支付结果**：后端确认支付状态，并更新订单信息。

#### 实际场景中的一些细节：
- **支付页面展示**：在微信支付页面，用户无需登录，因为微信会自动通过浏览器或微信客户端获取用户的身份信息（如微信账号）。
- **支付状态**：如果用户支付完成，支付页面通常会提示支付成功；如果支付失败，则会提示用户支付失败，并允许重新尝试支付。
- **支付中断或超时**：如果用户在支付过程中中断或支付超时，通常也会显示相关提示。

总结来说，H5支付的后续步骤相对简洁，重点在于后端如何接收支付回调并处理支付结果，前端则只需要提供支付链接并展示支付界面。微信支付会处理大部分细节，确保支付体验顺畅。

---

### 总结
接入 **QQ 小程序的微信 H5 支付** 涉及多个步骤，从商户号开通、回调地址配置，到接口的调用与签名。开发者需要了解 **V2版** 和 **V3版** 的区别，并根据需求选择合适的支付方式（直连模式、服务商模式、合单支付）。务必确保回调地址配置正确，且接口调用符合微信支付的要求。


## **二、jsapi转到h5支付的步骤**
### 微信支付接入 QQ 小程序指南

在将微信小程序的支付功能迁移到 QQ 小程序时，最大的挑战之一是支付方式的适配。微信支付的 JSAPI 支付方式在微信小程序中是直接支持的，但 QQ 小程序只支持 H5 支付方式。本文将详细讲解如何将微信支付的 JSAPI 支付转换为 H5 支付。

#### 1. **理解 H5 支付与 JSAPI 支付的区别**

- **JSAPI 支付**：专门为微信小程序提供的支付方式，允许用户直接在小程序内发起支付请求。JSAPI 支付的流程包括用户通过微信小程序发起支付，后端生成支付参数并返回给前端，由前端调用微信支付接口 `wx.requestPayment` 发起支付。

- **H5 支付**：适用于 Web 环境（包括 QQ 小程序的 WebView）。用户通过浏览器（如微信浏览器或 QQ 浏览器）发起支付，支付过程由后端生成支付链接，用户通过该链接进入微信支付页面进行支付。

#### 2. **H5 支付的流程**

H5 支付流程包括以下几个步骤：

1. **用户发起支付请求**：用户点击支付按钮，前端通过 API 向后端发送支付请求，后端生成订单。

2. **后端调用微信支付统一下单接口**：后端调用微信支付的统一下单接口，生成支付订单并获取 `prepay_id` 和 `mweb_url`。

3. **生成支付链接**：将支付信息拼接成支付 URL，供前端跳转到微信支付页面。

4. **前端跳转到支付页面**：前端使用 `window.location.href` 或 `wx.navigateTo`（通过 WebView）将用户跳转到微信支付页面。

5. **支付回调与订单更新**：微信支付完成后，会通过支付回调接口通知支付结果，后端接收通知并更新订单状态。

---

#### 3. **修改代码实现 H5 支付**

以下是如何将微信支付从 JSAPI 支付转换为 H5 支付的具体实现步骤。

##### 3.1 后端请求微信支付统一下单接口

后端通过调用微信支付的统一下单接口，生成订单，并获取支付的 URL (`mweb_url`)。

###### 后端实现（Node.js 示例）

```javascript
const axios = require('axios');

async function createWxPayOrder(amount, order_no) {
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

##### 3.2 前端跳转到支付页面

前端接收到 `mweb_url` 后，可以将其传递给用户，并通过 WebView 或跳转到微信支付页面。

###### 前端实现（QQ 小程序 WebView 示例）

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

##### 3.3 支付回调处理

微信支付完成后，会向你的 `notify_url` 发送支付结果通知。你需要在后端处理该回调，验证支付结果，并更新订单状态。

###### 后端处理支付回调

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

#### 4. **注意事项**

1. **支付状态检查**：H5 支付方式下，用户支付完成后，你需要在支付回调中确认支付状态。确保订单的状态被正确更新。回调通知是微信支付支付成功的确认依据。

2. **前端跳转**：QQ 小程序通过 WebView 打开支付页面时，要确保 WebView 设置正确，并且微信支付页面能够正常加载。对于其他 Web 环境，可以使用 `window.location.href` 跳转到支付页面。

3. **签名与证书管理**：确保后端签名与证书的安全性，使用 HTTPS 协议来保障数据传输的安全。

4. **微信支付的支付限制**：H5 支付仅支持部分场景，如用户必须使用微信支付的浏览器来完成支付。确保你的用户使用的浏览器支持微信支付。

---

#### 5. **总结**

将微信支付从 JSAPI 支付方式转接到 QQ 小程序的 H5 支付方式，主要涉及以下几个步骤：

1. **后端发起支付**：使用微信支付的统一下单接口，指定 `trade_type: 'MWEB'`，获取 `mweb_url` 支付链接。
2. **前端跳转**：将返回的支付链接通过 WebView 或 `window.location.href` 跳转给用户。
3. **支付回调处理**：在支付完成后，微信会向你的 `notify_url` 发送支付回调，你需要验证支付状态并更新订单状态。

通过上述修改，你可以将微信支付集成到 QQ 小程序中，实现 H5 支付功能。


### 三、步骤总结
### 1. 获取 `access_token`
首先，你需要使用 QQ 小程序的 `appid` 和 `secret`，以及 `grant_type` 为 `client_credential` 来获取 `access_token`。你会通过 `https://api.q.qq.com/api/getToken` 调用获取 `access_token`。

- **请求方法**：GET
- **请求URL**：`https://api.q.qq.com/api/getToken`
- **请求参数**：
  - `grant_type`: 固定为 `client_credential`
  - `appid`: 你的小程序的 AppID
  - `secret`: 你的小程序的 AppSecret

### 2. 生成支付订单
一旦获得 `access_token`，你就可以利用它来生成微信支付订单。具体步骤如下：

- **请求方法**：POST
- **请求URL**：`https://api.q.qq.com/wxpay/v3/pay/transactions/h5`
- **请求参数**：
  - `appid`: 你的 QQ 小程序的 `appid`
  - `access_token`: 获取的 `access_token`
  - `real_notify_url`: 支付成功后微信支付回调的通知地址
  - 其他支付相关的参数，比如订单号、商品描述、金额、用户IP、支付场景（H5支付等）

**请求体示例**：
```json
{
  "mchid": "1900006XXX",
  "out_trade_no": "H51217752501201407033233368018",
  "appid": "wxdace645e0bc2cXXX",
  "description": "Image形象店-深圳腾大-QQ公仔",
  "notify_url": "https://your.callback.url",
  "amount": {
    "total": 1,
    "currency": "CNY"
  },
  "scene_info": {
    "payer_client_ip": "127.0.0.1",
    "h5_info": {
      "type": "Wap"
    }
  }
}
```

### 3. 支付签名
在生成订单后，你需要按照微信支付的要求对订单参数进行签名。

- **签名算法**：通常使用 `RSA` 或者 `HMAC-SHA256`。
- 你需要对请求的部分参数进行签名，并将签名附加到请求中，以确保请求的安全性。

### 4. 支付跳转 URL
微信支付接口会返回一个支付跳转链接，你可以将该链接传递给前端，由前端调用微信支付。

**返回字段示例**：
```json
{
  "h5_url": "https://wx.tenpay.com/cgi-bin/mmpayweb-bin/checkmweb?prepay_id=wx2016121516420242444321ca0631331346&package=1405458241"
}
```

### 5. 支付回调地址
在支付完成后，微信支付会向你的回调地址发送支付结果通知。你需要设置回调地址来处理支付成功或失败的情况。

**回调地址示例**：
- **V2**：`https://api.q.qq.com/wxpay/notify`
- **V3**：`https://api.q.qq.com/wxpay/v3/notify/MCH_ID/OUT_TRADE_NO`

你需要在接收到回调时验证支付结果，确保订单状态正确，并根据支付结果进行后续处理。

### 总结
从你的描述来看，基本的步骤已经涵盖了：
1. 获取 `access_token`。
2. 通过 `access_token` 和其他支付参数生成微信支付订单。
3. 进行支付签名。
4. 返回支付跳转链接。
5. 配置支付回调地址并处理回调。

确认这些步骤后，主要的工作内容应该已经涵盖了。如果有任何步骤未完成，或有其它细节问题可以进一步沟通。


## 四、h5回调接口编写

### `h5callback.js` 修改总结

本次修改旨在将现有的微信小程序使用的 JSAPI 支付回调逻辑，迁移并适配到 QQ 小程序中使用的 H5 微信支付回调。主要目标是移除对 `openid` 的依赖，简化订单处理流程，并确保支付回调的安全性与数据完整性。

#### 关键修改点

1. **移除 `openid` 相关逻辑**：
   - H5 支付无需 `openid`，因此在回调处理中完全移除了获取和使用 `openid` 的代码。
   - 发货接口调用时，`openid` 参数设为 `null` 或根据业务需求使用其他标识（如 `userId`）。

2. **简化订单处理**：
   - 通过 `out_trade_no`（订单号）直接关联和处理订单，确保逻辑简洁且高效。
   - 从 Redis 获取订单信息，若不存在则从 MongoDB 中查找，避免重复处理。

3. **保持数据安全性**：
   - 继续使用签名验证和数据解密，确保回调请求的合法性和数据的完整性。
   - 使用最新的微信支付公钥证书进行验签，防止伪造请求。

4. **优化积分与优惠券处理**：
   - 积分处理基于 `userId`，无需依赖 `openid`。
   - 更新用户积分时，仅通过 `userId` 进行增减，确保操作的准确性。
   - 优惠券状态更新依然保留，确保用户优惠券的正确使用和记录。

5. **调整发货逻辑**：
   - 调用 `wx_deliver` 接口时，不再需要 `openid`，可根据 `orderNo` 或 `userId` 进行发货操作。
   - 确保发货过程与用户标识的无缝对接，提升系统的灵活性。

6. **错误处理与日志记录**：
   - 保持详细的错误处理和日志记录，便于后续调试和维护。
   - 捕捉并记录回调处理中的任何异常，确保系统的稳定性。

#### 代码结构调整

- **签名验证与数据解密**：
  - `verifyAndDecrypt` 中间件继续负责验证微信支付的签名，并解密加密的数据。
  - 确保解密后的数据结构与 H5 支付回调的数据一致，避免数据解析错误。

- **订单状态更新**：
  - 根据支付状态 (`SUCCESS` 或其他) 更新订单和配送订单的状态。
  - 清理 Redis 缓存，防止重复处理相同订单。

- **积分处理**：
  - 计算并更新用户的积分，不再依赖于 `openid`，而是通过 `userId` 直接操作用户积分。

- **发货操作**：
  - 调用 `wx_deliver` 时，不传递 `openid`，改用其他标识来执行发货逻辑，确保用户收到相应物品或服务。

#### 基础知识回顾

1. **H5 微信支付流程**：
   - **前端**：用户在 QQ 小程序中发起支付，前端获取 `mweb_url` 并跳转完成支付。
   - **后端**：生成预支付订单，返回 `mweb_url`，处理支付成功后的回调。

2. **签名验证**：
   - 验证回调请求的签名，确保数据未被篡改。
   - 使用微信支付提供的公钥证书进行验签。

3. **数据存储与缓存**：
   - **Redis**：用于快速访问和更新订单缓存，提升系统性能。
   - **MongoDB**：用于持久化存储订单、用户信息及相关数据，确保数据一致性和可靠性。

4. **异步编程与错误处理**：
   - 使用 `async`/`await` 简化异步操作，提高代码可读性。
   - 通过 `try...catch` 捕捉并处理异步操作中的错误，保持系统稳定运行。

#### 总结

通过上述修改，`h5callback.js` 现已成功适配 QQ 小程序的 H5 微信支付回调，不再依赖 `openid`，简化了订单处理流程，同时保持了系统的安全性和稳定性。关键调整包括：

- **移除 `openid` 依赖**，简化发货和积分逻辑。
- **优化订单处理逻辑**，确保仅通过 `out_trade_no` 正确关联订单状态。
- **保持数据安全**，继续使用签名验证和数据解密，确保回调数据的真实性。
- **提升系统灵活性**，通过使用 `userId` 等标识进行业务逻辑操作。


## 前后端完整步骤

要在 **QQ 小程序** 中接入 **微信 H5 支付**，需要以下完整步骤：

---

### 一、前置条件
1. **QQ 小程序配置**：
   - 确保 QQ 小程序已经完成账号注册，并获取了 AppID 和 AppSecret。

2. **微信支付配置**：
   - 微信支付商户号（MchID）已开通，并完成配置。
   - 获取微信支付的 API 密钥（API Key）和证书（如果需要）。

3. **后端准备**：
   - 准备一个中间服务器，用于生成预支付订单 (`prepay_id`) 和验证支付签名。

---

### 二、接入步骤

#### 1. 在微信支付平台添加域名
确保后端接口域名已经在微信支付后台 **支付设置** > **支付授权目录** 中配置。

---

#### 2. 前端请求支付接口

在 QQ 小程序中，用户触发支付行为时（如点击“立即支付”按钮）发送请求到后端：

```javascript
qq.request({
  url: 'https://your-server-domain/createOrder', // 请求后端接口
  method: 'POST',
  data: {
    orderId: 'your_order_id', // 订单号
    totalFee: 100, // 金额（分）
    body: '订单描述',
  },
  success: (res) => {
    if (res.data.success) {
      // 获取后端返回的支付参数
      const paymentData = res.data.paymentData;
      // 调用 H5 支付页面
      qq.navigateTo({
        url: `/pages/webview/webview?url=${encodeURIComponent(paymentData.mweb_url)}`,
      });
    } else {
      console.error('支付订单创建失败:', res.data.message);
    }
  },
  fail: (err) => {
    console.error('请求失败:', err);
  },
});
```

---

#### 3. 后端生成微信支付订单

在后端，调用微信支付统一下单接口生成 `prepay_id` 和 H5 支付链接 (`mweb_url`)。

**示例代码（Node.js）**：

```javascript
const axios = require('axios');
const crypto = require('crypto');
const xml2js = require('xml2js');

// 生成随机字符串
function generateNonceStr() {
  return crypto.randomBytes(16).toString('hex');
}

// 创建支付订单
async function createWeChatOrder(req, res) {
  const { orderId, totalFee, body } = req.body;

  const params = {
    appid: 'your_wechat_appid', // 微信公众账号 AppID
    mch_id: 'your_merchant_id', // 微信商户号
    nonce_str: generateNonceStr(),
    body,
    out_trade_no: orderId,
    total_fee: totalFee,
    spbill_create_ip: req.ip, // 用户的真实 IP
    notify_url: 'https://your-server-domain/wechat/notify', // 支付回调通知
    trade_type: 'MWEB', // H5 支付类型
  };

  // 签名生成
  const stringToSign = Object.keys(params)
    .sort()
    .map((key) => `${key}=${params[key]}`)
    .join('&') + `&key=your_api_key`;
  params.sign = crypto.createHash('md5').update(stringToSign).digest('hex').toUpperCase();

  // 发送请求到微信支付
  try {
    const xmlData = new xml2js.Builder().buildObject(params);
    const { data } = await axios.post('https://api.mch.weixin.qq.com/pay/unifiedorder', xmlData, {
      headers: { 'Content-Type': 'application/xml' },
    });

    // 解析返回 XML
    const result = await xml2js.parseStringPromise(data, { explicitArray: false });
    if (result.return_code === 'SUCCESS' && result.result_code === 'SUCCESS') {
      return res.json({ success: true, paymentData: result });
    }

    res.json({ success: false, message: result.return_msg });
  } catch (error) {
    console.error('微信支付请求失败:', error);
    res.json({ success: false, message: '微信支付请求失败' });
  }
}
```

---

#### 4. 配置支付结果回调接口

微信支付会在支付完成后调用你设置的 `notify_url`，携带支付结果数据。你需要验证签名并更新订单状态。

**回调验证示例代码**：

```javascript
app.post('/wechat/notify', async (req, res) => {
  const xmlData = req.body;

  // 解析 XML
  const result = await xml2js.parseStringPromise(xmlData, { explicitArray: false });

  // 验签
  const signData = { ...result };
  delete signData.sign;
  const stringToSign = Object.keys(signData)
    .sort()
    .map((key) => `${key}=${signData[key]}`)
    .join('&') + `&key=your_api_key`;
  const calculatedSign = crypto.createHash('md5').update(stringToSign).digest('hex').toUpperCase();

  if (calculatedSign !== result.sign) {
    console.error('签名验证失败');
    return res.send('<xml><return_code>FAIL</return_code></xml>');
  }

  // 更新订单状态
  if (result.result_code === 'SUCCESS') {
    // 更新数据库订单为已支付
    console.log(`订单 ${result.out_trade_no} 支付成功`);
  }

  res.send('<xml><return_code>SUCCESS</return_code></xml>');
});
```

---

#### 5. 用户完成支付跳转

在用户完成支付后，微信支付 H5 页面会返回到支付结果页面。确保页面状态处理得当，并引导用户返回 QQ 小程序。

---

### 三、注意事项

1. **跨域问题**：H5 支付链接需要通过 QQ 小程序的 WebView 显示，因此需要确保 WebView 页面和后端域名已配置。
   
2. **安全性**：
   - 严格验证微信支付回调的签名。
   - 后端接口需要通过 HTTPS 提供服务。

3. **兼容性**：
   - QQ 小程序内的 WebView 页面可能受限制，测试是否可以正常加载微信支付页面。

4. **测试环境**：
   - 使用微信支付提供的沙箱环境，进行测试订单创建和支付验证。

---

### 四、流程图

1. 用户发起支付请求 → 2. QQ 小程序调用后端生成订单 → 3. 后端请求微信统一下单接口 → 4. 返回 H5 支付链接 → 5. WebView 显示支付页面 → 6. 微信支付回调通知更新订单状态。

通过这些步骤，QQ 小程序可以顺利集成微信 H5 支付功能。