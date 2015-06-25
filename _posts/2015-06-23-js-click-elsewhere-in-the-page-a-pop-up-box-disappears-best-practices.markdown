---
layout: post
title:  "JS点击页面其他地方，弹出框消失的最优做法"
date:   2015-06-23 12:00:59
categories: 🐂js
excerpt: JS点击页面其他地方，弹出框消失的最优做法
---

一般为了做出点击弹出框后，再次点击弹出框为任意的地方，让弹出框消失，会写歌全局变量来判断，鼠标点得区域是否在弹出框上。

这种做法有2个缺点:

>* 1 弹出框内结构复杂的不好还的判断子div
>* 2 增加了一个全局变量

###其实有一种只要一条代码就可以实现这个效果

{% highlight js %}

  $('.div').click(function(e) {
   event.stopPropagation();//该方法将停止事件的传播
   
 });
 
 $(documednt).click(function(e) {
   $('.div').fadeOut('slow');//让弹出框消失吧
 });
 
{% endhighlight %}

原理就是让弹出层的点击事件只让div知道，documednt不知道自然就不会做出响应了。



