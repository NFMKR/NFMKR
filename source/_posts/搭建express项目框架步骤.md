---
title: 搭建express项目框架步骤
date: 2024-12-20 13:53:44
tags:
categories:
  - 文章
---
搭建 Express 后端项目的步骤：

1. **创建项目目录并初始化**
```bash
mkdir my-express-app
cd my-express-app
npm init -y
```

1. **安装必要的依赖包**
```bash
npm install express
# 安装一些常用的中间件
npm install cors body-parser dotenv
```

1. **创建基本的项目结构**
```bash:my-express-app/.env
my-express-app/
  ├── src/
  │   ├── routes/
  │   ├── controllers/
  │   ├── models/
  │   ├── middlewares/
  │   └── config/
  ├── .env
  ├── .gitignore
  └── app.js
```

1. **创建入口文件 app.js**
```javascript:app.js
const express = require('express');
const cors = require('cors');
const bodyParser = require('body-parser');
require('dotenv').config();

const app = express();

// 中间件配置
app.use(cors());
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

// 路由配置
app.get('/', (req, res) => {
  res.json({ message: '欢迎访问 API' });
});

// 启动服务器
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`服务器运行在端口 ${PORT}`);
});
```

5. **创建 .env 文件**
```plaintext:.env
PORT=3000
NODE_ENV=development
```

6. **创建 .gitignore 文件**
```plaintext:.gitignore
node_modules/
.env
```

7. **添加一个示例路由文件**
```javascript:src/routes/users.js
const express = require('express');
const router = express.Router();

router.get('/', (req, res) => {
  res.json({ message: '获取用户列表' });
});

module.exports = router;
```

8. **在 app.js 中引入路由**
```javascript:app.js
// ... 其他代码 ...

// 引入路由
const usersRouter = require('./src/routes/users');
app.use('/api/users', usersRouter);

// ... 其他代码 ...
```

9. **修改 package.json 添加启动脚本**
```json:package.json
{
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js"
  }
}
```

10. **安装开发依赖（可选）**
```bash
npm install nodemon --save-dev
```

### 启动项目
```bash
# 开发环境
npm run dev

# 生产环境
npm start
```