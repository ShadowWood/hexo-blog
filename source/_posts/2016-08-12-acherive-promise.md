---
layout: post
title:  "Promise的简单实现"
date:   2016-08-12 10:08:00
categories: javascript
tags: [javascript,node.js,Promise]
description: "Promise是js的一个异步流程控制标准，之前对async和promise的使用做了一个简单的对比，这次根据Promise/A+规范实现一个简单的Promise，以便对Promise有更深的了解"
img_url: "http://7xu027.com1.z0.glb.clouddn.com/own-world.jpg"
---

### 前言

Promise是js的一个异步流程控制标准，是为了解决js中异步回调过多导致的代码结构混乱的问题。
Promise标准已经被写入了ES6的语法中，ES6已经有了原生的Promise对象。之前对async和promise
的使用做了一个简单的对比，为了更好的理解Promise对异步回调的一个控制流程，这次根据Promise/A+规范实现一个简单的Promise。

<!-- more -->

### Promise/A+

Promise/A+是一个开放、健全且通用的Javascript Promise标准，是由Javascript的开发者制定的，以供其他开发者参考。

+ [Promise/A+英文原文](https://promisesaplus.com/)
+ [Promise/A+中文解读](http://malcolmyu.github.io/malnote/2015/06/12/Promises-A-Plus/)

上面是关于Promise/A+规范的资料，这里就不再对Promise/A+规范做进一步的解读了。

### 初始化Promise对象

首先定义一个Promise对象：

    function Promise(fn) {
        var __self = {
            status: 'Pending',
            value: undefined,
            options: ['Pending', 'Resolved', 'Rejected']
        }
    }

在这个Promise对象之中，定义了一个私有变量__self:

+ __self.status: 该值用来存储该Promise对象当前的状态
+ __self.value: 该值用来存储Promise的终值(resolved时的value)或者拒绝的原因(rejected时的reason)
+ __self.options: 该值为三个状态值的数组

因为Promise/A+中规定，Promise对象状态一旦处于resolved或者rejected，其状态和值便不能更改，所以其状态值和终值必须存储在私有变量中，防止外部对其进行操作更改。

但是这些私有变量也会涉及到值的更改，而且这些更改操作不一定在对象内部定义。所以在Promise对象定义的时候还需要定义一些set和get方法：

    function Promise(fn) {
        var __self = {
            status: 'Pending',
            value: undefined,
            options: ['Pending', 'Resolved', 'Rejected']
        }

        var self = this;

        self.__getStatus = function () {
            return __self.status
        }

        self.__setStatus = function (val) {
            var index = __self.options.indexOf(val);
            if (index === -1 || __self.status === 'Resolved' || __self.status === 'Rejected') {
                return false;
            }

            __self.status = val;
            return true;
        }

        self.__getValue = function () {
            return __self.value
        }

        self.__setValue = function () {
            if (__self.status === 'Pending') {
                __self.value = val;
                return true;
            } else {
                return false;
            }
        }

    }

首先是status的get和set函数，get函数直接返回__self.status的值，而在set函数之中则需要加一些限制。
在Promise/A+标准中规定状态(status)的更改只能是从pending到resolved或者rejected，当状态为resolved或者rejcted的时候，便不能再更改状态。

然后是value的get和set函数，get函数同理直接返回__self.value，而在set函数之中，要求只能是状态为pending的时候才能对值进行修改，否则修改无效。

### resolve和reject

在Promise对象实例化的时候，其参数函数中(例如的Promise对象参数fn)有两个参数: resolve和reject。
这两个参数是Promise对象实例化执行结束时调用的回调函数：
+ resolve(value): 在执行成功时调用，将Promise对象的状态修改为resolved并且终值修改为传入的value
+ reject(err): 在执行成功时调用，将Promise对象的状态修改为rejected并且将拒绝原因修改为传入的err

这两个函数既可以定义在Promise对象外面，也可以定义在Promise对象里面。
当定义的Promise对象外面的时在之后执行的时候需要使用.bind(this)方法将Promise对象的this传入，所以其最终结果和定义在内部没有多少差异，所以这里还是采用定义在内部的方式：

    function Promise(fn) {
        var __self = {
            status: 'Pending',
            value: undefined,
            options: ['Pending', 'Resolved', 'Rejected']
        }

        var self = this;

        self.onResolvedCallback = []; // Promise resolved 的回调函数集
	    self.onRejectedCallback = []; // Promise rejected 的回调函数集


        /*****此处省略了之前的status和value的set和get函数******/

        function resovle (val) {
            self.__setValue(val);
            self.__setStatus('Fulfilled')
            for (var i = 0; i < self.onResolvedCallback.length; i++) {
                self.onResolvedCallback[i](val);
            }
        }

        function reject (err) {
            if (err) {
                self.__setValue(err);
                self.__setStatus('Rejected');
            }
            for (var i = 0; i < self.onRejectedCallback.length; i++) {
                self.onRejectedCallback[i](err);
            }
        }

        try {
            fn(resolve, reject)
        } catch (err) {
            reject(err)
        }

    }

注意这次加入了两个特别的值，onResolvedCallback和onRejectedCallback，分别用来存储Promise对象revolved和rejected时的回调函数。
至于为什么需要使用这两个值以及这两个值实现了什么功能，这个在之后的构建.then方法的时候再详细解释。

