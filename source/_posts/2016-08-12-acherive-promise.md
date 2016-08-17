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
Promise标准已经被写入了ES6的语法中，ES6已经有了原生的Promise对象。之前对async和promise的使用做了一个简单的对比，为了更好的理解Promise对异步回调的一个控制流程，这次根据Promise/A+规范实现一个简单的Promise。

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
            if (!err) {
                return ;
            }
            self.__setValue(err);
            self.__setStatus('Rejected');
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

注意这次加入了两个特别的值，`onResolvedCallback` 和 `onRejectedCallback`，分别用来存储 Promise 对象 revolved 和 rejected 时的回调函数。
至于为什么需要使用这两个值以及这两个值实现了什么功能，这个在之后的构建.then方法的时候再详细解释。

其实`resolve`和`reject`两个操作流程基本是一致的，都进行了三步操作：

1. 设置终值或拒绝原因;
2. 修改状态为`Resovled`或 `Rejected`
3. 遍历回调函数集并执行回调函数；

这两者操作的区别只在于状态值和回调函数集不同，所以这两个函数完全可以合并成一个操作函数。这里只是为了语义更清晰一点，这里还是决定分开写。
之后直接执行Promise对象的fn参数函数，并且传入`resovle` 和 `reject` 当作fn的参数，之后在fn内部执行回调。
注意，这里执行fn的时候必须使用try catch捕捉错误并执行reject，因为完全可能在fn的执行过程中出现错误，这一部分也需要在Promise对象里捕捉到。

这里的`resolve`和`reject`还存在一个缺陷，就是在遍历`onRejectedCallback`和`onResolvedCallback`的时候可能会花费较大部分的时间，所以应该采用异步的写法，即加入setTimeout将其操作流程改成异步的:

    function resovle (val) {
        setTimeout(function() {
            self.__setValue(val);
            self.__setStatus('Fulfilled')
            for (var i = 0; i < self.onResolvedCallback.length; i++) {
                self.onResolvedCallback[i](val);
            }
        })
    }

    function reject (err) {
        if (!err) {
            return ;
        }
        setTimeout(function() {
            self.__setValue(err);
            self.__setStatus('Rejected');
            for (var i = 0; i < self.onRejectedCallback.length; i++) {
                self.onRejectedCallback[i](err);
            }
        })
    }


### .then方法

`Promise`的`.then`方法是`Promise`对象的核心部分之一，可以说这个链式法则是Promise广受推崇的一个重要原因吧。
对于`.then`方法，有几个重点要特别注意:

+ `.then`方法返回的是一个Promise对象；
+ `.then`方法有两个参数`onResolved`和`onRejected`，分别为Promise状态为Resolved和Rejected时的回调函数；
+ `.then`方法可以在Promise还未执行完成的时候就开始执行，也就是说当Promise为`Pending`状态的时候，`.then`方法也可以执行；

针对以上几个要点，这里先从调用`.then`方法时Promise已经Resovled来实现：
    
    Promise.prototype.then = function (onResolved, onRejected) {
        onResolved = typeof onResolved === 'function' ? onResolved : function (v) { return v };
        onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw e };
        var self = this;
        var status = self.__getStatus();
        var value = self.__getValue();

        if (status === 'Pending') {
            return new Promise(function (resolve, reject) {
                try {
                    var x = onResolved(value);
                    if (x instanceof Promise) {
                        x.then(resolve, reject)
                    } else {
                        resolve(x)
                    }
                } catch (e) {
                    reject(e)
                }
            })
        }
    }

首先判断传入的参数`onResolved`和`onRejected`是否是函数，如果不是则分别默认为 `function (v) { return v }` 和 `function (e) { throw e }`，以便实现值的穿透(后面详细介绍)。
然后获取到当前Promise对象的`status` 和 `value`，并返回一个新的Promise对象，在新的Promise对象中，执行`onResolved`函数。
之后判断`onResolved`函数的返回值是否为Promise对象，如果是Promise对象，直接调用其`.then`方法并传入`resolve`和`reject`做为其`onResolved`和`onRejected`参数，如果不是则直接执行`resolve`。
在这个操作流程之中，需要使用`try catch`包装代码块并将捕捉到的错误传入`reject`执行，因为完全可能在执行`onResolved`的过程中出现错误而阻止后续代码的执行。

当执行`.then`方法时Promise对象状态为`Rejected`的时候操作流程基本一致，只是将`onResovled`替换成`onRejected`就行了。
所以这里可以将生成新的Promise对象内部的操作流程写成一个新的函数：

    function thenDeal(fn, value, resolve, reject) {
        try {
            var x = fn(value);
            if (x instanceof Promise) {
                x.then(resolve, reject)
            } else {
                resolve(x)
            }
        } catch (e) {
            reject(e)
        }
    }

这个函数可以写在`.then`方法外面并直接调用：

    Promise.prototype.then = function (onResolved, onRejected) {
        onResolved = typeof onResolved === 'function' ? onResolved : function (v) { return v };
        onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw e };
        var self = this;
        var status = self.__getStatus();
        var value = self.__getValue();
        switch (status) {
            case 'Resolved':
                return new Promise(function (resolve, reject) {
                    thenDeal(onResolved, value, resolve, reject)
                })

            case 'Rejected':
                return new Promise(function (resolve, reject) {
                    thenDeal(onRejected, value, resolve, reject)
                })
        }
    }

接下来就是需要考虑当`.then`方法执行时，Promise对象还是`Pending`状态的情况了，这里就需要用到之前定义的回调函数集合`onResolvedCallback`和`onRejectedCallback`。
其实思路也并不复杂，只是需要注意一些值的调用，先给出完整的实现方法：

    Promise.prototype.then = function (onResolved, onRejected) {
        onResolved = typeof onResolved === 'function' ? onResolved : function (v) { return v };
        onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw e };
        var self = this;
        var status = self.__getStatus();
        var value = self.__getValue();
        switch (status) {
            case 'Pending':
                return new Promise(function (resolve, reject) {
                    self.onResolvedCallback.push(function () {
                        thenDeal(onResolved, self.__getValue(), resolve, reject)
                    });
                    self.onRejectedCallback.push(function () {
                        thenDeal(onRejected, self.__getValue(), resolve, reject)
                    })
                });
            case 'Revolved':
                return new Promise(function (resolve, reject) {
                    thenDeal(onResolved, value, resolve, reject)
                })

            case 'Rejected':
                return new Promise(function (resolve, reject) {
                    thenDeal(onRejected, value, resolve, reject)
                })
        }
    }

    function thenDeal(fn, value, resolve, reject) {
        try {
            var x = fn(value);
            if (x instanceof Promise) {
                x.then(resolve, reject)
            } else {
                resolve(x)
            }
        } catch (e) {
            reject(e)
        }
    }

在`Pending`状态的时候，因为完全不清楚Promise最终的执行结果是`Resolved`还是`Rejected`，所以需要把`onResolved`和`onRejected`分别弄进`onResolvedCallback`和`onRejectedCallback`里，以传入回调函数的形式最终在里面执行`thenDeal`。
这里需要注意一个特别的点，那就是`thenDeal`的value参数(即`Resolved`时的`value`或`Rejected`时的`reason`)必须是实时的Promise对象的值，即Promise的终值或拒绝原因，这部分和之前的`Resolved`及`Rejected`状态时的处理方式不同。

`.catch`可以直接用`.then`方法来定义：

    Promise.prototype.catch = function(onRejected) {
        return this.then(null, onRejected)
    }

对于这两行代码：

    onResolved = typeof onResolved === 'function' ? onResolved : function (v) { return v };
    onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw e };

条件判断的作用就不解释了，至于为什么要有`function (v) {return v}`和`function (e) {throw e}`的默认值，是为了实现Promise `.then`方法的值传递。例如：

    var p = new Promise((resolve, reject) => resolve(1));
    p.then()
     .then(v => {
         console.log(1);
         throw(2);
      })
     .then()
     .catch(e => console.log(e))

该代码的最终输出结果为 1 和 2。

### Promise的解决过程

在之前的`.then`方法实现里面的`thenDeal`方法是直接通过判断 `x instanceof Promise`来确实返回值是否是Promise对象，这种实现不符合Promise/A+规范的，并且无法和其他Promise交互。
在 `Promise/A+` 的2.3部分定义了Promise .then方法里的解决过程，并且阐述了`thenable`特性。原文翻译为：

> Promise解决过程是一个抽象的操作，其需输入一个`promise`和一个值，我们表示为 `[[Resolve]](promise, x)`，如果 `x` 有 `then` 方法且看上去像一个 `Promise` ，解决程序即尝试使 `promise` 接受 `x` 的状态；否则其用 `x` 的值来执行 `promise`。

根据这个标准，需要加入一个`promiseResovle`方法来实现Promise解决过程，首先先明确这个`promiseResovle`方法的在之前的Promise实现需要插入的位置：

    Promise.prototype.then = function (onResolved, onRejected) {
        onResolved = typeof onResolved === 'function' ? onResolved : function (v) { return v };
        onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw e };
        var self = this;
        var status = self.__getStatus();
        var value = self.__getValue();
        switch (status) {
            case 'Pending':
                return promise2 = new Promise(function (resolve, reject) {
                    self.onResolvedCallback.push(function () {
                        thenDeal(onResolved, self.__getValue(), resolve, reject, promise2)
                    });
                    self.onRejectedCallback.push(function () {
                        thenDeal(onRejected, self.__getValue(), resolve, reject, promise2)
                    })
                });
            case 'Revolved':
                return promise2 = new Promise(function (resolve, reject) {
                    setTimeout(function () {
                        thenDeal(onResolved, value, resolve, reject, promise2)
                    })
                })

            case 'Rejected':
                return promise2 = new Promise(function (resolve, reject) {
                    setTimeout(function () {
                        thenDeal(onRejected, value, resolve, reject, promise2)
                    })
                })
        }
    }

    function thenDeal(fn, value, resolve, reject, promise) {
        try {
            var x= fn(value);
            promiseResovle(promise, x, resolve, reject);
        } catch (e) {
            reject(e)
        }
    }

相较于之前的代码，主要在`thenDeal`进行了修改，传入了即将返回的Promise对象`promise2`，并且将其传入`promiseResovle`中。
`promiseResovle`的功能是对`onResolved`或者`onRejected`的返回值进行处理解决，使其能实现多种Promise的交互。

根据`Promise/A+`的规定，`promiseResovle`需要依次判断三种情况：

1. 参数 `x` 与 参数 `promise` 相等时；
2. 参数 `x` 为 Promise(即自己定义的Promise，不是通用的或者其他的Promise对象)；
3. 参数 `x` 为对象或者函数；

下面依次实现着三种情况的判断和要求。

#### 参数 `x` 与 参数 `promise` 相等 

`Promise/A+`中规定，当参数 `x` 与 参数 `promise` 相等时，以`TypeError`为`reason`拒绝执行`promise`，实现代码如下：

    function promiseResovle(promise, r, resolve, reject) {
        
        if (r === promise) {
            return reject(new TypeError('The promise has been circular used'))
        }
    }

#### 参数 `x` 为 Promise

接着当 `x`为Promise时：

+ 如果 `x` 处于等待态， `promise` 需保持为等待态直至 `x` 被执行或拒绝
+ 如果 `x` 处于执行态，用相同的值执行 `promise`
+ 如果 `x` 处于拒绝态，用相同的据因拒绝 `promise`

接着之前的实现：

    function promiseResovle(promise, r, resolve, reject) {
        
        if (r === promise) {
            return reject(new TypeError('The promise has been circular used'))
        }

        if (r instanceof Promise) {
            var status = r.__getStatus();
            if (status === 'Pending') {
                r.then(function(value) {
                    promiseResovle(promise, value, resolve, reject)
                }, reject)
            } else {
                r.then(resolve, reject)
            }

            return ;
        }
    }

当 `x` 处于 `Pending`状态的时候，调用 `x` 的 `.then` 方法并在其中再次调用`promiseResolve`以待获取 `x` 的终值或拒因；
当 `x` 处于 `onRejected` 或者 `onResolved` 的时候，就可以直接执行 `x.then` 并传入 `resolve` 和 `reject` 来执行 `promise`；


#### 参数 `x` 为对象或者函数

当 `x` 为对象或者函数的时候，`Promise/A+`规定操作方法及顺序如下：

1. 把 `x.then` 赋值给 `then`
2. 如果取 `x.then` 的值时抛出错误 `e` ，则以 `e` 为据因拒绝 `promise`
3. 如果 `then` 是函数，将 `x` 作为函数的作用域 `this` 调用之。传递两个回调函数作为参数，第一个参数叫做 `resolvePromise` ，第二个参数叫做 `rejectPromise`:
    + 如果 `resolvePromise` 以值 `y` 为参数被调用，则运行 `[[Resolve]](promise, y)`
    + 如果 `rejectPromise` 以据因 `r` 为参数被调用，则以据因 `r` 拒绝 promise
    + 如果 `resolvePromise` 和 `rejectPromise` 均被调用，或者被同一参数调用了多次，则优先采用首次调用并忽略剩下的调用
    + 如果调用 `then` 方法抛出了异常 `e`：
        + 如果 `resolvePromise` 或 `rejectPromise` 已经被调用，则忽略之
        + 否则以 `e` 为据因拒绝 `promise`
4. 如果 `then` 不是函数，以 `x` 为参数执行 `promise`
5. 如果 `x` 不为对象或者函数，以 `x` 为参数执行 promise

根据上述的规定，加上之前的实现，最终`promiseResolve`的定义如下：

    function promiseResovle(promise, x, resolve, reject) {
        var then;
        var thenCalled = false;

        if (x === promise) {
            return reject(new TypeError('The promise has been circular used'))
        }

        if (x instanceof Promise) {
            var status = x.__getStatus();
            if (status === 'Pending') {
                x.then(function(value) {
                    promiseResovle(promise, value, resolve, reject)
                }, reject)
            } else {
                x.then(resolve, reject)
            }

            return ;
        }

        function resolvePromise(y) {
            if(thenCalled) return;
            thenCalled = true;
            promiseResolve(promise, y, resolve, reject)
        }

        function rejectPromise(r) {
            if (thenCalled) return;
            thenCalled = true;
            reject(r)
        }

        if(x !== null && (typeof x === 'Object' || typeof x === 'function')) {
            try {
                then = x.then;
                if (typeof then === 'function') {
                    then.call(x, resolvePromise, rejectPromise)
                } else {
                    resolve(x)
                }
            } catch (e) {
                reject(e)
            }
        } else {
            resolve(x)
        }
    }

### 完整版

最后，贴出完整版的Promise实现：

    function Promise(fn) {
        var __self = {
            status: 'Pending',
            value: undefined,
            options: ['Pending', 'Resolved', 'Rejected']
        }

        var self = this;

        self.onResolvedCallback = []; // Promise resolved 的回调函数集
	    self.onRejectedCallback = []; // Promise rejected 的回调函数集


        /*****status和value的set和get函数******/

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

        /*********END**********/

        /*******resolve 和 reject *******/

        function resovle (val) {
            setTimeout(function() {
                self.__setValue(val);
                self.__setStatus('Fulfilled')
                for (var i = 0; i < self.onResolvedCallback.length; i++) {
                    self.onResolvedCallback[i](val);
                }
            })
        }

        function reject (err) {
            if (!err) {
                return ;
            }
            setTimeout(function() {
                self.__setValue(err);
                self.__setStatus('Rejected');
                for (var i = 0; i < self.onRejectedCallback.length; i++) {
                    self.onRejectedCallback[i](err);
                }
            })
        }

        /***********END**********/

        try {
            fn(resolve, reject)
        } catch (err) {
            reject(err)
        }

    }

    Promise.prototype.then = function (onResolved, onRejected) {
        /*************定义回调函数集************/
        onResolved = typeof onResolved === 'function' ? onResolved : function (v) { return v };
        onRejected = typeof onRejected === 'function' ? onRejected : function (e) { throw e };
        /************END************/
        var self = this;
        var status = self.__getStatus();
        var value = self.__getValue();

        switch (status) {
            
            case 'Pending':
                return promise2 = new Promise(function (resolve, reject) {
                    self.onResolvedCallback.push(function () {
                        thenDeal(onResolved, self.__getValue(), resolve, reject, promise2)
                    });
                    self.onRejectedCallback.push(function () {
                        thenDeal(onRejected, self.__getValue(), resolve, reject, promise2)
                    })
                });

            case 'Revolved':
                return promise2 = new Promise(function (resolve, reject) {
                    setTimeout(function () {
                        thenDeal(onResolved, value, resolve, reject, promise2)
                    })
                })

            case 'Rejected':
                return promise2 = new Promise(function (resolve, reject) {
                    setTimeout(function () {
                        thenDeal(onRejected, value, resolve, reject, promise2)
                    })
                })
        }
    }

    function thenDeal(fn, value, resolve, reject, promise) {
        try {
            var x= fn(value);
            promiseResovle(promise, x, resolve, reject);
        } catch (e) {
            reject(e)
        }
    }

    function promiseResovle(promise, x, resolve, reject) {
        var then;
        var thenCalled = false;

        if (x === promise) {
            return reject(new TypeError('The promise has been circular used'))
        }

        if (x instanceof Promise) {
            var status = x.__getStatus();
            if (status === 'Pending') {
                x.then(function(value) {
                    promiseResovle(promise, value, resolve, reject)
                }, reject)
            } else {
                x.then(resolve, reject)
            }

            return ;
        }

        function resolvePromise(y) {
            if(thenCalled) return;
            thenCalled = true;
            promiseResolve(promise, y, resolve, reject)
        }

        function rejectPromise(r) {
            if (thenCalled) return;
            thenCalled = true;
            reject(r)
        }

        if(x !== null && (typeof x === 'Object' || typeof x === 'function')) {
            try {
                then = x.then;
                if (typeof then === 'function') {
                    then.call(x, resolvePromise, rejectPromise)
                } else {
                    resolve(x)
                }
            } catch (e) {
                reject(e)
            }
        } else {
            resolve(x)
        }
    }

    Promise.prototype.catch = function (onRejected) {
        return this.then(null, onRejected)
    }

    module.exports = Promise;


> 参考文献:
[Promise/A+规范英文原文](https://promisesaplus.com/)
[Promise/A+规范中文翻译](http://malcolmyu.github.io/malnote/2015/06/12/Promises-A-Plus/)
[剖析Promise内部结构](https://github.com/xieranmaya/blog/issues/3)