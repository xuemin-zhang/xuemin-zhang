---
title: Atom -- MarkDown编辑、预览利器
date: 2015-06-10 16:48:05
copyright:
tags: [Atom,MarkDown]
categories: 工程、工具
---

## 一、背景
Atom是github专门为程序员推出的一个跨平台文本编辑器，于2015年1月8日[开源](https://github.com/atom/atom)，[官方](https://atom.io/)slogan为"A hackable text editor for the 21st Century"，其相对其他IDE算是比较轻量的，启动速度比较快，也提供了很多功能插件和主题。其用户界面简洁、直观，支持Windows、Mac、Linux 三大桌面平台，原生支持HTML、JavaScript、CSS、Node.js等前端应用编程语言，并且有Java、C#、PHP等语言的插件。相对其他IDE，其还有个特点，就是完全免费。

我没依赖其进行项目开发，就不在此详细聊其功能。我主要使用MarkDown功能，相对其他MarkDown编辑器，其支持类似Sublime等其他IDE项目管理的方式管理MarkDown文件，并且语言完全支持[GitHub Flavored Markdown](https://github.github.com/gfm/)。本文主要聊下Atom对MarkDown的支持及优化（插件化的功能），以及遇到的问题。

## 二、简要设置（for Mac用户）
### 2.1调整主题
{% asset_img atom1.gif %}

### 2.2安装插件
{% asset_img atom2.gif %}

## 三、MarkDown功能插件

### 3.1增强预览 markdown-preview-plus
Atom自带的Markdown预览插件markdown-preview功能比较简单，markdown-preview-plus对其做了功能扩展和增强。
* 支持预览实时渲染。(Ctrl + Shift + M)
* 支持Latex公式。(Ctrl + Shift + X)

使用该插件前，需要先禁用markdown-preview。
{% asset_img atom3.gif %}
{% asset_img atom4.gif %}

### 3.2同步滚动 markdown-scroll-sync
配合预览功能使用，预览后需要修改某个地方时候，能够方便的找到代码位置。
{% asset_img atom5.gif %}

### 3.3代码增强(language-markdown)
一般的MarkDown编辑器都会提供代码高亮功能，初此外该插件提供了快捷生产代码等功能。
{% asset_img atom6.gif %}

### 3.4其他插件
以上是几个与MarkDown比较相关的几个插件，Atom官方还提供了很多其他功能插件，详见：https://atom.io/packages

## 四、遇到的问题
按如上方式安装完markdown-scroll-sync后，有如下提示：
````
TypeError: Right-hand side of 'instanceof' is not callable
    at /packages/markdown-scroll-sync/lib/main.coffee:38:14
````
在官网issues中找到原因是与markdown-preview-plus有版本冲突，解决方案是卸载高版本的plus插件，命令行安装指定版本，命令如下：
````
apm install markdown-preview-plus@2.4.16
````



## 五、参考
https://github.com/vincentcn/markdown-scroll-sync/issues/334
