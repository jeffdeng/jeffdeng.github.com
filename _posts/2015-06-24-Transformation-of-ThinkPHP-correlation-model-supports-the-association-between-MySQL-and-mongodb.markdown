---
layout: post
title:  "改造thinkphp关联模型，支持mysql和mongodb之间的关联"
date:   2015-06-25 12:00:00
categories:  🐘php

---

* content
{:toc}

### 序

thinkphp对mongodb的关联模型一点也不支持，mysql和mongodb之间关联更谈不上了。

当你想关联两种不同不同数据库查询，或者mongo自身关联查询的时候，只有自己分开写了，非常浪费时间。

laravel目前是支持的
目前tp的生态还是很封闭，composer加载的第三方东西对TP可有可无，TP的一切依赖都是自己的。

## 重写一个Relation model

在项目的公共模块下面建立一个模型文件 `Application/Common/Model/RelationMongoTraint.class.php`

然后把系统的RelationModel复制过来，改 `class RelationModel extends ....`
为`trait  RelationMongoTraint` ,这个Traint怎么用呢，到时候直接到需要使用的关联模型的model里
   
    use RelationMongoTraint;

在文件里面如下：

    <?php
    namespace Common\Model;
    class  ImageModel extends MongoCommonModel {
    public $fields=[
    'id',//'int', NOT NULL,
    'data_type',//'tinyint',图片类型，0：奖品，1: 商品, 2：心愿',
     'sort',//排序
    'class',//图片分类 0形象图 1海报图
     'is_top',//置顶 0 1 
    'relate_id',//'int',图片关联主键，商品，心愿id等',
    'status',//'int',-1 删除 0 禁用，1正常,
    'create_time',//'int', NOT NULL,
    'update_time',//'int', NOT NULL
    'att_id',//附件id
    ];
    //HAS_ONE 当前模型没有外键，外键在管理表上 Image_id,
    protected $_link = array(
        'Attachment'=>array(
                'mapping_type'=> self::BELONGS_TO,
                'class_name' => 'Attachment',
                'foreign_key'=>'att_id',
                'mapping_name'=>'attachment',
                'mapping_fields'=>'id,type,url,describe',
        ),
    );

        use RelationMongoTraint;
    }

这么一句，Traint里面的代码就插入进去了，这个特效非常好用。下面开始一点修改这个traint.

### 修改关联函数，防止冲突

由于系统自带的关联模型用的是 `relation`函数，那么我们就用`relationMongo`,这样做可以保证万一你需要用到系统的关联呢，换个函数名称就行了。

    /**
     * 进行关联查询
     * @access public
     * @param mixed $name 关联名称
     * @return Model
     */
    public function relationMongo($name) {
        $this->options['linkMongo']  =   $name;
        return $this;
    }

这里把`$this->options['link]`换成`$this->options['linkMongo']`也是为了防止和原来的冲突，解析来把整个traint `$this->options['link]`的地方都换掉。

系统有个`__call`魔术方法下面有段代码并没什么卵用，这个方法已经被relation截住了，根本不会调用
    
    /**
     * 动态方法实现
     * @access public
     * @param string $method 方法名称
     * @param array $args 调用参数
     * @return mixed
    */
    public function __call($method,$args) {
        if(strtolower(substr($method,0,13))=='relationMongo'){
            $type    =   strtoupper(substr($method,13));
            if(in_array($type,array('ADD','SAVE','DEL'),true)) {
                array_unshift($args,$type);
                return call_user_func_array(array(&$this, 'opRelation'), $args);
            }
        }else{
            return parent::__call($method,$args);
        }
    }
    

else 部分是永远不会调用的（根据relation 到 relationMongo 我做了些替换，系统的思路没变 ）。


### 去掉mongo生成的_id

由于我们自定义了mongo的id为`id`,并且自增，系统的`_id`就基本没用，而且返回到APP还看起来不舒服
    
    protected function checkId(&$resu){
        unset($resu['_id']);
        return $resu;
    }

把它`unset`掉，把这个函数写在`_after_select` `_after_find`这俩钩子里面

        // 查询成功后的回调方法
    protected function _after_find(&$result,$options) {
        
        $this->checkId($result);
        
        // 获取关联数据 并附加到结果中
        if(!empty($options['linkMongo']))
            $this->getRelation($result,$options['linkMongo']);
    }

    // 查询数据集成功后的回调方法
    protected function _after_select(&$result,$options) {
        
        array_walk($result,array($this,'checkId'));
        
        // 获取关联数据 并附加到结果中
        if(!empty($options['linkMongo']))
            $this->getRelations($result,$options['linkMongo']);
    }


### 最后要修改的地方`getRelation`

TP最终开始关联的地方都是在这个函数里面

    /**
     * 获取返回数据的关联记录
     * @access protected
     * @param mixed $result  返回数据
     * @param string|array $name  关联名称
     * @param boolean $return 是否返回关联数据本身
     * @return array
     */
    protected function getRelation(&$result,$name='',$return=false){}


把第一句那个if的大括号给改掉，这么一层层嵌套不利于代码的可读性，改为：
    
    if(empty($this->_link)) return $result;

`foreach` 下面，一直到`switch`小改一下
    
    foreach($this->_link as $key=>$val) {
                    $mappingName =  !empty($val['mapping_name'])?$val['mapping_name']:$key; // 映射名称
                    if(empty($name) || true === $name || $mappingName == $name || (is_array($name) && in_array($mappingName,$name))) {
                    $mappingType = !empty($val['mapping_type'])?$val['mapping_type']:$val;  //  关联类型
                    $mappingClass  = !empty($val['class_name'])?$val['class_name']:$key;            //  关联类名
                    if (!empty($val['condition'])) {  // 关联条件
                        foreach ($val['condition'] as $k=>$v)
                        $mappingCondition[$k] = $v;
                    }      
                    
                    $mappingKey =!empty($val['mapping_key'])? $val['mapping_key'] : $this->getPk(); // 关联键名
                    if(strtoupper($mappingClass)==strtoupper($this->name)) {
                        // 自引用关联 获取父键名
                        $mappingFk   =   !empty($val['parent_key'])? $val['parent_key'] : 'parent_id';
                    }else{
                        $mappingFk   =   !empty($val['foreign_key'])?$val['foreign_key']:strtolower($this->name).'_id';     //  关联外键
                    }
                    // 获取关联模型对象
                    $model = D($mappingClass);
                    
                    $mappingFields = $val['mapping_fields'];
                    if ($val['mapping_fields']=='*') {  // 映射字段，Mongo不支持*
                        $mappingFields='';
                    }
                    switch($mappingType)

之前TP的`$mappingCondition`是用字符串来查询的，但是。。。TP的`mongoModel`只支持数组方式的条件查询，所以这里把 `mappingCondition`修改为数组方式，同时以后在模型的`link`里面condition也只能用数组方式（键值对）。

这里也是为什么TP关联模型不兼容mongo的主要原因。

在`HAO_ONE`部分改成这样
    
    case self::HAS_ONE:
                            $pk   =  $result[$mappingKey];
                            $mappingCondition[$mappingFk] = (int)$pk;
                            $relationData   =  $model->where($mappingCondition)->field($mappingFields)->find();
                            if (!empty($val['relation_deep'])){
                                $model->getRelation($relationData,$val['relation_deep']);
                            }
                            
                            break;

主要是改为数组查询方式，同时，主要把   `(int)$pk`强制转换为整形，`‘1’` 和 `1` 查询的结果在这个mongo扩展里面完全不同。

其他地方依法炮制 ,有个比较特别的是`MANY_TO_MANY`，系统完全用的`sql`拼的，要改为数组形式得花费好大功夫，算了，反正现在还用不上这个，先放着。

只改了查询部分，需要可以把关联插入 删除 更新也做了，在`opRelation`里面。

### 查询方式举例：

    D('Image')->relationMongo(true)->where($map)->select();


实际上 Traint是不支持const的，而relation模型有需要这4个常量
    
    const   HAS_ONE     =   1;
    const   BELONGS_TO  =   2;
    const   HAS_MANY    =   3;
    const   MANY_TO_MANY=   4;
 
 >* 我把几个放到了mongo模型的父类里面 `MongoCommonModel`,mysql model就直接集成relation model,否者只能在`use RelationMongoTraint;`同时把上面4个常量在写一遍了。

 >* 另外TP的关联不是lazy load的，10个select关联的也要查询10次，非常不经济。laravel对此做了优化，用in就能实现优化到2次查询。愿意折腾的倒是可以花1小时优化一下。


最终文件 [RelationMongoTraint.class.php](http://pan.baidu.com/s/1bn0gpRL)

