---
title: GitHub Pages + Hexo搭建博客
date: 2016-08-31 22:24:25
tags: 能工巧匠
---

这篇博客是使用**Github Pages + Hexo** 搭建本博客的总结。

作为一个有多年开发经验的人来说，没有搭建自己的博客，记录自己的开发点点滴滴，实在有点遗憾，以前总以为搭建博客得买服务器，域名等复杂的步骤，然后选择了国内老牌的博客CSDN或博客园来写博客，但可拓展性太低，千篇一律。

最近经过朋友的介绍使用GitHub Pages + Hexo搭建免费的独立博客，痛下决心来做一次。

## 前言
搭建博客需要先了解几个知识点
* [Git](https://git-scm.com/book/zh/v2)
* [GitHub](https://github.com/)
* [GitHub Pages](https://pages.github.com/)
* [Hexo](https://hexo.io/zh-cn/)
* [Markdown](http://www.appinn.com/markdown/#overview)

## 环境配置
### 安装Hexo
由于安装Hexo需要Node.js和Git，所以在[Hexo](https://hexo.io/zh-cn/)官网上有安装的详细说明（中文版），不在赘述（Mac电脑有自带Git的）。

当Node.js和Git都安装完成后，开始正式安装Hexo，在终端上执行
``` bash
$ sudo npm install -g hexo
```
需要输入管理员密码（Mac登录密码）。(`sudo`：Linux管理员指令，`-g`：全局安装)
> 坑一：[Hexo官网](https://hexo.io/zh-cn/)上的安装命令是`$ npm install -g hexo-cli`，安装时不要忘记前面加上`sudo`，否则会因为权限问题报错。

### 初始化博客
终端cd到你想创建博客的文件夹，然后中执行终端命令
```bash
$ hexo init blog
```
blog是你新建的博客文件夹，cd到这个文件夹，然后执行命令安装npm

```
$ npm install
```
最后执行命令，开启Hexo本地服务器

```
$ hexo s
```
终端中会提示在浏览器打开 http://localhost:4000 这个网址就可以看到（**因为我配置了主题内容，请忽略图片上的文字，之后将会讲到主题**）
{% asset_img Snip20160831_1.png hexo init image %}

## 关联Github
### 创建仓库
在前言的[GitHub Pages](https://pages.github.com/)中介绍了Pages仓库，所以要创建一个名为`用户名.github.io`固定写法的参考，例如`piglikeyoung.github.io`，如下图所示
{% asset_img Snip20160831_2.png github Pages image %}

本地的blog目录结构

```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

用文本编辑器打开`_config.yml`
修改下列代码

```
	 deploy:
    type: git
    repository: https://github.com/piglikeYoung/piglikeyoung.github.io.git
    branch: master
```
你需要把`repository`替换成你自己的github地址
> 坑二：在配置所有的`_config.yml`文件时（包括theme中的），在所有的`冒号:`后边都要加一个`空格`，否则执行hexo命令会报错，切记 切记

在 blog文件夹目录下执行下面命令生成静态页面
```
$ hexo g        全称：hexo generate
```

```
如果出现如下报错：
ERROR Local hexo not found in ~/blog
ERROR Try runing: 'npm install hexo --save'
则执行命令：
npm install hexo --save
若无报错，自行忽略此步骤。
```
再执行命令部署静态网页
```
hexo d            全称：hexo deploy
```
> 坑三：执行`hexo d`报错无法连接git或找不到git，执行这个命令 
```
$ npm install hexo-deployer-git --save
```

再次执行`hexo g`和`hexo d`命令。

如果你的电脑没有管理Github，执行`hexo d`时终端会提示你输入Github的用户名和密码

```
Username for 'https://github.com':
Password for 'https://github.com':
```
当`hexo d`执行成功，在浏览器上输入 http://piglikeyoung.github.io 就会显示 http://localhost:4000 一样的页面。

**为了避免每次都输入Github用户名和密码，可以参考下一章来配置SSH Keys**

## 添加ssh key到Github

