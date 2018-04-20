---
layout: post
title:  "node.js的安装与npm的使用"
date:   2016-04-19 15:35:30
categories: Nodejs
tags: [nodejs,npm]
img_url: "http://7xu027.com1.z0.glb.clouddn.com/node-js-wordpress.png"
---

### 什么是npm
npm, Node Package Manager, 是随同node.js一起安装的包管理工具。该管理工具可用于从第三方网站上下载node.js包，常用于：

+ 允许开发者从npm服务器下载别人编写的第三方包到本地使用。
+ 允许开发者从npm服务器下载并安装别人编写的命令行程序到本地使用。
+ 允许开发者将自己编写的包或命令行程序上传到npm服务器供别人使用。

从而方便开发者的项目构建和解决node.js项目部署上的很多问题。
<!-- more -->

### node.js的安装

node.js历史包版本下载地址 [https://nodejs.org/dist/](https://nodejs.org/dist/)

#### windows

1. 下载安装包安装:

在历史包版本里面找到最新版本的.msi文件，下载并执行安装；
<br/>在cmd里面输入path查看是否有npm的环境变量()，如果没有则需要将手动配置node.js和npm的环境变量，即将其安装路径添加到环境变量path之中就行（一般来说安装包里有选项会默认帮用户配置的）

2. 下载二进制文件安装:

在历史包版本里找到最新版本的.exe文件，下载并运行即可;

#### linux ubuntu

1. 从github上获取源码安装:

在 Github 上获取 Node.js 源码：

    $ sudo git clone https://github.com/nodejs/node.git

修改目录权限：

    $ sudo chmod 755 -R node

使用 ./configure 创建编译文件，并按照：

    $ cd node
    $ sudo ./configure
    $ sudo make
    $ sudo make install

查看 node 版本：

    $ node --version

2. 使用apt-get进行安装：

直接运行以下命令即可：

    sudo apt-get install nodejs
    sudo apt-get install npm


#### Mac Os

使用homebrew进行安装:

    brew install node

### npm常见命令与使用

#### 安装node.js的包

1. **npm install &lt;name&gt;**  安装nodejs的依赖包
2. **npm install &lt;name&gt; -g**  将包安装到全局环境中
<br/>但是代码中，直接通过require()的方式是没有办法调用全局安装的包的。全局的安装是供命令行使用的
3. **npm install &lt;name&gt; --save**  安装的同时，将信息写入package.json中
<br/>项目路径中如果有package.json文件时，直接使用npm install方法就可以根据dependencies配置安装所有的依赖包
4. **npm init**  引导你创建一个package.json文件，包括名称、版本、作者这些信息等
5. **npm remove &lt;name&gt;** 移除包
6. **npm update &lt;name&gt;** 更新包
7. **npm ls** 列出当前安装的了所有包
8. **npm root** 查看当前包的安装路径
9 **npm root -g**  查看全局的包的安装路径
10. **npm help**  帮助，如果要单独查看install命令的帮助，可以使用的**npm help install**
11. **npm config set registry &lt;源地址&gt;** npm换源命令，&lt;源地址&gt;的格式须以'http://'或'https://'开头
12. **npm view &lt;name&gt; versions** 查看npm包的历史版本
