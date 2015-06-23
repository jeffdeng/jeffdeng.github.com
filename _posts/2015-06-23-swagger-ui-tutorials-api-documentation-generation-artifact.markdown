---
layout: post
title:  "swagger ui教程，API文档生成神器"
date:   2015-06-23 12:06:59
categories: app
excerpt: "swagger ui教程，API文档生成神器"
---

[swagger ui](https://github.com/swagger-api/swagger-ui)是一个API在线文档生成和测试的利器，目前发现最好用的。[官方在线demo](http://petstore.swagger.io/)

![demo-1024x693.jpg](/static/img/demo-1024x693.jpg)

有了这个神器，以后就免去和APP前端反复解释字段和写繁琐的文档的痛苦了，从此远离写文档的苦力活，把精力用在该用的地方，不再和安卓和IOS扯蛋蛋。

##1.安装前端

这个神器分为2部分，前端+服务器端。前端是JS+HTML组成，她负责读取后端生成的JOSN文件来工作。后端可以是任意语言的，可以是PHP JAVA JS ,在这里有介绍，因为我一直用PHP开发就值介绍PHP比较好用的部分。

swagger-ui下载


    git clone https://github.com/swagger-api/swagger-ui.git
下载好之后找到 dist目录，这里面就是我们要用到的，这是官方编译好的，自己就不用去费力搞了。我们做的就是把这个目录重命名叫doc目录，放到网站根目录下。然后把index.html里面这一段的url改为你自己json资源文件输出路径,比如

    $(function () {
     var url = window.location.search.match(/url=([^&]+)/); 
     if (url && url.length > 1) { url = decodeURIComponent(url[1]); } 
     else { //url = "http://petstore.swagger.io/v2/swagger.json"; 
     url = "http://www.1.com/swagger-php/doc/index.php"; 
    }


至于官方的说本地化  `dist/lang: The swagger localization`，官方也没提供样本，第一个JS也不知道哪里去找，将就用英文吧。至此前端部分就搞定了，非常简单，用`www.1.com/doc/index`.html就能访问了因为没有配置后端现在会报错。

##2.安装后端

    git clone https://github.com/zircote/swagger-php.git

官方说事不支持git方式安装了，因为wagger-php需要依赖3个解析库，推荐compoer安装

    "require": { "zircote/swagger-php": "*" }

不过用git也是可以的。下载好了就可以用了，把swagger-php放在根目录下，用官方提供的Examples来生成我们的测试json吧。
  
    cd swagger-php makedir doc php swagger.phar Examples -o doc

OK,现在它就是做了一件事，扫描Examples下面的model还是有controller都生成了前端需要的JSON。现在刷新下`www.1.com/doc/index.html`，就能发现出现了PAI列表了。打开火狐浏览器firebug，会看到operations命名冲突的问题，这个事Examples下面有个文件叫operation.php引起的，删了，然后重新生成json就没问题了。那么开始测试一下API，点击页面try out，没反应？对，在firebug网络面板能看到有网络请求，不过不是当前域名的，是Examples的，以后自己修改Examples的basepath指向我们自己的API就好了。

那么如何写注释（Annotations）呢，看[官方文档](http://zircote.com/swagger-php/annotations.html)就行了，没几种

    /** * * @SWG\Model(id="Pet") */

比如这个是model类的声明，它的名字叫Pet,model之间还可以相互引用


    class Pet { /** * @SWG\Property(name="photoUrls",type="array",@SWG\Items("string")) */ public $photos;

    /**
    * @SWG\Property(name="tags",type="array",@SWG\Items("Tag"))
    */
    public $tags;
    }

`@SWG\Items`只有数组才能用。

好了，文档就算完毕了，我们平常做开发的时候只需要把这个两个目录拷过去，把example里面的对象修改为自己就行了。

##3.另外一种用法

这是一种方式，和框架无关的运行方式。还有有一种是集成到现有框架中，不过现在只支持laravel，github上提供了源代码，要是谁又精力+能力可以改一下用到thinkphp中。
github地址是这个`https://github.com/slampenny/Swaggervel`，不过这个就不能用`git clone`方式去按照了，配置太麻烦，用`composer`吧
    
    composer require "jlapp/swaggervel:dev-master"

然后会按照相关依赖和生产自动加载文件

![composer-1024x460.jpg](/static/img/composer-1024x460.jpg)

下一步
`Jlapp\Swaggervel\SwaggervelServiceProvider `复制这一句到 `app/config/app.php` 的 providers数组最上面，然后
    
    php artisan vender:publish

这一步把相关文件包括swagger ui复制到laravel框架public下面。OK，已经好了，试着访问根目录下，比如 `www.1.com/api-docs`试试，出现界面就成功了！没从先就用命令看下laravel的路由

    php artisan route:list

显示如下：

![ apiroot.jpg](/static/img/apiroot.jpg)

最上面2条就是刚刚添加的路由。
刷新页面是不是发现空白？要生产json需要你写@SWG的注释，再laravel的app目录下面任何文件写好就可以了，一般我们只需要写model和controller的，这个插件会扫描这个目录生产json文件。

##最后

我已经把demo上传网盘，需要的可以下载使用，laravel的需要自己配置数据库,再laravel下面的.env文件里面， 有什么不明白的可以留言。

[swagger-php laravel5](http://pan.baidu.com/s/1qWyqJjY)


