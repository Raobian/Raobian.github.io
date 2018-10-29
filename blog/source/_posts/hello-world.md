---
title: Hello World
date: 2018-06-21 10:09:18
---
Welcome to 老边'blog

### 创建文章
```
hexo new "名称"
```
### 标签 分类 描述
```
tags:
	- 标签1
	- 标签2
categories： “分类”
description： “描述”
```

### 创建标签页
```
hexo new page “tag_name”
```
<!-- more -->
##搭建
### 安装node.js
### 安装hexo
##### 安装
cmd 到指定目录
```
npm install -g hexo
```
##### 初始化
```
hexo init
```
##### 安装组件
```
npm install
```
##### 体验
```
hexo s -g
```
##### 关联github
创建github项目
```
项目名为：用户名.github.io
勾选 README

进入settings
可以看到 GitHub Pages

配好ssh
```
修改 /_config.yml
```
deploy:
  type: git
  repository: git@github.com:用户名/用户名.github.io.git
  branch: master
```

##### 创建文章
```
hexo new post “文章名”
```

##### 部署
```
安装扩展
npm install hexo-deployer-git --save
部署
hexo d -g
```

### Next
/_config.yml
```
title: 标题
subtitle: 副标题
description: 描述
author: 作者
language: 语言（简体中文是zh-Hans）
timezone: 网站时区（Hexo 默认使用您电脑的时区，不用写）

index_generator:
  path: ''
  per_page: 5
  order_by: -date
  
archive_generator:
  per_page: 20
  yearly: true
  monthly: true

tag_generator:
  per_page: 10

per_page: 5
pagination_dir: page

theme: next
```

/themes/next/_config.yml
```
// 菜单
menu:
  home: / || home
  archives: /archives/ || archive
  categories: /categories/ || th
  tags: /tags/ || tags
  
// 样式  
scheme: Pisces

avatar: /images/avatar.jpg
```
 
### 搜索功能
```
 npm install hexo-generator-searchdb --save

/_config.yml ：
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```