---
layout: post
title: Zookeeper学习（1）
date: 2021-03-23
Author: Jack
categories: [Zookeeper]
tags: [zookeeper]
comments: true
---

Zookeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

zookeeper可以做 配置管理，命名服务， 提供分布式同步，以及提供集群管理。

### **配置管理**

当配置文件很多的时候，很多服务器都需要配置的时候，或者配置文件是动态的情况下，配置服务需要提供很强的可靠性，这种可靠性可以依靠集群服务来实现，但是使用集群就需要解决一致性问题。这时候我们就可以使用zookeeper，他通过Zab（Zookeeper atomic broadcast）来实现一致性。

已知的例子有：1）HBase中，client通过连接zookeeper，获取HBase集群的配置信息。

​                           2）Kafka中，broker的信息也是通过zookeeper维护的。

### **命名服务**

当一个集群里面有数以千计的services运行在数以千计的服务上面，并且需要动态的注册和获取services信息的时候，我们就可以使用zookeeper。由于zookeeper提供watches，因此每当有新的service注册或者有service失去连接，client都会被notify。类似于DNS服务。

![Znode Structure](https://github.com/haotianlyu/haotianlyu.github.io/blob/master/images/Znode_for_naming_service.jpg?raw=true)

已知的例子有：1）Netflix的Curator

​                           2) zk_watcher也提供了client API来调用zookeeper的naming service

### **分布式锁**

当多个客户端向zookeeper请求一把分布式锁，每个客户端的请求相当于在锁的Znode节点上创建一个临时有序节点（ephemeral sequential node). 添加节点后，客户端会从zookeeper获取锁所在的Znode下面的所有子节点，然后他会查看自己的创建的节点的sequence number是不是最小的，如果是最小的，那么就相当于加锁成功，如果不是最小的，就对比自己创建的有序节点-1的那个节点设置watch，这样当上一个有序节点完成，上一个获得锁的客户端会通过在zookeeper删除节点来释放锁，这样watch会通知当前节点锁被释放，这时，客户端会重新从zookeeper获得锁所在的Znode的children节点，然后判断是否自己是第一位。如果一个客户端向zookeeper添加了上锁请求后失去响应，那么zookeeper会检测到该客户端失去响应，从而会自动释放节点。

例子：客户端A,B向zookeeper的my_lock分布式锁申请锁，A先发出了请求，那么zookeeper会在my_lock这个znode下面创建一个新的临时子node叫xxxxx-0000001。之后B又发出了请求，这时zookeeper会在my_lock下面创建另一个临时子node叫xxxxx-0000002。A发出请求后会向zookeeper请求getChild(my_lock), 得到[xxxxx-0000001, xxxxx-0000002], 由于自己的序号是最小的，因此相当于确认了自己获得了分布式锁。B发出请求后也向zookeeper请求getChild(my_lock), 得到[xxxxx-0000001, xxxxx-0000002], 由于自己不是最小的，所以对znode xxxxx-0000001添加watch，当A完成任务，释放锁，A会通过delete操作删除临时子节点，watch会notify B，此时B再次向zookeeper getChild(my lock)得到的就是[xxxxx-0000002], 自己是最小的，因此可以获得锁。

### **集群管理**

集群管理包括集群监控和集群控制，前者侧重于对集群运行时状态的收集，后者则是对集群进行操作与控制。

当我们希望知道集群中有多少machine在工作，对每台machine的状态进行记录或者是对machine进行on/off操作时，我们就需要集群管理系统。传统的集群管理系统是在每台machine上部署一个agent，agent定期主动向monitor center汇报当前的机器状态。当集群规模适中时，这个方案确实有效，但当业务场景增多，集群规模变大后，该方案会面临，大规模升级困难，agent客户端要support多种环境，以及对业务细节的状态无法监控等等问题。 这种情况下，就是zookeeper出场得时候了。由于客户端可以对zookeeper的节点添加watch，当该节点或子节点列表变化，zookeeper就会notify客户端。此外，对zookeeper上创建的临时节点，当客户端与zookeeper会话失效后，该临时节点会自动被清除。

例子1：分布式日志收集系统：

​	zookeeper创建收集器的根节点后（/logs/collector），每个日志收集器机器到zookeeper的收集器节点下面创建自己的持久节点(/logs/collector/host1) 以及用于存放状态的状态持久子节点（/logs/collector/host1/status）. 当所有的收集器完成注册，zookeeper将会将services分组，然后将分组后的services添加到收集器的节点上去，这样，收集器就可以通过getChild来获取自己需要获得日志的services列表，从而进行日志收集工作。因为要考虑收集器宕机的可能性，我们要让收集器定期向自己的status节点写入自己的状态信息（日志手机进度信息），相当于一种heart-beating， zookeeper可以通过状态节点的最后更新时间来判断收集器是否存活。当有收集器挂掉/有新机器加入，那么zookeeper就需要重新计算任务的分配。采用的策略有 全部动态分配（全部重新计算分组，然后重新分配）和 局部动态分配（状态节点也要记录收集器的负载情况，zookeeper选择负载轻的分配任务）当然，实际情况下，我们可以让zookeeper主动轮询收集器而不是让收集器主动汇报，这样虽然会有延时，但是可以降低网络压力。

例子2：在线云主机管理

​	我们使用zookeeper对机器的上下线进行全局监控。当新增机器的时候，首先需要将指定的Agent部署到机器上，Agent部署启动后，会首先向zookeeper对应的管理节点进行一个临时子节点的注册。（/XAE/machine/host1)。当Agent在zookeeper上创建完临时子节点之后，对/XAE/machine节点关注的监控中心就会接收到子节点变更的通知，从而可以添加对于新加入的机器开启管理。另一方面，当机器下线，zookeeper与该机器的连接会断开，从而导致临时子节点被删除，此时对/XAE/machine节点关注的监控中心就会得知机器下线。在机器运行过程中，Agent也会定期的将机器的运行状态更新在/XAE/machine/host1中，监控中心从而可以通过对这些节点设置watch来获得主机的运行时状态。