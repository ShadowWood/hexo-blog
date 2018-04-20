---
layout: post
title:  "浅谈js异步流程框架async与Promise的区别"
date:   2016-04-25 16:02:00
categories: Javascript
tags: [javascript,node.js,async,Promise,异步]
cover_index: http://7xu027.com1.z0.glb.clouddn.com/async_dribbble.png
---

### 前言

在node.js中执行异步操作时，操作的结果并不是通过返回函数来反馈，而是通过将结果传入回调函数，由回调函数来处理运行结果。
如果在一个程序之中，我们需要多数的异步I/O操作，那么我们的回调结构将会是一层套一层不断的嵌套下去形成"callback hell(回调地狱)"。
虽然我们也可以通过声明函数的方法来简化回调结构并避免callback hell的产生，但是如果遇到一些比较复杂的依赖关系的话，可能依然会有一些问题。
最近接触到node.js两个不同的异步模块async和Promise，给大家做一下对比和分享。
<!-- more -->

### async 和 Promise简介

async([github](https://github.com/caolan/async))是一个包含了许多js异步操作方法的node.js模块，其中共有70个函数，包括一些与数组类似的方法(map, each, filter, reduce...)，和一些异步流程方法(series, parallel, waterfall, ...)。
这些方法都必须要求在node环境在才能运行实现，并且运行结果都是传递到一个回调函数里，一般都是两个参数，一个参数是错误异常error，另一个是操作结果。

Promise应该算是一个异步操作标准。
Promise被定义为一个对象，来传递异步操作的消息，它代表了某个未来才会知道的结果的事件，并且这个事件提供统一的API，可供进一步的处理；Promise对象有两个特点：

+ 对象的状态不受外界影响；
+ 对象的状态一旦改变便不会再变，并且任何时候都可以得到这个结果。

对于Promise的实现有很多js库，并且对于这些js库也有一套评价标准[Promise/A+](https://promisesaplus.com/)。
本文介绍的Promise使用的bluebird([官方文档](http://bluebirdjs.com/docs/getting-started.html))这个库。


### 一个简单的异步实现

我们来进行一个简单的操作：使用fs.readFile从read.txt里面读取文件内容，然后将读取的内容用writeFile写入write.txt。

#### 不用async和Promise

    var fs = require('fs');
    fs.readFile('./read.txt', (err, data) => {
        fs.writeFile('./write.txt', data, (err) => {
            console.log("end");
        })
    })

#### async

    var async = require('async');
    var fs = require('fs');
    async.waterfall([
      callback => fs.readFile('./read.txt',
        (err, data) => {
          callback(null, data)
      }),
      (data, callback) => fs.writeFile('./write.txt',
        data,
        (err) => {
          callback(err)
        })
    ], (err, r) => console.log("end"));

#### Promise

    var Promise = require('bluebird');
    var fs = require('fs');
    var promise = new Promise((resolve, reject) => {
      fs.readFile('./read.txt', (err, data) => {
        if(err) reject(err);
        resolve(data)
      })
    });

    promise.then((data) => {
      fs.writeFile('./write.txt', data, (err) => {
        console.log("end")
      })
    });

 咋一看觉得不用async或者Promise好像结构还简单一些，那么如果我们是这样一个操作流程:

    function a(data1, data2 => {
        function b(data2, data3 => {
            function c(data3, data4 => {
                function d(data4, data5 => {
                    ....
                })
            })
        })
    })

也就是说当我们遇到有多层回调的时候，这个结构就相当难看了，当时如果用async或者Promise的话,可以将多层回调变成同步的样式来进行操作

    //async
    async.waterfall([
        cb => cb(err, data1),
        cb => cb(err, data1, data2),
        cb => cb(err, data2, data3),
        cb => cb(err, data3, data4),
        ...
    ], (err, r) => console.log("end"))

    // Promise
    var promise = new Promise((resolve, reject) => {
        resolve(data1)
    })
    promise
    .then(data1 => return data2)
    .then(data2 => return data3)
    .then(data3 => return data4)
    ...

### 简单对比

async的主要实现方式挺容易理解的，它本质是提供了一系列的工具函数，来帮助解决多层回调的问题，最常见的方式就是将多个异步操作集合到数组里，然后根据这些异步操作的相互间的关系来选择它提供的方法进行实现。

而Promise是将异步操作封装为对象，异步的结果输出到对象的.then()方法里，然后.then()方法的返回值又是一个Promise对象并且在.then()方法里return的值会传递到下一个.then()，即链式法则。

在async中我们如果在处理上一个异步回调的结果的时候，又需要用一个异步回调，并且还需要将这个异步回调的结果传递给下一个异步回调的话，我们只需要将结果放在callback()里就行了。

但是在Promise里，我们如果在.then()方法里还需要有异步操作，并且需要将这个异步操作的值传递出去，以便链式方法能够获取到这个参数该怎么办呢？

这里我们在稍微改一下上一个例子，我们先向write.txt里面写入一个'test'字符串，操作成功后读取write.txt的内容并打印，然后再读取read.txt里面的内容，然后将之前内容覆盖并写入write.txt里面：

    var Promise = require('bluebird');
    var fs = require('fs');
    var readFileAsync = Promise.promisify(fs.readFile);
    var writeFileAsync = Promise.promisify(fs.writeFile);

    writeFileAsync('./write.txt', 'test\n')
      .then(() => readFileAsync('./write.txt'))
      .then((data) => {
        console.log(data.toString());
        return readFileAsync('./read.txt')
      })
      .then(data => writeFileAsync('./write.txt', data))
      .then(() => readFileAsync('./write.txt'))
      .then(data => console.log(data.toString()));

首先我们将fs.readFile和fs.writeFile通过bluebird的promisify方法做Promise封装，命名为readFileAsync和writeFileAsync。
这样它们的返回值就变成了Promise对象，并且它们的本来应该返回到回调函数里的结果返回到了.then()方法里。
因为我们只要在.then()方法里return Promise对象，.then()的值就是我们内层返回的Promise对象，故我们又可以继续用.then()来使用数据并且操作，依次类推。

所以用Promise的进行多层异步回调的简化的一个基本思路就是，我们需要将这些异步回调想办法返回Promise对象，做Promise封装也好，重新构建一个Promise对象返回也好，都大同小异。

而async的对多层回调简化的思路就是只需要将值传递到callback里，就能被外层的函数来获取到结果。