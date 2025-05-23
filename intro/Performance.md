---
layout: page
title: 性能
parent: intro
nav_order: 4
---

* TOC
{:toc}


# Performance

## Cache命中率
Zeze 直接使用本进程内的Cache。在Cache命中的情况下，没有任何远程访问。此时性能
可以达到最高。Zeze 的性能核心就是Cache命中率。负载分配的一个原则就是需要提高
Cache命中率。

## 记录大小
Zeze 使用后台 key-value 数据库保存数据。记录读取和写入是作为整体保存到后台数据库
的。如果记录太大，只修改里面的少量数据，也需要整个记录一起保存到后台。这里有一定
的系列化开销。需要分析需求选择合适的记录大小。一般来说应用得到需求都是按模块给的，
开始的时候数据按模块划分即可。模块太大时最好分成子模块，或者模块内分成多个小一点
的记录。
记录可以包含容器，一般需要设定合适上限。如果数据需要很大，那么应用可能需要自己在
key-value记录的基础上实现list（多个记录来保存数据），然后自己实现遍历，增删等操作。

## 缓存同步和分布式事务
这个是全球同服的基础。当需要根据用户量增长不停增加服务器时，可能都有个疑问：吞吐
量能提高吗？如果全部的请求都要求互斥的访问同一个数据，那么系统吞吐量怎么弄都是是
上不去的。我相信这个世界是天然并发的。一般来说用户请求都访问自己的数据（局部数据）。
多个请求是可以并发的。由于Zeze的分布式事务还没有大规模全球同服的实际应用，虽然
Zeze的设计上没有限制规模，但有可能会碰到分布式瓶颈。目前考虑到的瓶颈主要是全局
单点并且无法并发的模块（这个模块的操作都访问同一个数据）。

## 全局模块的并发
全球同服的系统里面有些模块可能是全局的。这些请求都访问同一个数据，肯定是互斥申请
锁排队执行。最高性能就是单个线程全速运行的事务数，这是有上限的。随着用户增长，请
求量可能超过上限。此时需要采取一些方案提高数据的并发度。这种全局模块需要提高并发，
常见的方式是按某种规则把数据分成多个部分，参考ConcurrentHashMap的实现。通常的
数据分块方式例子如下：

1.	记录数据很大，可以分成小块。这样就直接提高了并发。
2.	把数据分成多份（如果可以）。比如某个公司账号有大量的并发转账请求，此时可以建多个
      子账号。转入操作根据转入者Id的Hash选择某个子账号，这样转入就并发了。转出操作也
      按这个规则找到开始的子账号。由于该子账号可能金额不够，这是按顺序继续扣后面的子账
      号。此时转出访问了多个记录，这是没问题的。但是多数情况应该只需要访问一个子账号，
      不够的情况肯定是少的。读取操作可以分别显示子账号，或者统计一下。读取会导致执行转
      入账号的服务器的Cache降级到Share。读请求很多的时候，可以用一个定时更新的cache
      减少实际的数据访问量。
3.	上面Arch部分提到的RedirectHash就是一种把数据分组的规则，也是Zeze目前提供的一
      种策略。乐观估计，RedirectHash可以解决相当一部分全局单点模块并发问题。

## 高并发设计应该按需行动
如果可以预见请求量，并且代价不大时，可以一开始就优化并发性能。否则可以等到请求量
大到快无法支撑了再来优化。一开始实现一个支持任意请求量是没有必要的。计算机都是在
有限资源有限时间解决问题。

## Benchmark

* 单线程顺序事务

tasks/s=1495740.77 time=6.69s cpu=8.27s concurrent=1.24
ZezeJavaTest::ABasicSimpleAddOneThread.java 循环执行存
储过程估计被java强烈优化，所以这个数值有点高。

* 多线程并发强冲突事务

tasks/s=252613.37 time=3.96s cpu=9.84s concurrent=2.49
ZezeJavaTest::BBasicSimpleAddConcurrentWithConflict.java
强烈冲突意味着事务几乎总是重做，由于乐观锁重做时保持
锁定状态，只会重做一次，所以concurrent为2.49符合预期。

* 多线程并发一般冲突事务
tasks/s=1140652.44 time=4.38s cpu=15.58s
concurrent=3.55
ZezeJavaTest::CBasicSimpleAddConcurrent.java

* GlobalAsync

&gt;50w/s

* Global虚拟线程

&gt;15w/s 当前锁不匹配虚拟线程，否则应该接近Async

* GlobalWithRaft虚拟线程
5w/s
