---
layout: post
title: "ZooKeeper运维"
date: 2017-05-05
description: "ZooKeeper运维"
tag: zookeeper
---   

### 一、zooKeeper 配置文件参数说明
|参数名|说明|
|:-:|:-:|
|clientPort|设置当前服务器对外的服务端口，客户端会通过该端口和Zookeeper服务器创建连接，一般设置为2181
|dataDir|ZooKeeper服务器存储快照文件的目录。默认情况下，如果没有配置参数dataLogDir,那么事物日志也会存储在这个目录中。由于事物日志的写性能直接影响Zookeeper整体的服务能力，因此同时通过参数dataLogDir来配置ZooKeeper事物日志的存储目录
|tickTime|该参数有默认值：3000，单位毫秒，可以不配置，很多运行时的时间间隔都使用ticketTime的倍数来表示的。例如：ZooKeeper中会话的最小超时时间默认是2*tickTime
|dataLogDir|用于配置ZooKeeper服务器存储事务日志文件的目录。默认情况下，ZK会将事务日志文件和快照参数存储在同一个目录中。应该要分开
|initLimit|该参数默认值为10.即表示是参数tickTime值的10倍，必须配置且为正数。作用：配置Leader等待Follower启动，并完成数据同步的时间。
|syncLimit|
|snapCount|
|preAllocSize|
|minSessionTimeout|
|maxSessionTimeout|
|maxClientCnxns|
|jute.maxbuffer|
|clientPortAddress|
|server.id=host:port:port|id：为serverID，与每台机器中maid文件中的数字一致port：指定Follower服务器与Leader进行运行时通信和数据同步时所使用的端口.port：端口则是专门用于进行Leader选举过程中的投票通信端口
|autopurge.snapRetainCount|
|autopurge.purgeInterval|ZK数据文件和事务日志文件的自动清理
|sync.warningthresholdms|
|forceSync|
|globalOutstandingLimit|服务端限制同时处理的请求书，即最大请求堆积数量
|leaderServes|默认情况下，Leader服务器接受并处理客户端的所有读写请求。在ZK的架构中，Leader服务器主要用来进行对事务更新请求的协调以及集群本身的运行时协调，因此，可以设置让Leader服务器不接受客户端的连接，专注于进行分布式协调。
|SkipAcl|参数默认值为：no，可以不配置，可选配置项为yes。是否跳过ACL权限检查
|cnxtimeolut|该参数默认值为：5000，单位毫秒，可以不配置，仅支持系统属性方式配置：zookeeper.cnxTimeout.该参数用于配置在Leader选举过程中各服务器之间进行TCP连接创建的超时时间
|electionAlg|无用

### 二、zooKeeper 基本命令
     conf：命令用于输出ZooKeeper服务器运行时使用的基本配置信息，包括clientPort、dataDir和tickTime等，以便查看参数
     cons：命令用于输出当前这台服务器上所有客户端连接的详细信息，包括每个客户端的客户端Ip、会话ID和最后一次与服务器交互的操作类型等
     crst：功能命令，用于重置所有的客户端连接统计信息。
     dump:用于输出当前集群的所有会话信息，包括这些会话的会话ID，以及每个会话创建的临时节点等信息。
     envi：用于输出ZooKeeper所在服务器运行时的环境信息，包括os.version java.version和user.home
     ruok:用于输出当前ZooKeeper服务器是否正常运行。
     stat：用于获取ZooKeeper服务器的运行时状态信息，包括基本的Zookeeper版本、打包信息、运行时角色、集群数据节点个数等信息，另外还会将服务器的客户端连接信息打印出来
     srvr:srvr不会将客户端的连接情况输出，仅仅输出服务器的自身信息
     srst:命令是一个功能行命令，用于重置所有服务器的统计信息
     wchs:命令用于输出当前服务器上管理的Watcher的概要信息
     wchc:输出当前服务器上管理的Watcher的详细信息，以会话为单位进行归组
     wchp:输出当前服务器上管理Watcher的详细信息，不同点在于wchp命令的输出信息以节点路径为单位进行归组。
     mntr:输出比stat命令更为详尽的服务器统计信息，包括请求处理的延迟情况、服务器内存数据库大小和集群的数据同步情况。

### 三、JMX(Java Management Extensions,Java管理扩展)
        JMX是一个为应用程序、设备、系统等植入管理功能，能够非常方便地让Java系统对提供运行时数据信息获取和系统管控的接口。从3.3.0版本开始，ZooKeeper也使用了标准的JMX方式来对外提供运行时数据信息和便捷的管控接口。
        开启远程JMX：在ZkServer.sh中配置：
            -Dcom.sun.management.jmxremote.port=5000
            -Dcom.sun.management.jmxremote.ssl=false
            -Dcom.sun.management.jmxremote.authenticate=false 