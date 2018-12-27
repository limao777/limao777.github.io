---
title: 使用hexo和X-pages搭建个人博客实践
date: 2017-10-13 16:36:55
category: IT技术
tags:
- git
- hexo
- 博客
---
才将基于github pages的博客系统搭建好，趁热先写写搭建过程中的一些过程，hexo是一个快速、简洁且高效的博客框架，能生成静态页面

# 前提
* 安装好nodejs
* 安装好git

### nodejs安装
进入[nodejs官方网站](http://nodejs.cn/)，直接下载安装包安装即可

### git安装
git也是比较好安装的，git官网下载git软件直接安装好即可，git官网是https://git-scm.com/

# 本地化搭建
本地化搭建将按照如下方式组织文章
* 安装hexo
* 初始化hexo
* 配置hexo
* 新增文章
* 生成静态页面
* 本地预览

### 安装hexo
```bash
npm install hexo-cli -g
```

### 初始化hexo
在准备写博客的文件夹处使用命令：
```bash
hexo init
npm install
```
此时将完成了hexo的初始化，如果一切默认配置则可以写文章了，通常我们要简单配置一下hexo

### 配置hexo
通常要配置的选项位于你建立博客根目录的_config.yml文件
必要的变更包括# Site、# URL，其他的则可以根据自身需要进行配置
其中# Site的配置关于language和timezone比较特殊，都不是随意配置的，须符合一定的language和timezone规则，如language通常配置zh-CN、en等，timezone则配置Asia/Shanghai，这些可以通过搜索引擎搜索Unix的locale语言以及timezone可选的配置
如果网站存放在子目录中，例如 http://yoursite.com/blog，则请将 url 设为 http://yoursite.com/blog 并把 root 设为 /blog/
最后建立一个about.html文件，通常很多主题模版需要这个文件，hexo中要手动建立一个：
```bash
hexo new page 'about'
```

### 新增文章
使用以下命令创建一篇新文章
```bash
hexo new '文章标题'
```
通过这个命令则在_posts文件夹下面建立了一个文件，该文件使用markdown进行编写，最后生成出来的就是你的博客了

### 生成静态页面
```bash
hexo g
```
此时public文件夹就是你的博客的静态文件了，可以上传到支持静态文件加载的地方，这样大家就能欣赏你的博客了

### 本地预览
```bash
hexo server
```
直接使用该命令即可启动本地服务器，照着提示浏览器访问那个端口就即可。如果不嫌麻烦，也可以使用apache、nginx指向public文件夹，效果是一样的

# 将博客上传到互联网
* 建立gitpages
* 代码上传到github

### 建立gitpages
当然，你得有一个github帐号，国内的如码云等git托管网站也有类似的功能
在github上建立一个和你名字一模一样的工程，选择该工程的配置，点击pages，点击生成即可，默认的访问方式就是{你的名字}.github.io

### 代码上传到github
```bash
git clone {你的项目}
```
再复制你的public文件夹到你的git项目中，如果使用了子目录则相应地建立子目录
```bash
git add --all
git commit -m 'update blog'
git push origin {你的项目}
```
# 一些小技巧
### 直接在hexo中使用静态文件
在hexo3中，静态文件可通常的markdown格式不一样，可以通过将 config.yml 文件中的 post_asset_folder 选项设为 true 来打开
打开过后每次新增文章则会相应地在_posts文件夹下建立静态资源目录，将静态资源目录放进去即可，引用方式：
```text
{% asset_path slug %}
{% asset_img slug [title] %}
{% asset_link slug [title] %}

e.g. {% asset_img example.jpg 这里是alt标签 %}
```

### 使用category和tags
e.g.
```text
---
title: test
date: 2017-10-13 13:58:50
category: ABC
tags:
- tag1
- tag2
- tag3
---
```
