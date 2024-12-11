---
title: C-路由定义
date: 2024-12-11 16:32:57
tags:
---
## 5. Express 路由配置

项目中通过 `src/router/` 目录组织不同的 API 路由，分模块管理各类功能：

### 5.1 `src/router/activity/registration.js`

**路径**: `src/router/activity/registration.js`

**作用**: 处理与活动登记相关的路由，如用户参加活动、提交登记信息等。

**使用步骤**:

1. **定义路由**:

   使用 Express Router 定义具体的路由处理函数。

2. **引入中间件和模型**:

   引入必要的中间件和 Mongoose 模型，用于认证和数据库操作。

   ```javascript
   const express = require("express");
   const { body } = require("express-validator");
   const Validate = require("../../middleware/Validate");
   const auth = require("../../middleware/auth");
   const Registration = require("../../models/activity/Registration");

   const router = express.Router();

   router.post(
     "/register",
     auth,
     [
       body("phoneNumber").isMobilePhone(),
       body("realName").isString().notEmpty(),
       // 其他验证规则...
     ],
     Validate,
     async (req, res) => {
       try {
         const { phoneNumber, realName, activityType } = req.body;

         // 创建新的登记记录
         const newRegistration = new Registration({
           userId: req.user._id,
           phoneNumber,
           realName,
           activityType,
           // 其他字段...
         });

         await newRegistration.save();

         res.status(201).json({ message: "登记成功", registration: newRegistration });
       } catch (error) {
         console.error("登记时出错：", error);
         res.status(500).json({ error: "服务器内部错误" });
       }
     }
   );

   module.exports = router;
   ```

3. **挂载路由**:

   在主应用文件中挂载该路由。

   ```javascript
   // src/index.js
   const registrationRouter = require("./router/activity/registration");

   app.use("/activity", registrationRouter);
   ```
