---
title: ZooKeeper会话管理
date: 2020-03-01 16:43:36
categories:
- ZooKeeper
tags: 
- ZooKeeper
---

会话（session）是 ZooKeeper 中最重要的概念之一，客户端与服务端之间的任何交互操作都与会话息息相关，这其中就包括临时节点的生命周期、客户端请求的顺序执行以及 Watcher 通知机制等等，这篇文章就带大家了解一下 ZooKeeper 会话的技术内幕。

### 会话信息

#### Session

在 ZooKeeper 客户端与服务端完成连接创建后，就建立了一个会话，这些会话的信息都是保存在 Session 这个实体里面的，它包含了四个重要的属性：

- sessionID
  会话 ID，用来唯一标示一个会话，每次客户端创建新会话的时候，ZooKeeper 都会为其分配一个全局的 sessionID，其实现的代码如下，最终得到的结果，高 8 位是所在机器的 id，后 56 位使用了当前时间的毫秒表示。

  ``` java
  /*
   * 创建 sessionId
   * id 就是配置在 myid 文件中的值
   */
  public static long initializeNextSession(long id) {
      long nextSid = 0;
      nextSid = (System.currentTimeMillis() << 24) >>> 8;
      nextSid = nextSid | (id << 56);
      return nextSid;
  }
  ```

- TimeOut： 会话超时时间

- TickTime： 下次会话超时时间点

- isClosing：用于标记该会话是否已经被关闭

#### SessionTracker

是 ZooKeeper 服务端的会话管理器，负责会话的创建、管理和清理等工作。

### 会话管理

#### 分桶策略

ZooKeeper 是通过分桶策略来管理会话的，这个策略是按照超时时间点进行分类。对于一个新创建的会话而言，其会话创建完毕后，ZooKeeper 就会为其计算 ExpirationTime，计算方式如下：

``` java
ExpirationTime = CurrentTime + SessionTimeout
ExpirationTime = (ExpirationTime / ExpirationInterval + 1) * ExpirationInterval
```

上面是计算过期时间的方法，其中 CurrentTime 是当前时间，SessionTimeout 指的是该会话设置的超时时间，ExpirationInterval 是服务器定期检查会话是否超时的时间间隔，通过上面的方法计算之后，得出的结果总是 ExpirationInterval 的整数倍，ZooKeeper 就可以定期的处理批量的会话，这是它的高明之处。

![ZooKeeper分通策略.png](http://ww1.sinaimg.cn/large/006avC6ggy1gcfvswiqqhj30f704imx4.jpg)

#### 会话激活

为了保持客户端会话的有效性，客户端会在会话超时时间过期范围内向服务端发送 PING 请求来保持会话的有效性，我们俗称 “心跳检测”。服务端接收到心跳检测之后，会重新设置该会话的过期时间，保证和该客户端保持长时间的连接状态。

心跳检测完毕，ZooKeeper 会将那些还存活这的会话迁移到对应的新区块中。

### 会话清理

1. 标记会话状态为 “已关闭”：由于整个清理会话清理过程需要一段时间，为了保证不再处理来自该客户端的新请求，SessionTracker 会首先将该会话的 isClosing 属性标记为 true，这样，即使在会话清理期间接收到该客户端的新请求，也无法继续处理了
2. 发起 “会话关闭” 请求：为了使对该会话的关闭操作在整个服务器集群中都生效，ZooKeeper 使用了提交 “会话关闭” 的请求方式，并立即交付给 PrepRequestProcessor 处理器进行处理。
3. 收集需要清理的临时节点
4. 添加 “节点删除” 事务变更
5. 删除临时节点
6. 移除会话
7. 关闭 NIOServerCnxn