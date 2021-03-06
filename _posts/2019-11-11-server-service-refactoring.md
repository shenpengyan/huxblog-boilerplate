---
layout:     post
title:      "服务端框架重构"
subtitle:   "server-service-framework-refactor"
date:       2019-11-11 12:00:00
author:     "沈歌"
header-img: "img/home-bg-o.jpg"
tags:
        - springboot
        - rose
---

由于公司业务还是使用的paoding rose + jade + resin的技术架构，导致新员工学习成本高，且框架本身已经很多年没人维护了，所以决定迁移至springboot2 + mybatis + tomcat.

## 前言

以下是我的迁移实践，任何开发一般都是分为三步：写代码、测试、监控。

很多程序员，并不关注测试和监控，这就是俗称的“管杀不管埋”，写完代码就认为结束了，默认程序已经好使了，问题全靠接口调用方反馈或者上线后再排查，一方面会影响个人的声誉，别人会觉得你不靠谱；另一方面容易给公司造成严重损失。

对于框架迁移，重构类的工作来说，测试和监控尤其重要，因为所有的接口，都有可能出问题。

## 准备工作

1) 调研springboot框架与paoding-rose框架的不同语法，尽可能找出代码中paoding-rose的特殊规则，列表进行对比。如paoding-rose的json数据需要加一个"@"符号，@Get，@Post这样的特殊标签，以及ControllerInterceptorAdapter这样的Intercepter等等。paoding-rose是基于spring的，底层实现是一样的，但是为了易用做了一层自己的包装和适配，对于一些项目中一些特殊用法，高度重视。

2) 调研mybatis和jade框架的异同。之前公司的jade框架是通过zookeeper获取配置，jade提供主从分离，分表查询等功能，但是mybatis原生并不支持；jade是基于注解sql的，mybatis要改成xml配置的形式。

3) 部署相关的异同。从resin到spring内置的tomcat，相关部署配置是否可以替换，如何适配公司的部署系统。

## 分阶段迁移

参考其他项目的迁移经验，决定分两阶段迁移，方便定位问题，确认影响。

第一阶段：迁移dao层，jade迁移到mybatis

第二阶段：paoding rose + resin迁移到springboot2 + 内置tomcat
主要为controller层及相关配置文件等。

### 迁移dao层

这一阶段遇到的问题主要是sql文件改写和主从分离。

#### 1) sql文件改写

sql文件改写主要是体力活，怎么在改写70多张表的sql情况下还不出错，真的很难。

前期计划使用[通用mapper](https://github.com/abel533/Mapper)这个框架来做，这样可以生成部分常用方法。但是后来深入了解后发现，业务sql都比较复杂，基本默认的sql不能用，而且使用通用mapper必须重新生成model文件，这样的话所有原来的model文件都需要更改，controller，service里的业务逻辑都得修改。使用通用mapper修改了几张表后，发现费时费力，而且对代码改动太大，风险太大。

因此放弃使用通用mapper，改为手写mybatis相关文件，之前已经自动生成了一堆mapper文件，里面的model字段名和数据库表字段名的映射map还是可以使用的，方便不少。

我是采用边改写，边自测的方式来做的，在test目录下写对比测试，对比新旧框架在相同请求参数下的返回值异常(select语句)。重复性劳动很容易粗心/疲劳导致出错，对比测试帮助我测出了很多问题。

#### 2) mybatis实现主从分离

这里我们采用的是AbstractRoutingDataSource + mybatis插件的形式，通过mybatis插件，拦截Executor的query方法和update方法，对所有的select语句，通过ThreadLocal变量的形式，指定走从库的DataSource。

### 迁移整体框架

迁移整体框架主要分为几部分：
1. Controller层的迁移，主要是注解和返回值。
对于注解，一位同事写了一个python脚本，将两个框架的对应注解进行了自动化改写，极大提升了工作效率。

2. 一些paoding rose的特殊使用。比如返回值@符号，代码里面一些其他用到老框架的地方。

返回值@前缀这个，让我深刻感受到了整洁代码的重要性。由于项目代码很老经手了很多人，到处都是@符号，代码搜索又不好匹配。在paoding rose框架里面，@符号是标识json结构返回的，返回数据会去掉它，但是springboot并不会，这种问题没有报错日志，一旦带到线上会导致客户端无法解析返回结果，问题非常严重。

先改了能找到的用老框架的地方，然后去掉maven依赖，找不到包的自然会报错，一个一个改。

3. 启动相关的配置

由于线上存在同一台机器部署多个节点服务的情况，因此服务端口、log配置文件必须是可配置的。这里我的实现是，log4j2.xml文件放在打的fat jar相同目录下。
项目中resource目录放一个application.yml文件，fat jar同级目录配置一个application.properties, 包含以下三个值的配置

```
server.port=<%= scope["config::http_port"] %>
server.tomcat.accesslog.directory=<%= scope["config::prog_logdir"] %>
logging.config=<%= scope["config::basedir"] %>/log4j2.xml

```
读取配置文件时，application.properties中的配置优先级更高，这样确保服务启动的端口、access日志目录和其他日志目录都是可配置化的。

代码改写之后，就是繁重的测试工作了。

首先还是对比测试，对所有重要接口都做了新老框架下接口的对比测试，黑盒对比，直接请求http请求对比返回值，请求参数尽可能抓包用真实数据。有些接口有特殊的行为逻辑，或者里面有一些动态参数，每次请求都不一样，这种就需要灵活处理了。总共对比了80个左右的接口（日请求量top80的接口）。

然后在测试机器上部署，使用客户端抓包看返回值是否有异常，同时观察服务器错误日志。（这里顺便提一句：error级别日志放到单独的文件里，这样查错误日志会非常方便）。因为是很老的项目，客户端经过了很多个版本迭代，最新版客户端可能已经不用这些代码了，一定是测不全的。

## 上线

在测试环境测试没有新问题产生，这种情况下就可以开始着手上线了。

迁移dao层时, 分为两个阶段，第一阶段上线了少数几张表，确认mybatis相关配置没有问题，之后又对所有表的替换做了上线。

线上共有60多个节点，先上线一个节点，观察错误日志，几个小时后没有什么sql相关异常抛出，再上线其他机器。

上线之后一定要及时观察各种报警和错误日志。

这里顺便提一下线上监控及日志的一些实践。

### 当前线上主要分两部分：

#### 机器和nginx

运维同学建立了机器指标监控看板和阈值报警，以及基于nginx日志的http请求的pivot统计。

比较重要的机器指标包括：
- cpu负载
- 内存剩余量
- /home分区磁盘剩余
- 跟分区磁盘剩余
- 入网流量
- 出王流量

比较重要的域名服务指标包括：
域名下http接口请求4xx，5xx的数量及占比

#### 服务端业务

业务由服务端同学负责，主要分两部分：
第一部分是metric监控及基于监控的报警。

第二部分是接入了公司的elk和druid数据。将业务日志，如接口请求日志，服务异常日志等打入elk中，可以方便的定位和查询问题。另外，服务端同学还基于elk日志做了Exception的报警机制，当某类型的Exception超过正常值，会触发短信邮件报警。

完善的监控和报警体系是重构的可靠保障，本次重构没有太大的线上问题，但是上线后靠监控和报警，发现了不少小问题，及时进行了修正。

## 总结

总体来说，这次重构比较顺利。

一方面得益于前期的充分准备和其他模块的部分经验，另一方面得益于对比测试和线上监控报警体系。

重构一个3000万日活，每天17亿请求，峰值qps32k的服务，一定要小心小心再小心。要确保不出问题，一定要做好对比测试和线上报警。

我的对比测试主要是dao层和接口级别的线下、线上对比测试，如果能够在线上空跑一段时间用真实流量做对比是最好的(成本比较高，且需要运维同学支持)。

另外：***有效的监控报警永远不嫌多***。





























