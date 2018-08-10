---
title: hexo&github-pages
date: 2018-05-21 17:35:08
tags:
  - hexo
  - 前端
---

<!-- Hexo博客使用google-code-prettify代码高亮 http://masikkk.com/article/hexo-12-google-code-prettify/ -->
<!--  为Hexo博客加入prettify插件
http://jumpbyte.cn/2016/07/02/use-and-install-prettify/ -->

## 关于 hexo

hexo 是基于`nodejs`的博客搭建工具。使用者可以用 markdown 语法来书写博客，hexo 能生成根据配置漂亮的静态文件。

- [hexo 官网](https://hexo.io/)
- [hexo 文档](https://hexo.io/docs/)
- [知乎：有哪些好看的 Hexo 主题？ ](https://www.zhihu.com/question/24422335)
  <!-- more -->

### 基本使用

```shell
npm install hexo-cli -g
hexo init blog
cd blog
npm install
hexo server
```

此时打开[http://localhost:4000/](http://localhost:4000/)即可看到博客页面。

### 配置

不过由于 hexo 默认的主题比较丑，建议使用其他主题。在 github 上搜索`hexo theme`选择一个排名较前的主题[yilia](https://github.com/litten/hexo-theme-yilia)。

### 部署到 github 上

1.配置`_config.yml`
`yml deploy: type: git repo: git@github.com:losymear/losymear.github.io.git branch: master`
由于我推送的提交到`gh-pages`时不能开启 git pages,因此将之部署到 master 分支。

2.新建`losymear.github.io`项目

3.建立本地 git 仓库

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

4.部署

```
hexo clean
hexo deploy
```

## code prettify 语法高亮

1.禁用默认高亮
在主`_config.yml`文件中把`highlight`相关的配置设置为`false`:

```yml
highlight:
  enable: false
  line_number: false
  auto_detect: false
  tab_replace:
```

然后禁用主题的高亮。yilia 是在`main.scss`中注释掉`@import "./highlight";`。同时还需要注意`article-entry`中`p code,li code`的样式，可以考虑用`pre.prettyprint li code{color: inherit;....}`覆盖。
我的做法是添加如下样式：

```css
pre.prettyprint li code {
  padding: inherit;
  margin: inherit;
  background: inherit;
  border: inherit;
  font-family: inherit;
  word-wrap: inherit;
  font-size: inherit;
}
```

2.复制`code prettify`的相关文件
先在[code prettify](https://github.com/google/code-prettify)中下载最新的 release 文件，解压，然后把`prettify.js`复制到`themes/yilia/source/js`文件夹下，把`prettify.css`复制到`themes/yilia/source/css`文件夹下。

3.加载`code prettify`
首先需要使用`jQuery`。在`themes/yilia/layout/_partial/head.ejs`中添加

```html
<script src="<%- config.root %>js/jQuery.js"></script>
```

然后同一文件中添加

```html
<link rel="stylesheet" href="<%- config.root %>css/prettify.css" media="screen" type="text/css">
```

在`themes/yilia/layout/_partial/foot.ejs`中添加

```html
<script src="<%- config.root %>js/prettify.js"></script>
```

同时添加如下脚本：

```html
  <script type="text/javascript">
    $(document).ready(function () {
      $('pre').addClass('prettyprint linenums');
      prettyPrint();
    })
</script>
```

4.additional

也可以`prettify.css`换成[其他主题文件](https://jmblog.github.io/color-themes-for-google-code-prettify/)，我选择的是 tomorrow-night-bright.css。

## 添加目录


## 其他

~~目前图片使用七牛云,[chrome 插件 qiniu upload files](https://chrome.google.com/webstore/detail/qiniu-upload-files/emmfkgdgapbjphdolealbojmcmnphdcc?utm_source=chrome-ntp-icon)。可登陆[七牛云后台](https://portal.qiniu.com/bucket)查看内容。~~
图片目前使用[聚合图床](http://photo.ishield.cn/)。

## tips

### 使用插件

- [hexo-footnotes](https://github.com/LouisBarranqueiro/hexo-footnotes)
- [hexo-filter-mermaid-diagrams](https://github.com/webappdevelp/hexo-filter-mermaid-diagrams)
- [hexo-filter-flowchart](https://github.com/bubkoo/hexo-filter-flowchart)
