---
layout: post
title: "ZooKeeper技术内幕"
date: 2017-05-04
description: "ZooKeeper技术内幕"
tag: zookeeper
---   


### 一、系统模型
     在本节中，我们首先将从数据模型、节点特性、版本、Watcher和ACL五个方面来讲述ZooKeeper的系统模型
     1.1.数据模型
          ZooKeeper的视图结构和标准的Unix文件系统非常类似，但是使用的是数据节点概念，我们称为ZNode。ZNode是ZooKeeper中数据的最小单元，每个ZNode上都可以保存数据，同时还可以挂载子节点，因此构成了一个层次化的命名空间，我们称为树。
         树：
          在ZooKeeper中，每个数据节点都被称为一个ZNode，所有ZNode按层次化结构进行组织，形成一棵树。ZNode的节点路径标识方式和Unix文件路径非常相似，都是由一系列使用斜杠(/)进行分割的路径标识
         事务ID
          在ZooKeeper中，事务是指能够改变ZooKeeper服务器状态的操作，我们也称之为事务操作或更新操作，一般包括数据节点创建与删除、数据节点内容更新和客户端会话创建与失效等操作。对于每一个事务请求，ZooKeeper都会为其分配一个全局唯一的事务ID，用ZXID来表示，通常是一个64位的数字。每个ZXID对应一次更新操作，从这些ZXID中可以间接地识别出ZooKeeper处理这些更新操作请求的全局顺序。
     1.2.节点特点
          ZooKeeper的命名空间是由一系列数据节点组成的。
         节点类型：
          在ZooKeeper中，每个数据节点都是有声明周期的，其生命周期的长短取决于数据节点的节点类型。在ZooKeeper中，节点类型可以分为持久节点(PERSISTENT)、临时节点(EPHEMERAL)和顺序节点(SEQUENTIAL)三大类，通过组合使用，可以生成以下四种组合型节点类型：
         持久节点(PERISSITENT):
          持久节点是ZooKeeper中最常见的一种节点类型。创建之后就会一直存在ZooKeeper服务器上，直到主动删除操作来主动清除这个节点
         持久顺序节点(PERSISTENT_SEQUENTIAL)
          持久顺序节点的基本特性和持久节点是一致的，额外增加的特性表现为顺序性上。
         临时节点(EPHEMERAL)
          临时节点的声明周期和客户端的会话绑定在一起，会话结束后，就会删除节点
         临时顺序节点(EPHEMERAL_SEQUENTIAL)
          临时顺序节点的基本特性和临时节点也是一致的，同样式在临时节点的基础上，添加了顺序的特性
         状态信息：
          ZooKeeper上的数据节点进行数据的写入和子节点的创建。实际上，每个数据节点除了存储了数据内容之外，还存储了数据节点本身的一些状态信息。
     1.3 版本 ——   保证分布式数据原子性操作
          ZooKeeper中为数据节点引入了版本的概念，每个数据节点都有三种类型的版本信息，对数据节点的任何更新操作都会引起版本号的变化，version cversion aversion 
     1.4 Watcher - 数据变更的通知
          ZooKeeper提供了分布式数据的发布/订阅功能。在ZooKeeper中，引入了Watcher机制来实现这种分布式的通知功能。ZooKeeper允许客户端向服务端注册一个Watcher监听，当服务端的一些指定事件触发了这个Watcher，那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。整个Watcher注册与通知过程如图7-4所示：

### 二、ZooKeeper的Watcher
    机制主要包括客户端线程，客户端 WatchManager 和 ZooKeeper 服务器三部分。在具体工作流中，简单地：客户端在向ZooKeeper服务器注册Watcher的同时，会将 Watcher 对象存储在客户端的 WatchManager 中。当 ZooKeeper 服务器触发 Watcher 事件后，会向客户端发送通知。客户端线程从 WatchManager 中取出对应的Watcher对象来执行回调逻辑.
     Watcher接口：
          在ZooKeeper中，接口类Watcher用于表示一个标准的事件处理器，其定义了事件通知相关的逻辑，包含KeeperState和EventType俩个枚举类，分别代表了通知状态和事件类型，同时定义了事件的回调方法：proccess(WatchedEvent event)
     Watcher事件
          同一个事件类型在不同的通知状态中代表的含义有锁不同，下表常见的通知状态和事件类型。
    KeeperState
    EventType
    触发条件
        说明
        None(-1)
        客户端端与服务器成功建立会话
        NodeCreated(1)
        Watcher监听的对应数据节点被创建
        SyncConnected(3)
        NodeDeleted(2)
        Watcher监听的对应数据节点被删除
        此时客户端和服务器处于连接状态
        NodeDataChanged(3)
        Watcher监听的对应的数据节点的内容发生改变
        NodeChildrenChanged(4)
        Watcher监听的对应数据节点的子节点列表发生变更
        Disconnected(0)
        None(-1)
        客户端与ZooKeeper服务器断开连接
        此时客户端和服务器处于断开连接状态
        Expired(-112)
        None(-1)
        会话超时
        此时客户端会话失效，通常同时也会接收到SessionExpiredException异常
        AuthFailed(4)
        None(-1)
        通常有俩种情况：
            1.使用错误的scheme进行权限检查
            2.SAS权限检查失败
            通常同时也会收到AuthFailedException异常
            UnKnown(-1)
            从3.1.0版本开始已放弃
            NoSyncConnected（1）
                ZooKeeper中最常见的几个通知状态和事件类型。其中，针对NodeDataChanged事件，节点的数据内容和数据的版本号dataVersion.因此，即使使用相同的数据内容来跟更新还是会出发这个事件通知，因为对于ZK而言，无论数据内容是否发生了变更，一旦有客户端调用了数据更新的接口，且更新成功，就会更新dataVersion值。
        NodeChildrenChanged事件会在数据节点的子节点【列表】发生变更的时候被触发，这里说的子节点列表变化特指【子节点个数】和【组成情况的变更】，即新增子节点或删除子节点，而子节点内容的变化是不会出发这个事件的。
                AuthFailed这个事件，需要注意的地方是，它的触发条件并不是简单因为当前客户端会话没有权限，而是授权失败。
        使用正确的Scheme进行授权：
                zkClient = new ZooKeeper(SERVER_LIST,3000,new Sample_AuthFailed1());
                zkClient.addAuthInfo(“”,””.getBytes());
        回调方法process()
                process()方法是Watcher接口中的一个回调方法，当ZooKeeper向客户端发送一个Watcher事件通知时，客户端就会对相应的process方法进行回调，从而实现对事件的处理。process方法定义如下：
                abstract public void process(WatchedEvent event);
        WatchedEvent包含了每一个事件的三个基本的属性：通知状态(KeeperState)、事件类型(eventType) 和 节点路径(path),ZooKeeper使用WatchedEvent对象来封装服务端事件并传递给Watcher，从而方便回调方法process对服务器事件进行处理
                WatchedEvent和WatcherEvent，笼统讲，俩者表示是同一个事务，都是对一个服务端事件的封装。不同的是，WatchedEvent是一个逻辑事件，用于服务端和客户端程序执行过程中所需的逻辑对象，而WatcherEvent因为实现了序列化接口，因此可以用于网络传输，其数据结构为7-6:
                服务端在生成WatchedEvent事件之后，会调用getWrapper方法将自己包装成一个可序列化的WatcherEvent事件，以便通过网络传输到客户端。客户端在接收到服务端的这个事件对象后，首先将WatcherEvent事件还原成一个WatchedEvent事件，并传递给process方法处理，回调方法proccess根据入参就能够解析出完整的服务端事件。无论是WatchedEvent还是WatcherEvent，其对ZooKeeper服务端事件的封装都是极其简单的。当/zk-book这个节点的数据发生变更时，服务端会发送给客户端一个”ZNode数据内容变更”事件，客户端只能够接收到如下信息：
                KeeperState:SyncConnected
                EventType:NodeDataChanged
                Path:/zk-book

### 三、工作机制：
     ZooKeeper的Watcher机制，总的来说可以概括一下三个过程：客户端注册Watcher、服务端处理Watcher和客户端回调Watcher，其内部各组件之间关系：
    客户端注册Watcher
        创建一个ZooKeeper客户端对象实例时，可以向构造方法中传入一个默认的Watcher:
            public ZooKeeper(String connectString ,int sessionTimeout,Watcher watcher);
        Watcher将作为整个ZK会话期间的默认Watcher，会一直被保存在客户端ZKWatchManager的defaultWatcher中。另外ZooKeeper客户端也可以通过getData、getChildren和exist三个接口来向ZK服务器注册Watcher，无论使用哪种方式，注册Watcher的工作原理都是一致的，如：
            public byte[] getData(String path,boolean watch,Stat stat)
             public byte[] getData(final String path,Watcher watcher,Stat)
        在这俩个接口上都可以进行Watcher的注册，第一个接口通过一个boolean参数来标识是否使用上文中提到的默认Watcher来进行注册，具体的注册逻辑和第二个接口是一致的。
        在向getData接口注册Watcher后，客户端首先会对当前客户端请求request进行标记，将其设置为使用Watcher监听，同时会封装一个Watcher的注册信息WatchRegistration对象，用于暂时保存数据节点的路径和Watcher的对应关系，具体的逻辑：
            public Stat getData(final String path,Watcher watcher,Stat stat){
                         WatchRegistration web = null;
                         if(watcher != null ){
                              web = new DataWatchRegistration(watcher,clientPath);
                         }
                         request.setWatch(watcher != null);
                    }
        在ZooKeeper中，【Packet】可以被看作一个最小的通信协议单元，用于进行客户端与服务端之间的网络传输，任何需要传输的对象都需要包装成一个Packet对象。因此，在ClientCnxn中WatchRegistration又被封装到Packet中去，然后放入发送队列中等待客户端发送：
### 四、ACL-保障数据的安全
     ZooKeeper作为一个分布式协调框架，其内部存储的都是一些关乎分布式系统运行时状态的元数据，尤其是一些涉及分布式锁、Master选举和分布式协调等应用场景的数据，会直接影响基于ZooKeeper进行构建的分布式系统的运行状态。因此如何有效的保障ZooKeeper中数据的安全，从而避免因误操作而带来的数据随意变更导致的分布式系统异常就很重要。ZooKeeper提供了一套完整的ACL(Access Control List)权限控制机制来保障数据的安全。
     1.ACL介绍：
        ZooKeeper的ACL权限控制和Unix/Linux操作系统中的ACL有一些区别，读者可以从三个方面来理解ACL机制，分别是：权限模式(Scheme)、授权对象(ID)和权限(Permission),通常使用”scheme:id:permission”来标识一个有效的ACL信息。
     权限模式:Scheme
          权限模式用来确定权限验证过程中使用的校验策略。在ZooKeeper中，开发人员使用最多的就是以下四种权限模式：
          IP：
               IP模式通过IP地址粒度进行权限控制，例如配置了”ip:192.168.0.110”,即表示权限控制都是针对这个IP地址的。同时，IP模式支持按照网段的方式进行配置，例如：”ip:192.168.0.1/24”，表示针对192.168.0.*这个IP段进行权限控制、
          Digest：
               Digest是最常见的权限控制模式，也更符合我们对于权限控制的认识，其以类似于”username:password”形式的权限标识来进行权限配置，便于区分不同应用来进行权限控制。
               当我们通过”username:password”形式配置了权限标识后，ZK会对其先后进行俩次编码处理，分别是SHA-1算法加密和BASE64编码，其具体实现由：
               DigestAuthenticationProvider.generateDisgest(String idPassword)函数进行封装。
          World
               World是一种开放的权限控制模式，从起名字中也可以看出，事实上这种权限控制方式几乎没有任何作用，数据节点的访问权限对所有用户开发，即所有用户都可以在不进行任何权限校验的情况下操作ZooKeeper上的数据。另外，World模式也可以看做是一种特殊的Digest模式，它只有一个权限标识，即”world:anyone”.
          Super
               Super模式，故名思意，就是超级用户。也是一种特殊的Digest模式。在Super模式下，超级用户可以对任意ZK上的数据节点进行任何操作
       2.授权对象：ID
          授权对象指的是权限赋予的用户或一个执行实体，例如IP地址或是机器等。在不同的权限模式下，权限对象不同的，下标列出各个权限模式和授权对象之间的对应关系：
    权限模式
        授权对象
    IP
        通常是一个IP地址或是IP段，例如”192.168.0.110”或”192.168.0.1/24"
    Digest
        自定义，通常是username:BASE64(SHA-1(username:password))”
    World
        只有一个ID:”anyone"
    Super
        与Digest模式一致
    权限：Permission
        权限就是指那些通过权限检查后可以被允许执行的操作。在ZK中，所有对数据的操作权限分为以下五大类：
        CREATE(C):数据节点的创建权限，允许授权对象在该数据节点下创建子节点
        DELETE(D):子节点的删除权限，允许授权对象删除该数据节点的子节点
        READ(R):数据节点的读取权限，允许授权对象访问该数据节点并读取其数据内容或子节点列表等。
        WRITE(W):数据节点的更新权限，允许授权对象对该数据节点进行更新操作
        ADMIN(A):数据节点的管理权限，允许授权对象对该数据节点进行ACL相关设置操作

### 五、权限扩展体系：
        ZK默认提供的IP、Digest、World和Super这四种权限模式，在绝大部分场景下，这四种权限模式已经能够很好地实现权限控制的目的。同时，ZK提供了特殊的权限控制插件体系，允许开发人员通过指定方式对ZK的权限进行扩展。这些扩展的权限控制方式就插件一样插入到ZK的权限体系中去。在ZK的官方文档中，也称为该机制为”Pluggable ZooKeeper Authentication”。
        注册自定义权限控制器：
            完成自定义权限控制器的开发后，接下去就需要将该权限控制器注册到ZK服务器中去了。ZK支持通过系统属性和配置文件俩种方式注册自定义的权限控制器。
            系统属性-DZooKeeper.authProvider.X
            在ZooKeeper启动参数中配置类似如下的系统属性：
            -Dzookeeper.authProvider.1 = com.zkbook.CustomAuthenticationProvider
        配置文件方式
            在zoo.cfg配置文件中配置类似于如下的配置项：
               authProvider.1 =com.zkbook.CustomAuthenticationProvider
        对于权限控制器的注册，ZK才有管理延迟加载的策略，即只有在第一次处理包含权限控制的客户端请求时，才会进行权限控制器的初始化。同时，ZK还会降所有的权限控制器都注册到ProviderRegistry中区。在具体的实现，ZK首先将DigestAuthenticationProvider和IPAuthenticationProvider这俩个默认的控制器初始化，然后通过扫描zookeeper.authProvider.这一系统属性，获取到所有用户配置的自定义权限控制器，并完成其初始化。

### 六、ACL管理
        设置ACL
            通过zkCli脚本登陆ZK服务器后，可以通过俩种方式进行ACL的设置。一种是在数据节点创建的同时进行ACL权限的设置，命名格式如下：
          create [-s] [-e] path data ACL

### 七、序列化与协议：
        ZooKeeper的客户端和服务端之间会进行一系列化的网络通信以实现数据的传输。对于一个网络通信，首先需要解决的就是对数据的序列化和反序列化处理，在ZooKeeper中，使用了Jute这一序列化组件来进行数据的序列化和反序列化操作。同时，为了实现一个高效的网络通信程序，良好的通信协议涉及也就是很关键。ZK的序列化组件Jute以及通信协议的设计原理来讲解ZK在网络通信底层的一些设计内容。
     1.Jute介绍
          Jute是ZK中的序列化组件，最初也是Hadoop中的默认序列化组件，其前身是Hadoop Record IO中的序列化组件，后来由于Apache Avro具有出众的跨语言特性、丰富的数据结构和对MapReduce的天生支持，并且能够非常方便地用于RPC调用，从而在Hadoop0.21.0版本开始Avro这个序列话框架。
          Record接口：定义了自己独特的序列化格式Record、ZooKeeper中所有需要进行网络传输或是本地磁盘存储的类型定义，都实现了该接口。分别是:serialize和deseriablize，分别用于序列化和反序列化
          OutputArchive和InputArchive:分别是Jute底层的序列化器和反序列化器接口定义。在最新版的Jute中，分别有BinaryOutputArchive/BinaryInputArchive,CsvOutputArchive/CsvInputArchive和XmlOutputArchive/XmlInputArchive三种实现。
        无论哪种实现，都是基于OutputStream和InputStream进行操作。
        BinaryOutputArchive对数据对象的序列化和反序列化，主要用于进行网络传输和本地磁盘的存储，是ZooKeeper底层最主要的序列化方式。
     2.通信协议
              基于TCP/IP协议，ZooKeeper实现了自己通信协议来完成客户端与服务器、服务器与服务器之间的网络通信。ZooKeeper通信协议整体上的设计非常简单，对于请求、主要包括请求头和请求体，而对于响应，则主要包含响应头和响应体.
          请求头：RequestHeader.java包含了请求最基本的信息，包含xid和type；xid用于记录客户端请求发起的先后序号，用于确保单个客户端请求的响应顺序。type代表请求的操作类型。常见的包括创建节点(OpCode.create:1)、删除节点(OpCode.create:2)和获取节点数据(OpCode.getData:4)等，所有这些操作类型都是定义在类org.apache.zookeeper.ZooDefs.OpCode中。根据协议规定，除非是”会话创建”请求，其他所有的客户端请求都会带上请求头。
          请求体：Request.java，协议的请求题部分是指请求的主题内容部分，包含了请求的所有操作内容。不同的请求类型，其请求体部分的结构是不同的，下面以会话创建、获取节点数据和更新节点这三个经典的哥请求体进行详解：
                         ConnectRequest:创建会话，ZK客户端和服务端在创建会话的时候，会发送ConnectRequest请求，该请求体重包含了协议的版本号protocolVersion,最近一次接收到的服务器ZXID lastZxidSeen,会话超时TimeOut、会话标识sessionId和会话密码passed,其数据结构定义如下：
                        GetDataRequest:获取节点数据，客户端在向服务器发送获取节点数据请求的时候，会发出GetDataRequest请求，该请求体中包含了数据节点路径path和是否注册watcher的标识watch。
     请求协议实例：获取节点数据
     协议解析：响应部分
          响应头：ReplyHeader.java
                       响应体：Response
               协议的响应体部分是指响应的主体内容部分，包含了相应的所有返回数据。不同的响应类型，其响应体部分的结构是不同的。
               ConnectResponse:创建会话
               GetDataResponse：获取节点数据，针对客户端的获取节点请求，服务端会返回客户端一个GetDataResponse响应，该响应体体中包含了及诶单的数据内容data和节点状态stat,其数据结构定义如下：
               SetDataResponse：更新节点数据
               针对客户端的更新节点数据请求，服务端会返回客户端一个SetDataResponse响应，该响应体中包含了最新的节点状态stat。
           3.客户端
                客户端是开发人员使用ZooKeeper最主要的途径，必须对ZooKeeper客户端的内部原理进行详细讲解。ZooKeeper的客户端主要由下面的核心组件组成。
                1.ZooKeeper实例：客户端的入口
                2.ClientWatchManager:客户端Watcher管理器
                3.HostProvider：客户端地址列表管理
                4.ClientCnxn：客户端核心线程，其内部又包含俩个线程，即SendThread和EventThread。前者是IO线程，主要负责ZooKeeper客户端和服务端之间的网络IO通信；后者是一个事件线程，主要负责对服务端事件进行处理。
                
        ZooKeeper客户端的初始化与启动环节，实际上就是ZooKeeper对象的实例化过程，因此我们首先看看ZK客户单的构造方法：
            ZooKeeper(String connectString ,int sessionTimeout,Watcher watcher);
            ZooKeeper(String connectString ,int sessionTimeout,Watcher watcher,boolean canBeReadOnly);
            ZooKeeper(String connectString ,int sessionTimeout,Watcher watcher,long sessionId,byte[] sessionPasswd);
            ZooKeeper(String connectString ,int sessionTimeout,Watcher watcher,long sessionId,byte[] sessionPasswd,boolean canBeReadOnly);
             1.设置默认Watcher
             2.设置ZooKeeper服务器地址列表     
             3.创建ClientCnxn
        如果在ZooKeeper的构造方法中传入一个Watcher对象的话，那么Zookeeper就会将这个Watcher对象保存在ZKWatchManager的defaultWatcher中，作为整个客户端会话期间的默认Watcher。
    
    会话的创建过程：
        1.初始化ZooKeeper对象
            通过调用Zookeeper的构造方法来实例化一个ZooKeeper对象，在初始化过程中，会创建一个客户端的Watcher管理器：ClientWatchManager.
        2.设置会话默认Watcher
            如果在构造方法中传入了一个Watcher对象，那么客户端会将这个对象作为默认Watcher保存在ClientWatchManager中.
     3.构造ZooKeeper服务器地址列表管理器：HostProvider.
            对于构造方法中传入的服务器地址、客户端会将其存放在服务器地址列表管理器HostProvider中.
     4.创建并初始化客户端网络连接器：ClientCnxn。
            ZooKeeper客户端首先会创建一个网络连接器ClientCnxn,用来管理客户端与服务器的网络交互。另外，客户端在创建ClientCnxn的同时，还会初始化客户端俩个核心的队列outgoingQueue和pendingQueue,分别作为客户端的请求发送队列和服务端响应的等待队列。ClientCnxn连接器的底层IO处理器时ClientCnxnSocket，因此在这一步中，客户端还会同时创建ClientCnxnSocket处理器.
     5.初始化SendThread和EventThread
            客户端创建俩个核心网络线程SendThread和EventThread，前者用于管理客户端和服务端之间的所有网络IO，后者则用于进行客户端的事件处理。同时，客户端还会将ClientCnxnSocket分配给SendThread作为底层网路IO处理器，并初始化EventThread的等待处理事件队列waitingEvents，用于存放所有等待被客户端处理的事件.
     6.启动SendThread和EventThread
          SendThread首先会判断当前客户端的状态，进行一系列清理性工作，为客户端发送”会话创建”请求做准备。
     7.获取一个服务器地址
          在开始创建TCP连接之前，SendThread首先需要获取一个ZooKeeper服务器的目标地址，这通常是从HostProvider中随机获取出一个地址，然后委托给ClientCnxnSocket去创建与ZooKeeper服务器之间的TCP链接。
     8.创建TCP连接
          获取到一个服务器地址后，ClientCnxnSocket负责和服务器创建一个TCP长连接。
     9.构造ConnectRequest请求
          在TCP连接创建完毕后，可能有的读者会认为，这样是否就说明已经和ZooKeeper服务端完成连接了呢？步奏8只是纯粹地从网络TCP层面完成了客户端与服务端之间的Socket连接，但远远未完成ZooKeeper客户端的会话创建。
          SendThread会负责根据当前客户端的实际设置，构造出一个ConnectRequest请求，该请求代表了客户端试图与服务器创建一个会话。同时，ZooKeeper客户端还会进一步将该请求包装成网络IO层的Packet对象，放入请求发送队列outgoingQueue中去。
     10.发送请求
          当客户端请求准备完毕后，就可以开始向服务端发送请求了。ClientCnxnSocket负责从outgoingQueue中取出一个待发送的Packet对象，将其序列化成ByteBuffer后，向服务端进行发送。
          
     响应处理阶段：
     11.接收服务端响应。
          ClientCnxnSocket 接收到服务端的响应后，会首先判断当前的客户端状态是否是”已初始化”，如果尙未完成初始化，那么就认为响应一定是会话创建请求的响应，直接交由readConnectResult方法来处理响应。
     12.处理Response
          ClientCnxnSocket 会接收到的服务端响应进行反序列化，得到ConnectResponse对象，并从中获取到ZooKeeper服务端分配的会话sessionId.
     13.连接成功
          连接成功后，一方面需要通知SendThread线程，进一步对客户端进行会话参数的设置，包括readTimeout和conenctTimeOut等，并更新客户端状态；另一方面，需要通知地址管理器HostProvider当前成功连接的服务器地址。
     14.生成事件：SyncConnected-None
          为了能够让上层应用感知到会话的成功创建，SendThread会生成一个事件SyncConnected-None,代表客户端与服务器会话创建成功，并将该事件传递给EventThread线程。
     15.查询Watcher
          EventThread线程接收到事件后，会从ClientWatchManager管理器中查询出对应的Watcher，针对SyncConnected-None事件，那么就直接找出步骤2中存储的默认Watcher，然后将其放到EventThread的waitingEvents队列中去。
     16.处理事件     
          EventThread不断地从waitingEvents队列中取出待处理的Watcher对象，然后直接调用该对象的process接口方法，以达到触发Watcher的目的。
    ZooKeeper客户端完整的一次会话创建过程已经全部完成了。地址列表管理器、ClientCnxn和ClientCnxnSocket等这些ZooKeeper客户端的核心组件及其之间的关系。
    
    服务器地址列表：
        使用ZooKeeper构造方法时，用户传入的ZooKeeper服务地址列表，即connectString参数，通常是这样一个使用逗号隔开。192.168.0.1:2181,192.168.0.2:2181,192.168.0.3:2181,
    
    Chroot:
        客户端隔离命名空间该特性允许每个客户端为自己设置一个命名空间(namespace)。如果一个ZK客户端设置了Chroot，那么客户端对服务器的任何操作，都会被限制在其自己的命名空间下。
        客户端可以在connectString中添加后缀的方式来设置Chroot.192.168.0.1:2181,192.168.0.2:2181,192.168.0.3:2181/apps/X
    
    HostProvider:
        地址列表管理器.在ConnectStringParser解析器中会对服务器地址做一个简单的处理，并将服务器地址和相应的端口封装成一个InetSocketAddress对象，以ArrayList形式保存在ConnectStringParser.serverAddress属性中。经过处理地址列表会被进一步的封装到StaticHostProvider类中。HostProvider：
        
    StaticHostProvider解析服务器地址：
        针对ConnectStringParser.serverAddress集合中那些没有被解析的服务器地址，StaticHostProvider首先会对这些地址进行解析，然后再放入serverAddress集合中去。同时，使用Collections工具类的shuffle方法来将这个服务器地址列表进行随机的打散。
    
    获取可用的服务器地址：
        通过调用StaticHostProvider的next()方法，能够从StaticHostProvider中获取一个可用的服务器地址。这个next()方法并非常简单地从serverAddresses中依次获取一个服务器地址，而是先将随机打散后的服务器地址列表拼装成一个环形队列。
     StaticHostProvider只是ZK官方提供的对于地址列表管理器的默认实现方式，也是最通用和最简单的一种实现。如果有需求，可以满足HostProvider三要素的前提下，实现自己的服务器列表地址管理器。
     配置文件方式：
          在ZK默认的实现方式中，是通过在构造方法中传入服务器地址列表的方式来喜爱吸纳地址列表的设置。
     动态变更的地址列表管理器
          在ZK的使用过程中，我们会碰到这样的问题：ZK服务器集群的整体迁移或个别机器的变更，会导致大批客户端应用也跟着一起进行变更。实现动态变更的地址列表管理器，对于提升ZK客户端用户体验非常重要。最简单的实现方式就是实现这样一个HostProvider：地址列表管理器能够定时从DNS或一个配置中心上解析出来ZK服务器地址列表，如果这个列表变更了，那么就同时更新到serverAddresses集合中去，这样就在下一次需要获取服务器地址(next()方法)的时候，就自然使用了新的服务器配置地址，随着时间的推移，慢慢地就能够在保证客户端透明的情况下ZK服务及其的变更了。
     
     实现同机房优先策略
     4.ClientCnxn:网络IO
          ClientCnxn是ZK客户端的核心工作类，负责维护客户端与服务器之间的网络连接并进行一系列网络通信。ClientCnxn内部的工作原理：
     Packet:
          Packet是ClientCnxnCnxn内部定义的一个对协议层的封装，作为ZK中请求与响应的载体。Packet中包含了最基本的请求头(requestHeader)、响应头(replyHeader)、请求头(request)、响应体(response)、节点路径(clientPath/serverPath)和注册的Watcher(watchRegistration)等信息。在具体底层传输中，只会将requestHeader、request和readOnly三个属性进行序列化，其余属性保存在客户端的上下文中，不会进行与服务端之间的网络传输。
     outgoingQueue和pendingQueue
     ClientCnxn中，有俩个核心的队列outgoingQueue和pendingQueue,分别代表客户端的请求发送队列和服务端响应的等待队列。outgoing队列是一个请求发送队列，专门用于存储那些需要发送到服务端的Packet集合。Pending队列是为了存储那些已经从客户端发送到服务端的，单丝需要等待服务端响应的Packet集合。
     ClientCnxnSocket：底层Socket通信层
          ClientCnxnSocket定义了底层通信的接口。在ZooKeeper中，器默认的实现是ClientXnxSocketNIO.该实现是用的Java原生的NIO接口，核心是doIO逻辑，主要负责请求的发送和响应请求的接收的过程。
     请求发送：
          在正常请情况，会从outgoingQueue队列中提取出一个可发送的Packet对象，同时生成一个客户端请求序号XID并将其设置到Packet请求头中去，然后将其序列化后进行发送。这里提到了”获取一个可发送的Packet对象”，那么什么样的Packet是可发送的呢？在outgoingQueue队列中的Packet整体上按照先进先出的顺序被处理的，但是如果检测到客户端与服务器端之间正在处理SASL权限的话，那么那些不含请求头(requestHeader)的Packet是可以被发送的，其余的不能被发送。请求你发送完后，会立即将Packet保存到pendingQueue队列中，以便的等待服务端响应返回进行相应的处理。
     响应接收：
          客户端获取到来自服务端的完整响应数据后，根据不同的客户端请求类型，会进行不同的处理：
          1.如果检测到当前客户端还尚未进行初始化，那么说明当前客户端与服务端之间正在创建会话，那么就直接将接收到的ByteBuffer(incomingBuffer)序列化成ConnectResponse对象。
          2.如果当前客户端已经处于正常的会话周期，并且接收到的服务端响应是一个事件，那么ZK客户端会将接收到的ByteBuffer(incomingBuffer)序列化成WatcherEventThread对象，并将该事件放入等待处理队列中。
          3.如果是一个常规的请求响应(Create、GetDataNode和Exist等操作请求),那么会从pendingQueue队列中取出一个Packet来进行处理。ZK客户端首先通过检验服务端响应中包含的XID值来确保请求处理的顺序性，然后再将接收到的ByteBuffer(incomingBuffer)序列化成相应的Response对象。
    最后，会在finishPacket方法中处理Watcher注册等逻辑。
     SendThread：
          SendThread是客户端ClientCnxn内部一个核心的IO调度线程，用于管理客户端和服务端之间的所有网络IO操作。在ZK客户端的实际运行过程中，一方面，SendThread维护了客户端与服务端之间的会话生命周期，其通过在一定的周期频率内向服务端发送一个PING包来实现心跳检测。同时，在会话周期内，如果客户端与服务端之间出现TCP连接断开的情况，那么就会自动且透明化地完成重连操作。另一个方面，SendThread管理了客户端所有的请求发送和响应接收操作，其将上层客户端API的操作转化为响应的请求协议并发送到服务端，并完成对同步调度的返回和异步调度的回调。同时，SendThread还负责将来自服务端的事件传递给EventThread去处理。
     EventThread：
          EventThread是客户端ClientCnxn内部的另一个核心的线程，负责客户端的事件处理，并处罚客户端注册的Watcher监听。EventThread中有一个waitingEvents队列，用于临时存放那么被触发的O被揭穿他，包括那些客户端注册的Watcher和异步接口中注册的回调器AsyncCallback。同时，EventThread会不断地从waitingEvents这个接口中取出Object，识别出其具体类型(Watcher或AsyncCallback),并分别调用process和processResult接口方法来实现对事件的触发和回调。
          
    四、会话
     会话(Session)是ZK中重要概念之一，客户端与服务端之间的任何交互操作都与会话息息相关，这其中就包括临时节点的声明周期、客户端请求的顺序执行以及Watcher通知机制等。
     1.会话状态
          在ZK客户端与服务端成功完成连接创建后，就建立了一个会话。ZK会话在整个运行期间的声明周期中，会在不同的会话状态之间进行切换，这些状态一般可以分为CONNECTING、CONNECTED、RECONNECTING、RECONNECTED和CLOESE等。
        在ZK运行期间，客户端的状态总是介于CONNECTING和CONNECTED俩者之间。另外，如果出现会话超时、权限检查失败或是客户端主动退出程序等情况，那么客户端的状态机会会直接变为CLOSE。
     2.会话创建
          会话创建过程中ZK服务端的工作原理：
     Session是ZK中的会话实体，代表了一个客户端会话。其中包含四个基本属性。
     sessionID:会话ID，用来唯一标识一个会话，每次客户端创建新会话的时候，ZK都会为其分配一个全局唯一的sessionID。
     TimeOut:会话超时时间，客户端在构造ZK的实例的时候，会分配一个sessionTimeOut参数用于指定会话的超时时间。ZK客户端向服务器发送这个超时时间后，服务器会根据自己的超时时间限制最终确定会话的超时时间。
     TickTime:下次会话超时时间点。
     isClosing：该属性用于标记一个会话是否已经被关闭。通常当服务端检测到一个会话已超时失效的时候，会将该会话的isClosing属性标记为”已关闭”，这样能确保不再处理来自会话的新请求了。
     SessionID：
          由于SessionID必须是全局唯一的。生成的策略是什么？在SessionTracker初始化的时候，会调用initalizeNextSession方法来生成初始化sessionID,之后在ZK的正常运行过程中，会在该SessionIDE的基础上为每个会话进行分配，其初始化算法如下：
     long nextSid = 0;
     nextSid = (System.currentTimeMillis()<<24)>>8;
     nextSid = nextSid | (id << 56)
     return nextSid;
    SessionTracker:
            SessionTracker是ZK服务端的会话管理器，负责会话的创建、管理和清理的工作，可以说，整个会话的生命周期离不开SessionTracker的管理。每个会话在SessionTracker内部都保留了三分，具体：
     sessionsById:这是一个HashMap<Long,SessionImpl>类型的数据结构，用于根据sessionID来管理Session实体。
     sessionWithTimeOut:这是一个ConcurrentHashMap<Long,Integer>类型的数据结构，用于根据sessionID来管理会话的超时时间。该数据结构和ZK内存数据库相连通，会被定期持久化到快照文件中去。     
     sessionSets:这是一个HasMap<Long,SessionSet>类型的数据结构，用于根据下次会话超时时间点来归档会话，便于进行会话管理和超时检查。在下文”分桶策略”会话管理的介绍中，我们还会对该数据结构进行详细讲解。
     创建连接
          服务端对于客户端的”会话创建”请求的处理，大体分为四大步骤，分别是处理ConnectRequest请求，会话创建、处理器链路处理和会话响应。在ZK服务端，首先将会由NIOServerCnxn来负责接收来自客户端的”会话创建”请求，并发序列化出connectRequest请求，然后根据ZK服务端的配置完成会话超时时间的协商。随后，SessionTracker将会为会话分配一个sessionID,并将其注册到sessionsById和sessionsWithTimeout中去，同时进行会话的激活。之后，该”会话请求”还会在ZK服务端的各个请求处理器之间进行顺序流转，最终完成会话的创建。
     会话管理
          ZK的服务端是如何管理这些会话的？
          分桶策略：ZK的会话管理主要是由SessionTracker负责的，其采用了一种特殊的会话管理方式，我们称为分桶策略。所谓的分桶策略，是指将类似的会话放在同一区块中进行管理，以便于ZK对会话进行不同区块的隔离处理以及同一区块的统一处理。
    
    五、服务端启动
          单击版服务器启动：
               ZK服务器启动，大体可以分为以下五个不要步骤：配置文件解析、初始化数据管理器、初始化网络IO管理器、数据恢复和对外服务。
          预启动：
               预启动的步骤如下：
               1.统一由QuorumPeerMain作为启动类：无论是单击版还是集群模式启动ZK服务器，在zkServer.cmd和zkServer.sh俩个脚本中，都配置了使用了org.apache.zookeeper.server.quorum.QuorumPeerMain作为启动入口类。
               2.解析配置文件zoo.cfg
                    ZK首先会进行配置文件的解析，配置文件的解析其实就是对zoo.cfg文件解析。
               3.创建并启动历史文件清理器DatadirCleanupManager:自动清理历史数据文件的机制，包括对事物日志和快照数据文件进行定时清理。
               4.判断当前集群模式还是单机模式的启动
                         ZK根据步奏2 中解析出来的集群服务地址列表来判定当前是集群模式还是单击模式，如果是单击模式，那么就委托给ZooKeeperServerMain进行启动处理
               5.再次进行配置文件zoo.cfg的解析
               6.创建服务器实例ZooKeeperServer.ZK服务器首先会进行服务器实例的创建，接着下去的步奏则都是对该服务器实例的初始化工作，包括连接器、内存数据库和请求处理器等组件的初始化
       初始化
               1.创建服务器统计器ServerStats:     ServerStats是ZK服务器运行时的统计器，包含了基本的运行时信息，
               2.创建ZK数据管理器FileTxnSnapLog
                    FileTxnSnapLog是ZK上层服务器和底层数据存储之间的对接层，提供了一系列操作数据文件的接口，包括事务日志文件和快照数据文件。ZK根据zoo.cfg文件中解析出的快照数据目录dataDir和事务日志目录dataLogDir来创建FileTxnSnapLog
               3.设置服务器tickTime和会话超时时间限制
               4.创建ServerCnxnFactory:ZK都是自己实现NIO框架，3.4.0版本开始，引入了Netty。读者可以通过配置系统属性zooKeeper.serverCnxFactory来指定使用ZK自己实现的NIO还是Netty框架作为ZK服务端网络工厂。
               5.初始化ServerCnxnFactoryBean
                    ZK首先会初始化一个Thread，作为整个ServerCnxnFactory的主线程，然后再初始化NIO服务器
               6.启动ServerCnxnFactory主线程
                    主线程ServerCnxnFactory的主逻辑(run方法)。ZK的NIO服务器已经对外开放端口，客户端能够访问ZK的客户端的端口2181，但是此时ZK服务器是无法正常处理客户端请求。
               7.恢复本地数据     
                    每次在ZK启动的时候，都需要从本地快照数据文件和事务日志文件中进行数据恢复。ZK的本地数据恢复比较复杂。
               8.创建并启动会话管理器
                    在ZK启动阶段，会创建一个会话管理器SessionTracker。它负责ZK服务端的会话管理。创建SessionTracker的时候，会初始化expirationInterval,nextExpirationTime和sessionWithTimeout，同时还会计算一个初始化的sessionID.
               9.初始化ZK的请求处理链
                    ZK的请求处理方式是典型的责任链模式的实现，在ZK服务器上，会有多个请求处理器一次来处理一个客户端请求。在服务端启动的启动的时候，会将这些请求处理器串联起来形成一个请求处理链。
               10.注册JMX服务：ZK会将服务器运行时的一些信息以JMX的方式暴露给外部，关于ZK的JMX
               11.注册ZK服务器实例
   
### 八、集群版本ZK服务器启动流程：
    Leader选举
        Leader选举的步骤如下：
            1.初始化Leader选举：然后ZK会根据zoo.cfg中的配置，创建相应的Leader选举算法实现。在ZK中，默认提供了三种Leader选举算法的实现，分别是LeaderElection、AuthFastLeaderElection和FastLeaderElection，可以通过在配置文件zoo.cfg中使用electionAlg属性来指定，分别使用数字0~3来表示。目前3.4.0版本只支持FastLeaderElection选举算法
        Leader和Follower启动期交互过程
            ZK已经完成了Leader选举，并且集群中每个服务器都已经确定了自己的角色-通常情况下分为Leader和Follower俩种角色。
            1.创建Leader服务器和Follower服务器：完成Leader选举之后，每个服务器都会根据自己服务器角色创建相应的服务器实例，并开始进入各自角色的主流程。
            2.Leader服务器启动Follower接收器LeaderCnxAcceptor：LearnerCnxAcceptor接收器用来负责接收所有非Leader服务器的连接请求
            3.Leader服务器开始和Leader建立连接。
            4.Leader服务器创建LearnerHandler:Leader接收到来自其它机器的连接创建请求后，会创建一个LearnerHandlers实例。每个LearnerHandlers实例都对应了一个Leader与Learner服务器之间的链接，其负责Leader和Learner服务器之间几乎所有的消息通信和数据同步。
            5.向Leader注册
               当和Leader建立起连接后，Learner就会开始向Leader进行注册——— 所谓的注册，其实就是将Learner服务器自己的基本信息发送给Leader服务器，LearnerInfo，包括当前服务器的SID和服务器处理的最新的ZXID。
            6.Leader解析Learner信息，计算新的epoch.     Leader服务器在接收到Learner的基本信息后，会解析出该Learner的SID和ZXID，然后根据该Learner的ZXID解析出其对应的epoch_of_learner,和当前Leader服务器的epoch_of_leader进行比较，如果该Learner的epoch_of_learner更大的话，那么就更新Leader的epoch:
     epoch_of_leader = epoch_of_learner + 1
          7.发送Leader状态：计算出新的epoch之后，Leader会将该信息以一个LEADERINFO消息的形式发送给Learner，同时等待Learner的响应
          8.Learner发送ACK消息：Follower在收到来自Leader的LEADERINFO消息后，会解析出epoch和ZXID，然后向Leader反馈一个ACKEPOCH响应。
          9.数据同步:Leader服务器接收到Learner的这个ACK消息后，就可以开始与其进行数据同步了。
          10.启动Leader和Learner服务器:当有过半的Learner已经完成了数据同步，那么Leader和Learner服务器实例就可以开始启动了。
          
    Leader选举：
        ZK集群中的三种服务器角色：Leader、Follower和Observer。Leader选举概述、算法分析和实现细节三个方面看看ZK是如何进行Leader选举的
        Leader选举概述：
          Leader选举是ZK这种最重要的技术之一，也是保证分布式数据一致性的关键所在。
    术语：
        SID：服务器ID     用来标识一台ZK集群中的机器，每台机器不能重复，和myid的值一致
        ZXID：事务ID      是一个事务ID，用来唯一标识一次服务器状态的变更。
     Vote：投票
     Quorum:过半机器数
 
    算法分析：
         进入Leader选举
               1.服务器初始化启动             
               2.服务器运行期间无法和Leader保持连接,集群中本来就已经存在一个Leader,集群中确实不存在Leader