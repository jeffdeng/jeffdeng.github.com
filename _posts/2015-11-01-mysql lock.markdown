---
layout: post
title:  "mysql的乐观锁和悲观锁 手动kill死锁id"
date:   2015-11-01 12:00:00
categories:  🐬mysql

---

* content
{:toc}

## mysql的乐观锁和悲观锁

有一种情况 A B 两个进程同时操作一条数据库的一条记录，最后结果会出现不确定性。
    
    mysql> select * from    users  where id=8 \G;
    *************************** 1. row ***************************
            id: 8
          name: abc
         phone: 98765001
      nickname: NULL
         email: 
        avatar: NULL
        cityid: NULL
    coordinate: NULL
           sex: 0
      password: $2y$10$on.ZcqIKnfn8jq.M4SFx/uiS2nghpioa9Mxeqb2R463mgq1VTKs6C
    phone_type: NULL
    comment_num: 0
     share_num: 0
     login_num: 0
    remember_token: NULL
        vesion: 0
    created_at: 2015-11-05 17:38:55
    updated_at: 2015-11-05 17:38:55
    1 row in set (0.00 sec)



然后A执行了一次update
    
    update   users set phone='123'  where id=8;
    

B执行了一次update

    update   users set phone='321'  where id=8;


那么结果是 phone 等于 123 还是321 呢，就看哪个最后执行了。

为了保证结果的可控性，有2种方式可以解决这个问题。

### 1.悲观锁

 悲观锁就是只能让一个人在同一时间操作同一行数据，最终每个人都成功。

 实现方式

    update   users set phone='123'  where id=8 for update;##在这条语句执行过程中，其他进程的更新操作是在等待中


或者


    set autocommit=0;
    begin;
    update   users set phone='123'  where id=8;
    ##在这个事务update到commit执行过程中，其他进程的更新操作是在等待中
    commit;

本质都一样

### 2.乐观锁

乐观锁是不会阻塞任何一个用户，但是会让第二个操作这行记录的人更新失败，数据结果是第一个人的。
 实现方式

    select * from users  where id=8;
    update   users set phone='123',vesion=vesion+1 where id=8 and vesion='上一次查询结果';



优缺点：
悲观锁会把别的进程阻塞住，让你等啊等，直到更新完了或者提交了，适合那种让每个进程最终结果都成功的情况。

乐观锁只会让并发的N个人中1个人执行成功，其余的都失败，好处是数据一致性保证了，马上知道结果，不用被阻塞。


大并发量下完整的玩法：
用把订单或者抢购提交到redis，用redis队列异步跑。假如没有先来后到的顺序，那么就用对id取余，1，2，3。。。分别开进程去跑，每个进程里面开事务保证数据一致性。


## 如何干掉死锁

网上看到一篇文章，用`mysql> show processlist \G;`里面 `State: locked`的进程kill掉了，然后因为kill掉了死锁。
其实状态为locked只是被真正的罪魁祸首阻塞掉的进程而已，正真的死锁进程`show processlist;`是看不到的。
有个一个简单有快速的方式，就是查看`information_schema.INNODB_TRX`表，里面就是阻塞了别人的查询了。








