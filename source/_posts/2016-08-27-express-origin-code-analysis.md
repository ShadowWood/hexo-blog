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

	exports = module.exports = createApplication;

	function createApplication() {
		var app = function(req, res, next) {
			app.handle(req, res, next);
		};

		mixin(app, EventEmitter.prototype, false);
		mixin(app, proto, false);

		app.request = { __proto__: req, app: app };
		app.response = { __proto__: res, app: app };
		app.init();
		return app;
	}

文件一开始便引入了一些基础模块，基本来说根据这些模块的引入基本对express结构基本有一些理解了，之后的分析思路也会沿着这些引入的文件来进行。
先大概介绍下这些模块做了哪些事情：

+ EventEmitter: node.js的events模块
+ mixin: 用来合并对象的工具
+ proto: express应用的原型对象，在application.js里详细定义
+ Route: 定义最基本Route对象，包括app.post,app.all等以及Router对象的http方法都是从这里继承的
+ Router: 完整的Router对象，继承了Route的http方法，也集合了./router/layer.js下的路由初始化方法及路由处理方法，相当于是路由功能的整合
+ req: request对象
+ res: response对象

在引入了这些模块之后，就有了整个express的应用创建方法`createApplication`，也相当于是express的一个main函数。
在`createApplication`函数中一开始就把需要返回的值`app`定义为一个函数，该函数有`req`, `res`, `next`三个参数，这不得不使人联想起app.get('/', function(req, res, next) {})里的req,res,next。在这个函数的内部则直接调用`app`的handle方法，并将`req`,`res`,`next`作为参数传入其中。
`app.handle`这个方法是在`./application`中定义的并通过mixin将该方法合并到app对象上。

	app.handle = function handle(req, res, callback) {
		var router = this._router;

		// final handler
		var done = callback || finalhandler(req, res, {
			env: this.get('env'),
			onerror: logerror.bind(this)
		});

		// no routes
		if (!router) {
			debug('no routes defined on app');
			done();
			return;
		}

		router.handle(req, res, done);
	};

该方法的作用是将res, res逐级分发到express应用每个路由中，以便执行各个路由相匹配的操作。
其实这个函数最关键的部分是在router.handle(req, res, done)这个方法的执行上，这是真正的路由分发执行操作，在./router/index.js里面定义，之后会对这部分进行分析讲解。

接着createApplication里的操作。在定义了app之后，执行了两个`mixin`方法，分别将 `EventEmitter` 和 `./application.js` 中的属性和方法合并到`app`之中。

在 mixin 之后，createApplicaion 又使用了对象字面量的定义方法，定义了`app.request`和`app.response`对象，分别以 `./request.js` 和 `./response.js`为原型对象，并赋值app属性且指向app本身(这里是为了在之后的response对象或request对象中，能够使用this.app访问已经创建的express实例)。

然后就是执行`app.init()`，这个`init()`方法也是在`./application.js`中定义的，用来初始化express应用的设置

	app.init = function init() {
		this.cache = {};
		this.engines = {};
		this.settings = {};

		this.defaultConfiguration();
	};

在createApplicaion的最后返回组建好的`app`对象。

在createApplicaion之后，便是一些公共API的导出：

	exports.application = proto;
	exports.request = req;
	exports.response = res;

	/**
	* Expose constructors.
	*/

	exports.Route = Route;
	exports.Router = Router;

	/**
	* Expose middleware
	*/

	exports.query = require('./middleware/query');
	exports.static = require('serve-static');

对于这些公共api模块的功能和作用也不多说了，express的文档有相应的说明。

在`./express.js`的最后，遍历了一个数组：

	[
		'json',
		'urlencoded',
		'bodyParser',
		'compress',
		'cookieSession',
		'session',
		'logger',
		'cookieParser',
		'favicon',
		'responseTime',
		'errorHandler',
		'timeout',
		'methodOverride',
		'vhost',
		'csrf',
		'directory',
		'limit',
		'multipart',
		'staticCache',
	].forEach(function (name) {
		Object.defineProperty(exports, name, {
			get: function () {
				throw new Error('Most middleware (like ' + name + ') is no longer bundled with Express and must be installed separately. Please see https://github.com/senchalabs/connect#middleware.');
			},
			configurable: true
		});
	});

这是因为express 4.x之后，很多中间件依赖没有在express内部导入了，但是express有时会用到这些中间件，这里是一个中间件检测，告诉开发者数组内的中间件需要从外部install进来。