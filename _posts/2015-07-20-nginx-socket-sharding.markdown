---
layout: post
title: NGINX 1.9.1 新特性：套接字端口共享
date: 2015-07-20 20:13:00
author: hy05190134
comments: true
---

> 原文地址： <https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/>

NGINX 1.9.1 发布版本中引入了一个新的特性 —— 允许套接字端口共享，该特性适用于大部分最新版本的操作系统，其中也包括 DragonFly BSD 和内核 3.9 以后的 Linux 操作系统。套接字端口共享选项允许多个套接字监听同一个绑定的网络地址和端口，这样一来内核就可以将外部的请求连接负载均衡到这些套接字上来。（对于 [NGINX Plus](https://www.nginx.com/) 的用户来说，该特性将会在年底发布的版本 7 中得到支持）

实际上很多潜在的用户希望使用端口重用功能。其他服务也可以简单的进行热更新升级（NGINX 支持 [多种方式](http://nginx.org/en/docs/control.html#upgrade) 的热更新）。对于 NGINX 来说，打开该选项可以通过在特定场景下减少连接锁竞争而提高性能。

如下图所示，在该选项没有生效的前提下，一个单独的监听套接字会通知所有的工作进程，每个进程则会试图争抢接管某个连接：

![NGINX+LUA]({{ site.remoteurl }}/assets/trandition-listen.png!/fw/600)

当该选项生效时，这个时候对于每个网络地址和端口就会有多个监听套接字，每个工作进程对应一个套接字，内核会决定由哪个监听套接字（也就是决定哪个工作进程）接管进来的连接。这个特性可以减少进程与进程之间为接收连接产生的锁竞争而提高多核系统的性能。但是，如果当一个工作进程处于阻塞操作时，这个时候不仅会影响已经被该进程接收的连接，还会阻塞由系统准备分配给该进程的连接请求：

![NGINX+LUA]({{ site.remoteurl }}/assets/new-listen.png!/fw/600)

### 配置套接字共享

如下面配置所示，可以通过在 listen 指令后添加新的参数 reuseport 来为 HTTP 或者 TCP（**流**模块）打开套接字端口共享功能：

~~~
http {
    server {
        listen 80 reuseport;
        server_name  localhost;
        ...
    }
}

stream {
    server {
        listen 12345 reuseport;
        ...
    }
}
~~~

对于套接字来说，添加 reuseport 参数也就等于禁止了 [accept\_mutex](http://nginx.org/en/docs/ngx_core_module.html#accept_mutex) 指令了，因为 mutex 对于 reuseport 来说是多余的，不过对于那些不希望使用套接字端口共享特性的端口来说，我们则有必要设置 accept\_mutex。

### reuseport 性能压力测试

我通过运行 [wrk](https://github.com/wg/wrk) 压测工具来压测一个跑在 36 核亚马逊实例上并开启 4 个工作进程的 NGINX。为了减少网络对于测试结果的影响，我们的客户端和 NGINX 都是运行在本地的，并且 NGINX 只返回 `OK` 字符串而不返回文件。我对比了三个不同的配置，一个是默认的（等同于 accept\_mutex on），另一个是 accept_mutex off，还有一个则是 reuseport。正如下图所示，使用 reuseport 后每秒能处理的请求数比其他的两个增加了 2-3 倍，并且同时减少了平均延迟和平均延迟的标准差。

![NGINX+LUA]({{ site.remoteurl }}/assets/bench-result.jpg)

我还尝试客户端和 NGINX 分别运行在不同的主机上跑相关的压力测试，但这一次 NGINX 返回的是一个 HTML 文件。正如下表所示，与上图的结果类似，使用端口重用能减少请求的处理延迟，并且延迟标准差减少的更加明显（大概是 10 倍）。其他结果（并没有在下表中显示）则更加让人欣喜。通过端口重用，请求的压力被各个工作进程均衡掉，而在默认条件下（也就是 accept\_mutex 打开），一些工作进程会得到比较高的负载，而在 accept\_mutex 关闭的时候，所有的进程都会显示负载比较高。

![NGINX+LUA]({{ site.remoteurl }}/assets/result-table.jpg)

在上述的压力测试中，请求的频率很高但是每个请求并不需要复杂的处理。其他一些初步的测试也表明端口重用特性在网络流量匹配的时候最能提高性能。（该参数并不能在邮件模块的上下文监听指令中生效，因为邮件的网络流量并不能匹配这个特性）。我们建议你在使用 reuseport 这个指令前先在你的 NGINX 实例上进行性能测试，而不是全部的 NGINX 实例都使用。对于测试 NGINX 的一些小提示，可以参考 [Konstantin Pavlov](http://www.youtube.com/watch?v=eLW_NSuwYU0) 在 2014 NGINX 大会上的演讲。

### 致谢

感谢在英特尔工作的卢英奇和 Sepherosa Ziehau，他们每个人各自贡献了使得套接字端口共享生效的解决方案。NGINX 开发小组合并了他们俩的想法从而创造出目前看来比较理想的解决方案。
