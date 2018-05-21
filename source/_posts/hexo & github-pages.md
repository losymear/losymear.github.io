---
title: hexo&github-pages
date: 2017-05-21 17:35:08
tags:
    - hexo
    - 前端
---
## about hexo

hexo是基于`nodejs`的博客搭建工具。用户使用markdown语法来书写博客，hexo提供能生成漂亮的静态文件。

- [hexo官网](https://hexo.io/)
- [hexo文档](https://hexo.io/docs/)
- [知乎：有哪些好看的 Hexo 主题？ ](https://www.zhihu.com/question/24422335)

### 基本使用
```shell
$ npm install hexo-cli -g
$ hexo init blog
$ cd blog
$ npm install
$ hexo server
```
此时打开http://localhost:4000/即可看到博客页面。

### 配置

不过由于hexo默认的主题比较丑，建议使用其他主题。在github上搜索`hexo theme`选择一个排名较前的主题[yilia](https://github.com/litten/hexo-theme-yilia)

### 部署到github上
.1 配置`_config.yml`
    ```yml
    deploy:
    type: git
    repo: git@github.com:losymear/losymear.github.io.git
    branch: master
    ```
由于我推送的提交到`gh-pages`时不能开启git pages,因此将之部署到master分支。

.2 新建`losymear.github.io`项目

.3 建立本地git仓库
```shell
git init
git checkout -b source 
git add --all
git commit -m 'init'
git remote add origin
git@github.com:losymear/losymear.github.io.git
git push -u origin source
```
保存项目源文件到`source`分支。

.4 部署
``` 
hexo clean
hexo deploy
```



### 其他
目前图片使用七牛云,[chrome插件qiniu upload files](https://chrome.google.com/webstore/detail/qiniu-upload-files/emmfkgdgapbjphdolealbojmcmnphdcc?utm_source=chrome-ntp-icon)。可登陆[七牛云后台](https://portal.qiniu.com/bucket)查看内容。



### tips
#### 使用插件
- [hexo-footnotes](https://github.com/LouisBarranqueiro/hexo-footnotes)
- [hexo-filter-mermaid-diagrams](https://github.com/webappdevelp/hexo-filter-mermaid-diagrams)
- [hexo-filter-flowchart](https://github.com/bubkoo/hexo-filter-flowchart)