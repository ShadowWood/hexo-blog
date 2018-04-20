---
layout: post
title:  "javascript实现按钮计时"
date:   2015-04-18 18:15:30
categories: Javascript
tags: [javascript]
img_url: "http://7xu027.com1.z0.glb.clouddn.com/button-timer.jpg"
---

最近实验和项目忙得不可开交，终于可以闲下心来写写博客了。

这段时间都在做后台管理的web端，自然也离不开短信验证那一套东西，最开始对js不熟，做按钮计时花了大半时间。
今天就拿这个下手吧。为了方便操作html中的元素，就直接用jquery框架了。
<!-- more -->

首先先确定需要输入的东西，一个是手机号，一个是验证码。

    <script src="http://cdn.bootcss.com/jquery/1.11.1/jquery.min.js"></script>
    <label>手机号：</label>
    <input id="phone">
    <br/>
    <label>验证码：</label>
    <input>
    <button id="get_verify_code">获取验证码</button>

 首先我们得实现在用户点击意义“获取验证码”按钮后，得有一个计时机制，防止用户在短时间内多次获取验证码，而且同时也有暗示用户获取验证码这步操作已经执行。计时的功能用setInterval()函数来实现。先贴出代码：

    <script type="text/javascript">
        var count;
        var countdown;
        var button = $('#get_verify_code')
        function Count(){
            count--;
            button.text("请等待"+count+"秒");
            if(count == 0){
                clearInterval(countdown);
                button.text("获取验证码");
                button.removeAttr('disabled');
            }
        }
        button.click(function(){
            count = 30;
            $(this).text("请等待"+count+"秒");
            $(this).attr('disabled', true);
            countdown = setInterval(Count, 1000);
        })
    </script>


OK，这段代码的两个重点就是setIntercval()函数和全局变量的使用。首先我们来了解下setInterval()函数。

函数定义：

+ setInterval() 方法可按照指定的周期（以毫秒计）来调用函数或计算表达式。
+ setInterval() 方法会不停地调用函数，直到 clearInterval() 被调用或窗口被关闭。由 setInterval() 返回的 ID 值可用作 clearInterval() 方法的参数。

函数语法：

+ setInterval(code,millisec[,"lang"])

参数：

+ code必须。要调用的函数或要执行的代码串。
+ millisec必须。周期性执行或调用 code 之间的时间间隔，以毫秒计。

返回值:

+ 一个可以传递给 Window.clearInterval( ) 从而取消对 code 的周期性执行的值。

我们将计时函数 Count( )  用 setInterval( ) 函数来不断执行，知道计时标志 count <= 0 时，停止计时，将按钮恢复如初。在代码中我用了setInterval(Count, 1000), 其中 Count 时 Count( )函数的函数名，“1000”是指定的周期，即1000毫秒＝1秒。

至于为什么我要将 count 和 countdown 定义为全局变量，以下便是我的理由：

+ count 作为计时标志，每次都会被函数 Count( ) 调用操作，故应将其定义为全局变量避免出现错误。

+ countdown 是 setInterval( ) 返回的 id 值，用来作为 clearInterval( ) 的参数停止计时，而在这里clearInterval( ) 被用在 Count( ) 之中，而 Count( ) 又是在 setInterval( ) 里被调用，为了使countdown有效，其必须定义在 setInterval( ) 之外，故将其定义为全局变量。

这样一看其实还挺明显的，但是之后等我们加入下一个功能，你就会明白全局变量的重要性了。

***

相信大家已经发现这个按钮计时有个明显的缺陷了，那就是只要你将当前的页面一刷新，按钮又可以使用了，那么做的这一切就没用了。

要解决这个问题有两种方案：

+ 在后台存储用户点击按钮时的时间，每次用户向后台发出页面请求的时候，判定当前的时间距离上一次点击按钮的时间相差是否有30秒，然后进行对应的操作；
+ 在本地存储用户点击按钮时的时间，用法同上。

这里我们为了减少向服务器请求的数据（我不会告诉你其实是我想偷懒！＝。＝），便采用第二种方案。

本地存储的方式有两种，一是使用 localStorage, 二是使用 sessionStorage。前者是永久存储，后者如果关闭浏览器就清空。这里我们用 localStorage。 ok，想法就这样，先贴出代码：

    <script type="text/javascript">
        $(document).ready(function(){
            var count;
            var countdown;
            var button = $('#get_verify_code');
            function Count(){
                count--;
                button.text("请等待"+count+"秒");
                if(count == 0){
                    clearInterval(countdown);
                    button.text("获取验证码");
                    button.removeAttr('disabled');
                    localStorage.removeItem('verify_time')
                }
            }
            if(localStorage['verify_time']){
                var last_time = parseInt(localStorage['verify_time']);
                var now_time = parseInt(new Date().getTime());
                if(now_time-last_time < 30000){
                    count = parseInt(30-(now_time - last_time)/1000);
                    button.text("请等待"+count+"秒");
                    button.attr('disabled', true);
                    countdown = setInterval(Count, 1000);
                }
                else{
                    localStorage.removeItem('verify_time');
                }
            }
            button.click(function(){
                count = 30;
                $(this).text("请等待"+count+"秒");
                $(this).attr('disabled', true);
                localStorage['verify_time'] = new Date().getTime();
                countdown = setInterval(Count, 1000);
            })
        });
    </script>

首先用  $(ducument).ready(function( ){ })  来实现页面的初始化操作，保证需要的代码操作执行完后，才能对页面进行下一步操作。

这里我们将点击“获取验证码按钮”的时间用时间戳存在localStorage['verify_time'] 里，每次计时结束 或者 下一次刷新页面的时候已经距离上次操作30秒后，就将其删除。

每次刷新页面的时候，先判定本地是否记录有上次获取验证码操作的时间，若有，则计算距离上次操作时间是否超过30秒，如果低于30秒，则继续计时；否则，删除本地的记录。

由于 localStorage 都是存的字符串类型，所以在计算时间差和对 count 进行赋值的时候，都要将其转换为int类型。

在这里，我们就能发现重视全局变量带来的好处了，count 和 countdown 虽然都会被函数以及函数的内层调用，但是这些函数都不是在同时对这两个变量进行操作，所以理论上不会引起冲突，故而将 count 和 countdown 定义为全局变量，不仅使得逻辑清楚，还使得代码简洁，减少了很多不必要的操作。

在这之前，我犯过许多错，其中就包括对全局变量的使用不当，也是由于python用惯了，没重视这方面问题，老是喜欢使用局部变量，使得代码逻辑混乱，半天找不出问题。

最后爽性全删掉，重新整理思路，删繁就简，最后才发现，其实完全可以用全局变量，不仅清晰易懂，还减少了很多不必要的操作。
