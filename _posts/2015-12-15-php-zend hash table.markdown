---
layout: post
title:  "php内核分析之最重要的数组实现方式"
date:   2015-12-15 12:00:00
categories:  🐘php

---

* content
{:toc}

## 序
要说PHP最好用的是什么，我肯定说PHP的数组，可以非常方便的组装你想要的数据，然后给模板。
而PHP的数组的实现方式其实用到了2中简单的数据结构完成的
    1是哈希表
    2是双向链表

## HashTable — PHP最核心数据结构


PHP的hash table具有如下特点：

1 支持典型的key->value查询
2 可以当做数组使用
3 添加、删除节点是O（1）复杂度
4 key支持混合类型：同时存在关联数组合索引数组
4 Value支持混合类型：array (“string”,2332)
5 支持线性遍历：如foreach
Zend hash table实现了典型的hash表散列结构，同时通过附加一个双向链表，提供了正向、反向遍历数组的功能。其结构如下图暂无：
![demo-1024x693.jpg](暂无)

可以看到，在hash table中既有key->value形式的散列结构，也有双向链表模式，使得它能够非常方便的支持快速查找和线性遍历。

散列结构：Zend的散列结构是典型的hash表模型，通过链表的方式来解决冲突。
zend的hash table是一个自增长的数据结构，初始大小均为8，当hash表数目满了之后，其会动态以2倍的方式扩容 
在进行key value快速查找时候，zend在每个元素中都会用一个变量nKeyLength标识key的长度以作快速判定元素是再hash table位置。
    
    //用字符key和字符key的lenth来计算出h
    h = zend_inline_hash_func(arKey, nKeyLength);
    
    //这里用了掩码而不是mod,因为位运算更快
     nIndex = h & ht->nTableMask;

     //从哈希表里面取出链表节点指针
     p = ht->arBuckets[nIndex];
  


双向链表：Zend hash table通过一个链表结构，实现了元素的线性遍历。理论上，做遍历使用单向链表就够了，之所以使用双向链表，主要目的是为了快速删除，避免遍历。


  
....以后再写




