---
layout: post
title:  "python网络爬虫入门"
date:   2015-02-10 8:13:30
categories: Python
tags: [python,网络爬虫]
description: "本文将介绍python基本的网络请求方式，以及网页抓包分析，适合于刚开始接触python的人学习"
img_url: "http://7xu027.com1.z0.glb.clouddn.com/web-spider.jpg"
---

### 前言

对于刚刚接触python的人并且没有什么编程基础的人来说，从网络爬虫开始入门是个不错的选择。学习一门编程语言要从实践中去学习，必须要自己动手操作，光是看一系列的编程书籍，不动动手做几个实际的程序运行运行，是完全不可能掌握一门语言的。就好比武侠小说里学习一门武功一样，光是看别人操练，自己不比划比划，难道就能练就一身好功夫么？这样的话以后出身江湖不被别人砍死都是万幸了。

网络爬虫是我学习python(甚至可以说学习编程)的第一个程序(“Hello World”什么的就算了把)。所以在此极力推荐初学python的人选择从网络爬虫作为入门学习的选择。

### 网络爬虫简介

简单来说网络爬虫就是按照一定的算法和规则，自动的抓取网络信息的程序或者脚本。比较常用的也是这里要给大家介绍的基于HTTP协议的抓取网页并从中提取特定信息的脚本。

我们平常用浏览器打开一个网页，都是想网页的服务器端发送一个请求，然后服务器端做一系列的判断和处理，返回给我们一个HTML文件，然后我们的浏览器再将HTML文件解析成可视化的视图供我们浏览和操作。而这里要给大家介绍的爬虫就是将获取到HTML文件中的字符进行匹配和分析，提取我们需要的数据。

### 获取网页

对于python的介绍和安装，百度或谷歌一搜一大片，这里我也不再做介绍，我们直接切入正题。

python作为一门脚本语言，自身集合了很多WEB操作的库，再结合其简洁的语法，往往只需要简单几行代码，就能完成一些复杂的请求和分析。

现在我们以抓取天涯论坛的帖子为例。

+ [http://bbs.tianya.cn/post-culture-308737-1.shtml](http://bbs.tianya.cn/post-culture-308737-1.shtml)

这是一篇天涯论坛上的小说，现在我们准备写一个简单的网络爬虫，把这个网页的帖子抓取下来。

    import urllib

    url = "http://bbs.tianya.cn/post-culture-308737-1.shtml"
    re_f = urllib.urlopen(url)
    page = re_f.read()
    print page


urllib为python系统库专门用来处理WEB请求的一个包，用这个包可以实现向服务器端发送HTTP请求。

代码运行的结果是一整串的HTML代码，也就是我们做访问网页的源代码,就这样我们像浏览器一样获取网页的请求就完成了。

### 匹配字符串获取数据

接下来我们要对获取到的网页信息做处理，只抓取我们需要的数据。这里以抓取文章标题为例。

    import urllib
    import re

    url = "http://bbs.tianya.cn/post-culture-308737-1.shtml"
    re_f = urllib.urlopen(url)
    page = re_f.read()
    content = re.findall("<title>(.*?)_.*?</title>", page)
    print content[0]

这里使用了正则表达式，关于正则表达式我之后会写一篇文章详细介绍。

re为python系统库用来处理正则表达式匹配的一个包。通过对网页源码的分析，我们不难发现，我们所需要的文章标题在标签<title>之中，但是这个标签的具体内容是这样的：

    <title>男人密码:《妻子，请原谅我》(已出版)_舞文弄墨_天涯论坛</title>

我们所需要的内容只有“男人密码:《妻子，请原谅我》(已出版)”，title标签和“_舞文弄墨_天涯论坛”我们都不需要，所以我们将 "<title>"与"_舞文弄墨_天涯论坛"之间的内容用"()"提取出来即可，其中.*?表示匹配任意字符串。

re.findall()返回的是一个列表，我们只需要将第一个元素提取出来即可。

到这里，仅仅只用几行代码，一个简单的小爬虫就完成了。

***

上一部分主要讲解了如何使用python发送HTTP  GET 请求并简单处理获取到的数据，其中用到了 urllib 和 re 库。http协议常用的请求中除了 GET 方法，还有 POST 方法。POST 方法与 GET 方法的区别网上已经有很多讲解了，这里笔者也不再赘述，简单的来说 GET 方法用于信息获取，POST 方法可能会对服务器上的资源进行修改。本篇文章以发送手机短信为例，讲解如何发送http POST请求。

### 准备工作

这里笔者选用的短信服务商为云片网，关于云片网的api文档请移步这里：

+ [http://www.yunpian.com/api/sms.html](http://www.yunpian.com/api/sms.html)

在api文档里已经有调用短信接口的python代码示例，不过示例代码中使用的是urllib库。因此笔者在这里使用urllib2进行编写，这个库也是python的标准库之一，用来处理http请求非常方便。

### 代码实现

    # -*- coding: utf-8 -*-
    import urllib2
    import urllib


    url = "http://yunpian.com/v1/sms/send.json"
    data = urllib.urlencode({'apikey': "***********************",
                             'mobile': '****************',
                             'text': "【云片网】您的验证码是1234"})
    response = urllib2.urlopen(url, data)
    print response.read()

### 代码讲解

apikey 对应云片网账号的 apikey，mobile 为需要发送短信的手机号码。

urllib.urlencode() 是将字典或包含两个元素的列表或元组转换为url参数，读者可以print一下本例的 data，结果输出为:

    “mobile=15682513909&text=%E3%80%90%E4%BA%91%E7%89%87%E7%BD%91%E3%80%91%E6%82%A8%E7%9A%84%E9%AA%8C%E8%AF%81%E7%A0%81%E6%98%AF1234&apikey=d25fbaa9ed5025eef7b777ca73c56af3”(text输出格式为unicode)。

urllib2.urlopen()创建一个表示远程url的类文件对象，然后像本地文件一样操作这个类文件对象来获取远程数据。当只传入url参数时，该函数将使用HTTP GET向url发送请求,当同时传入url和data参数时，该函数则使用HTTP POST方法向url发送请求。返回为一个类文件对象，该对象提供如下方法：

+ read(), readline(), readlines(), fileno(), close()：这些方法的使用方式与文件对象完全一样，读者可以自行了解python对于文件对象的处理；

+ info()：返回一个httplib.HTTPMessage对象，表示远程服务器返回的头信息；

+ getcode()：返回HTTP状态码；

+ geturl()：返回请求的url；

### 代码进阶

对于以上的代码urllib2也能做到，而且在使用上几乎没有区别，那么这里我们对代码进行一些修改：

    # -*- coding: utf-8 -*-
    import urllib2
    import urllib


    url = "http://yunpian.com/v1/sms/send.json"
    data = urllib.urlencode({'apikey': "***********************",
                         'mobile': '****************',
                         'text': "【云片网】您的验证码是1234"})
    req = urllib2.Request(url, data)
    response = urllib2.urlopen(req)
    print response.read()

其中我们引入了Request对象，并且直接在urllib2.urlopen()中传入Request对象，代码依旧能执行。在python官方的urllib2文档中，有这么一句话“Open the URL url, which can be either a string or a Request object.” 对于传入的url参数，可以直接是字符串，也可以是Request对象，而在urllib之中没有这种用法。

对于Request对象，我们可以通过它来修改 HTTP 请求的 header 和 proxy，这个我们在之后的文章会讲解，这里先提出这个用法，以免在以后的使用和介绍中显得突兀。

以上代码的最终结果是一个json字符串，使用type()方法返回值为str。可以使用python的json库，使用json.loads()将json字符串转换为数据原本的格式，本例的最终原本的数据为字典。

***

有了以上的基础，我们可以尝试实现一个简单的模拟登录

### 什么是模拟登录

在很多网站上都会有一套用户系统，那么肯定就免除不了会有登录操作，并且许多信息和操作只有在登录之后才能实现。目前网站上判断用户是否登录有大概两种方式：

+ 使用cookie来对应记录用户的登录状态以及有效时间

+ 自定义一个token，发送给已经登录的用户，里面存储了用户的一些信息已经登录状态有效时间

这两种方式的思想都大同小异吧，目前我们主要对使用cookie进行登录判定的网站举例讲解。

### 需要准备的工具

用户抓包的工具：chrome、firefox的firebug插件。至于抓包工具如何使用，大家可以自行搜索。

### 抓包

首先我们需要通过抓包工具获取如下三个信息：

+ post请求的网址

+ post请求的参数

+ post请求的返回值

然后我们就针对以上获取到的数据进行下一步操作

### 代码实现

以电子科技大学的信息门户登录为例，需要用到的包:urllib，urllib2，cookielib

    # -*- coding: utf-8 -*-
    import urllib
    import cookielib
    import urllib2


    loginUrl = "https://uis.uestc.edu.cn/amsever/UI/Login"
    postUrl = "https://uis.uestc.edu.cn/amserver/UI/Login"
    # 初始化一个CookieJar来处理cookie信息
    cookieJar = cookielib.CookieJar()

    # 初始化一个opener
    cJHandler = urllib2.HTTPCookieProcessor(cookieJar)
    opener = urllib2.build_opener(cJHandler, urllib2.HTTPHandler)
    urllib2.install_opener(opener)

    # 用get请求访问网站，获取必要的cookie值

    res1 = opener.open(loginUrl)

    # 用post请求向url提交表单

    postData = {"IDToken0": "",
                # 登录用户名
                "IDToken1": "这里填写你的登录用户名",
                # 登录密码
                "IDToken2": "这里填写你的登录密码",
                "IDButton": "Submit",
                "goto": "aHR0cDovL3BvcnRhbC51ZXN0Yy5lZHUuY24vbG9naW4ucG9ydGFs",
                "encoded": "true",
                "gx_charset": "UTF-8"}
    postData = urllib.urlencode(postData).encode(encoding='UTF8')
    req = urllib2.Request(postUrl, postData)
    response = opener.open(req)
    print response.read()

不出意外的话，打印的结果就是登录成功后返回的网页。

### 代码讲解

相对于之前的爬虫代码，这里多了一个cookie的管理，所以我们使用了cookielib包来生成一个CookieJar对象来自动处理返回的cookie值。

之后我们需要将获取到的cookie值写入我们的每一次http请求中，于是我们使用了urllib2.build_opener()来生成一个OpenerDirector对象，而在一个OpenerDirector里面有很多的处理类，我们称之为handler。这些handler可以帮我们完善我们的http请求，比如本例中的向http请求中加入获取到的cookie。在调用了urllib2.builde_opener()之后，我们需要再调用urllib2.install_opener()来将生成的OpenerDirector实例化，这样在之后我们就可以使用OpenerDirector的open方法来代替urllib2.urlopen()方法了(当然你之后继续使用urllib2.urlopen()也不会有问题)

在模拟登录操作成功之后，我们就可以通过get或者post请求直接访问获取我们登录后才能看到的信息了，如个人学籍信息等，不过一定要使用urllib2.urlopen()或者OpenerDirector.open()方法来发送请求，因为我们之后的操作必须使用到我们之前登录成功获取到的cookie值，所以我们必须用我们生成的OpenerDirector对象的open方法来发送请求（其实urllib2.urlopen()也是默认调用当前OpenerDirector对象的open()方法）。

### 结语

对于模拟登录的方法，实现方式不止以上的一种，不过基本思想都是一致的。归根结底就是用户先向服务器发送需要认证的信息，然后服务器返回给你一个身份标识，之后用户每次向服务器发送请求的时候，都将这个身份标识一起发给服务器，向服务器证明你是一个登录的合法用户，然后服务器返回给用户应该看到的和想看到的数据。