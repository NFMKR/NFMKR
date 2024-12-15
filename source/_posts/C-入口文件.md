---
title: 入口文件
date: 2024-12-10 10:59:41
categories: 
  - 笔记
---
# 入口文件
  - 作用：一个基于 Express 框架的 Node.js 服务的入口文件，主要作用是启动和配置一个服务器，处理各种路由请求，并集成一些外部模块与功能。

## 引入依赖模块
  - express: 引入 Express 框架，Express 是一个 Web 应用框架，用于快速构建 Web 应用程序，简化 HTTP 请求的处理、路由管理等功能。`const express = require("express");`
  - path: Node.js 内置模块，用于处理文件路径。`const path = require("path");`
  - dotenv: 用于加载 .env 文件中的环境变量,通过环境变量可以配置不同的设置，如数据库连接信息、API 密钥等。`const  dotenv = require("dotenv");`
  - fs: Node.js 内置模块，用于文件系统操作，如读写文件。`const fs = require("fs");`

## 加载环境变量
  - dotenv.config(): 读取 .env 文件中的环境变量，并将其加载到 process.env 中。
  ```
  dotenv.config();
  rrquire("dotenv").config({
    path: `.env.${process.env.NODE_ENV}`,
  });
  ```
## 输出 ASCII 艺术与环境信息
  - 输出 ASCII 艺术，并显示当前环境信息。
  - 这部分用于输出一段 ASCII 艺术的字符（通常用于调试或增加趣味性），以及当前的运行环境 (process.env.NODE_ENV)，有助于了解当前服务器是在哪个环境下运行的。
  ```
  console.log("...");  // 输出一段 ASCII 艺术的文字
  console.info("当前环境是", process.env.NODE_ENV);
  ```

## 设置模块别名
  - module-alias: 这是一个第三方包，它可以让你为文件路径设置别名。比如，通过 @router 可以直接引用 ./router 文件夹中的内容，而不需要写长的相对路径。
  - path.resolve(__dirname, "./"): 这是获取当前文件目录的绝对路径，用于配置别名。
```javascript
require("module-alias/register");
const aliases = {
  "@": path.resolve(__dirname, "./"),
  "@router": path.resolve(__dirname, "./router),
  "@model": path.resolve(__dirname, "./model"),
};
require("module-alias").addAliases(aliases);
```

## 设置CORS（跨域资源共享）
  - 这段代码设置了跨域请求的响应头（CORS）。CORS 允许或限制不同源（域名、协议、端口）之间的请求。这里配置为所有来源都可以访问（* 表示允许所有域名访问）。
  - next() 用来传递请求给下一个中间件，通常是处理请求的路由处理器。
```javascript
app.all("*", function (req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "*");
  res.header("Access-Control-Allow-Methods", "*");
  res.header("Access-Control-Allow-Credentials", "true");
  next();
});
```

## 根据用户代理判断请求来源
  - 这段代码会检查请求头中的 User-Agent 字段，根据该字段判断请求是来自小程序（如微信小程序）还是 App，并将这个信息存储在 req.agent 中，方便后续的处理中使用。
```javascript
app.use((req, res, next) => {
  const userAgent = req.headers["user-agent"];
  if (userAgent.includes("MiniProgram") || userAgent.includes("MicroMessenger")) {
    req.agent = "miniprogram";
  } else {
    req.agent = "app";
  }
  next();
});
```

## 路由定义和使用
  - 这部分代码是定义和加载路由。在 Express 中，路由是用于定义请求 URL 和处理函数之间映射关系的。
  - app.use("/c", cRouter) 意味着将 @/router/a/c 这个模块的路由挂载到 /c路径下，当访问该路径时，执行对应的回调函数。
```javascript
const cRouter = require("@/router/a/c");
app.use("/c", cRouter);
```

## 定时任务
  - 这段代码用于启动一些定时任务（可能是通过 cron 工具实现），如定期更新产品、优惠券等。这些功能通常与定时任务库（如 node-cron）结合使用。
```javascript
const { cronA, cronB, cronC, cronD } = require("@/utils/_cron");
cronA();
cronB();
cronC();
cronD();
```

## 初始化操作
  - 这段代码从数据库中获取配置信息（例如，收货价格等），并输出到控制台。它可能是为了验证配置是否正确或进行一些初始化设置。
```javascript
Config.findOne().select("DeliverAmount").then((res) => {
  console.log("收货价格", res?.DeliverAmount);
  console.log("支付回调地址:", process.env.WX_CALLBACK_PAY_URL);
});
```

## 启动服务器
  - startServer 函数用于启动 Express 服务器。通过 app.listen(port) 启动服务，监听指定端口。
  - process.env.PORT 是环境变量中指定的端口号，通常可以在 .env 文件中配置。
```javascript
const startServer = async (port) => {
  // ... 启动逻辑
};
startServer(process.env.PORT);
```

##  处理错误和未找到的路由

  - 这部分是为了处理 404 错误和服务器内部错误。
    - 当访问一个不存在的路由时，返回 404 错误和“path not found”信息。
    - 对于未处理的异常（如代码错误），会返回 500 错误和错误信息。
```javascript
app.use("*", (req, res) => {
  res.status(404).json({
    error: "path not found",
  });
});

app.use((err, req, res, next) => {
  return res.status(500).json({
    code: 500,
    error: err.message,
  });
});
```

## 总结

- 模块化：项目将不同的功能分成了多个路由模块，每个模块负责不同的业务逻辑。
- 配置管理：通过 .env 文件和环境变量管理配置信息。
- 中间件：使用中间件（如 CORS、用户代理识别等）来处理请求前的逻辑。
- 路由管理：通过 app.use() 来管理各种路由。
- 错误处理：使用全局的错误处理机制，确保服务器能优雅地处理错误。