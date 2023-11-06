---
title: "[openresty/nginx实践]P998延时从45ms降到15ms"
date: 2023-11-06T15:11:58+08:00
draft: false
tags: ["openresty"]
categories: [""]
---

## 问题背景

系统使用openresty对外提供API，客户端过来的请求先到openresty，经过openresty转发给其后端的go服务。

流量特点是高并发，且要求低时延，除去网络时延的时间，请求要在20-40ms之内返回。

系统上线之后观察监控发现，P95：:5ms，P99：:20ms，但是P998达到了45ms。虽然总体上是满足性能要求的，但考虑到目前流量规模还不大，后续一旦规模增长有可能出现大量超时现象，还是应该对其做一些优化。


![系统时延-优化前](/images/rta_adapter_p998_1.png)


## 排查过程

首先想到是否是后端的go服务存在不稳定的情况，于是完善监控面板，发现后端服务的P998时延为10ms，遂排除这一因素。

那问题就出在openresty/nginx这一层了。

> 此处省略一些徒劳的排查过程。。。😳

在access log中我们记录的nginx的`$connection`与`$connection_requests`这两个变量。其中`$connection`表示当前的连接号，`$connection_requests`表示当前连接处理过的请求数量。

过滤出响应时间超过30ms的请求记录，观察发现这些记录都发生在`$connection_requests`达到100的时候，然后对应的连接就被关闭了。


![access_log](/images/rta_adapter_access_log.png)

所以应该是连接关闭导致了这些高时延的请求出现。

那么为什么会是100呢？keep-alive的长连接会受到哪些nginx配置项的影响呢？


遇事不决，问问大模型(ChatGPT)吧😄，见下图：

![聊天截图](/images/rta_adapter_keepalive_requests.png)


了解一下`keepalive_requests`配置项，官方描述如下：

>Syntax:	keepalive_requests number;
>
>Default:	keepalive_requests 1000;
>
>Context:	http, server, location
>
>Sets the maximum number of requests that can be served through one keep-alive connection. After the maximum number of requests are made, the connection is closed.
>
>Closing connections periodically is necessary to free per-connection memory allocations. Therefore, using too high maximum number of requests could result in excessive memory usage and not recommended.
>
>**Prior to version 1.19.10, the default value was 100.**

它用于设置一个`keep-alive`连接上可以处理的请求的最大数量。当达到最大请求数量时，连接被关闭。且在`1.19.10`版本之前，这个值默认为100. 之后默认值为1000.

而我们用的版本是openresty 1.15，且之前忽略了这项配置。。。

在高QPS的场景下，100就显得太低了，这会导致长连接被频繁的关闭、新建。从另一个角度看，机器上也会存在大量的`TIME_WAIT`状态的TCP连接。

参考官方默认值将其设置为1000，上线后观察指标，P998降到了15ms左右。


![系统时延-优化后](/images/rta_adapter_p998_2.png)
