---
layout: post
title:  "API接口常用的测试+查看利器"
date:   2015-06-23 12:06:59
categories: 📱app
excerpt: "API接口常用的测试+查看利器"
---

## Postman && HttpRequester

  一个火狐HTTP插件，可以自定义参数，Header,上传文件，查看cookie，用来测试PAI接口的再好不过了。

  chrome下也有对应的插件叫`Postman`。

  后来用了下，发现HttpRequester有BUG，接口本来没问题得，用这个会出现问题，还是用postman好些

##JSONView

  一款JOSN格式化插件，只要你返回的HTTP头里面有application/json，就会帮你格式化成可视化的形式，方便查看接口数据正确性。

  同样的插件还有json handle，这个会加一些鲜亮的色彩，不过不方便复制上面的字段。

![jsonview](/static/img/jsonview.jpg)

##Firebug

  这个插件是火狐里面最常用的插件，调试JS CSS必备神器，有了它就不要再用alert这么落后的调试方式了，console.log，打印对象、字符串，一清二楚。另外还能查看cookie，网络，对API调试也很有帮助。



