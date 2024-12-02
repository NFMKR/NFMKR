---
title: qq小程序接入微信支付
date: 2024-12-02 10:04:47
tags:
---
### QQ 小程序接入微信支付（H5支付）的详细教程

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
   - **V2版**（H5支付统一下单）：
     - 适用于传统的微信支付方式，只有统一下单接口。
   - **V3版**（包含直连模式、服务商模式、合单支付API）：
     - **直连模式**：适用于商户号直接与微信支付连接。
     - **服务商模式**：适用于服务商为多个商户提供支付服务。
     - **合单支付**：适用于多个子订单合并支付。

#### 7. **调用下单API：V2 版与 V3 版区别**
   - **V2 版（统一下单）**：
     - API 地址：`https://api.q.qq.com/wxpay/unifiedorder?appid=YourQQAppID&access_token=YourAccessToken&real_notify_url=UrlEncodedNotifyUrl`
   - **V3 版（H5下单 API）**：
     - 直连模式：`https://api.q.qq.com/wxpay/v3/pay/transactions/h5?appid=YourQQAppID&access_token=YourAccessToken&real_notify_url=UrlEncodedNotifyUrl`
     - 服务商模式：`https://api.q.qq.com/wxpay/v3/pay/partner/transactions/h5?appid=YourQQAppID&access_token=YourAccessToken&real_notify_url=UrlEncodedNotifyUrl`
     - 合单支付：`https://api.q.qq.com/wxpay/v3/combine-transactions/h5?appid=YourQQAppID&access_token=YourAccessToken&real_notify_url=UrlEncodedNotifyUrl`

#### 8. **请求参数说明**
   - **appid**：QQ 小程序的 appid。
   - **access_token**：通过调用 QQ 小程序后台接口获取的 access_token。
   - **real_notify_url**：微信支付通知回调地址。如果不填写，默认为在 QQ 小程序后台设置的支付回调地址。
   - **notify_url**（V2版）或 **notify_url**（V3版）：支付回调的地址，直接接收微信支付的异步通知。

#### 9. **微信支付的支付回调地址**
   - **V2 版**：`https://api.q.qq.com/wxpay/notify`
   - **V3 版**：`https://api.q.qq.com/wxpay/v3/notify/MCH_ID/OUT_TRADE_NO`
     - `MCH_ID` 和 `OUT_TRADE_NO` 分别为下单时的商户号和订单号。
     - 直连模式填普通商户号和订单号；服务商模式填服务商商户号和订单号；合单支付填合单商户号和订单号。

#### 10. **签名问题**
   - 在 **V3版支付** 中，签名时应使用 **微信支付的原版URL**，不应包含 `appid`, `access_token`, `real_notify_url` 等参数。

#### 11. **错误码及解决方法**
   - **9030**：商户号未绑定。解决方法：在 QQ 小程序开发者管理端绑定商户号。
   - **9031**：notify_url 设置错误。解决方法：按照文档说明正确设置 `notify_url`。
   - **9032**：没有开通微信支付。解决方法：去 QQ 小程序开发者管理端开通微信支付。
   - **9034**：请求 body 解析失败。解决方法：检查请求 body 内容，确保格式正确。

#### 12. **发起微信支付**
   - 通过 `qq.requestWxPayment` 发起支付请求，传入返回的 **H5 支付链接**。当支付完成后，支付结果会通过配置的回调地址发送给你的服务器。

#### 13. **支付结果回调处理**
   - 服务器根据微信支付回调的内容，验证签名，处理支付结果（如更新订单状态）。
   - 确保支付回调地址和服务器逻辑能够正确处理支付状态的变化（如支付成功、支付失败等）。

---

### 总结
接入 **QQ 小程序的微信 H5 支付** 涉及多个步骤，从商户号开通、回调地址配置，到接口的调用与签名。开发者需要了解 **V2版** 和 **V3版** 的区别，并根据需求选择合适的支付方式（直连模式、服务商模式、合单支付）。务必确保回调地址配置正确，且接口调用符合微信支付的要求。

通过以上步骤，你可以成功接入并实现 **微信 H5 支付** 功能，方便 QQ 小程序用户完成支付流程。