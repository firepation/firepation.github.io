---
title: ZooKeeper客户端启动过程
date: 2020-02-29 09:37:59
categories:
- ZooKeeper
tags: 
- ZooKeeper
---

作为一个开发人员，我们和 ZooKeeper 服务器交互主要还是通过官方提供的客户端，这样的话我们就有必要了解下其内部的实现原理了。底下的话是客户端的整体架构，其中：

- ZooKeeper 是客户端的入口
- ClientWatchManager 是客户端 Watcher 管理器
- HostProvider 是客户端地址列表管理器
- ClientCnxn 是客户端的核心线程，其包含了 SendThread 和 EventThread，前者的话负责客户端和服务器之间的网络通信，后者的话负责对服务端事件的处理。

![ZooKeeper客户端类图.jpg](http://ww1.sinaimg.cn/large/006avC6ggy1gcdattw18pj312r0jlgo0.jpg)

其实客户端的启动过程就是创建 ZooKeeper 类的过程，我们就来看看它在构造方法里面都做了什么东西，底下的话是启动的整个流程图，我们一步一步来看。其中白底的话我们可以看作第一阶段，我们称初始化阶段；灰底的话是第二阶段，我们称为会话创建阶段；黄底的话则是客户端在连接收到响应后对应的处理，我们称为响应处理阶段。

![ZooKeeper 会话创建过程.jpg](http://ww1.sinaimg.cn/large/006avC6ggy1gcdaad0rdfj30y914gwic.jpg)

### 初始化阶段

1. 初始化 ZooKeeper 对象
初始化过程中会创建客户端的 Watcher 管理器：ClientWatchManager。
2. 设置会话默认 Watcher
如果在构造方法中传入一个 Watcher 对象，这个对象就会被当作整个会话的默认 Watcher，并且该 Watcher 会被保存在 ClientWatchManager 中。
3. 构造 HostProvider
   对于构造方法中传入的服务器地址，客户端会将其存放在服务器地址列表管理器 HostProvider 中
4. 创建网络连接器 ClientCnxn
   ZooKeeper 客户端创建 ClientCnxn 用于管理客户端与服务器的网络交互。同时还会初始化两个核心队列 outgoingQueue 和 pendingQueue，分别作为客户端的请求发送队列和服务端响应的等待队列。
5. 初始化 SendThread 和 EventThread
   SendThread 用于管理客户端和服务器之间的所有网络 I/O，后者用于进行客户端的事件处理。同时客户端还会将 ClientCnxnSocket 分配给 SendThread 作为底层网络 I/O 处理器，并初始化 EventThread 的待处理队列 waitingEvents，用于存放所有等待被客户端处理的事件。

### 会话创建阶段

6. 启动 SendThread 和 EventThread
7. 获取一个服务器地址
8. 创建 TCP 连接
   ClientCnxnSocket 负责和服务器创建一个 TCP 长连接
9. 构造 ConnectRequest 请求
   步骤 8 只是纯粹从网络 TCP 层面完成了客户端与服务器之间的 Socket 连接，但远未完成 ZooKeeper 客户端的会话创建
   SendThread 会构造出一个 ConnectRequest 请求，该请求代表了客户端试图与服务器创建一个会话。同时，ZooKeeper 客户端还会进一步将请求包装成网络 I/O 层的 packet 对象，放入请求发送队列 outgoingQueue 中去。
10. 发送请求
    ClientCnxnSocket 负责从 outgoingQueue 中取出一个待发送的 Packet 对象，将其序列化成 ByteBuffer 后，发送给服务器端。

### 响应处理阶段

11. 接收服务端响应
    ClientCnxnSocket 接收到服务器的响应后，会首先判断当前的客户端状态是否是 “已初始化”，如果尚未完成初始化，那么就认为该响应一定是会话创建请求的响应，直接交由 readConnectResult 方法来处理该响应。
12. 处理 Response
    ClientCnxnSocket 会对接收到的服务端响应进行发序列化，得到 ConnectResponse 对象，并从中得到 ZooKeeper 服务端分配的会话 sessionId
13. 连接成功
    连接成功后，一方面需要通知 SendThread 线程，进一步对客户端及逆行会话参数的设置，包括 readTimeout 和 connectTimeout 等，并更新客户端状态；另一方面，需要通知地址管理器 HostProvider 当前成功连接的服务器地址
14. 生成事件：SyncConnected-None
    为了能够让上层应用感知会话的成功创建，SendThread 会生成一个事件 SyncConnected-None，代表客户端与服务器会话创建成功，并将该事件传递给 EventThread 线程。
15. 查询 Watcher
    EventThread 线程收到事件后，会从 ClientWatchManager 管理器中查询出对应的 Watcher，针对 SyncConnected-None 事件，那么就直接找出步骤 2 中存储的默认 Watcher，然后将其放到 EventThread 的 waitingEvents 队列中去。
16. 处理事件
    EventThread 不断从 waitingEvents 队列中取出待处理的 Watcher 对象，然后直接调用该对象的 process 方法。

**注：**以上内容来自《从 Paxos 到 ZooKeeper -- 分布式一致性原理与实践》