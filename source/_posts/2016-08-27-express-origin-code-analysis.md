---
layout: post
title:  "express源码分析"
date:   2016-08-27 21:00:00
categories: Javascript
tags: [javascript,node.js,express]
description: "express是node.js的一个轻量级后端框架，本文将对express的源码实现逻辑及主要服务的实现方式做一些分析"
img_url: "http://7xu027.com1.z0.glb.clouddn.com/own-world.jpg"
---

### 前言

express是node.js的一个轻量级后端框架，本文将对express的源码实现逻辑及主要服务的实现方式做一些分析

<!-- more -->

### 基本结构

直接`npm install express`，可以看到express的包结构如下：

    ├── lib
    │   ├── middleware
    │   │   ├── init.js
    │   │   └── query.js
    │   ├── router
    │   │   ├── index.js
    │   │   ├──  layer.js
    │   │   └── route.js
    │   ├── application.js
    │   ├── express.js
    │   ├── request.js
    │   ├── response.js
    │   ├── utils.js
    │   └── view.js
    ├── History.md
    ├── index.js
    ├── LICENSE
    ├── package.json
    └── Readme.md

express模块的入口是 index.js，该文件中又引入了./lib/express.js，并将其用module.exports导出。

### express.js
`lib/express.js`是对express所有定义及功能模块的整合，该文件的开头如下：


    var EventEmitter = require('events').EventEmitter;
    var mixin = require('merge-descriptors');
    var proto = require('./application');
    var Route = require('./router/route');
    var Router = require('./router');
    var req = require('./request');
    var res = require('./response');

基本来说根据这些包的引入基本对express结构基本有一些理解了，之后的分析思路也会沿着这些引入的文件来进行。
先大概介绍下这些模块做了哪些事情：

+ EventEmitter: node.js的events模块
+ mixin: 用来合并对象的工具
+ proto: express应用的原型对象，在application.js里详细定义
+ Route: 定义最基本Route对象，包括app.post,app.all等以及Router对象的http方法都是从这里继承的
+ Router: 完整的Router对象，继承了Route的http方法，也集合了./router/layer.js下的路由初始化方法及路由处理方法，相当于是路由功能的整合
+ req: request对象
+ res: response对象


