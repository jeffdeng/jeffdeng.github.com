---
layout: post
title:  "PHP flock实现文件加锁——PHP多个进程同时写入同一个文件"
date:   2016-03-15 12:00:00
categories:  🐘php

---

* content
{:toc}

代码如下


    $handle = fopen('1.txt', 'a+');
    if (flock($handle, LOCK_EX)) {//这一步加锁不成功会阻塞，不要写什么while true了
         $random ="sleep(10) ";
         fwrite($handle, 'time is  '.microtime(true)." $random \r\n");
         //flock($handle, LOCK_UN);//测试就没必要加这个， fclose($handle);会自动解锁
         sleep(10);
    }else{
        header("Status: 404 Not Found"); //写其他的也行
    }
    fclose($handle);


第一次用siege运行`siege -c10 -r1 -b http://www.laravel.com/1.php`

同时马上修改代码如下，在10s内刷新浏览器，访问`http://www.laravel.com/1.php` 2次
    
     $handle = fopen('1.txt', 'a+');
     if (flock($handle, LOCK_EX)) {//这一步加锁不成功会阻塞，不要写什么while true了
         $random ="sleep(1) ";
         fwrite($handle, 'time is  '.microtime(true)." $random \r\n");
         sleep(1);
    }else{
        header("Status: 404 Not Found"); 
    }
    fclose($handle);

siege 执行成功了6次
![flock.png](/static/img/flock1.png)

同时，浏览器也在转圈等待，说明被flock阻塞住了
查看写入结果 也发现只有等sleep 10秒的执行完了才sleep 1的
![flock.png](/static/img/flock2.png)


