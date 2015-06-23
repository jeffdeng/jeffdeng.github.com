---
layout: post
title:  "mongodb3.0.2 在mac和ubuntu上开启客户端权限过程"
date:   2015-06-23 12:06:59
categories: mongodb
excerpt: "mongodb3.0.2 在mac和ubuntu上开启客户端权限过程"
---

mongo从2.4以后的版本很多命令发生了变化，网上搜到的教程已经有点落后了，经过看官方文档一番折腾终于配置成功。

### ubuntu下用apt自动安装
安装完成后系统会生成一个用`service mongodb start`启动的脚本，普通程序用这个没什么问题，但是遇到mongo就比较坑了稍后讲。

###  mac下安装
`brew install mongodb`
不过现在提供的最新版是2.6.9的，可以手动去
`/usr/local/homebrew/Library/Formula/mongodb.rb`
把url地址换为
`url "https://fastdl.mongodb.org/src/mongodb-src-r3.0.3.tar.gz"`
，然后安装。

### ubuntu上mongodb默认的问题

* 在ubuntu上service mongodb start默认启动是mongod这样的命令启动的，没有加任何参数，然后mongo就用自动的默认参数去运行了 。。了。。。。

*  那`/etc/mongod.conf`文件呢，我还配置了？抱歉，你不用`mongod --config /etc/mongod.conf`，mongo是不会去扫描的，你配置任何参数都不回起效。

*  而且默认的数据库路径是在`/data/mondgodb/`下面，和默认提供的`/etc/mongod`.conf里面提供的dbpath路径不在一个地方。

这样就会导致一个非常让人困惑的低级问题，你用`service mongodb start `启动服务，配置了`user password`，然后用`mongod --config /etc/mongod.conf -auth`，去验证权限始终不成功，因为数据库不在一个地方。


##创建一个权限测试用户，命令也变了

    use amdin db.createUser( {user: "admin",pwd: "admin",roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] })

现在可以用这个用户去测试了
mac下的配置文件格式用了YML格式，在`/usr/local/etc/mongod.conf`最后加2行

    security: authorization: enabled

启动的时候指定配置文件就可以开启权限了。

最后把IP限制打开，内网可以指定IP，远程把`bind_ip`改为`0.0.0.0`



