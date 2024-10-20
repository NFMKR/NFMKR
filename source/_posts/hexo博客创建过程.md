---
title: hexo博客创建过程
date: 2024-10-19 19:29:04
tags:
---
# 创建hexo的步骤

1. 通过下载和GitHub这个库去搭建博客
[hexo博客框架](https://github.com/theme-shoka-x/hexo-theme-shokaX?tab=readme-ov-file)
2. 在hexo官方文档进行具体搭建自己的博客
[hexo官方文档](https://hexo.io/zh-cn/docs/themes)
3. 建站用了`hexo init nfmkr` `cd nfmkr` `npm install`
4. 在_config.yml中配置了title,subtitle,description,keywords,author
5. 下载主题theme
[下载的博客:里面有很多博客使用教程](https://xaoxuu.com/wiki/stellar/#start)
[下载的github仓库链接](https://github.com/xaoxuu/hexo-theme-stellar)
6. 写博客文章
   命令`hexo new myfirstpost`
7. 代码的输写方式
```bash
console.log("dfadad")
```
8. [themne](https://hexo.io/themes/#responsive)
9. `npm install hexo-cli -g`在服务器终端进行安装hexo博客框架
10. 遇到在网页中打开博客是空白的页面，错误首先设置网站，网站目录，网站目录设置，将设置成public文件夹，然后在_config.yml中的theme设置成stellar，再在服务器终端进行下载`npm i hexo-theme-stellar`，下载后重新`npm run build` ，成功在服务器上访问博客。