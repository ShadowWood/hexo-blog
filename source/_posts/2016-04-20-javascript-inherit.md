---
layout: post
title:  "浅析javascript的继承方式"
date:   2016-04-20 17:08:30
categories: Javascript
tags: [javascript,继承]
description: "本文将列举出javascript的6种继承方式及举例一些简单的应用场景"
img_url: "http://7xu027.com1.z0.glb.clouddn.com/1736__new-york-in-blue_p.jpg"
---

### 前言

继承是面向对象语言的一个最为重要的概念。许多面向对象语言都支持两周继承方式：接口继承和实现继承。
接口继承只继承方法签名，实现继承则继承实际的方法。
然而，函数并没有签名，所以在ES中无法实现接口继承，只有实现继承，并且实现方式主要是依靠原型链来实现。
<!-- more -->

### 原型链

原型链的基本思想是**利用原型让一个引用类型继承另一个引用类型的属性和方法**，其基本模式如下：

    function SuperType () {
        this.property = true;
    }

    SuperType.prototype.getProperty = function () {
        console.log(this.property)
    }

    function SubType () {
        this.subProperty = false;
    }

    SubType.prototype = new SuperType();
    SubType.prototype.getSubProperty = function () {
        console.log(this.subProperty)
    }

    var instance = new SubType();
    console.log(instance.property); // true
    console.log(instance.subProperty); // false
    instance.getProperty(); // true
    instance.getSubProperty(); // false

在以上代码中，SubType继承了SuperType的属性和方法，实现的方法是通过将SuperType的实例赋于SubType.prototype，即重写SubType的原型对象。

我们可以通过instanceof操作符或者isPrototypeOf()方法来确定原型和实例之间的关系，对于以上代码：

    console.log(instance instanceof Object); // true
    console.log(instance instanceof SuperType); // true
    console.log(instance instanceof SubType); // true

    console.log(Object.prototype.isPrototypeOf(instance)); // true
    console.log(SuperType.prototype.isPrototypeOf(instance)); // true
    console.log(SubType.prototype.isPrototypeOf(instance)); //true

原型链继承是最常见的继承方式，但是也存在一些问题。最主要的问题来自包含引用类型的原型，例如:

    function SuperType () {
        this.arr = [1, 2, 3, 4]
    }

    function SubType () {}

    SubType.prototype = new SuperType();

    var instance1 = new SubType();
    instance1.arr.push(4);
    console.log(instance1.arr); // 1, 2, 3, 4

    var instance2 = new SubType();
    console.log(instance2.arr); // 1, 2, 3, 4

在这个例子中，我们想对instance1的arr数组添加元素4，但是却影响到了instance2。
这是因为arr这个属性是来自于SuperType的同一个实例，也就是说instance1.arr和instance2.arr指向的是同一个数组，当对instance1.arr操作时，就会一样影响到instance2.arr。

原型链的第二个问题是：当我们创建子类型的实例时，没有办法在不影响所有超类型的对象实例的情况下向继承于超类型的构造函数中传递参数。

基于这两个主要问题，所以在实际运用中很少单独用原型链的方式来实现继承。

### 借用构造函数

借用构造函数的基本思想是**在子类型构造函数的内部调用超类型构造函数**，如下所示:

    function SuperType (name) {
        this.arr = [1, 2, 3];
        this.name = name;
    }

    function SubType () {
        // 继承 SuperType
        SuperType.call(this, 'subtype')
    }

    var instance0 = new SuperType('supertype');
    console.log(instance0.name); // 'supertype'

    var instance1 = new SubType();
    instance1.arr.push(4);
    console.log(instance1.arr); // 1, 2, 3, 4
    console.log(instance1.name); // 'subtype'
    console.log(instance0.name); // 'supertype'

    var instance2 = new SubType();
    console.log(instance2.arr); // 1, 2, 3
    console.log(instance2.name); // 'subtype'


上例中，通过在SubType中使用SuperType.apply()方法来操作SubType的作用域this，使得子类型定义了与超类型相同的属性，从而实现了SubType继承了SuperType的属性，并且我们还能在不影响SuperType所有对象实例的情况下，通过传递参数来定义继承于SuperType的属性。

借用构造函数的继承方式解决了原型链中存在的两个主要问题，即：

+ **包含引用类型引起的问题**
+ **无法在不影响超类型的所有实例对象的情况下向子类型构造中向超类型构造函数传递参数**

但是借用构造函数由于是用在内部调用构造函数来实现继承的，所以它的方法都是在构造函数里定义，那么就无法像原型链一样使用**Object.prototype.attr = function** 的定义方法的方式来实现函数的复用，所以在实际运用中也很少单独使用借用构造函数的方式来实现继承


### 组合继承

结合原型链和借用构造函数的继承方式，取长补短便产生了组合继承，即**使用原型链实现对原型属性和方法的继承，而通过借用构造函数来实现对实例属性的继承**，如下所示:

    function SuperType (name) {
      this.name = name;
      this.arr = [1, 2, 3];
    }

    function getName () {
      console.log(this.name)
    }

    SuperType.prototype.getName = getName;

    function SubType (name, age) {
      // 借用构造函数，实现继承实例属性
      SuperType.call(this, name);
      this.age = age;
    }

    // 原型继承，实现对方法的继承
    SubType.prototype = new SuperType();
    SubType.prototype.constructor = SubType;
    SubType.prototype.getAge = function () {
      console.log(this.age)
    };

    var instance1 = new SubType('in1', 10);
    instance1.arr.push(4);
    console.log(instance1.arr); // 1, 2, 3, 4
    instance1.getName(); // 'in1'
    instance1.getAge(); // 10

    var instance2 = new SubType('in2', 20);
    console.log(instance2.arr); // 1, 2, 3
    instance2.getName(); // 'in2'
    instance2.getAge(); //20

在本例中，SubType既使用了原型链的继承方式继承了SuperType的getName方法，并且可以用函数复用的方式来定义getName方法又通过借用构造函数的继承方式继承name和arr属性，让SubType的两个不同实例分别拥有自己的属性。

组合继承避免了原型链和借用构造函数的缺陷，因而在实际应用中是JS中最常见的继承模式。


### 原型式继承

2006年，Douglas Crockford提出原型式继承的方式，即**借助原型可以基于已有的对象创造新对象，同时还不必自己因此创建自定义类型**，他给出了如下代码:

    function object (o) {
        function F () { }
        F.prototype = o;
        return new F()
    }

在object()函数内部定义了一个临时的构造函数F，将传入的实例对象o作为F的原型，然后返回F的一个新实例。

在ES5中，这个方法被规范化，并且用Object.create()来实现，即我们可以通过Object.create()来实现与构造函数实现原型链一样的效果：

    var whiteCar = {
        color: 'white',
        users: ['Tom', 'Jim']
    }

    var blackCar = Object.create(whiteCar)
    blackCar.color = 'black'
    blackCar.users.push('Black')

    console.log(whiteCar.users) // 'Tom', 'Jim', 'Black'

如上例所示原型式继承和原型链一样存在因包含引用类型引起的问题，因为它的实现方式本质和之前提到的原型链一样，只不过它没有采用构造函数来实现，而是直接操作对象来实现继承。

### 寄生式继承

寄生式继承是基于原型式继承思想的一种实现方法，即**创建一个仅用于封装继承过程的函数，该函数在内部以某种方式来增强对象，最后再像真的是它做了所有的工作一样返回对象**，如下所示：

    function createCar (original) {
        var clone = Object.create(original);
        clone.getColor = function () {
            console.log(this.color)
        }
        return clone;
    }

    var whiteCar = {
        color: 'white',
        users: ['Tim', 'Tom']
    }

    var blackCar = createAnother(whiteCar);
    blackCar.color = 'black';
    blackCar.getColor() // 'black'

寄生式继承相对于原型式继承只是将增强新对象（添加属性和方法）在一个函数中实现，其中，不一定要使用Object.create()来创造新对象，任何可以返回新对象的方法都可以。
### 寄生组合式继承

用寄生式继承将组合式继承改进，即**使用寄生式继承来继承超类型的原型，然后再将结果指定的子类型的原型**，就形成了寄生组合式继承的方法如下所示：

    // 寄生式组合式继承核心
    function inheritPrototype (subType, superType) {
        var prototype = Object.create(superType.prototype);
        // 为创建的副本重写constructor属性，弥补因重写原型而失去默认的construct属性
        prototype.constructor = subType;
        SubType.prototype = prototype;
    }

    function SuperType (name) {
        this.name = name;
        this.arr = [1, 2, 3]
    }

    SuperType.prototype.getName = function () {
        console.log(this.name)
    }

    function SubType (name, age) {
        SuperType.call(this, name)
        this.age = age
    }

    inheritPrototype(SubType, SuperType);

    SubType.prototype.getAge = function () {
        console.log(this.age);
    }

其中inheritPrototype函数是整个寄生组合式继承过程的核心，即通过类似寄生式继承的方式实现Subtype对SuperType的原型继承，然后在子类型实例化的时候，又因为内部构造函数实现借用构造函数方式来继承了SuperType的属性。

这个例子的高效率体现在它只调用了一次 SuperType构造函数，而在组合式继承的例子里面调用了两次(SuperType.call(), new SuperType)，并且因此避免了在SubType.prototype上面创建不必要的，多余的属性。与此同时还保证了原型链的不变，从而能正常使用 instanceof 和 isPrototypeOf方法。

 故寄生组合式继承是目前最理想的继承方法。

***

参考资料：

+ 《JavaScript 高级程序设计(第三版)》(Nicholas C.Zakas著，李松峰 曹力译)



