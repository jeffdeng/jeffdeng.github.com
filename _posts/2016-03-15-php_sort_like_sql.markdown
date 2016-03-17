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
```

 
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
```

解释如下：

status为 5,4,6,7,8 在这个序列中，5排在最前面，8在最后，假如状态status一样，就比较 last_modified 大小，以此类推，可以实现多字段的排序。





