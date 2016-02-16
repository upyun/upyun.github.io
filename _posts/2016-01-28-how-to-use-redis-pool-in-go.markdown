---
layout: post
title:  "如何在 Go 语言中使用 Redis 连接池"
date:   2016-01-28 20:00:00
author: Akagi201
comments: true
---

## TOC

- [关于连接池](#关于连接池)
- [使用连接池遇到的坑](#使用连接池遇到的坑)
- [使用连接池的正确姿势](#使用连接池的正确姿势)
- [最后的解决方案](#最后的解决方案)
- [References](#References)

<a name="关于连接池"/>

## 关于连接池

一个数据库服务器只拥有有限的资源，并且如果你没有充分使用这些资源，你可以通过使用更多的连接来提高吞吐量。一旦所有的资源都在使用，那么你就不能通过增加更多的连接来提高吞吐量。事实上，吞吐量在连接负载较大时就开始下降了。通常可以通过限制与可用的资源相匹配的数据库连接的数量来提高延迟和吞吐量。

如果不使用连接池，那么，每次传输数据，我们都需要进行创建连接，收发数据，关闭连接。在并发量不高的场景，基本上不会有什么问题，一旦并发量上去了，那么，一般就会遇到下面几个常见问题：

* 性能普遍上不去
* CPU 大量资源被系统消耗
* 网络一旦抖动，会有大量 TIME_WAIT 产生，不得不定期重启服务或定期重启机器
* 服务器工作不稳定，QPS 忽高忽低

要想解决这些问题，我们就要用到连接池了。连接池的思路很简单，在初始化时，创建一定数量的连接，先把所有长连接存起来，然后，谁需要使用，从这里取走，干完活立马放回来。 如果请求数超出连接池容量，那么就排队等待或者直接丢弃掉。

<a name="使用连接池遇到的坑"/>

## 使用连接池遇到的坑

最近在一个项目中，需要实现一个简单的 `web server` 提供 `Redis` 的 `HTTP interface`，提供 `JSON` 形式的返回结果。考虑用 `Go` 来实现。

首先，去看一下 `Redis` 官方推荐的 [Go Redis driver](http://redis.io/clients#go)。官方 Star 的项目有两个：`Radix.v2` 和 `Redigo`。经过简单的比较后，选择了更加轻量级和实现更加优雅的 [Radix.v2](https://github.com/mediocregopher/radix.v2)。

`Radix.v2` 包是根据功能划分成一个个的 `sub-package`，每一个 `sub-package` 在一个独立的子目录中，结构非常清晰。我的项目中会用到的 `sub-package` 有 `redis` 和 `pool`。

由于我想让这种被 fork 的进程最好简单点，做的事情单一一些，所以，在没有深入去看 `Radix.v2` 的 `pool` 的实现之前，我选择了自己实现一个 `Redis pool`。这里，就不贴代码了。(后来发现自己实现的 `Redis pool` 与 `radix.v2` 实现的 `Redis pool` 的原理是一样的，都是基于 `channel` 实现的, 遇到的问题也是一样的。)

不过在测试过程中，发现了一个诡异的问题。在请求过程中经常会报 `EOF` 错误。而且是概率性出现，一会有问题，一会又好了。通过反复的测试，发现 bug 是有规律的，当程序空闲一会后，再进行连续请求，会发生3次失败，然后之后的请求都能成功，而我的连接池大小设置的是3。再进一步分析，程序空闲300秒后，再请求就会失败，发现我的`redis-server` 配置了`timeout 300`，至此，问题就清楚了。是连接超时 `redis-server` 主动断开了连接。客户端这边从一个超时的连接请求就会的到 EOF 错误。

然后我看了一下 `radix.v2` 的 `pool` 包的源码，发现这个库本身并没有检测坏的连接，并替换为新的连接的机制。也就是说我每次从连接池里面 `Get` 的连接有可能是坏的连接。所以，我当时临时的解决方案是通过增加失败后自动重试来解决了。不过，这样的处理方案，连接池的作用好像就没有了。技术债能早点还的还是早点还上。

<a name="使用连接池的正确姿势"/>

## 使用连接池的正确姿势

想到我们的 `ngx_lua` 项目里面也大量使用 `redis` 连接池，他们怎么没有遇到这个问题呢。只能去看看源码了。

经过抽象分离， `ngx_lua` 里面使用 `redis` 连接池部分的代码大致是这样的

~~~
server {
    location /pool {
        content_by_lua_block {
            local redis = require "resty.redis"
            local red = redis:new()

            local ok, err = red:connect("127.0.0.1", 6379)
            if not ok then
                ngx.say("failed to connect: ", err)
                return
            end

            ok, err = red:set("hello", "world")
            if not ok then
                return
            end

            red:set_keepalive(10000, 100)
        }
    }
}
~~~

发现有个 `set_keepalive` 的方法，查了一下[官方文档](https://github.com/openresty/lua-resty-redis#set_keepalive)，方法的原型是 `syntax: ok, err = red:set_keepalive(max_idle_timeout, pool_size)` 貌似 `max_idle_timeout` 这个参数，就是我们所缺少的东西，然后进一步跟踪源码，看看里面是怎么保证连接有效的。

~~~
function _M.set_keepalive(self, ...)
    local sock = self.sock
    if not sock then
        return nil, "not initialized"
    end

    if self.subscribed then
        return nil, "subscribed state"
    end

    return sock:setkeepalive(...)
end
~~~

至此，已经清楚了，使用了 `tcp` 的 `keepalive` 心跳机制。

于是，通过与 `radix.v2` 的作者一些[讨论](https://github.com/mediocregopher/radix.v2/issues/21)，选择自己在 `redis` 这层使用心跳机制，来解决这个问题。

<a name="最后的解决方案"/>

## 最后的解决方案

在创建连接池之后，起一个 `goroutine`，每隔一段 `idleTime` 发送一个 `PING` 到 `redis-server`。其中，`idleTime` 略小于 `redis-server` 的 `timeout` 配置。

连接池初始化部分代码如下：

~~~
p, err := pool.New("tcp", u.Host, concurrency)
errHndlr(err)
go func() {
    for {
        p.Cmd("PING")
        time.Sleep(idelTime * time.Second)
    }
}()
~~~

使用 `redis` 传输数据部分代码如下：

~~~
func redisDo(p *pool.Pool, cmd string, args ...interface{}) (reply *redis.Resp, err error) {
	reply = p.Cmd(cmd, args...)
	if err = reply.Err; err != nil {
		if err != io.EOF {
			Fatal.Println("redis", cmd, args, "err is", err)
		}
	}

	return
}
~~~

其中，`radix.v2` 连接池内部进行了连接池内连接的获取和放回，代码如下：

~~~
// Cmd automatically gets one client from the pool, executes the given command
// (returning its result), and puts the client back in the pool
func (p *Pool) Cmd(cmd string, args ...interface{}) *redis.Resp {
	c, err := p.Get()
	if err != nil {
		return redis.NewResp(err)
	}
	defer p.Put(c)

	return c.Cmd(cmd, args...)
}
~~~

这样，我们就有了 `keepalive` 的机制，不会出现 `timeout` 的连接了，从 `redis` 连接池里面取出的连接都是可用的连接了。看似简单的代码，却完美的解决了连接池里面超时连接的问题。同时，就算 `redis-server` 重启等情况，也能保证连接自动重连。

<a name="References"/>

## References
* [Connection pool](https://en.wikipedia.org/wiki/Connection_pool)
* [Database connection](https://en.wikipedia.org/wiki/Database_connection)
* [OpenResty 连接池](http://wiki.jikexueyuan.com/project/openresty/web/conn_pool.html)
