---
layout: post
title:  "ubuntu上搭建PPPoE认证服务器"
date:   2015-04-18 13:13:37
categories: Linux
tags: [Linux, PPPoE]
---


### 前言

 对于PPPoE很多人应该不陌生，拨号上网都碰过这玩意儿，但是在后台到底是怎么实现这个认证的，了解的人应该不多。
正好这学期综合课程设计老师要求搞这玩意儿，那就搞搞咯，时间上的话大概一天就搞定了，但是之后去理解这些配置做了什么东西还是花了很久时间。
<!-- more -->


### 配置

+ 系统：ubuntu14.04
+ PPPoE软件：ppp
+ 虚拟机网络模式：PPPoE服务器(双网卡：host-only模式 + 桥接模式), radius服务器(host-only模式)

### 搭建PPPoE服务器

首先直接通过apt-get安装ppp软件：

    sudo apt-get install ppp

然后修改ppp的配置文件options，该文件在/etc/ppp文件夹下

    cd /etc/ppp
    sudo gedit options

在这里我们需要修改几个地方：

+ 设置ms-dns： 这个参数是设置给客户端的dns服务器地址，和 PPPoE 服务器的dns服务器地址对应，可用查看自己服务器网络的dns地址并复制上去；
+ 注释掉+pap，取消掉-pap的注释，表示 PPPoE 服务器不使用pap认证方式；
+ 取消掉+chap的注释，表示PPPoE服务器使用chap认证方式；


然后我们设置 chap-secrets 文件，直接在最后一行按 用户名服务器名密码ip地址 的格式插入，例如：

    test * 123456 *

表示用户名为test ，服务器为任意，密码为123456，ip地址任意

接下来配置 pppoe-server-options 文件，如果/etc/ppp目录下没有这个文件就新建一个，在文件中其他的删除或者注释掉，写入以下内容：

     auth
     require-chap
     logfile /var/log/pppd.log

到这里，PPPoE的服务器的基本配置算完成了，但是为了让客户端连上外网，我们还需要进一步的配置。

开启ip转发功能，在超级用户的身份下(即先使用 sudo su 切换到超级用户)，再执行以下命令：

     cat 1 > /proc/sys/net/ipv4/ip_forward

这个命令只能暂时将ip转发功能打开，重启之后便会自动关闭，若需要一直打开ip转发功能，需要编辑 /etc/sysct1.conf , 将 net.ipv4.ip_forward=1 的注释取消掉，然后再执行 sudo sysct1 -p便可永久开启ip转发功能。

配置iptables，使用超级用户执行以下命令：

     iptables -A POSTROUTING -t nat -s 11.11.11.0/24 -j MASQUERADE

11.11.11.0/24 是我们之后给客户端分配的ip地址池，可以根据自己的喜好替换，只需要与之后开启pppoe-server的 地址相对应即可。

关于iptables，是一个内核防火墙模块，可以实现数据包的过滤和转发，具体请参见iptables的man手册。

最后开启 PPPoE 服务器：

      sudo pppoe-server -I eth1 -L 11.11.11.1 -R 11.11.11.10 -N 20

注意

+ -I 后的参数用于指定监听哪个网络端口。可以使用ifconfig命令查看当前工作的端口名称。笔者的虚拟机的host-only的网卡为eth1，所以使用这个网卡。
+ -L 后的参数用于指定在一个PPP连接中PPPoE服务器的IP地址，之前我们在配置网关的时候确定的ip地址池为 11.11.11.0/24，所以这里我们要与之对应，变使用11.11.11.1作为PPPoE服务器地址。
+ -R 后的参数用于指定当有客户连接到服务器上时，从哪个IP地址开始分配给客户。
+ -N 后的参数用于指定至多可以有多少个客户同时连接到PPPoE服务器上。


至此，PPPoE服务器搭建完毕，可以直接在另一个虚拟机的windows客户端上进行拨号连接，账号名为test，密码为123456，如果PPPoE服务器的桥接网卡能用来上网的话，那么windows客户端也能上网。

可能的问题

+ 如果windows客户端连接的时候无法检测到PPPoE服务器的存在，即无法通过wan口发现PPPoE服务器，检查一下自己的PPPoE服务器是否启动成功。
+ 如果windows客户端连接上了PPPoE服务器但是无法连上外网，先检查自己的PPPoE服务器是否能连上外网，若没有问题，再检查自己PPPoE的ip转发配置、options中的ms-dns是否配置正确 或者 iptables中的配置是否和pppoe-server分配给客户端的ip地址池一致

***

参考博客

+ [http://blog.chinaunix.net/uid-9525959-id-4008338.htm](http://blog.chinaunix.net/uid-9525959-id-4008338.html)