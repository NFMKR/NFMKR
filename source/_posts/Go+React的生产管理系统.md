---
title: 基于go和react的restful生产管理系统开发
date: 2024-10-24 19:18:29
tags:
---
## 搭建Gin和React环境

1. 安装go，初始化go模块```go mod init pms```
2. 安装Gin框架 ```go get -u github.com/gin-gonic/gin```
3. 创建基本的 Gin 应用,在项目目录下创建一个名为 main.go 的文件，添加以下内容：

```package go

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})

	r.Run(":8080") // 启动服务，监听8080端口
}
```

4. 运行Gin应用：```go run main.go```
5. 安装node.js和npm
6. 创建 React 应用```npx create-react-app my-react-app```
7. 进入项目目录```cd my-react-app```
8. 安装 Axios（用于 HTTP 请求）```npm install axios```
9. 启动 React 应用```npm start```
10. 部署：在服务器上运行npm install 生成相应配置，然后运行npm run build 生成build文件夹，打开宝塔面板在该域名的的网站目录处指向相应的build文件夹即可。




