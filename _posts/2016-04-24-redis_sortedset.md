---
layout: post
title: "数据结构-sortedset"
date: 2015-04-24
description: "数据结构-sortedset"
tag: 分布式缓存 
---   



### 1.redis数据结构-SortedSet
        sorted set是set的一个升级版本，它在set的基础上增加了一个顺序属性，这一个属性在添加修改元素的时候，可以指定，每次指定后，zset会自动重新按新的值调整顺序，可以简单的理解为两列的mysql表，一列存value，一列存顺序。
        sorted set也是一个string类型元素的集合，不同的是每个元素都会关联一个double类型的score.sorted 
        set的实现是skip list 和 hash table的混合体当元素被添加到集合中时，一个元素到score的映射被添加到hash table中，所以给定一个元素获取score的开销是O(1),另外一个socre到元素的映射被添加到skip list,并按照score排序，所以就可以有序的获取集合中的元素。添加、删除操作开销都是O(log(n))和skip list的开销一致.
        redis的skipList实现用的是双向链表，这样那个可以逆序从尾部取元素。sorted set最经常的使用方式应该是作为索引来使用。我们可以把要排序的字段作为score存储，对象的id当元素的存储。

### 2.命令：
     1.zadd      向名称key的zset中添加元素member,socre用于排序。如果该元素已经村粗，则根据socre更需你该元素的顺序
     2.zrem      删除名称为key 的 zset中的元素member
     3.zincrby  如果名称为key的zset中已经存在元素member,则该元素的score增加increment;否则向集合中添加该元素，其score的值为increment.
                zincrby myzset2 2 ‘one’ :将one的score 从1 增加了2，增加到3
     4.zrank    返回名称为key的zset中member元素的排序名(按score从小到大排序)即下标
     5.zrevrank  返回名称为key的zset中member元素的排序名(按score从大到小排序)即下标
     6.zrevrange  返回名称为key的zset(按score从大到小排序)中的index从start到end的所有元素
     7.zrangebyscore 返回集合中score在给定区间的元素
     8.zcount 返回集合中score在给定区间的数量
     9.zcard  返回集合中元素个数
     10.zscore 返回给定元素对应的score   zscore  key element
     11.zremrangebyrank 删除集合中排名在给定区间的元素  zremrangebyrank key score element
     12.zremrangebyscore 删除集合中score在给定区间的元素
    
### 3.数据结构


### 4.实战