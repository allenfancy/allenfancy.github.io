---
layout: post
title: "ZooKeeper集群"
date: 2017-05-02
description: "ZooKeeper集群"
tag: zookeeper
---   


### 一、Leader和Follower启动
     1.创建并启动会话管理器
     2.初始化ZK的请求处理链
     3.注册JMX服务
     
### 二、Leader选举
     SID：服务器ID，用来唯一标识一台ZK集群中的机器，每台机器不能重复和myid一致。
     ZXID：事务ID，用来唯一标识一次服务器状态的变更。在某一个时刻，集群中每台集群的ZXID值不一定全都一致。，这和ZK服务端对于客户端更新请求的处理逻辑相关
     Vote：投票
     Quorum：过半机器数，Quorum = (n/2+1)
     进入Leader选举：
          当ZK集群中的一台服务器出现以下俩种情况之一时，就会开始进入leader选举。
               1.服务器初始化启动
               2.服务器运行期间无法和Leader保持连接
     变更投票：集群中的每台机器发出自己的投票后，也会接收到来自集群中其他机器的投票。每台机器都会根据一定的规则，来处理收到的其他机器的投票，并此来决定是否需要变更自己的投票。这个规则也称为了整个Leader选举算法的核心所在。术语：
          vote_sid:接收到的投票中所推举Leader服务器的SID
          vote_zxid:接收到的投票中所推举Leader服务器的ZXID
          self_sid:当前服务器自己的SID
          self_zxid:当前服务器自己的ZXID
     规则1：如果vote_zxid大于self_zxid，那么就认可当前接收到的投票，并再次将该投票发送出去。
     规则2：如果vote_zxid小于self_zxid，那么就坚持自己的投票，不做任何变更
     规则3：如果vote_zxid等于self_zxid，那么就对比俩者的SID。如果Vote_sid大于self_sid,那么就认可当期接收到的投票，并再次将该投票发送出去
     规则4：如果vote_zxid等于self_zxid,并且vote_sid小于self_sid,那么同样坚持自己的投票，不做变更
     
### 三、Leader选举的实现细节
     FastLeaderElection算法的设计不复杂，但在正在的实现过程中，对于一个需要应用在生产环境的产品来说，需要解决实际问题。
     服务器状态：
          为了能够情况的对ZK集群中每台机器的状态进行标识，在org.apache.zookeeper.server.quorum.QuorumPeer.ServerState类中列举了4种服务器状态分别为：LOOKING、FOLLOWING、LEADING和OBSERVING
     ZK设计了一种建立TCP连接的规则：只允许SID大的服务器主动和其他服务器建立连接，否则断开连接。

### 四、选举的核心
    Starts a new round of leader election.
        开始Leader选举
        Whenever our QuorumPeer
        changes its state to LOOKING, this method is invoked, and it
        sends notifications to all other peers.
     
        1.自增选举轮次
            logiccalClock属性，用于标识当前Leader的选举轮次，ZK规定了所有有效的投票都必须在同一轮次中。ZK开始操作新一轮的投票时，会首先对logicalclock进行自增属性
        2.初始化选票
            在开始进行新一轮的投票之前，每个服务器都会首先初始化自己的选票。初始化选票也就是对Vote属性的初始化。在初始化阶段，每台服务器都会将自己推荐为Leader
        3.发送初始化选票
            在完成选票的初始化后，服务器会发起第一次投票。ZK会将初始化好的选票放入sendQueue队列中，由发送器WorkerSend负则发送出去。
        4.接收外部投票
            每台服务器都会不断地从recvqueue队列中获取外部投票。如果服务器发现无法获取到任何的外部投票，那么就会立即确认自己是否和集群中其他服务器保持着有效连接。如果发现没有建立连接，那么就会马上建立连接,如果已经建立了连接，那么就再次发送自己当前的内部投票
        5.判断选举轮次
            当发送完初始化选票之后，接下来就要开始初始化外部投票了。在处理外部投票的时候，会根据选举轮次进行不同的处理外部投票的选举轮次大于内部投票
            如果服务器发现自己的选举轮次已经落后于该外部投票对应服务器的选举轮次，那么就会立即更新自己的选举轮次(logicalclock),并且情况所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内部投票
            最终再将内部投票发送出去
            外部投票的选举轮次小于内部投票
            如果接收到的选票的选举轮次落后于服务器自身，那么ZK就会直接忽略外部投票，不做任何处理，回复到第四步
            外部投票的选举轮次等于内部投票
            开始进行选票PK
        6.选票PK
            totalOrderPredicate方法的核心逻辑。选票PK的目的是为了确定当前服务器是否需要变更投票，主要从选举轮次、ZXID和SID三个因素来考虑，具体条件如下：在选票PK的时候依次判断，符合任意一个条件就需要进行投票变更
            如果外部投票中被推举的Leader服务器的选举轮次大于内部投票，那么就进行投票变更
            如果选举轮次一致的话，那么就对比俩者的ZXID。如果外部投票的ZXID大于内部投票，那么就需进行投票变更
            如果俩者的ZXID一致，那么就比较俩者的SID。如果外部投票的SID大于内部投票，那么就需要进行投票变更
     7.变更投票
        通过选票PK后，如果确定了外部投票由于内部投票，那么就进行投票变更
     8.选票归档
        无论是否进行了投票变更，都会将刚刚收到的那份外部投票放入recvset中进行归档。recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票，按照服务器对应的SID来分区
     9.统计投票
     10.更新服务器状态
     
### 五、服务器角色介绍
     ZK集群中，分别有Leader、Follower和Observer三种类型的服务器角色：
     1.Leader服务器是整个ZK集群工作机制中的核心，其主要工作有：
          .事务请求的唯一调度和处理者，保证集群事务处理的顺序性
          .集群内部各服务器的调度者
     2.请求处理链
       使用责任链模式来处理每一个客户端请求是ZK的一大特色。Leader服务器的请求处理链如图：
    Leader服务器请求处理链：
     从PrepRequestProcessor到FinalRequestProcessor，前后一个7个请求处理器组成了Leader服务器的请求处理链。
     1.PreRequestProcessor
          PreRequestProcessor是Leader服务器的请求预处理器，也是Leader服务器的第一个请求处理器。在ZK中，那些会改变服务器状态的请求称为事务请求———通常指的是创建节点，更新数据，删除及诶单以及创建会话等请求，PrepRequestProcessor能识别出当前客户端请求是否是事务请求，对于事务请求，PrepRequestProcessor处理器会对其进行一系列处理，诸如创建请求事务头，事务体，会话检查、ACL检查和版本检查等
     2.ProposalRequestProcessor
          ProposalRequestProcessor处理Leader服务器的事务投票处理器，也就是Leader服务器事务处理流程的发起者。对于非事务请求，ProposalRequestProcessor会直接将请求提交给CommitProcessor处理器，还会根据请求类型创建对应的Proposal提议，并发给所有的Follower服务器来发起一次集群内的事务投票。同时，ProposalRequestProcessor还会将事务请求交付给SyncRequestprocessor进行事务日志记录
     3.SyncRequestprocessor
          SyncRequestprocessor是事务日志记录处理器，该处理器主要用于将事务请求记录到事务日志文件中去，同时还会触发ZK进行数据快照。
     4.AckRequestProcessor
          AckRequestProcessor处理器是Leader特有的处理器，其主要负责在SyncRequestProcessor处理器完成事务日志记录后，向Proposal的投票收集器发送ACK反馈，已通知拓扑收集器当前服务器已经完成了对该Proposal的事务日志记录
     5.CommitProcessor
          CommitProcessor是事务提交处理器。对于非事务请求，该处理器会直接将其交付给下一级处理器进行处理；而对于事务请求，CommitProcessor处理器会等待集群内针对Proposal的投票直到该Proposal可被提交。利用CommitProcessor处理器，每个服务器都可以很好的控制对事务请求的顺序处理
     6.ToBeCommitProcessor
          ToBeCommitProcessor是一个比较特别的处理器，根据其命名规则，ToBeCommitProcessor处理器中有一个toBeApplied队列，专门用来存储那些已经被CommitProcessor处理过的可被提交的Proposal。ToBeCommitProcessor处理器将这些请求逐个的交付给FinalRequestProcessor处理器进行处理---等待FinalRequestProcessor处理器处理完之后，再将其从toBeApplied队列中移除
     7.FinalRequestProcessor
          FinalRequestProcessor是最后一个请求处理器。该处理器主要用来进行客户端请求返回之前的收尾工作，包括创建客户端请求的响应；针对事务请求，该处理器还会负责将事务应用到内存数据库中去。
    LearnerHandlers
     为了保持整个集群内部的实时通信，同时也使为了确保可以控制所有的Follower/Observer服务器，Leader服务器会与每一个Follower、Observer服务器都建立一个TCP长连接，同时也会为每个FollowerObserver服务器都创建一个名为LearnerHandler的实体。LearnerHandler是ZK集群中Learner服务器的管理器，主要负责Follower Observer服务器和Leader服务器之间的一些类网络通信，包括数据同步、请求转发和Proposal提议的投票等。Leader服务器中保存了所有的Follower Observer对应的LearnerHandler

### 六、Follower
     Follower服务器是ZooKeeper集群状态的跟随者，其主要的工作有一下三个：
          .处理客户端非事务请求，转发事务请求给Leader服务器
          .参与事务请求Proposal的投票
          .参与Leader选举投票
    和Leader服务器一样，Follower也同样采用了采用责任链模式组装请求处理链来处理每一个客户端请求，由于不需要负责事务请求的投票处理
     FollowerRequestProcessor ——————>  CommitProcessor ——————> FinalProcessor
     Leader Server      ——————> SyncRequestProcessor  ———————> SendAckRequestProcessor
    与Leader服务器的请求处理链最大的不同点在于，Follower服务器的第一个处理器转成FollowerRequestProcessor处理器，同时由于需要处理事务请求的投票，因此也没有了ProposalRequestProcessor处理器。
     1.FollowerRequestProcessor
          FollowerRequestProcessor是Follower服务器的第一个请求处理器，其主要功能就是识别出当前请求是否是事务请求。如果是事务请求，那么Follower就会将该事务请求转发给Leader服务器，Leader服务器在接收到这个事务请求后，就会将其提交给请求处理链，按照正常事务请求进行处理。
     2.SendAckRequestProcessor
          SendAckRequestProcessor是Follower服务器上另外一个和Leader服务器有差异的请求处理器。事务日志记录反馈的角色，在完成事务日志记录后，会向Leader服务器发送ACK消息以表明自身完成了事务日志的记录功能。

### 七、Observer服务器
     在ZK的3.3.0版本开始引入的一个全新的服务器角色。Observer的请求处理链路：
     ObserverRequestProcessor ——————> CommitProcessor —————>FinalProcessor
     Leader Server      ——————> SyncRequestProcessor ————> SendAckRequestProcessor

### 八、集群间消息通信
     在整个ZK集群中，都是由Leader服务器来负责进行个各服务器之间的协调，同时，各服务器之间的网络通信，都是通过不头痛类型的消息传递来实现的。ZK集群各服务器之间是如何进行协调的？ZK的消息类型大体可以分为四类：数据同步类型、服务器初始化类型、请求处理型和会话管理型
     数据同步型：
          数据同步型消息是指在Learner和Leader服务器进行数据同步的时候，网络通信所用到的消息，通常有DIFF、TRUNC、SNAP和UPTODATE四种。
    ZK集群间数据同步过程中的消息类型表：
    消息类型
    发送方->接收方
    说明
    DIFF，13
    Leader  —> Learner
    用于通知learner服务器，Leader即将与其进行DIFF方式进行数据同步
    TRUNC，14
    Leader —>Learner
    用于触发Learner服务器进行内存数据库的回滚操作
    SNAP，15
    Leader ——>Learner
    用于通知Learner服务器，Leader即将与其进行全量方式的数据同步
    UPTODATE，12
    Leader——>Learner
    用于告诉Learner服务器，已经完成了数据同步，可以开始对外提供服务了
    服务器初始化型
    服务器初始化型消息是指整个集群或是某些新机器初始化的时候，，Leader和Learner之间相互通信所使用的消息类型，常见的有OBSERVERINFO、FOLLOWERINFO、LEADERINFO、ACKEPOCH和NEWLEADER五种。
    消息类型
    发送方-接收方
    说明
    OBSERVERINFO，16
    Observer—>Leader
    该信息是由Observer服务器在启动的时候发送给Leader的，用于向Leader服务器注册自己，同时向Leader服务器表明当前Learner服务器的角色是Observer。消息中心包含了当前Observer服务器的SID和已经处理的最新的ZXID
    FOLLOWERINFO，11
    Follower—>Leader
    该信息通常是由Follower服务器在启动的时候发送给leader的，用于向Leader服务器注册自己，同时向Leader服务器表明当前Learner服务器角色是Follower。消息中包含了当前Follower服务器的SID和已经处理的最新ZXID
    LEADERINFO，17
    Learner——>Leader
    Learner连接上Leader后，会向Leader发送LearnInfo消息(包含了OBSERVERINFO和FOLLOWERINFO俩类消息),Leader服务器在接收到消息后，也会将L恩爱的人服务器的基本信息发送诶这些Learner，这个消息就是LEADERINFO，通常包含了当前Leader最新的EPOCH值
    ACKEPOCH，18
    Learner——>Leader
    Learner在接收到Leader发来的LeaderInfo消息后，会将自己最新的ZXID和EPOCH以ACKEPOCH消息的形式发送给Leader
    NewLeader，10
    Leader ->>>>>Learner
    该消息通常用于Leader服务器向Learner发送一个阶段性的标识消息-Leader会在和Learner完成一个交流后，向Learner发送NEWLEADER消息，同时带上当前Leader服务器上的最新的ZXID。
    
    请求处理型：
         请求处理型消息是指在进行请求处理的过程中，Leader和Learner服务器之间互相通信所有使用的消息，常见的有REQUEST PROPOSAL ACK COMMIT INFORM SYNC
         REQUEST,1          Learner->Leader     该消息是ZK的请求转发消息。在ZK中，所有的事务请求必须由Leader服务器来处理。当Learner服务器接收到客户端的事务请求后，就会将请求以REQUEST消息的形式转发给Leader服务器来处理
         PROPOSAL,2          Leader->Follower     该消息是ZK实现ZAB算法的核心，即ZAB协议中的提议。在处理事务请求的时候，Leader服务器会将事务请求以PROPOSAL消息的形式创建投票发送给集群中所有的Follower服务器来进行事务日志的记录
         ACK，3     Follower -> Leader      Follower服务器在接收到来自Leader的PROPOSAL消息后，进行事务日志的记录。如果完成了事务日志的记录，那么就会以ACK消息的形式反馈给Leader
         COMMIT，4     Leader -> Follower     该消息用于通知集群中所有的Follower服务器，可以进行事务请求的提交了，Leader服务其在接收到过半的Follower的ACK消息后，就进入事务请求的饿最终提交流程，生成COMMIT消息，告诉所有的Follower服务器进行事务请求的提交
         INFORM，8     Leader->Observer     在事务请求提交阶段，针对Follower服务器，Leader仅仅只需要发送一个Commit消息，Follower服务器就可以完成事务请求的提交了，因为在这之前的事务请求投票阶段，Follower已经接收过PROPOSAL消息，该消息中包含了事务请求的内容，因此Follower可以从之前的Proposal缓存中再次获取到事务请求。而对于Observer来说，没有参见过投票的阶段，因此无法知道事务请求是否提交，因此设计INFORM消息，该消息不仅仅能够通知Observer已经提交事务请求，同时还会在消息中携带事务请求的内容
         SYNC，7     Leader-Learner     该消息用于通知Learner服务器已经完成了Sync操作
    
    会话管理型
        会话管理型消息是指ZK在进行会话管理的过程中，和Learner服务器之间互相通信所有的消息，常见的有PING和REVALIDATE
    PING，5     Leader - > Learner该消息用于Leader同步Learner服务器上的客户端心跳检测，用于激活存活的客户端。ZK的客户端往往会随着服务器无法直接接收到所有客户端的心跳检测，需要委托给Learner来保存这些客户端的心跳检测记录。Leader会定时地想Learner服务器发送PING消息，Learner服务器在接收到PING消息后，会将这段时间内保持心跳检测的客户单列表，同样以PING消息的形式反馈给Leader服务器，由于Leader服务器来负责逐个对这些客户端进行会话激活。REVALIDATE,6     Learner -> Leader该消息用于Learner校验会话是否有效，同时也会激活会话。这童话村发生在客户单重连的过程中，新的服务器需要向Leader发送REVALIDATE消息以确定该会话是否已经超时=============================请求处理========================================
    1.会话创建请求
        ZooKeeper服务端对于会话创建的处理，大体可以分为请求接收，会话创建，预处理，事务处理，事务应用和会话响应6大环节，其大体流程如图下：

    请求接收：
     1.I/O层接收来自客户端的请求
          在ZK中，NIOServerCnxn实例维护每一个客户端连接，客户端与服务端的所有通信是由NIOServerCnxn负责的—其负责统一接收来自客户端的所有请求，并将请求内容从底层网络IO中完整地读取出来。
     2.判断是否是客户端 会话创建 请求
          NIOServerCnxn在负责网络通信的同时，也是客户端会话的载体----每一个会话都会对应一个NIOServerCnxn实体。因此，对于每个请求，ZK都会检查当前NIOServerCnxn实体是否已经被初始化。如果尙未被初始化，那么就可以确定该客户端请求一定是会话创建请求。
     3.反序列化ConnectRequest请求
          一旦确定当前客户端请求是 会话创建 请求，那么服务端就可以对其进行反序列化，并生成一个ConnectRequest请求实体。
     4.判断是否是ReadOnly客户端
          在ZK的设计实现中个，如果当前ZK服务器是以ReadOnly模式启动的，那么所有来自非ReadOnly型客户端将无法被处理。因此针对ConnectRequest，服务端会首先检查其是否是ReadOnly客户端，并以此来决定师傅接收该 会话创建请求
     5.检测客户端ZXID
          正常情况下，同一个ZK集群中，服务端的ZXID必定大于客户端的ZXID，因此如果发现客户端的ZXID值大于服务端的ZXID值，那么服务端将不会接受该客户端的 会话创建 请求
     6.协商sessionTimeout
          客户端在构造ZK实例的时，会有一个sessionTimeOut参数用于指定会话的超时时间。客户端向服务器发送这个超时时间后，服务器会根据自己的超时时间限制最终确定该会话的超时时间———————即sessionTimeout协商过程、默认情形下，ZK服务端对超时时间的限制介于2个tickTime到20个tickTime之间。可以通过zoo.cfg来配置对应的超时时间限制
     7.判断是否需要重新创建会话
          服务端根据客户端请求中是否包含SessionID来判断该客户端是否需要重新创建会话。如果客户端请求中已经包含了sessionID，那么就认为该客户端正在进行会话重连。在这种情况下，服务端只需要重新打开这个会话，否则需要重新创建。

    会话创建：
     8.为客户端生成sessionID
          SessionTracker生成全局唯一的sessionID.sessionID的算法
     9.注册会话
          创建会话最重要的工作就是向SessionTracker中注册会话。SessionTracker中维护了俩个重要的数据结构：sessionWithTimeOut(sessionID所有会话的超时时间)和sessionByID(所有会话的实体)。在会话春关键初期，就应该将该客户端会话的相关信息保存到这俩个数据结构中，方便后续会话管理器进行管理
     10.激活会话
          向SessionTracker注册完会话后，需要对会话进行激活操作。会话激活设计到ZK会话管理分桶策略；核心为：Wie会话安排一个区块，以便会话清理程序能够快速高效地进行会话清理
     11.生成会话密码
          服务端在创建一个客户端会话的时候，会同时为客户端生成一个会话密码，连同sessionIDE一起发送给客户端，作为会话在集群中不同机器间转移的凭证。会话密码的生成算法如下：
     static final private long superSecret = 0XB3415C00L;
     Random r = new Random(sessionId ^ superSecret);
     r.nextBytes(passed)；
     
    预处理
     12.将请求交给ZK的PrepRequestProcessor处理器进行处理
     13.创建请求事务头
          对于事务请求，ZK首先会为其创建请求事务头。请求事务头是每个ZK事务请求中非常重要的部分。服务端后续的请求处理器都是基于该处理请求头识别当前请求是否是事务请求。请求事务头包含了一个事务请求最基本的一些信息：
     clientId                    客户端ID，用来唯一标识该请求所属的客户端
     cxid                        客户端的操作序列号
     xxid                        该事务请求对应的事务ZXID
     time                        服务器开始处理该事务请求的时间
     type                        事务请求的类型，例如：create delete setData和createSession等
     14.创建请求事务体
               对于事物请求，Zk还会为其创建IQ你刚起UI的事务体。在此处于是会话创建请求，因此会创建事务体CreateSessionTxn
     15.注册与激活会话
    
    事务处理：
        16.将请求交给ProposalRequestProcessor处理器
          对于完成请求预处理后，PrepRequestProcessor处理器会将请求交付给自己下一级处理器：ProposalRequestProcessor：是一个和提案有关的处理器，提案是ZooKeeper中针对事务请求所展开的一个投票流程中对事务操作的包装：从ProposalRequestProcessor处理器开始，请求的处理将会进入三个字流程：分别为Sync流程、Proposal流程和Commit流程；
     Sync流程：其核心就是使用SyncRequestProcessor处理器记录事务日志的过程。ProposalRequestProcessor处理器在接收到一个上级处理器流转过来的请求后，首先会判断是否是事务请求，针对事务请求，都会通过事务日志的形式将其记录下来。Leader服务器和Follower服务器的请求处理链路都会有这个处理器，俩者在事务日志上的工是一致的。在完成事务日志记录后，每个Follower服务器都会向Leader服务器发送ACK消息，表明自身完成了事务日志的记录，以便Leader服务器统计每个事务请求的投票情况。
     Proposal流程：在ZK的实现中，每一个事务请求都需要集群中过半机器投票认可才能被真正应用到ZK的内存数据库中去。
          (1)发起投票：如果当前请求是事务请求，那么Leader服务器就会发起一轮事务投票。在发起事务投票之前会先检测服务端的ZXID是否可用。如果ZXID不可用，会抛出XIDRolloverException异常
          (2)生成提议Proposal：如果当前服务端的ZXID可用，就会开始事务投票了。ZK会将之前创建的请求和事务体，以及ZXID和请求序列化到Proposal对象中
          (3)广播提议：生成提议后，Leader服务器会以ZXID做为标识，将该提议放入投票箱outstandingProposals中，同时广播给所有的Follower服务器
          (4)收集投票：Follower服务器在接收到leader发来的这个提议后，会进入Sync流程来进行事务日志的记录，一旦记录完成后，就会发送ACK消息给Leader服务器，Leader服务器根据这些ACK消息统计每个提议的投票情况，票数过半就进入Commit流程
          (5)将请求放入toBeApplied队列：在该提议被提交之前，ZK首先会将其放入toBeApplied队列中去
          (6)广播Commit消息：
     Commit流程：
           (1)将请求交付给CommitProcessor处理器
               CommitProcessor处理器在收到请求后，并不会立即成立，而是会想起放入queueRequest队列中
           (2)处理queuedRequests队列请求：CommitProcessor处理器会有一个单独的线程来处理从上一级处理器流转下来的请求。当加测到queuedRequests队列中已经有新的请求进来，就会逐个从队列中取出请求进行处理
           (3)标记nextPending: 如果queuedRequests队列中取出的请求是一个事务请求，那么就需要进行集群中各个服务器之间的投票处理，同时需要将nextPending标记为当前请求。标记nextPending的作用，一方面是为了确保事务请求的顺序性，另一个方面是便于CommitProcessor处理器检测当前集群中是否正在进行事务请求的投票
           (4)等待Proposal投票
           (5)投票通过：通过投票，ZK将会计入请求提交阶段。ZK会将请求放入committedRequests队列中，同时唤醒Commit流程
           (6)提交投票：一旦发现committedRequests中有了提交的请求，那么Commit流程就会开始提交请求，请求之前先检测，Committed流程会对比之前标记的nextPending和committedRequests队列中第一个请求是否一致。如果检测通过，那就Commit流程就会将该请求放入toProcess队列中，然后交付给下一个请求处理器:FinalRequestProcessor
    事务应用：
     17.交付FinalRequestProcessor处理器：首先检测outstandingChanges队列中请求的有效性，如果发现这些请求已经落后于当前正在处理的请求直接移除、
     18.事务应用：
          将事务日志中的数据变更到内存数据中。【会话创建】这类事务请求，ZK做了特殊的处理，因为会话的管理是由SessionTracker负责的，而在会话创建的步骤9总，ZK已经将会话信息注册到SessionTracker中，因此无需对内存数据库做任何处理，只要再次向SessionTracker进行会话注册即可
     19.将事务请求放入队列：commitProposal
          一旦完成事务请求的内存数据库应用，就可以将该请求放入commitProposal队列中。commitProposal队列用来保存最近被提交的事务请求，以便集群间机器进行数据的快速同步。
    会话响应
     20.统计处理：到这里ZK的请求的处理链路上的所有请求处理器间完成了流转。ZK会计算请求在服务端处理所花费的时间，同时还会统计客户端连接的一些基本信息，包括lastZxid（最新的ZXID）、lastOp（最后一次和服务端的操作）和lastLatency(最后一次请求处理所花费的时间)等信息
     21.创建响应ConnectReponse:
          ConnectReponse就是一个会话创建成功后的响应，包含了当前客户端与服务器端之间的通信协议版本号protocolVersion 会话超时时间 sessionID 和 会话密码
     22.序列化ConnectReponse
     23.IO成发送响应给客户端
    SetData请求：可以大致分为4大步骤：分别是请求的预处理、事务处理、事务应用和请求响应；
    
    预处理：
     1.I/O层接收来自客户端的请求
     2.判断是否是客户端[会话创建]请求，如果不是会话创建请求，就按照事务请求进行处理
     3.将请求交个ZooKeeper的PrepRequestProcessor处理器进行处理
     4.创建请求事务头
     5.会话检查：客户端会话检查是指检查该会话是否有效，即是否已经超时。如果该会话已经超时，那么服务端就想客户端抛出SessionExpiredException异常。
     6.反序列化请求，并创建ChangeRecord记录
          面对客户端请求，ZooKeeper首先会将其进行反序列化并生成特定的SetDataRequest请求。SetDataRequest请求中通常包含了数据节点路径path、更新的数据内容data和期望的数据节点版本version.同时请求中对应的path,ZooKeeper会生成一个ChangeRecord记录，并放入outstandingChanges队列中。outstandingChanges队列中存放了当前ZooKeeper服务器正在进行处理的事务请求，以便ZooKeeper在处理后续请求的过程中需要针对之前的客户端请求的相关处理，例如对于会话关闭请求来说，其需要根据当前正在处理的事务请求来收集需要清理的临时节点，关于会话清理相关的内容
     7.ACL检查
          由于当前请求是数据更新请求，因此Zookeeper需要检查该客户端是否具有数据更新的权限。如果没有权限，就会抛出NoAuthException异常。
     8.数据版本检查
          ZK可以依靠version属性来实现客观锁机制中的写入校验。如果ZK服务器发现当前数据内容的版本号与客户端预期的版本不匹配的话，那么将会抛出异常
     9.创建请求事务体SetDataTxn
     10.保存事务操作到outstandingChanges队列中去
    
    事务处理：
     对于事务请求，ZooKeeper服务端都会发起事务处理流程。无论对于会话创建请求还是SetData请求，或是其他事务请求，事务处理流程都是一致的，都是由于ProposalRequestProcessor处理器发起，通过Sync、Proposal和Commit三个子流程相互协作完成的。
    
    事务应用：
     11.交付给FinalRequestProcessor处理器
     12.事务应用：ZK会将请求头和事务体直接交给内存数据库ZKDatabase进行事务应用，同时返回ProcessTxnResult对象，包含了数据节点的内容更新后的stat
     13.将事务请求放入队列：commitProposal
     
    请求响应：
         14.统计处理
         15.创建响应体：SetDataResponse是一个数据更新成功后的响应，主要包含了当前数据节点的最新状态stat
         16.创建响应头：响应头是每个请求响应的基本信息，方便客户端对相应进行快速的解析，包含当前响应对应的事务ZXID和请求处理是否成功的标识err.
         17.序列化响应
         18.I/O层发送响应给客户端
         
    事务请求转发：
        在事务请求的处理过程中，需要我注意的一个细节是，为了保证事务请求被顺序执行，从而确保ZooKeeper集群的数据一致性，所有的事务请求必须由Leader服务器来处理。ZooKeeper实现了非常特别的事务请求转发机制：所有非Leader服务器如果接收到了来自客户端的事务请求，那么必须将其转发给Leader服务器来处理。在Follower或是Observer服务器中，第一个请求处理器分别是FollowerRequestProcessor和ObserverRequestProcessor,无论是哪个处理器，都会检查当前请求是否是事务请求，如果是事务请求，那么就将该客户端请求以REQUEST消息的形式转发给LEADER服务器。Leader服务器在接收到这个消息后，会解析出客户端的原始请求，然后提交到自己的请求处理链中开始进行事务请求的处理。
    GetData请求：以GetData请求为实例，展示非事务请求的处理流程：
    
    预处理：
         1.IO层接收来自客户端的请求
         2.判断是否是客户端【会话创建】请求
         3.将请求交给ZooKeeper的PrepRequestProcessor处理器进行处理
         4.会话检查：由于GetData请求是非事务请求，因此省去了许多事务预处理逻辑，包括创建请求事务头，ChangeRecord和事务体等，以及对数据节点版本的检查
    非事务处理：
         5.反序列化GetDataRequest请求
             6.获取数据节点：反序列化出的完整GetDataRequest对象(包括了数据节点的path和watcher注册情况),ZooKeeper会从内存数据库中获取到该节点以及ACL信息
         7.ACL检查
         8.获取数据内容和stat，注册Watcher
    请求响应
         9.创建响应体GetDataResponse
              GetDataResponse是一个数据获取成功后的响应，主要包括了当前数据界定啊的内容和状态stat信息
         10.创建响应头
         11.统计处理
         12.序列化响应
         13.IO层发送响应给对应的客户端
        ===========数据与存储============
    在ZK中，数据存储分为俩部分：内存数据和磁盘数据存储。
    1.内存数据
        ZK的数据模型是一棵树。在这个内存数据库中，存储了整颗树的内容，包括所有的节点路径、节点数据以及ACL信息等，ZooKeeper会定时将这个数据存储到磁盘上。
        DataTree：是ZooKeeper内存数据存储的核心，是一个树的数据结构，代表了内存中的一份完整的数据。DataTree不包含任何与网络、客户端连接以及请求处理等相关的业务逻辑，是一个非常独立的ZooKeeper组件：
        DataNode：是数据存储的最小单元
         DataTree用于存储所有ZooKeeper节点的路径、数据内容及其ACL信息等，底层的数据结构其实是一个典型的ConcurrentHashMap键值对结构：
        private final ConcurrentHashMap<String,DataNode> nodes = new ConcurrentHashMap<String,DataNode>();
        在nodes这个Map中，存放了Zookeeper服务器上所有的数据节点，可以说，对于ZooKeeper数据的所有操作，底层都是对这个Map结构的操作。nodes以数据节点的路径(path)为key，value则是节点的数据内容：DataNode，另外对临时节点，为了便于实时访问和及时清理，DataTree中还单独将临时节点保存起来：
        private final Map<Long,HashSet<String>> ephemerals = new ConcurrentHashMap<Long,HashSet<String>>();
        ZKDatabase:是ZooKeeper的内存数据库，负责管理ZooKeeper的所有会话、DataTree存储和事务日志。ZKDataBase会定时向磁盘Dump快照数据，同时在ZooKeeper服务器启动的时候，会通过磁盘上的事务日志和快照数据文件恢复成一个完整的内存数据库。
    2.事务日志
        事务日志的存储、日志格式和日志写入等过程进行了解：
        文件存储：在zoo.cfg中配置的目录：dataDir。这个目录是ZK中默认用于存储事务日志文件的，其实在ZooKeeper中可以为事务日志单独分配一个文件存储目录：dataLogDir。在对应的XXXX/XXXX/xxxx/version-2目录下会生成类似下面文件：
     。。。。。。。。 log.1 …..
     。。。。。。。。 log.1c …..
    这些文件就是ZooKeeper的事务日志了。
        1.文件大小一致
        2.文件后缀非常有规律，都是一个十六进制数字，同时随着文件修改时间的推移
     日志格式：
        日志写入：FileTxnLog负责维护事务日志对外的接口，包括事务日志的写入和读取等，首先来看日志的写入。将事务操作写入事务日志的工作主要由append方法来负责：
        public synchronized boolean appen(TxnHeader hdr , Record txn)
    ZooKeeper在进行事务日志的写入过程中，会将事务头和事务体传给该方法。事务日志的写入过程大体分为6个步骤：
        1.确定是都有事务日志可写
            当ZK服务器启动完成需要进行第一次事务日志的写入，或是上一个事务日志写满的时候，都会处于与事务日志文件断开的状态，即ZK服务器没有和任意一个日志文件相关关联。ZK首先会判断FileTxnLog组件是否已经关联上一个可写的事务日志文件。如果没有关联上事务日志文件，那么就会使用与该事务操作关联的ZXID作为后缀创建一个事务日志文件，同时构建事务日志文件头(magic,事务日志格式版本version和dbid),并立即写入这个事务日志文件中去。同时，将该文件的文件流写入一个集合：streamsToFlush.streamsToFlush集合是ZK用来记录当前需要强制进行数据落盘的文件流。
      2.确定事务日志文件是否需要扩容(预分配)
            ZooKeeper的事务日志文件会采取磁盘空间预分配的策略。预分配大小可以用zookeeper.preAllocSize来进行设置
      3.事务序列化
           事务序列化包括对事务头和事务体的序列化，分别是对TxnHeader(事务头)和Record(事务体)的序列化。其中事务体又可分为会话创建事务(CreateSessionTxn)、节点创建事务(CreateTxn)、节点删除事务(DeleteTxn)和节点数据更新事务(SetDataTxn)等
      4.生成Checksum
           为了保证事务日志文件的完整性和数据的准确性，Zookeeper在将事务日志写入文件前，会根据步骤3中序列化生产的字节数组来计算Checksum.在ZooKeeper默认使用Adler32算法来计算Checksum值
      5.写入事务日志文件流
           系列化后的事务头，事务体及CheckSum值写入到文件流中去。此时由于ZooKeeper使用的是Buffer额度OutputStream，因此写入的数据并非真正被写入到磁盘文件上
      6.事务日志刷入磁盘 
          将每个事务日志文件对应的文件流放入了streamsToFlush,因此这里会从streamsToFlush中提取出文件流，并调用FileChannel.force()接口来强制将数据刷入磁盘文件中去。force接扣对应的其实是底层的fsync接口，是一个比较耗费磁盘IO资源的接口，因此ZooKeeper允许用户控制是否需要主动调用接口，可以通过系统属性ZooKeeper.forceSync来设置
    日志截断：
        在Zookeeper运行过程中，可能会出现这样的情况，非Leader机器上记录的事务ID(peerLastZxid)比Leader服务器大，无论这个情况是如何发生的，都是一个非法的运行时状态。同时，ZooKeeper遵循一个原则：只要集群中存在Leader,那么所有机器都必须与该Leader的数据保持同步。一旦某台机器碰到上述情况，Leader会发送TRUNC命令给这个机器，要求进行日志截断。Learner服务器在接收到该命令后，就会删除所有包含或大于peerLastZxid是的事务日志文件。
    snapshot ———— 数据快照
        数据快照是ZooKeeper数据存储中另一个非常核心的运行机制。数据快照用来记录ZooKeeper服务器上某个时刻的全量内存数据内容，并将其写入到指定的磁盘文件中。文件存储和事务日志类似
    
    初始化过程：
            在ZK服务器启动期间，首先会进行数据初始化工作，用于将存储在磁盘上的数据文件加载到ZooKeeper服务器内存中。
    
    初始化流程：
        数据的初始化工作，其实就是从磁盘中加载数据的过程，主要包括了从快照文件中加载快照数据和根据事务日志进行数据订正俩个过程。
        1.初始化FileTxnSnapLog
            FileTxnSnapLog是ZooKeeper事务日志和快照数据访问层，用于衔接上层业务与底层数据存储。底层数据包括了事务日志和快照数据俩个部分，因此FileTxnSnapLog内部分为FileTxnLog和FileSnap的初始化，分别代表事务日志管理器和快照数据管理器的初始化
        2.初始化ZKDatabase
            完成FileTxnSnapLog的初始化后，我们就完成了ZooKeeper服务器和底层数据存储的对接，接下来就是要开始构建内存数据库ZKDatabase了。在初始化过程中，首先会构建一个初始化的DataTree，同时将步奏1中初始化的FileTxnSnapLog交给ZKDatabase，以便内存数据库能够对事务日志和快照参数进行访问
            
    数据同步：
        ZK集群服务器启动的过程中，整个集群完成Leader选举之后，Learner会向Leader服务器进行注册。当Leader服务器向Leader完成注册后，就进入数据同步环节。简单地讲，数据同步过程就是Leader服务器将那些没有在Learner服务器上提交过的事务请求同步给Learner服务器。
    获取Learner状态：
        在注册Learner的最后阶段，Learner服务器会发送给Leader服务器一个ACKEPOCH数据包，Leader会从这个数据包中解析出该Learner的currentEpoch和castzxid.
    数据同步初始化：
        Leader服务器会进行数据同步初始化，首先会从ZooKeeper的内存数据库中提取出事务请求对应的提议缓存：proposals,同时完成对一下三个ZXID值的初始化：
     peerLastZxid:该Learner服务器最后处理的ZXID
     minCommittedLog:Leader服务器提议缓存队列committedLog中的最小ZXID
     maxCommittedLog:Leader服务器提议缓存队列committedLog中的最大ZXID

### 九、ZooKeeper集群数据同步通常分为四类：
    1.直接差异化同步(DIFF同步)
        场景：peerLastZxid介于minCommittedLog和maxCommittedLog之间.Leader服务器会首先向这个Learner发送一个DIFF指令，用于通知Learner进入差异化数据同步阶段，Leader服务器即将把一些Proposal同步给自己。在实际Proposal同步过程中，针对每个Proposal，Leader服务器都会通过发送俩个数据包来完成，分别是PROPOSAL内容数据包和COMMIT指令数据包————这和ZooKeeper运行时Leader和Follower之间的事务请求的提交过程是一致的
        Demo：假如某个时刻Leader服务器的提议缓存队列对应的ZXID依次是:0x500000001,0x500000002,0x500000003,0x500000004,0x500000005
        而Learner服务器最后处理的ZXID为0x500000003，于是Leader服务器就会一次将0x500000004，0x500000005俩个提议同步给Learner服务器，同步过程中的数据包发送顺序如下：
        发送顺序
        数据包类型
        对应的ZXID
        1
        PROPOSAL
        0x500000004
        2
        COMMIT
        0x500000004
        3
        PROPOSAL
        0x500000005
        4
        COMMIT
        0x500000005
        Learner服务器就可以接收到自己和Leader服务器的所有差异数据。Leader服务器在发送完差异数据之后，就会将Learner加入到forwardingFollowers或observingLearners队列中，这俩个队列在ZK运行期间的事务请求处理过程中都会使用到。。随后Leader还会立即发送一个NEWLEADER指令，用于通知Learner，已经将提议缓存队列中Proposal都同步给自己
    
    2.先回滚再差异化同步(TRUNC+DIFF同步)
        仅回滚同步(TRUNC同步)
        全量同步(SNAP同步)
        场景1：peerLastZxid小于minCommittedLog
        场景2：Leader服务器上没有提议缓存队列，peerLastZxid不等于lastProcessedZxid(Leader服务器数据恢复后得到最大ZXID)。
        所谓全量同步就是Leader服务器将本机上的全局内存数据都同步给Learner。Leader服务器首先向Learner发送一个SNAP指令，通知Learner即将进行去全量数据同步。
        在整个ZK集群间机器的数据同步流程。真个数据同步流程的代码实现主要是LeaderHandlers和Learner俩个类中

### 十、ZooKeeper核心技术的总结：
        ZooKeeper以树作为其内存数据模型，树上的每一个节点是最小的数据单元，即ZNode。ZNode具有不同的节点特性，同时每个节点都具有一个递增的版本号，一次可以实现分布式数据的原子性更新。
        ZooKeeper的序列化层使用Hadoop中遗留下来的Jute组件
        ZooKeeper的客户端和服务端之间会建立起TCP长连接进行网络通信，基于该TCP连接衍生出来的会话概念，是客户端和服务端之间所有请求与响应交互的基石。在会话的生命周期中，会出现断开、重连或是会话失效等一系列问题，这些都是ZK的会话管理器需要处理的问题======Leader服务器会负责管理每个会话的生命周期，包括会话的创建、心跳检测和销毁等。
        在服务器启动阶段，会进行磁盘数据的恢复，完成数据恢复后就会进行Leader选举。一旦选举产生Leader服务器后，就立即开始进行集群间的数据同步。在整个过程中，ZK都处于不可用状态，直到数据同步完毕，ZooKeeper才可以对外提供正常服务。在运行期间，如果Leader服务器所在的机器挂掉或是集群中绝大部分服务器断开连接，那么就触发新一轮的Leader选举。同样，在新的Leader服务器选举产生之前，ZK无法对外提供服务。
        一个正常运行的ZooKeeper集群，其机器角色通常由Leader、Follower和Observer组成。ZooKeeper对于客户端请求的处理，严格按照ZAB协议规范来进行。每一个服务器在启动初始化阶段都会组装一个请求处理链，Leader服务器能够处理所有类型的客户端请求，而对于Follower或是Observer服务器来说，可以正常处理非事务请求，而事务请求规则需要转发给Leader服务器来处理，同时，对于每个事务请求，Leader都会为其分配一个全局唯一且递增的ZXID，次来保证事务处理的顺序性。在事务请求的处理过程中，Leader和Follower服务器都会进行事务日志的记录。
        ZooKeeper通过JDK的File接口简单地实现了自己的数据存储系统，其底层数据存储包括事务日志和快照数据俩部分，这些都是ZooKeeper实现数据一致性非常关键的部分。