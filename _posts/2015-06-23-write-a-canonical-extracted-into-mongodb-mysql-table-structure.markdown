---
layout: post
title:  "写个正则把mysql表结构提取出来导入mongodb"
date:   2015-06-23 22:14:54
categories: mysql
excerpt: 把部分mysql表结构导入到mongodb
---


最近要把部分mysql表结构导入到mongodb，由于mongodb没有msyql固定的表结构概念，所以需要搞个文档规范一下。于是就把字段记录入mongo的model里面。
  

    $table=ucwords('goodsCategory');
    $subject=
    <<<EOF
    CREATE TABLE IF NOT EXISTS a_goods_category (
      id int(11) NOT NULL,
      pid int(11) NOT NULL COMMENT '商品分类对象父id',
      name varchar(50) NOT NULL COMMENT '商品分类名称',
      create_time int(11) NOT NULL,
      update_time int(11) NOT NULL
    ) ENGINE=MyISAM DEFAULT CHARSET=utf8 COMMENT='商品分类对象';

    //上面是导出的mysql表结构
    EOF;


    $php_doc=
    <<<EOF
        <?php
        namespace Common\Model;
        class  {$table}Model extends MongoCommonModel {

    EOF;

    $field='public $feild=['."\n";
  
    $pattern="/`(\w+)`\s(\w+)\(\d+\)(?: NOT NULL COMMENT )?'?(.*+)?'?,?/";
    $callback=function($matches) use (&$field){
    // var_dump($matches); 可以打印看下匹配到内容
    
    $field.="\t'{$matches[1]}',//{$matches[2]},{$matches[3]}\n";
      
    };
  
  
    preg_replace_callback($pattern, $callback, $subject);
  
  
    $field.="];\n";
  
    $php_doc.=$field."}";
  
    file_put_contents("{$table}Model.class.php", $php_doc);
     //生成Model文件


-----

这个正则难度不大，不过写的时候还是得查手册，不经常写的东西分分钟忘掉了。

#### 简单分析下这个正则

>* `(\w+)`捕获 字母、数字、下划线和汉字这样的文字，1个或者多个
>* `\d+` 匹配一个或者多个数字
>* `(?: NOT NULL COMMENT )`这么写的目的是不捕获这个小括号里面的内容，这样就不会出现在$matches数组中
>* `,?` 逗号可有可无


`var_dump($matches)`的结果如下

![vardump.jpg](/static/img/vardump.jpg)



