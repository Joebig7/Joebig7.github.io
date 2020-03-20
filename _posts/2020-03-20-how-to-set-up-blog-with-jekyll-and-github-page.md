---
layout:     post
title:      "如何使用Github和Jekyll搭建博客"
subtitle:   "记录贴"
date:       2020-03-20 12:00:00
author:     "JoeBig7"
header-img: img/post/post-jekyll.jpg
catalog: true
tags:
    - Blog
    - Github
    - Jekyll
---

## 前言
因为最近刚刚把博客搬迁,并且使用Jekyll模板和Github Page建立了新的博客，特地记录一下主要的搭建过程。

## 准备环境

### 注册Github账号
首先我们必须要申请一个Github的账号，[传送门](https://github.com)。

### 搭建Ruby环境
搭建Ruby环境主要是因为Jekyll基于Ruby安装编译，如果我们要在本地调试就需要安装。步骤如下：
#### 下载Ruby
如果是window环境，[下载地址](https://rubyinstaller.org/downloads)：

![8gQ9tH.png](https://s1.ax1x.com/2020/03/20/8gQ9tH.png)

> 需要注意的是，选择带有DEVKIT进行安装，不然运行时会缺少msys组件。并且安装路径不能有空格，或者默认即可。

#### 安装Ruby
安装完成后，使用`ruby -v`验证下Ruby是否已经安装成功
![8glQaD.png](https://s1.ax1x.com/2020/03/20/8glQaD.png)

#### 安装Jekyll和Bundler
使用`gem install jekyll bundler`命令安装jekyll和Bundler插件，这边可能因为'墙'的原因安装比较慢，可以通过`gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/`设置为国内的源。同样安装完成后，通过`jekyll -v`和`bundler -v`分别验证是否安装成功。

![8g3F39.png](https://s1.ax1x.com/2020/03/20/8g3F39.png)

### 选取主题
环境搭建好以后，我们需要取选取jekyll的主题。官网首先提供了很多的主题[传送门](http://jekyllthemes.org)，如果想要使用别人的主题，可以通过fork别人项目来使用，不过fork完成后记得把项目的名字
改成**username.github.io**，其中username是你Github名字。

### 修改配置
最后要做的就是对模板进行修改，删除你不需要的东西，主要的配置文件是**_config.ymal**,关于配置文件的属性可以参考[官网](http://jekyllcn.com/docs/configuration/)


### 域名设置
如果自己买了域名并且做了解析后，只要修改CNAME文件将地址改成你自己的就可以了。

## 本地运行
如果想本地编译和运行jekyll，使用`bundle exec jekyll build`和`bundle exec jekyll servve`两个命令就可以实现,效果如下：

![8gtHhT.png](https://s1.ax1x.com/2020/03/20/8gtHhT.png)

## 效果
我的模板引用自[huxpro](https://github.com/Huxpro/huxpro.github.io)(感谢)，有兴趣的可以看下。最后我的博客实现效果如下：

![82FDsJ.png](https://s1.ax1x.com/2020/03/20/82FDsJ.png)
## 总结
使用jekyll搭建博客比较简单，主要是搭建Ruby环境的时候由于墙的原因下载文件比较慢(实际操作过程中可以使用迅雷下载有些资源下载速度还是比较快的)，其他的步骤都比较简单如果有问题查看一下官方文档基本都能解决。