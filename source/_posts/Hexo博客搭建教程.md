---
title: hexo博客创建过程
date: 2024-10-19 19:29:04
categories: 
  - 文章
---
# 创建hexo的步骤

1. 通过下载和GitHub这个库去搭建博客
[hexo博客框架](https://github.com/theme-shoka-x/hexo-theme-shokaX?tab=readme-ov-file)
2. 在hexo官方文档进行具体搭建自己的博客
[hexo官方文档](https://hexo.io/zh-cn/docs/themes)
3. 建站用了`hexo init xxxxx` `cd xxxxx` `npm install`
4. 在_config.yml中配置了title,subtitle,description,keywords,author
5. 下载主题theme
[下载的博客:里面有很多博客使用教程]我使用的主题(https://xaoxuu.com/wiki/stellar/#start)
[下载的github仓库链接](https://github.com/xaoxuu/hexo-theme-stellar)
6. 写博客文章
    如果命令不能用，需要npm install -g hexo-cli来安装相应的框架
   命令`hexo new myfirstpost`
7. 代码的输写方式
`bash
console.log("dfadad")
`
1. [themne](https://hexo.io/themes/#responsive)
2. `npm install hexo-cli -g`在服务器终端进行安装hexo博客框架
3.  遇到在网页中打开博客是空白的页面，错误首先设置网站，网站目录，网站目录设置，将设置成public文件夹，然后在_config.yml中的theme设置成stellar，再在服务器终端进行下载`npm i hexo-theme-stellar`，下载后重新`npm run build` ，成功在服务器上访问博客。
4.  先进入博客文件目录中再进行github推送到服务器的命令`git pull  origin main` ,生成博客文件hexo的静态文件`npm run build`，启动 Hexo 服务器：`npm start`,使用合并（merge）:`git config pull.rebase false`,使用变基（rebase）:git config pull.rebase true`.
5.  git pull origin main是在服务器上拉，git push origin main是在本地上推送，npm run build是打包。
6.  单行代码：使用单个反引号（`）包围，多行代码块：使用三个反引号（`）包围（
`javascript
function greet(name) {
    return `Hello, ${name}!`;
}
console.log(greet('World'));
  // 输出: Hello, World!

），并指定语言类型，避免嵌套反引号：在多行代码中不使用单个反引号，留空行：在代码块与标题或文本之间留空行，以确保正确解析。
