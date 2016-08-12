---
layout: post
title:  "js实现图片异步上传"
date:   2016-04-22 13:23:00
categories: javascript jquery
tags: [javascript,图片上传,异步]
description: "最近在做express练习项目的时候需要解决一个图片异步上传的问题，简单分享下几种的实现方式"
img_url: "http://7xu027.com1.z0.glb.clouddn.com/1085__shining-together_p.jpg"
---

### 前言
最近在学习node.js的express框架，其中包含了一个用户系统，之前在做用户系统的时候对用户信息的编辑更新都是用的
同步的方式来进行表单提交，这一次兴趣使然想用异步的方式来实现信息编辑的表单提交，用jquery框架的ajax方法的对普通
的文本信息都能很好的实现，但是在图片的异步请求提交的时候遇到了一些问题。因为jquery框架的ajax方法本身对二进制数据
的表单提交没有很好的支持，网上的很多推荐实现方式都说用额外的插件来实现，但是由于‘’不可描述“的原因，当时下载这些插件
来使用有些不方便，所以便一直探索不用其他插件的情况下，用jquery的ajax或者原生的js来实现.
<!-- more -->

### 使用jquery的ajax方法实现

#### 前端表单

    <form id="avatar_form" enctype="multipart/form-data">
        <input type="file" name="avatar">
    </form>

#### js代码

    function uploadAvatar () {
        var data = new FormData();
        var files = $("input[name='avatar']")[0].files;
        if (files) {
            data.append("file", files[0])
        } else {
            alert("please choose a picture")
        }

        $.ajax({
            type: 'put',
            dataType: 'json',
            url: "/user/change_avatar"
            data: data,
            contentType: false,
            processData: false,
            success: function (data) {
                alert('success');
            }
        })
    }

使用jquery的ajax方法实现的主要思想是**我们重新构建一个表单，用jquery读取html中表单的图片信息，然后向新表单之中加入文件，
并且不让ajax来自动处理我们的表单数据，直接将原始表单数据提交**

相信大家也注意到了，在ajax的设置参数里，我们多了几个参数的设置：

+ contentType
+ processDate

根据jquery ajax的官方文档[http://api.jquery.com/jquery.ajax/](http://api.jquery.com/jquery.ajax/)，这几个参数的设置主要是一下用途：

1. contentType (值类型:Boolean or String, 默认值:'application/x-www-form-urlencoded; charset=UTF-8')
<br/>这个值用来设置HTTP请求的content-type头信息，默认是'application/x-www-form-urlencoded; charset=UTF-8‘。在jQuery1.6版本之后，这个值可以设置
为false，用来告诉jQuery不要设置请求的content-type头信息。由于我们这里需要使用原始的表单，不需要jQuery帮我们设置content-type信息，
FormData对象会根据表单的值来设置，故而将这个值设置为false。

2. processData (值类型:Boolean, 默认值: true)
<br/>这个值的设定是整个过程最重要的一部分，因为当这个值为true的时候，ajax会将data里的值处理成contentType里我们设置的类型，默认是"application/x-www-form-urlencoded".
当这个值为false的时候，ajax就不会对data里的值进行处理，这正是我们所需要的


其实整个过程最重要的部分还是FormData.
<br/>FormData是XMLHttpRequest Level 2添加了一个新的接口.利用FormData对象,我们可以通过JavaScript用一些键值对来模拟一系列表单控件,
<br/>构造函数

    new FormData(optional HTMLFormElement form)

form 参数是一个可选的HTML表单元素,可以包含任何形式的表单控件,包括文件输入框。

FormData对象还可以使用append方法添加键值对，如

    var form = new FormData();

    form.append("name", "testName");
    form.append("num", 123456); // 数字123456被立即转换成字符串"123456"

    // fileInputElement中已经包含了用户所选择的文件
    form.append("file", fileInputElement.files[0]);

使用FormData最主要的优势便是我们可以用它构造表单来传输二进制文件，故而是异步上传图片的较好选择。
在浏览器的支持方面

Chrome | FireFox(Gecko) | IE | Opera | Safari
--------|------------------|----|--------|--------
7+ | 4.0(2.0)  | 10+ | 12+ | 5+

#### XMLRequest实现
当然除了ajax异步的方式，也可以是用XMLRequest的方式实现异步请求，如果是要传输二进制文件的话，
一样是可以用到FormData对象来构造表单，然后用XMLRequest异步请求的方式将FormData对象当作表单提交即可。

    var formEle = document.getElementById("avatar_form");
    var form = new FormData(formEle);

    var xReq = new XMLHttpRequest();
    xReq.open("PUT", "/user/change_avatar", true);
    xReq.onload = function(oEvent) {
        if (oReq.status == 200) {
            alert("success");
        } else {
             alert("failed")
        }
    };

    xReq.send(form);

