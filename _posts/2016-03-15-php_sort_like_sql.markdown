---
layout: post
title:  "php实现对数组多字段排序"
date:   2016-03-15 12:00:00
categories:  🐘php

---

* content
{:toc}

特殊原因，需要合并两张表，然后进行排序，分页，所以有了一下代码

实现效果如SQL :   `'field(status,5,4,6,7,8) ASC,t.last_modified DESC'`


php代码如下：

 
    $obj = new stdClass();
    $obj->status = 4;
    $obj->last_modified = 11111111;

    $obj2 = new stdClass();
    $obj2->status = 4;
    $obj2->last_modified = 22222222;

    $allBorrow[] = $obj;
    $allBorrow[] = $obj2;
 
    usort($allBorrow, function($a,$b){
        $i1 = array_search($a->status,array(5, 4, 6, 7, 8));
        $i2 = array_search($b->status,array(5, 4, 6, 7, 8));
        if ($i1 == $i2) {
            return $a->last_modified > $b->last_modified ? -1 : 1;
        }
        if ($i1 > $i2) {//索引位置越大，排名越后，所以返回1
            return 1;
        } else {
            return -1;
        }
     });

解释如下：

status为 5,4,6,7,8 在这个序列中，5排在最前面，8在最后，假如状态status一样，就比较 last_modified 大小，以此类推，可以实现多字段的排序。


## 面试遇到的一个类似的题

最近面试做到一个题，让做一个类似PHP原生的usort的函数，当时有个关键点没有想明白，就是 返回1要交换，-1 和0 还有什么意义呢。。。所以写了一半没写了
回来后试着用Php写了一个,发现不用管-1 和 0 就行，反正排序只管大小关系就交
用最简单的冒泡排序实现，PHP原生的使用桶排序。
    

    function swap(&$a,&$b){
          $temp = $a;
          $a = $b;
          $b = $temp;
    } 
    function nsort(&$data,$callable){  
        for($i=0; $i<count($data); $i++) {
            for($j=$i+1; $j<count($data); $j++) { 
            $re = $callable($data[$i],$data[$j]);
            if( $re == 1 ) {//最小的换到最底部
                 swap($data[$i], $data[$j]);
            }
         }
      }
    }
    // $callable = function($a,$b){
    //     if ($a > $b) {//索引位置越大，排名越后，所以返回1
    //         return 1;
    //     } else {
    //         return -1;
    //     }
    //  };

    $callable = function($a,$b){
        $i1 = array_search($a->status,array(5, 4, 6, 7, 8));
        $i2 = array_search($b->status,array(5, 4, 6, 7, 8));
        if ($i1 == $i2) {
            return $a->last_modified > $b->last_modified ? -1 : 1;
        }
        if ($i1 > $i2) {//索引位置越大，排名越后，所以返回1
            return 1;
        } else {
            return -1;
        }
    };


      //  $data = array(5,1,4,3,2);

        $obj = new stdClass();
        $obj->status = 4;
        $obj->last_modified = 11111111;

        $obj2 = new stdClass();
        $obj2->status = 4;
        $obj2->last_modified = 22222222;

        $obj3 = new stdClass();
        $obj3->status = 5;
        $obj3->last_modified = 22222222;

        $obj4 = new stdClass();
        $obj4->status = 5;
        $obj4->last_modified = 1111111;

        $allBorrow[] = $obj;
        $allBorrow[] = $obj2;
        $allBorrow[] = $obj3;
        $allBorrow[] = $obj4;

        nsort($allBorrow,$callable);

        var_dump($allBorrow);





