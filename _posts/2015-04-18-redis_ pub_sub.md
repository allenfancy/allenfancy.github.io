---
layout: post
title: "redis-pub/sub"
date: 2015-04-18 
description: "redis-pub/sub"
tag: 分布式缓存 
---   


### 1.Redis数据结构-发布订阅
    发布订阅(pub/sub)是一种消息通信模式，主要的目的是解耦消息发布者和消息订阅者之间的耦合，这点和设计模式中的观察者模式比较相似。pub/sub不仅仅解决发布者和订阅者直接代码级别耦合也解决俩者在物理部署上的耦合。redis作为一个Pub/sub的server。在订阅者和发布者之间起到了消息路由的功能。订阅者可以通过subscribe和psubscribe命令向redis server订阅自己感兴趣的消息类型，redis将消息类型称为通道(channel)。当发布者通过publish命令向redis server发送特定类型的消息时。订阅该消息类型的全部client都会收到此消息。这里的消息传递是多对多的。一个client可以订阅多个channel也可以向多个channel发送消息
### 2.基本命令：
    1.PUBLISH
         PUBLISH channel message
         将信息message发送到指定的频道channel。
         时间复杂度：O(N+M) : n是订阅者的数量，而M是使用模式订阅的客户端的数量
         Demo：publish chatroom hello everyOne
    2.PSUBSCRIBE
         PSUBSCIBE pattern [pattern ...]
         订阅一个或多个符合给定模式的频道。
         每个模式以*作为匹配符，比如it*匹配所有it开头的频道(it.news,ti.blog等)
         时间复杂读:O(N)，N是订阅的模式的数量
    
    3.PUNSUBSCRIBE
         PUNSUBSRCRIBE [pattern [pattern ...]]
         指示客户端退订给所有给定模式。
         如果没有模式被指定，那么客户端使用【PSUBSCRIBE】命令订阅的所有模式都会被退订
    4.SUBSCRIBE
         订阅给定的一个或者多个频道
         SUBSCRIBE channel [channel ...]
         DEMO: subscribe msg chatroom
    5.UNSUBSCRIBE
         指示客户端退订给定的频道
         UNSUBCRIBE [channel [channel ...]]
    6.PUBSUB
         PUBSUB <subcommand> [argument [argument]]
         PUBSUB 是查看订阅与发布系统状态的内部命令，它是由几个不同格式命令组成的。
         1.PUBSUB CHANNELS [pattern]
              列出当前活跃的频道。
              活跃频道指的是那些至少有一个订阅者的频道，订阅模式的客户端不计算在内。
         2.PUBSUB NUMSUB [channel …. ]
              返回给定频道的订阅者数量，订阅者模式的客户端不计算在内
         3.PUBSUB NUMPAT 
              返回订阅模式的数量
              注意，这个名返回的不是订阅模式客户端的数量，而是客户端订阅的所有模式的数量总和

    Pipleline批量发送请求：
        redis是一个cs模式的tcp server，使用和http类似的请求响应协议。一个client可以通过一个socket连接发起多个请求命令。每个请求命令发出后client通常会阻塞并等待redis服务处理，redis处理完请求命令后会将结果以报文的形式返回给redis

### 3.虚拟内存的使用
        目的：暂时把不经常访问的数据从内存交换到磁盘中，从而腾出宝贵的内存用于其他需要访问的数据。尤其是对于redis这样的内存数据库，内存总是不够用的。除了可以将数据分割到多个redis server外。另外能够提高数据库容量的办法就是使用虚拟内存把那些不经常访问访问的数据交换到磁盘上，如果我们的存储的数据总是有很少部分的数据被将从访问。大部分数据很少被访问，对于网站来说：活跃的用户量一般不大。虚拟内存不但能够提高单台redis server数据库的容量，而且也不会对性能造成太多的影响
        redis为什么没有使用操作系统提供的虚拟内存机制而是自己实现虚拟内存机制呢？
            1.操作系统的虚拟内存是4K页面为最小单位进行交换的。而redis的大多数数据对象远远小于4K，所以一个操作系统页面上可能有多个redis对象。另外redis的集合对象类型如list set 可能存在与多个操作系统页面上。最终可能造成只有10%key被经常访问，但是所有操作系统页面都会被操作系统认为是活跃的。这样只有内存真正耗尽时操作系统才会交换页面
             2.相比操作系统的交换方式，redis可以将被交换到磁盘的对象进行压缩，保存到磁盘的对象可以去除指针和对象元数据信息，一般压缩后的对象会比内存中的对象小10倍，这样redis虚拟内存会比操作系统虚拟内存少很多IO操作
   
    VM相关配置：
         vm-enabled yes                  #开启VM功能
         vm-swap-file /temp/redis.swap  #交换出来的value保存的文件路径
         vm-max-memory 100000   #redis 使用的最大内存上限
         vm-page-size 32        #每个页面的大小32个字节
         vm-pages 134217728     #最多使用多少页面
         vm-max-threads 4       #用于执行value对象换入换出的工作线程量
    
    redis的虚拟内存在设计上为了保证key的查找速度，只会将value交换到swap文件中。所以如果是内存问题是由于太多value很小的key造成的，那么虚拟内存并不能解决，和操作系统一样redis也是按照页面来交换对象的。redis规定同给一个页面只能保存一个对象。但是一个对象可以保存在多个页面中。在redis使用内存没有超过vm-max-memory 之前是不会交换任何value的。
     
    
### 3.数据结构


### 4.实战