---
layout: post
title:  "PPPoE服务器添加Radius支持"
date:   2015-05-19 8:46:00
categories: Linux
tags: [Linux,PPPoE,Radius]
img_url: "http://7xu027.com1.z0.glb.clouddn.com/PPPoE-Radius.jpg"
---

### 前言

本片文章紧接之前的PPPoE服务器配置和Radius服务器配置，如果还没进行过这两步的同志请移步这里：

+ [ubuntu上搭建PPPoE认证服务器](/linux/2015/04/18/build-PPPoE-On-Ubuntu.html)
+ [ubuntu上安装和配置FreeRadius](/linux/2015/04/29/build-FreeRadius-On-Ubuntu.html)

<!-- more -->

### PPPoE服务器配置

我们只需要在之前的PPPoE服务器上修改和添加一些东西即可。

首先，确认 /etc/ppp/   路径下是否有radius文件夹，若没有，则安装radiusclient1，即：

    sudo apt-get install radiusclient1

然后在 /etc/ppp/路径下新建一个radius文件夹，将/etc/radiusclient/ 文件夹下的所有文件拷贝到 /etc/ppp/radius/ 目录下(若已有radius文件夹，请忽略此步骤)

之后，修改 /etc/ppp/radius/radiusclient.conf 文件，修改以下两项：

    authserver     10.37.129.5:1812
    acctserver     10.37.129.5:1813

其中  10.37.129.5  为笔者配置的Freeradius服务器ip地址，读者需根据自己的Freeradius服务器地址来配置该项。

修改 /etc/ppp/pppoe-server-options 文件：

    auth
    require-chap
    default-mru
    default-asyncmap
    lcp-echo-interval 60
    lcp-echo-failure 5
    ms-dns 192.168.0.1
    ms-dns 10.132.129.1
    noipdefault
    noipx
    nodefaultroute
    noproxyarp
    noktune
    netmask 255.255.255.0
    logfile /var/log/pppd.log
    plugin radius.so  #配合radius 使用
    plugin radattr.so
    radius-config-file /etc/ppp/radius/radiusclient.conf

注意

+ ms-dns为 DNS 服务器ip地址，请读者根据自己实际情况配置；
+ radius.so 和 radattr.so为radius支持文件，请读者仔细检查 /etc/ppp/ 路径下是否有这两个文件，若没有就去百度或者谷歌下载吧＝。＝；


接着修改/etc/ppp/radius/server 文件，添加freeradius服务器ip地址和共享密钥(需和freeradius服务器配置一致)：

    10.37.129.5  testing123

其中 10.37.129.5 为笔者的Freeradius服务器地址，请读者根据自己的实际情况配置；testing123为readius客户端与freeradius服务器的共享密钥，需要与之后的freeradius服务器配置一致。

### Freeradius服务器配置

接着之前Freeradius服务器的配置。

修改 /etc/freeradius/client.conf 文件，添加PPPoE服务器的信息：

    client 10.37.129.4 {
    ipaddr = 10.37.129.4
    secret = testing123
    require_message_authenticator = no
    }
其中10.37.129.4为PPPoE服务器的ip地址，请读者根据自己的实际情况配置。

### 启动服务器

启动freeradius服务器

    freeradius -X

启动PPPoE服务器

    sudo pppoe-server -I eth1 -L 11.11.11.1 -R 11.11.11.10 -N 20

***

参考博客

+ [http://blog.chinaunix.net/uid-21651676-id-3030200.html](http://blog.chinaunix.net/uid-21651676-id-3030200.html)
+ [http://blog.atime.me/note/freeradius_daloradius_install_config_on_ubuntu.html](http://blog.atime.me/note/freeradius_daloradius_install_config_on_ubuntu.html)