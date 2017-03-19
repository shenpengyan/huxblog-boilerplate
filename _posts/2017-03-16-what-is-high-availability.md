---
layout:     post
title:      "（转）究竟啥才是互联网架构“高可用”"
subtitle:   "what is High Availiability"
date:       2017-03-16 20:42:00
author:     "沈歌"
header-img: "img/home-bg-o.jpg"
tags:
- 高可用
- 架构
---


### 一、什么是高可用

高可用HA（High Availability）是分布式系统架构设计中必须考虑的因素之一，它通常是指，通过设计减少系统不能提供服务的时间。

假设系统一直能够提供服务，我们说系统的可用性是100%。

如果系统每运行100个时间单位，会有一个时间单位无法提供服务，我们说系统的可用性是99%。很多公司的高可用目标是4个9，也就是99.99%,这就意味着，系统的年停机时间为8.76个小时。百度的搜索首页，是业内公认搞高可用保障非常出色的系统，甚至人们会通过<www.baidu.com>能不能访问来判断“网络的连通性”。前两天手机百度宕机4小时，人们一度以为是自己的网络坏了。这是对百度HA最高的褒奖了。

### 如何保障系统的高可用

我们都知道，单点是系统高可用的大敌，应该尽量在系统设计的过程中避免单点。方法论上，高可用保证的原则是“集群化”，或者叫“冗余”：只有一个单点，挂了服务会受影响。如果有冗余备份，挂了还有其他backup能够顶上。

保证系统高可用，架构设计的核心准则是：冗余。

有了冗余还不够，每次出现故障需要人工介入恢复势必会增加系统的不可服务时间。所以，又往往是通过“自动故障转移”来实现系统的高可用。

接下来我们看下典型互联网架构中，如何通过冗余+自动故障转移来保证系统的高可用特性。

### 常见的互联网分层架构

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/layer-architecture.jpeg)

常见互联网分布式架构如上分为：

**（1） 客户端层**：典型调用方是浏览器browser或者手机应用APP

**（2） 反向代理层**：系统入口，反向代理。

**（3） 站点应用层**：实现核心应用逻辑，返回html或者json

**（4） 服务层**：如果实现了服务化，就有这一层

**（5） 数据-缓存层**： 缓存加速访问存储

**（6） 数据-数据库层**：数据库固化数据存储

整个系统的高可用，又是通过每一层的冗余+自动故障转移来综合实现的。

### 四、分层高可用架构实践

##### 【客户端层-> 反向代理层】的高可用

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/client-to-proxy.png)

【客户端层】到【反向代理层】的高可用，是通过反向代理层的冗余来实现的。以nginx为例：有两台nginx，一台对线上提供服务，另一台冗余以保证高可用，常见的实践是keepalived存货探测，相同virtual IP提供服务。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/client-to-proxy-down.png)

自动故障转移，当nginx挂了的时候，keepalived能够探测到，会自动的进行故障转移，将流量自动迁移到sandow-nginx，由于使用的是相同的virtual IP，这个切换过程对调用方是透明的。

##### 【反向代理层-> 站点层】的高可用

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/proxy-to-website.png)

【反向代理层】到【站点层】d嗯高可用，是通过站点层的冗余来实现的。假设反向代理层是nginx，nginx.conf里能配置多个web后端，并且nginx能够探测到多个后端的存活性。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/proxy-to-website-down.png)
自动故障转移：当web-server挂了的时候，nginx能给探测到，会自动的进行故障转移，将流量自动迁移到其他的web-server,整个过程由nginx自动完成，对调用方是透明的。

#### 【站点层-> 服务层】的高可用

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/website-to-service.png)

【站点层】到【服务层】的高可用，是通过服务层的冗余来实现的。“服务连接池”会建立与下游服务的多个连接，每次请求会“随机”选取连接来访问下游服务。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/website-to-service-down.png)

自动故障转移：当service挂了的时候，service-connection-pool能够探测到，会自动的进行故障转移，将流量自动迁移到其他的service，整个过程由连接池自动完成，对调用方是透明的（所以说RPC-client中的服务连接池是很重要的基础组件）

#### 【服务层-> 缓存层】的高可用

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/service-to-cache.png)

【服务层】到【缓存层】的高可用，是通过缓存数据的冗余来实现的。

缓存层的数据冗余又有几种方式：一种是利用客户端的封装，service对cache进行双读或者双写。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/service-to-cache-redis.png)

缓存层也可以通过支持主从同步的缓存集群来解决缓存层的高可用问题。

以redis为例，redis天然支持主从同步，redis官方也有sentinel哨兵机制，来做redis的存活性检测。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/service-to-cache-redis-down.png)

自动故障转移：当redis主挂了的时候，sentinel能够探测到，会通知调用方访问新的redis，整个过程由sentinel和redis集群配合完成，对调用方是透明的。

说完缓存的高可用，这里要多说一句，业务对缓存并不一定有“高可用”的要求，更多的对缓存的使用场景，是用来“加速数据访问”：把一部分数据放到缓存里，如果缓存挂了或者缓存没有命中，是可以去后端的数据库中再取数据的。

这类允许“cache miss”的业务场景，缓存架构的建议是：

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/cache-miss.png)

将kv缓存封装成服务集群，上游设置一个代理（代理可以用集群冗余的方式保证高可用），代理的后端根据缓存访问的key水平切分成若干个实例，每个实例的访问并不做高可用。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/cache-miss-down.png)

缓存实例挂了屏蔽：当有水平切分的实例挂掉时，代理层直接返回cache miss，此时缓存挂掉对调用方也是透明的。key水平切分实例减少，不建议做re-hash，这样容易引发缓存数据的不一致。

#### 【服务层>数据库层】的高可用

大部分互联网技术，数据库层都用了“主从同步，读写分离”架构，所以数据库层的高可用，又分为“读库高可用”和“写库高可用”两类。

#### 【服务层> 数据库层“读”】的高可用

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/service-to-db-read.png)

【服务层】到【数据库读】的高可用，是通过读库的冗余来实现的。既然冗余了读库，一般来说就至少有两个从库，“数据库连接池”会建立与读库多个连接，每次请求会路由到这些读库。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/service-to-db-read-down.png)

自动故障转移：当读库挂了的时候，db-connection-pool能够探测到，会自动的进行故障转移，将流量自动迁移到其他的读库，整个过程由连接池自动完成，对调用方是透明的（所以说DAO中的数据库连接池是很重要的基础组件）

#### 【服务层> 数据库层“写”】的高可用

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/service-to-db-write.png)

【服务层】到【数据库写】的高可用，是通过写库的冗余来实现的。以mysql为例，可以设置两个mysql双主同步，一台对线上提供服务，另一台冗余以保证高可用，常见的实践是keepalived存活探测，相同virtual IP提供服务。

![](http://shenpengyan.github.io/img/in-post/what-is-high-availiability/service-to-db-write-down.png)

自动故障转移：当写库挂了的时候，keepalived能够探测到，会自动的进行故障转移，将流量自动迁移到shadow-db-master，由于使用的是相同的virtual IP，这个切换过程对调用方是透明的。

#### 五、总结

高可用HA（High Availiability）是分布式系统架构设计中必须考虑的因素之一。它通常是指，通过设计减少系统不能提供服务的时间。

方法论上，高可用是通过**冗余+自动故障转移**来实现的。

整个互联网分层系统架构的高可用，又是通过每一层的冗余+自动故障转移来综合实现的，具体是指：

1. 【客户端层】到反向代理的高可用，是通过反向代理层的冗余实现的，常见实践是keepalived+virtual IP自动故障转移
2. 【反向代理层】到【站点层】的高可用，是通过站点层的冗余实现的，常见实践是nginx与web-server之间的存活性探测和自动故障转移
3. 【站点层】到【服务层】的高可用，是通过服务层的冗余实现的，常见实践是通过service-connection-pool来保证自动故障转移。
4. 【服务层】到【缓存层】的高可用，是通过缓存数据的冗余实现的，常见实践是缓存客户端双读双写，或者李勇缓存集群的主从数据同步与sentinel保活与自动故障转移；更多的业务场景，对缓存没有高可用要求，可以使用缓存服务化来对调用方屏蔽底层复杂性。
5. 【服务层】到【数据库“读”】的高可用，是通过读库的冗余实现的，常见实践是通过db-connection-pool来保证自动故障转移
6. 【服务层】到【数据库“写”】的高可用，是通过写库的冗余实现的，常见实践是keepalived + virtual IP自动故障转移。


> 转自：58沈剑的《架构师之路》公众号，[究竟啥才是互联网架构“高可用”](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651959728&idx=1&sn=933227840ec8cdc35d3a33ae3fe97ec5&chksm=bd2d046c8a5a8d7a13551124af36bedf68f7a6e31f6f32828678d2adb108b86b7e08c678f22f&scene=21#wechat_redirect)





