---
layout: post
title: "数据结构-list"
date: 2015-04-08 
description: "数据结构-list"
tag: 分布式缓存 
---   



### 1. Redis数据结构-List
        list是一个链表结构，主要功能是push pop 获取一个范围的所有值等等，操作中key理解为链表的名字Redis的list类型其实就是一个每个子元素都是string类型的双向链表。
        链表的最大长度为2^32。可以通过Push pop操作从链表的头部或者尾部添加删除元素。这样使得list既可以用作栈，也可以用做队列关于list的pop操作还有阻塞版本的，当我们lrpop一个list对象时，如果list是空，或者不存在，会立即返回nil。但是阻塞版本的，可以添加超时时间，超时后会返回nil。为什么要阻塞版本的pop呢？主要是为了避免轮询。

### 2.命令
    1.lpush        在key对应的list的头部添加字符串元素
    2.rpush        在key对应的list的尾部添加字符串元素
                   先在list尾部插入一个hello,然后在hello的尾部插入一个world
    3.linsert      在key对应list的特定位置之前或之后添加字符串元素
    4.lset         设置list中指定下标的元素值(下标从0开始)
    5.lrem         从key对应list中删除count个和value相同的元素
                   count>0时，按从头到尾的顺序删除
                   count<0时，按从尾到头的顺序删除
                   count=0时，删除全部
    6.ltrim        保留指定key的值范围内的数据
    7.lpop         从list的头部删除元素，并返回删除元素
    8.rpop         从list的尾部删除元素，并返回删除元素
    9.rpoplpush    从第一个list的尾部移除元素并添加到第二个list的头部，最后返回被移除的元素值，整个操作是原子的，如果第一个list是空或者不存在则返回nil
    10.lindex      返回名称为key的list中index位置的元素
    11.llen        返回key对应list的长度
    
### 3.数据结构
    zipList 和 quickList的组合结构，当数据量较小的情况下，使用ZipList数据结构，它将所有的元素紧挨着一起存储，分配的是一块连续的内存。
    当数据量比较多的时候才会改成quickList.因为普通的链表需要的附加指针空间太大，会比较浪费内存，而且会加重内存碎片化。

### 4.实战