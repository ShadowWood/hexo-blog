---
layout: post
title:  "ubuntu上安装和配置FreeRadius"
date:   2015-04-29 15:09:00
categories: Linux
tags: [Linux,FreeRadius]
img_url: "http://7xu027.com1.z0.glb.clouddn.com/build-Radius.jpg"
---

### 前言

前面笔者分享了在ubuntu上搭建PPPoE服务器的步骤和一些问题及解决办法，但是这学期的综合课程设计内容是要求搭建和配置由radius协议实现认证和计费的PPPoE服务器，之前的东西只是第一步，本文接着分享如何在ubuntu上安装配置FreeRadius来实现基于radius协议的认证和计费。
关于radius的介绍可以看一下这里：

+ [http://blog.csdn.net/liang13664759/article/details/1574367/](http://blog.csdn.net/liang13664759/article/details/1574367/)

<!-- more -->

### 系统配置及需要安装的软件

+ 系统：ubuntu 14.04
+ 安装软件：
    + freeradius-mysql(自动安装freeradius本体)
    + mysql-server

安装命令：

    sudo apt-get install freeradius-mysql mysql-server

### 进行FreeRadius基本配置

修改 /etc/freeradius/clients.conf，这是FreeRadius的客户端配置文件，在这里我们要将我们之后要与服务器连接的客户端的信息写入配置文件，例如加入如下几行：

    client 192.168.0.2 {
      ipaddr = 192.168.0.2
      secret = testing123
      require_message_authenticator = 0
    }

表示有一个 ip地址为192.168.0.2的客户端之后要接入Freeradius服务器，它与服务器的共享密钥为 testing123。
该配置文件一般已经配置好了一个本地测试的客户端，即ip地址为 localhost 的客户端。

修改 /etc/freeradius/radiusd.conf，这是FreeRadius的配置文件，取消下一行的注释来包含FreeRadius的 sql 配置文件：

    $INCLUDE  sql.conf

修改/etc/freeradius/sql.conf，这是FreeRadius与 mysql 数据库相关的配置文件，注意一下几行：

    server = "localhost"
    login = "root"
    password = "123456"

这里为mysql数据库的ip地址，用户名和密码，默认使用3306端口，当然也可以自己加一行 port="端口号" 来根据自己的喜好配置。(这几行应该配置文件里有，不需要修改，如果没有就自己加上)

接着取消掉文件中下面一行，来让FreeRadius从数据库里面读取客户端的信息：

    readclients = yes

接着切换到 /etc/freeradius/sql/mysql，该文件下的众多sql脚本文件用于构建FreeRadius的数据库，首先在admin.sql里修改数据库名称，用户名和密码等内容(默认配置是radius, root, 123456，若之前数据不是默认配置，需对sql语句进行相关修改)，这些内容必须和/etc/freeradius/sql.conf的设置相同。

最后修改/etc/freeradius/sql/mysql/dialup.conf文件，取消掉以下几行的注释来提供在线人数统计功能：

    simul_verify_query  = "SELECT radacctid, acctsessionid, username, \
                            nasipaddress, nasportid, framedipaddress, \
                            callingstationid, framedprotocol \
                            FROM ${acct_table1} \
                            WHERE username = '%{SQL-User-Name}' \
                            AND acctstoptime IS NULL"


### 配置mysql数据库

先切换到/etc/freeradius/sql/mysql文件夹下(以便进行之后的操作)，然后登录mysql数据库：

    cd /etc/freeradius/sql/mysql
    mysql -u root -p

先创建数据库radius，然后导入文件下的sql文件：

    create database radius;

    source admin.sql;
    source cui.sql;
    source ippool.sql;
    source schema.sql;
    source wimas.sql;

再在mysql运行下面几句sql语句加入用户组：

    insert into radgroupreply (groupname,attribute,op,values) values ('user'，'Auth-Type',':=','Local');
    insert into radgroupreply (groupname,attribute,op,values) values ('user'，'Service-Type',':=','Framed-User');
    insert into radgroupreply (groupname,attribute,op,values) values ('user'，'Framed-IP-Address',':=','255.255.255.254');
    insert into radgroupreply (groupname,attribute,op,values) values ('user'，'Framed-IP-Netmask',':=','255.255.255.0');

然后加入用户账号和密码（以账号test，密码123456为例）：

    insert into radcheck (username,attribute,op,value) values ('test','User-Password',':=','123456')
      将账号加入用户组：

    insert into usergroup (username,groupname) values ('test','user');
      修改/etc/freeradius/sites-enabled/default：

找到authorize {}模块，注释掉files，去掉sql前的#号

找到accounting {}模块，注释掉radutmp,注释掉去掉sql前面的#号。

找到session {}模块，注释掉radutmp，去掉sql前面的#号。

找到post-auth {}模块，去掉sql前的#号，去掉sql前的#号（Post-Auth-Type REJECT内）

修改/etc/freeradius/sites-enabled/inner-tunnel：

找到authorize {}模块，注释掉files，去掉sql前的#号。

找到session {}模块，注释掉radutmp，去掉sql前面的#号。

找到post-auth {}模块，去掉sql前的#号，去掉sql前的#号（Post-Auth-Type REJECT内）。


到这里就基本配置完毕了。

***

### 本地测试

使用调试模式启动 FreeRadius：

    freeradius -X

使用radtest模拟向服务器发送请求：

    radtest test 123456 localhost 1812 testing123

注意

**test**为用户账号，**123456**为密码，这两个必须和我们之前在mysql数据里面写入的用户账号和密码对应，否则服务器会返回Access-reject响应，**localhost**和**1812**为FreeRadius服务器的 ip地址和端口，
**testing123**是我们之前在**/etc/freeradius/clients.conf**里面配置的与本地客户端的共享密钥。