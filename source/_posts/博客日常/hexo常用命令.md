---
title: hexo常用配置及命令
top: false
date: 2024-12-10 14:12:28
author: chengjt
summary:
img:
cover:
coverImg:
tags:
    - 博客日常
    - hexo
categories:
    - 博客日常
---

# 在 Hexo 博客的 Butterfly 主题中添加文章置顶功能
参考链接：cc

卸载 hexo-generator-index

执行命令: npm uninstall hexo-generator-index

安装 hexo-generator-index-pin-top

执行命令: npm i hexo-generator-index-pin-top --save

文件置顶

在需要置顶的文章的 Front-matter 中加上 top: true/数字 即可, 数字越大，文章越靠前。

```
---
title: 123
date: 2024-12-04 19:33:44
top: 10
categories:
- flutter
tags:
- flutter
- android studio
top_img:
cover: https://www.voidking.com/dev-hexo-categories/#:~:text=1%E3%80%81%E6%96%B0%E5%BB%BAcategories%E9%A1%B5%E9%9D%A2%20hexo
---

```
# Hexo添加categories页面
[好好学Hexo：Hexo添加categories页面](https://www.voidking.com/dev-hexo-categories/#:~:text=1%E3%80%81%E6%96%B0%E5%BB%BAcategories%E9%A1%B5%E9%9D%A2%20hexo)
