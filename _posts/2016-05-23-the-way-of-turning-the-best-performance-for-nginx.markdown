---
layout: post
title:  "Nginx并发性能不断调优的正确姿势"
date:   2016-05-24 12:00:00
categories:  🐧linux

---

* content
{:toc}

先啰嗦一下几个举例

##Nginx调优举例

Nginx调优可以调的参数有很多，常见的参数比如:

Worker Processes ,一般和CPU核数一致

Worker Connections ，每个进程允许的最多连接数， 理论上每台nginx 服务器的最大连接数为worker_processes*worker_connections，这个参数尽量大些，否者并发多了会没有多余的Worker Connections去连接。

    client_header_buffer_size 4k;

    keepalive_timeout 60; 

    open_file_cache max=65535 inactive=60s;

等等这些。

##linux 内核调优举例

linux 默认值 open files 比较小。
    #ulimit -n
    1024
`/etc/security/limits.conf` 最后增加

    - soft nofile 65535
    - hard nofile 65535

`/etc/pam.d/login`加入 
 
    session    required     pam_limits.so

另外还有一个开打文件最大值需要修改 `vim etc/sysctl.conf`
    
    fs.file-max = 100000

还有些tcp的参数 `vim /etc/sysctl.conf`

    # 调高系统的 IP 以及端口数据限制，从可以接受更多的连接 32768 61000
     net.ipv4.ip_local_port_range = 32768 65000
    #
     net.ipv4.tcp_window_scaling = 1

    # 调高系统的 IP 以及端口数据限制，从可以接受更多的连接 32768 61000
     net.ipv4.ip_local_port_range = 32768 65000
     net.ipv4.tcp_window_scaling = 1

     # # 调高 socket 侦听数阀值
    net.core.somaxconn = 65535
    net.ipv4.tcp_max_tw_buckets = 1440000
    #
    # # Increase TCP buffer sizes
    # # 调大 TCP 存储大小
    net.core.rmem_default = 8388608
    net.core.rmem_max = 16777216
    net.core.wmem_max = 16777216
    net.ipv4.tcp_rmem = 4096 87380 16777216
    net.ipv4.tcp_wmem = 4096 65536 16777216
    net.ipv4.tcp_congestion_control = cubic

    net.ipv4.tcp_tw_reuse = 1
    net.ipv4.tcp_tw_recycle = 1
    fs.file-max = 65000

##调优的正确姿势

网上写调优基本上上都是千篇一律，一堆参数就贴出来了，怎么来的，具体怎么调的，没说
知道这些参数也有用，但是遇到新问题的时候有些问题还是不好解决的，那么作为一个nginx初学者如何调优呢---压测。

基本流程：

1.修改nginx或者Linux配置

2.压测看并发和响应上去没有

3.重复1，2

那么问题来了，怎么压测？

1是压测标准是什么，比如
   
    ab -c200 -n1000 localhsot 

    ab -c300 -n1000 localhsot 

发现100并发服务器也扛得住，200并发也扛得住,300+并发也扛得住，怎么衡量这台服务器更优了，写报告的时候这台服务器的并发值是多少？或者有什么好压测工具可以告诉我们？

我认为http请求的性能标准就是：
    
     在一个持续请求时间内，一定并发用户下N，服务器能提供稳定的服务，那么就可以认定这台服务器的并发量是N

这是对http请求，websocket这种无效，websocket有更简单的性能标准。

那么就可以用

    $ ab -c3000 -t10 http://192.168.92.148/
    This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
    Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
    Licensed to The Apache Software Foundation, http://www.apache.org/

    Benchmarking 192.168.92.148 (be patient)
    Completed 5000 requests
    Completed 10000 requests
    Completed 15000 requests
    Completed 20000 requests
    Completed 25000 requests
    Completed 30000 requests
    Completed 35000 requests
    Completed 40000 requests
    Completed 45000 requests
    Completed 50000 requests
    Finished 50000 requests


    Server Software:        nginx/1.10.0
    Server Hostname:        192.168.92.148
    Server Port:            80

    Document Path:          /
    Document Length:        25 bytes

    Concurrency Level:      3000
    Time taken for tests:   8.142 seconds
    Complete requests:      50000
    Failed requests:        0
    Write errors:           0
    Total transferred:      12800000 bytes
    HTML transferred:       1250000 bytes
    Requests per second:    6140.98 [#/sec] (mean)
    Time per request:       488.522 [ms] (mean)
    Time per request:       0.163 [ms] (mean, across all concurrent requests)
    Transfer rate:          1535.24 [Kbytes/sec] received

    Connection Times (ms)
                  min  mean[+/-sd] median   max
    Connect:        1  203 211.9    170    1281
    Processing:    34  249 165.0    236    1127
    Waiting:        0  201 155.0    162    1127
    Total:         82  451 280.4    444    2182

    Percentage of the requests served within a certain time (ms)
      50%    444
      66%    508
      75%    531
      80%    545
      90%    653
      95%   1016
      98%   1440
      99%   1492
     100%   2182 (longest request) 


第一个是单个并发用户的响应时间，
    
     Time per request:       488.522 [ms] (mean)
 
 一般一个网页反应在0.5s下比较好，然后继续加并发用户数，用户增加，而反应时间还可以接受，那么这个优化就是有效的

     $ ab -c1000 -t100 http://192.168.92.148/
     $ ab -c2000 -t100 http://192.168.92.148/
     $ ab -c3000 -t100 http://192.168.92.148/

多几次就测出来了。

压测工具除了 `ab` ，还有`webbench` ,`tsung`。

`webbench` 提供的结果太过简单，那个并发数据没有太大意义。

`tsung`是个重量级工具，可以测试http外的其他协议，而且测试结果更接近真实值,而且可以虚拟上百万用户，分布式测试。




