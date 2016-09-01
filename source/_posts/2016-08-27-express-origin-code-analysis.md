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
    │   │   ├── layer.js
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

### router
./router文件夹下包括三个文件：

+ `layer.js`：定义中间件的基本数据结构
+ `route.js`：定义express的路由中间件Route;
+ `index.js`：定义一个中间件容器，也就是Router对象，用来存放路由中间件(Route)以及其他功能中间件

> `Router` 和 `Route` 的区别：Router可以看作是一个中间件容器，不仅可以存放路由中间件（Route），还可以存放其他中间件；而Route仅仅是路由中间件，封装了路由信息。
Router和Route都各自维护了一个stack数组，该数组就是用来存放中间件和路由的。

#### layer.js

首先来看layer.js中对于中间件的初始定义：
	
	var pathRegexp = require('path-to-regxp');

	function Layer(path, options, fn) {
		if (!(this instanceof Layer)) {
			return new Layer(path, options, fn)
		}

		debug('new %s', path);
		var opts = options || {};

		this.handle = fn;
		this.name = fn.name || '<anonymous>';
		this.params = undefined;
		this.path = undefined;
		this.regexp = pathRegexp(path, this.keys = [], opts);

		if (path === '/' && opts.end === false) {
			this.regexp.fast_slash = true;
		}
	}

`path`参数不用多说，就是传入的url字符串，这里使用了`path-to-regexp`这个库，用来匹配url字符串，`options`是`path-to-regexp`需要的配置参数，即为 {sensitive: Boolean, stric: Boolean, end: Boolean}。npm上有该库的详细使用说明，这里就不再讲解了。
`fn`也就是中间件里的回调处理函数，在Layer初始化的时候将它赋值给了自己的`handle`属性。

之后Layer还定义了三个操作方法：`handle_error`, `handle_request`, `match`。

`handle_error`就是定义的express应用中的错误处理部分，例如app.use(fuction(err, req, res, next){})最后就会执行到这里。
`handle_request`定义的就是express应用中的路由中间件请求处理函数，也就是例如app.get('/test', function(req, res, next){})的操作最后的执行位置。
`match`定义的是匹配`path`参数的操作，使用`path-to-regexp`的操作方法，例如在请求过程中`/foo/23`与就会和之前定义的`/foo/:id`相匹配，并最终将对应的`23`赋值`req.params.id`，这一部分的操作需要结合`path-to-regexp`的操作方法去了解。

整个Layer的定义其实并不复杂，它定义了中间件的基本数据结构，是后面Router和Route对象实现的基础。

#### route.js

同样，先从Route对象的初始化入手：

	function Route(path) {
		this.path = path;
		this.stack = [];

		debug('new %s', path);

		this.methods = {};
	}

`path`参数不用多说，`stack`是一个存放layer组件的数组，`methods`是存放HTTP方法的Object，例如{'get': true, 'post': true}，即表示该Route中间件只能接受get和post方法。

紧接着Route通过原型链的方式定义了两个与`methods`紧密相关的方法：

+ _handles_method: 判断Route对象中是否存在method(传入参数)方法，并且如果method值为`head`，当作get方法处理；
+ _options：返回Route对象的methods值，并且如果存在`get`，则再添加一个`head`值。

然后就是比较重要的部分，中间件的派发操作：

	Route.prototype.dispatch = function dispatch(req, res, done) {
		var idx = 0;
		var stack = this.stack;
		if (stack.length === 0) {
			return done();
		}

		var method = req.method.toLowerCase();
		if (method === 'head' && !this.methods['head']) {
			method = 'get';
		}

		req.route = this;

		next();

		function next(err) {
			if (err && err === 'route') {
				return done();
			}

			var layer = stack[idx++];
			if (!layer) {
				return done(err);
			}

			if (layer.method && layer.method !== method) {
				return next(err);
			}

			if (err) {
				layer.handle_error(err, req, res, next);
			} else {
				layer.handle_request(req, res, next);
			}
		}
	};

在知道了`layer`和`stack`这两个事物的基础上，这个函数的操作流程就很好理解了，其实就是通过函数递归的方法，对Route对象的`stack`按插入顺序进行遍历，然后依次执行`stack`里的`layer`的过程。
当然，首先`req.method`也就是请求的http方法必须在Route对象中的methods之中。

最后就是定义如何调用Route对应的HTTP方法,也就是`router.get`,`router.post`等最终执行的地方

	Route.prototype.all = function all() {
		var handles = flatten(slice.call(arguments));

		for (var i = 0; i < handles.length; i++) {
			var handle = handles[i];

			if (typeof handle !== 'function') {
				var type = toString.call(handle);
				var msg = 'Route.all() requires callback functions but got a ' + type;
				throw new TypeError(msg);
			}

			var layer = Layer('/', {}, handle);
			layer.method = undefined;

			this.methods._all = true;
			this.stack.push(layer);
		}

		return this;
	};

	methods.forEach(function(method){
		Route.prototype[method] = function(){
			var handles = flatten(slice.call(arguments));

			for (var i = 0; i < handles.length; i++) {
				var handle = handles[i];

				if (typeof handle !== 'function') {
					var type = toString.call(handle);
					var msg = 'Route.' + method + '() requires callback functions but got a ' + type;
					throw new Error(msg);
				}

				debug('%s %s', method, this.path);

				var layer = Layer('/', {}, handle);
				layer.method = method;

				this.methods[method] = true;
				this.stack.push(layer);
			}

			return this;
		};
	});

两个代码块一个是定义了`Route.all`,一个是通过遍历`methods`(require('methods')，存储了各种HTTP请求方法)将其中的元素赋值到成Route对象的属性，也就有了`Route.get`,`Route.post`等方法。
其实这两个代码块其中的执行流程都大同小异。
在这里需要注意的是定义`Route.method`(这里method指代all,get,post等)时，其中第一行代码

	var handles = flatten(slice.call(arguments))

这里的handles就是app.get('/path', fn1, fn2, fn3)中的`fn1`,`fn2`,`fn3`等，也就是中间件的回调函数。
但是如果这样使用，有人会问不应该是slice.call(arguments, 1)，也就是从第二个参数开始截取吗？(slice 是 Array.prototype.slice，在route.js开头定义的)
刚开始看到这里的时候，笔者也有这个疑问，后来在index.js里面，也就是定义Router对象的地方，找到这么一段代码：

	proto.route = function route(path) {
		var route = new Route(path);

		var layer = new Layer(path, {
			sensitive: this.caseSensitive,
			strict: this.strict,
			end: true
		}, route.dispatch.bind(route));

		layer.route = route;

		this.stack.push(layer);
		return route;
	};

	// create Router#VERB functions
	methods.concat('all').forEach(function(method){
		proto[method] = function(path){
			var route = this.route(path)
			route[method].apply(route, slice.call(arguments, 1));
			return this;
		};
	});

这里可以看出Route是被放在了Router的`stack`里的`layer.route`，然后在调用类似Router\[method\](path, fn1, fn2)的时候，已经将其中path提取出来，并且直接通过调用`this.route(path)`赋值到`Route`中的`path`属性，之后将`fn1`,`fn2`通过`slice.call(arguments,1)`的方式截取出来，使用route[method].apply调用到Route.method方法。
所以在Route.method调用的时候，其arguments已经是回调函数`fn1`，`fn2`等的数组了。

OK，解决了这个疑问，继续下一个重点，可以看见在Route.method的定义中，最终都返回了`this`，加上之前对于arguments的处理，就形成了路由中间件的灵活调用方法：

	router.get('/path', fn1, fn2, fn3);

	router.get('/path', [fn1, [fn2, [fn3]]]);

	router.get('/path', fn1).get('/path', fn2).get('/path', fn3)

这三个最终实现的结果是一样的，第一个和第二个没有什么区别，第三个有些许不一样，第一个和第二个在Router中'fn1,fn2,fn3'都是在同一个layer.route之中，而第三个则是在不同的layer.route之中。
第一个和第二个是通过遍历Route的stack来找到fn进行执行，而第三个是遍历Router的stack来找到fn进行执行。简单来说就是一个放在外层的stack，一个放在内层的stack。

其实在route.js这部分必须要结合index.js来看，不然对于一些实现方法不是很好理解。

#### index.js

老规矩，还是先从导出对象的基本定义开始。

	var proto = module.exports = function(options) {
		var opts = options || {};

		function router(req, res, next) {
			router.handle(req, res, next);
		}

		// mixin Router class functions
		router.__proto__ = proto;

		router.params = {};
		router._params = [];
		router.caseSensitive = opts.caseSensitive;
		router.mergeParams = opts.mergeParams;
		router.strict = opts.strict;
		router.stack = [];

		return router;
	};

这里的初始化定义应该不难看懂，options参数就是`pathRegexp`要求的三个配置参数`caseSensitive`,`mergeParams`,`strict`。
`router.stack`前面也解释得比较多了，这里也不再赘述。`router.params`和`router._params`是定义app.params(param, fn)中会使用到存储对象。
需要注意的是Router最终返回的是 `router.handle(req,res,next)`的执行函数，`router.handle`是定义的Router中的路由派发操作，类似Route.dispatch，之后这里会详细解释。

在这之后在Router对象上定义了param(name, fn)方法，这其实就是app.param(name, fn)定义的部分，这部分会在appliction.js绑定到app.param上，也就成了express文档里的app.param。这部分没有什么难以理解的地方，结合文档看基本可以理解，所以这里就跳过proto.param(name, fn)。

之后就是在Router的初始化里提到过的router.handle的定义：

	proto.handle = function handle(req, res, out) {
		var self = this;

		debug('dispatching %s %s', req.method, req.url);

		var search = 1 + req.url.indexOf('?');
		var pathlength = search ? search - 1 : req.url.length;
		var fqdn = req.url[0] !== '/' && 1 + req.url.substr(0, pathlength).indexOf('://');
		var protohost = fqdn ? req.url.substr(0, req.url.indexOf('/', 2 + fqdn)) : '';
		var idx = 0;
		var removed = '';
		var slashAdded = false;
		var paramcalled = {};

		// store options for OPTIONS request
		// only used if OPTIONS request
		var options = [];

		// middleware and routes
		var stack = self.stack;

		// manage inter-router variables
		var parentParams = req.params;
		var parentUrl = req.baseUrl || '';
		var done = restore(out, req, 'baseUrl', 'next', 'params');

		// setup next layer
		req.next = next;

		// for options requests, respond with a default if nothing else responds
		if (req.method === 'OPTIONS') {
			done = wrap(done, function(old, err) {
				if (err || options.length === 0) return old(err);
				sendOptionsResponse(res, options, old);
			});
		}

		// setup basic req values
		req.baseUrl = parentUrl;
		req.originalUrl = req.originalUrl || req.url;

		next();

		function next(err) {
			/**next 定义，暂时省略 **/
		}

		function trim_prefix(layer, layerError, layerPath, path) {
			/**暂时省略 **/
		}
	}

因为这部分加上内部定义的函数的话，代码量比较多，一下贴出来不太容易理清思路，所以先将`next`和`trim_prefix`函数省略(`trim_prefix`是在`next`内部调用的)，先看看handle大概做了一件什么事情。
之前也说过`router.handle`是派发req和res到每个路由中间件的操作，联系之前提到的Route中的`dispatch`方法是通过函数调用的方法来遍历`stack`的中间件实现的req和res的派发，所以这里也是用了同样的操作：“通过递归调用`next`来实现路由派发”。
但是在调用next()之前，`router.handle`对`req.url`也就是原始的url进行了处理，把请求协议和search参数以及host都提取了出来，所以可以理解为在派发路由前，先对url做了一个预处理操作。

接下来就是仔细看一下看`next`函数是具体定义了一个怎么样的操作：

	function next(err) {
		var layerError = err === 'route'
			? null
			: err;

		// remove added slash
		if (slashAdded) {
			req.url = req.url.substr(1);
			slashAdded = false;
		}

		// restore altered req.url
		if (removed.length !== 0) {
			req.baseUrl = parentUrl;
			req.url = protohost + removed + req.url.substr(protohost.length);
			removed = '';
		}

		// no more matching layers
		if (idx >= stack.length) {
			setImmediate(done, layerError);
			return;
		}

		// get pathname of request
		var path = getPathname(req);

		if (path == null) {
			return done(layerError);
		}

		// find next matching layer
		var layer;
		var match;
		var route;

		while (match !== true && idx < stack.length) {
			layer = stack[idx++];
			match = matchLayer(layer, path);
			route = layer.route;

			if (typeof match !== 'boolean') {
				// hold on to layerError
				layerError = layerError || match;
			}

			if (match !== true) {
				continue;
			}

			if (!route) {
				// process non-route handlers normally
				continue;
			}

			if (layerError) {
				// routes do not match with a pending error
				match = false;
				continue;
			}

			var method = req.method;
			var has_method = route._handles_method(method);

			// build up automatic options response
			if (!has_method && method === 'OPTIONS') {
				appendMethods(options, route._options());
			}

			// don't even bother matching route
			if (!has_method && method !== 'HEAD') {
				match = false;
				continue;
			}
		}

		// no match
		if (match !== true) {
			return done(layerError);
		}

		// store route for dispatch on change
		if (route) {
			req.route = route;
		}

		// Capture one-time layer values
		req.params = self.mergeParams
			? mergeParams(layer.params, parentParams)
			: layer.params;
		var layerPath = layer.path;

		// this should be done for the layer
		self.process_params(layer, paramcalled, req, res, function (err) {
			if (err) {
				return next(layerError || err);
			}

			if (route) {
				return layer.handle_request(req, res, next);
			}

			trim_prefix(layer, layerError, layerPath, path);
		});
	}

在这里主要关注一下stack的遍历操作。在`next`之中stack的遍历先是通过while循环判断`match`值来确定是否找到匹配的`layer`, `match`是通过调用`matchLayer`返回结果，该方法定义为：

	function matchLayer(layer, path) {
		try {
			return layer.match(path);
		} catch(err) {
			return err;
		}
	}

其实就是调用`layer.match`，只是加了个try catch而已。

如果while循环如果找到了匹配的layer，那么会就会调用`proto.process_params`的方法，也就是self.process_params()这部操作；

`proto.process_params`的源码这里就不贴出来了，了解一下操作流程就好。这个方法需要对应`proto.param`来理解，其实就是***查询并调用***`proto.param`里面***定义且匹配的***与`req.param`绑定的function，这些回调函数都是存储在`proto.params`里面的，所以这里就又会有一个通过函数递归的遍历操作。
在遍历完了之后就是调用

	function (err) {
			if (err) {
				return next(layerError || err);
			}

			if (route) {
				return layer.handle_request(req, res, next);
			}

			trim_prefix(layer, layerError, layerPath, path);
	}

如果`layer.route`是存在的，结合之前分析的`Layer`和`Route`以及在Route的分析中提前提到这部分后面定义的`Router.route`，也就是：

	proto.route = function route(path) {
		var route = new Route(path);

		var layer = new Layer(path, {
			sensitive: this.caseSensitive,
			strict: this.strict,
			end: true
		}, route.dispatch.bind(route));

		layer.route = route;

		this.stack.push(layer);
		return route;
	};

`layer.handle_request`会执行`layer.route`的`dispatch`操作，也就是在Route中派发路由最终执行到在express中定义的对应的路由操作函数，之后又执行`next()`就又到了这里的`proto.stack`中的下一个遍历操作。
但是如果`layer.route`不存在，说明这只是一个中间件而不是路由中间件，也就是用`route.use`定义的中间件，所以就调用`trim_prefix`方法来执行路由中间件：

	function trim_prefix(layer, layerError, layerPath, path) {
		var c = path[layerPath.length];
		if (c && '/' !== c && '.' !== c) return next(layerError);

			// Trim off the part of the url that matches the route
			// middleware (.use stuff) needs to have the path stripped
		if (layerPath.length !== 0) {
			debug('trim prefix (%s) from url %s', layerPath, req.url);
			removed = layerPath;
			req.url = protohost + req.url.substr(protohost.length + removed.length);

			// Ensure leading slash
			if (!fqdn && req.url[0] !== '/') {
				req.url = '/' + req.url;
				slashAdded = true;
			}

			// Setup base URL (no trailing slash)
			req.baseUrl = parentUrl + (removed[removed.length - 1] === '/'
				? removed.substring(0, removed.length - 1)
				: removed);
		}

		debug('%s %s : %s', layer.name, layerPath, req.originalUrl);

		if (layerError) {
			layer.handle_error(layerError, req, res, next);
		} else {
			layer.handle_request(req, res, next);
		}
	}

这个方法是在`proto.handle`里定义，和`next`同级。

OK，这里再回到while，如果while循环没有找到匹配的layer就进行收尾工作，要么就根据layerError也就是next(err)中的err参数判断是否已经匹配到结果正常首尾，或者就是404没找到匹配。

分析到这里，这里使用两张图对上面的分析进行一个整理。

首先是layer,Route和Router这三者的关系（原图片来自文章[从express源码中探析其路由机制](https://cnodejs.org/topic/545720506537f4d52c414d87)）

![layer,Route,Router](http://7xu027.com1.z0.glb.clouddn.com/router_route_layer.png)


然后是`proto.handle`的一个执行流程：

![proto.handle](http://7xu027.com1.z0.glb.clouddn.com/express_router_handle.png)

有了这两张图，再配合之前的文字描述，应该对路由这部分的处理不会有什么问题了。

### application


### request 和 response


### view


### 从请求到响应