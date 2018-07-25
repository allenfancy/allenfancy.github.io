---
layout: post
title: "数据结构-hash"
date: 2015-04-14 
description: "数据结构-hash"
tag: 分布式缓存 
---   



### 1.redis数据结构-> Hash
    Redis hash是一个String类型的field 和 value的映射表，它的添加、删除操作都是O(1).hash特别适合用于存储对象。相较而言将对象的每个字段存储单个string类型，将一个对象存在hash类型中会占用更少的内存，并且可以更加方便的存储整个对象。省内存的原因：新建一个hash对象时开始是用zipmap(又称为small hash)来存储的。这个zipmap其实并不是hash table,但是zipmap相比正常的hash实现可以节省不少hash本身需要的一些元数据存储的开销。尽管zipmap的添加、删除、查找都是O(n),但是由于一般对象的field数量都不太多。所以使用zipmap也是很快的，也就是说添加删除平均还是O(1).如果field或者value的大小超出一定限制后，Redis会在内部自动将zipmap替换成正常的hash实现。这个限制在配置文件中指定：
    hash-max-zipmap-entries 64 #配置字段最多64个
    hash-max-zipmap-value 512 #配置value最大为512字节

### 2.命令
    1.hset 设置hash field为指定值，如果key不存在，则先创建。
    2.hsetnx 设置 hash field为指定值，如果key不存在，则先创建。如果field已经存在，返回0
    3.hmset 同时设置多个hash 的多个field
    4.hget 获取指定的hash field
    5.hmget 获取全部指定的hash field
    6.hincryby 指定的hash field 加上给定的值
    7.hexists 测试指定field是否存在
    8.hlen 返回指定hash的field数量
    9.hdel 删除指定hash 的field数量
    10.hkeys 返回 hash的所有field
    11.hvals 返回 hash的所有的value
    12.hgetall 返回某个hash中全部的field及value
    
### 3.数据结构


### 4.实战