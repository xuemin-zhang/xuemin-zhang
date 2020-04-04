---
title: Git忽略不想上传的文件、目录
date: 2015-10-21 16:48:05
copyright:
tags: [Git]
categories: 工程、工具
---
项目开发工程中，总会有些不需要上传到仓库的文件，为了避免这些文件被误上传，git支持两种配置模式（配置方法完全相同），可以直接忽略配置的文件和目录。
* 配置文件的名称：.gitignore
* 配置忽略的内容：在.gitignore中添加想要忽略的文件或者目录，支持*等通配符方式。
* 示例：
```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
tools/
```


* 两种模式：
1、全局配置，即对本地仓库所有Git管理的项目都有效
将.gitignore放置某路径下，然后执行命令
```
git config --global core.excludesfile 你的路径/.gitignore
```
  2、私有配置，即仅对某个Git项目有效
  只需要将.gitignore文件放置在项目根路径下即可。
