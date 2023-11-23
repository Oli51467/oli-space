## 一、分布式锁

锁的预备知识：[乐观锁和悲观锁详解 | JavaGuide(Java面试 + 学习指南)](https://javaguide.cn/java/concurrent/optimistic-lock-and-pessimistic-lock.html#aba-问题)

方案总结：

[分布式锁介绍 | JavaGuide(Java面试 + 学习指南)](https://javaguide.cn/distributed-system/distributed-lock.html#为什么需要分布式锁)

[分布式锁常见实现方案总结 | JavaGuide(Java面试 + 学习指南)](https://javaguide.cn/distributed-system/distributed-lock-implementations.html#redis-如何解决集群情况下分布式锁的可靠性)

#### Redis实现分布式锁：

- [【精选】Redis：Redisson分布式锁的使用（推荐使用）_redisson分布式锁使用_穿城大饼的博客-CSDN博客](https://blog.csdn.net/chuanchengdabing/article/details/121210426)

- [Redis分布式锁-这一篇全了解(Redission实现分布式锁完美方案)-CSDN博客](https://blog.csdn.net/asd051377305/article/details/108384490)

#### Zookeeper实现分布式锁：

- [Zookeeper + Curator实现分布式锁 - 掘金 (juejin.cn)](https://juejin.cn/post/7038596797446651940)
- [SpringBoot整合zookeeper、curator，实现分布式锁功能 - 简书 (jianshu.com)](https://www.jianshu.com/p/575d1813fa34)

##### 优点：

1. ZooKeeper分布式锁（如InterProcessMutex）能有效地解决分布式问题，不可重入问题，使用起来也较为简单

##### 缺点：

1. ZooKeeper实现的分布式锁，性能并不太高。因为每次在创建锁和释放锁的过程中，都要动态创建、销毁暂时节点来实现锁功能，
2. Zk中创建和删除节点只能通过Leader（主）服务器来执行，然后Leader服务器还需要将数据同步到所有的Follower（从）服务器上，这样频繁的网络通信，系统性能会下降。
3. 在高性能、高并发的应用场景下，不建议使用ZooKeeper的分布式锁，而由于ZooKeeper的高可用性，因此在并发量不是太高的应用场景中，还是推荐使用ZooKeeper的分布式锁。

#### 题外：

- [【求锤得锤的故事】Redis锁从面试连环炮聊到神仙打架。 (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzg3NjU3NTkwMQ==&mid=2247505097&idx=1&sn=5c03cb769c4458350f4d4a321ad51f5a&source=41#wechat_redirect)



