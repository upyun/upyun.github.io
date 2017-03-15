---
layout: post
title:  "mqkv 一个通用的基于分布式消息队列的分布式存储"
date:   2016-09-21 17:50:00
author: Akagi201
comments: true
---

公司直播系统的源站集群需要一个中心存储服务来提供流的元数据信息的存储和查询功能。由于源站集群是部署在全国各地的多个机房, 多个节点上的。所以, 这里是一个典型的分布式存储的应用场景。这个服务的稳定性非常重要, 也直接影响到直播系统整个服务整体的可用性。

下面, 我来与大家分享交流一下, 我们源站集群共享存储方案经历了哪些变化, 最后, 介绍下我们的下一代共享存储方案 mqkv, 也欢迎熟悉分布式与存储的小伙伴提提建议。

由于, 我们公司的 CDN 系统是使用 Redis 的主从同步机制来进行配置元数据的同步和分发到全国各个 CDN 节点的, 我们对 Redis 的各种特性比较熟。所以, 这里, 我们最先考虑的也是 Redis 的方案。

## 应用场景

分布式存储也是一个很宽泛的概念, 可以选择的技术很多。首先, 要分析好我们的应用场景是怎样的, 哪些是我们 care 的, 哪些是我们不太 care 的。

推流源站会对流的元数据信息进行写操作, 而拉流源站会进行流的元数据的读操作。而直播是个典型的一对多提供服务的场景, 所以, 相应的, 我们的共享存储方案也是写少读多。同时, 我们可以允许少量的数据丢失, 我们更看重的是服务无SPOF, 稳定性与读写性能。

## 方案一: Redis Cluster

Redis 3.0 之后推出了自己的 Redis Cluster 集群方案, 所以, 在最开始, 我们还是优先去尝试官方的集群方案。但是, 我们发现 Redis Cluster 仍然不能实现跨机房容灾, 跨机房高可用的功能还是需要自己来实现。所以, 我们源站集群共享存储的最初版本是每台源站的读写都是去操作部署在一个 BGP 机房的 Redis Cluster。但这个方案会导致读写性能都不理想, 所以, 后面我们考虑了在程序中引入缓存, 来减少读压力。

## 方案二: Redis Cluster + TTL MemoryCache

为了优化读性能, 我们首先考虑在程序中加入缓存, 结合我们的业务场景, 我们开发引入了基于 [groupcache](https://github.com/golang/groupcache) 的超时缓存方案。我们比较专注最新的数据, 过期的数据没有意义, 反而会影响我们的业务逻辑, 所以, 元数据的每个 key 都可以配置一个 TTL 过期时间, 当时间到达时, 这个 key 就会 expire 掉。在一个 key expire 的同时有大量访问这个 key 的请求这个临界点时, [groupcache](https://github.com/golang/groupcache) 的内部有锁机制保障, 不会出现大量的回源请求, 给中心存储造成压力, 这个方案在我们的测试环境中, 测试结果跟我们预期基本一致。

## 方案三: Redis Master/Slave 读写分离

而其实我们上面的方案并没有上线, 就有人提出了 Redis 读写分离的方案。写到一个中心 Redis Master 节点, 然后每台源站只去读本机的 Redis Slave 节点, 通过 Redis 的主从同步机制来确保数据一致性。最开始没有用这套方案是因为 Redis 的主从同步机制与 Redis Cluster/Sentinel 有冲突, 不能共存。后来, 运维提供了通过 keepalived 来保证 redis 的高可用, 所以, 我们线上采用了这套方案, 通过运行实际效果来看, 比较理想。

## 方案四: etcd

我们也在调研一些其他的分布式 kv 存储的方案, 下面是 etcd 的 benchmark 结果。

![etcd_test]({{ site.remoteurl }}/assets/etcd_test.png)

etcd 并发量10, 100, 1000 分别测试 PUT, GET, DELETE 连续三个操作。结果平均响应时间也跟着上去了。这个结果我们是无法接受的, 所以, 这个方案没有再继续深入研究了。

## 方案五: mqkv

在对现有的一些分布式存储以及集群方案测试结果非常失望后, 我们开始考虑自研适用于我们这种业务场景的分布式存储方案。

我们预期要达到的效果:

* 每台源站读写操作都在本地, 要有较好的读写性能。写操作可以异步化, 尽快返回。
* 无中心节点, 无 SPOF。(写在一个中心的方案, 写的这个主 Redis 还是一个单节点, 一旦机房断网或者断电, 那么, 整个直播服务就不可用了)
* 允许出现少量的写数据失败的情况。

基于以上几点我设计了 mqkv:
* 支持跨机房部署, 避免的单点问题。
* 读写操作都在同一个源站的本机, 读写性能均达到最佳。
* mqkv 提供给应用的接口使用的协议是 Redis 协议, 兼容大量的 redis client driver。

## mqkv 架构图

![mqkv_arch]({{ site.remoteurl }}/assets/mqkv_arch.png)

## 实现过程

首先, 我实现了一个 Go 语言版的 redis server 的 api 框架 [RedFace](https://github.com/Akagi201/redface)。设计主要参考了 `net/http` 的接口, 由于, 目前的业务逻辑还比较简单, 所以, 太复杂的代码并不多。

比较巧的是, 在我实现了这个包的那个周末, 我看到了 hacker news 上有个跟我的项目功能非常类似的一个项目上了头条, 叫 [redcon](https://github.com/tidwall/redcon)。不过从接口可以明显的看出, 我实现的版本接口更加简洁, 友好。具体地, 可以对比下 redcon 的 [example](https://github.com/tidwall/redcon/blob/master/example/clone.go) 和 redface 的 [example](https://github.com/Akagi201/redface/blob/master/example/clone/main.go)

不过, redcon 的 benchmark 性能确实比我实现的要好, 这里, 我暂时还没有找到具体的原因, 哈哈, 欢迎高手帮忙分析下。

接下来, 就是 mqkv 的实现了, 其实在架构与逻辑确定好了, 轮子也造好了之后, 写代码就变成很简单的事情了。简单的说, 我就是将应用的 write 操作都异步化, 通过分布式消息队列将消息发送出去, read 操作直接 proxy 本地的 kv 存储。其中利用了 nsq 的 PUB/SUB 模型, 所有 write 操作都 produce 到 `mqkv_topic` 这个 topic 下, 同时, 每个 mqkv 也作为消费者注册消费 topic 为 `mqkv_topic`, channel 为本机 hostname 的消息。这样, 就实现一写多读的消息分发模型了。每个 mqkv 在本地的 redis 进行全量的 kv 存储, 这里的 Redis 连接, 我也是用了 [Redis 中间件](https://godoc.org/github.com/upyun/utilgo/radixutil) 来兼容 normal redis/redis sentinel/redis cluster 各种集群与高可用 redis 方案。

这样, mqkv 就完成了, 是不是很简单。

## benchmark
* 当 mqkv 启动后, 其实, 对于应用来说, 他本身就化身成为了一个 normal redis, 所以, 可用 `redis-benchmark` 进行压测。

```
❯ redis-benchmark -p 6389 -t set,get -n 1000000 -q -P 512 -c 512

SET: 28659.04 requests per second

GET: 23171.21 requests per second

```

以上是在我的 macbook pro 上性能测试结果。

## 监控
* 支持 pprof 性能监控: `GET /debug/pprof/profile`
* 支持 stats channel 信息 api: `GET /api/v1/consumer_stats` 可以参看当前 mqkv 自己所连 channel 的消息消费情况。

## 其他的一些还需解决的问题

当然 mqkv 还存在很多不完美的地方。

* nsq 部署依赖 dns, 需要将 机器的 hostname 与 ip 关系写到 `/etc/hosts` 里面。不知道有没有更简单的方法。
* 机器扩容, 源站宕机一段时间后恢复, 元数据如果恢复, 如何保证数据一致性。
* 也在考虑开发一个 kafka 版本进行对照, 看是否能保证更好的数据一致性。
