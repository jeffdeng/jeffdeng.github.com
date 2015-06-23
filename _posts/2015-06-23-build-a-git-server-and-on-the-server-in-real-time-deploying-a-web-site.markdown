---
layout: post
title:  "搭建Git服务器并且在服务器实时部署网站"
date:   2015-06-23 12:06:59
categories: git
excerpt: "搭建Git服务器并且在服务器实时部署网站"
---

* content
{:toc}

最近想要部署一个网站到阿里云上，同步本地git提交到服务器，服务器实时更新。首先搭建服务器，git服务器和客服端就叫git，安装一个就OK了。

####第一步，服务器上安装git：
yum install git

####第二步，服务器上创建一个git用户，用来运行git服务：
    $ sudo adduser git

####第三步，创建证书登录：

在本机上
    $ ssh-keygen -t rsa -C "随便写@suibian.com"

在用户主目录里找到.ssh目录，里面有`id_rsa`和`id_rsa.pub`两个文件，`id_rsa`是私钥，不能泄露出去，`id_rsa.pub`是公钥
拷贝`id_rsa.pub`到服务器， 然后把`id_rsa.pub`的内容写到服务器的认证文件
服务器上:
    
    $ id_rsa.pub >> /home/git/.ssh/authorized_keys

####第四步，服务器上初始化Git仓库

注意这里不要初始化裸仓库，否者你还得从服务器checkkout出来到web服务器的指定的根目录下：
    
    $ cd html $ sudo git init

把owner改为git：

    $ sudo chown -R git:git html

####第五步，服务器上禁用shell登录：
编辑`/etc/passwd`，找到最后一行
    
    git:x:1001:1001:,,,:/home/git:/bin/bash

改为

    git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell

好了，可以在本机上克隆版本库了

    $ git clone git@server:/var/html

到这一步后提交会出现问题，因为我们创建的不是裸仓库，相对于提交到另外一个人得工作版本上，所以需要设置一下

到服务器的html目录下`.git/config`，在其中添加如下内容：
    
    [receive] denyCurrentBranch = false

OK，这样就可以提交到服务器了，但是有个问题，提交了之后服务器没有更新当前版本库。需要设置一个钩子shell

    $ cd html/.git//hooks/ $ vim post-receive

内容为

    #!/bin/sh cd /var/html || exit unset GIT_DIR git reset —hard

这里一定要`unset GIT_DIR`，否者不起作用，这样服务器端就自动更新了，刷新你的浏览器就能看的刚刚更新的网站了。


