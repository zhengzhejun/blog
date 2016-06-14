---
layout: post
title: githut jekyll 创建个人博客
comments: true
categories: blog
tags: [jekyll, github]
---
这里我总结一下这两天用github+jekyll创建个人博客的过程，也算是我的zhengzhejun.com的第一篇博客。首先，我们需要github账号，安装jekyll，最好有一个自己的域名（我花了49/年买了我的域名，名字拼音长，域名也便宜。）

## 主要参考的博客
* <http://www.ezlippi.com/blog/2015/03/github-pages-blog.html>

大部分内容可以参考上面的博客，这里我主要罗列一下我踩得坑，和一些jekyll的用法。
我是直接下载了一个jekyll theme，我是从这里选了一个<http://jekyllthemes.org/>。也可以在github上搜索jekyll theme，有好多好看的主题。我选了一个相对简介一下的，先上上手，以后再看看用一个比较酷炫的主题。

## 遇到的坑
在post目录下添加了一个自己的markdown文件，其中头文件中指定了date属性

{% highlight vi%}
---
layout: blabla
title: blabla
date: 2016-06-09 18:19:01 -0500
---
{% endhighlight %}

但是

{% highlight vi %}
jekyll serve
{% endhighlight %}

无法生成该文件的html文件，后来发现date属性日期不能写当天日期，写昨天就行。不知道这算不算一个bug。

## jekyll用法总结（后面陆续更新）
* [中文官方文档](http://jekyllcn.com/docs/installation/)

### 套用其他theme（主题）的一些样式
* 使用其他主题的效果到自己的博客，我们需要把css，和js文件拷贝到自己的博客项目中
* 主题的layout中会有类似head.html的文件，这里包含了引用css和js文件的引用语句，可以直接拷贝到自己的目录中，或添加到自己的文件中。
* ico文件是浏览器地址栏中的小图标
