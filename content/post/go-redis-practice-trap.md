---
title: "go-redis 实践中遇到的一个坑"
date: 2021-05-28T15:22:48+08:00
draft: true
tags: ["go"]
categories: [""]
---


我们的一个线上服务既使用到了redis单点同时也使用到了redis集群，因此在服务中既有redis client又有cluster client。

## 问题表现

线上服务的机器中总是会保留有一定数量的状态为`CLOSE_WAIT`的tcp连接,而且占比很高。

```shell
  1692 CLOSE_WAIT
  4913 ESTABLISHED
    4 LISTEN
    1 SYN_SENT
    75 TIME_WAIT
```

关于CLOSE_WAIT连接出现的原因，已经建立的tcp连接，被动关闭的一方收到对方发来的FIN信号，就会进入CLOSE_WAIT状态，同时回复ACK。当己方关闭这个断开的连接后，CLOSE_WAIT状态消失（转移到别的状态，直至连接完全关闭）。更具体的信息请查阅tcp的4次挥手。

线上有很多CLOSE_WAIT状态的连接，说明是与这个服务关联的其他服务主动断开了连接，而对应的连接没有被及时关闭。
与这个服务关联的其他服务有它的调用方和redis server，所以可能是它们二者之一主动断开了连接（也可能是二者皆有）。

然后通过抓包验证了是redis server主动断开了空闲的连接（redis server设置了连接空闲5分钟超时）。

```shell
15:24:10.345156 IP ip1.28648 > ip2.8028: Flags [.], ack 4305, win 58, options [nop,nop,TS val 874347092 ecr 1924047787], length 0
15:24:11.953025 IP ip2.8028 > ip1.28648: Flags [F.], seq 4305, ack 34906, win 57, options [nop,nop,TS val 1924049479 ecr 874347092], length 0
15:24:11.992657 IP ip1.28648 > ip2.8028: Flags [.], ack 4306, win 58, options [nop,nop,TS val 874348740 ecr 1924049479], length 0
15:25:19.780244 IP ip1.28648 > ip2.8028: Flags [F.], seq 34906, ack 4306, win 58, options [nop,nop,TS val 874416527 ecr 1924049479], length 0
15:25:19.780365 IP ip2.8028 > ip1.28648: Flags [R], seq 1599025402, win 0, length 0
```

但是本机并没有及时返回FIN信号，也就是没有及时关闭这个连接。

## 寻找原因

问题基本弄清楚了，下面就要确认原因了。

首先想到的就是go-redis的连接池实现，在我之前的文章[go-redis 连接池的实现](https://charter652.netlify.app/post/go-redis-conn-pool)中学习过相关内容。

在连接池中涉及到空闲连接和过期连接清理的就是`reaper`函数，这个函数的启动依赖于两个参数`IdleTimeout`和`IdleCheckFrequency`，在单点的client options中，这两个参数是有默认值的，也就是说即使不显式的设置这两个参数，因为其有默认值也会致使reaper协程的启动。

接下来就是坑点了，因为我们没有仔细阅读ClusterOptions的源码，一开始以为cluster client中，这两个参数的行为与单点的client中是一致的，所以在线上的服务中都没有设置这两个参数。

但其实在cluster client中，reaper协程是在`NewClusterClient()`函数中启动的，且在没有设置`IdleCheckFrequency`参数的情况下，默认是不启动该协程的，也就是不会自动去检查空闲超时和过期的连接。若连接已经断开，只能是等到实际请求中取到该连接发现已经不可用了才会去关闭它。

```go
func NewClusterClient(opt *ClusterOptions) *ClusterClient {
	opt.init()

	c := &ClusterClient{
		clusterClient: &clusterClient{
			opt:   opt,
			nodes: newClusterNodes(opt),
		},
		ctx: context.Background(),
	}
	c.state = newClusterStateHolder(c.loadState)
	c.cmdsInfoCache = newCmdsInfoCache(c.cmdsInfo)
	c.cmdable = c.Process

	if opt.IdleCheckFrequency > 0 {
		go c.reaper(opt.IdleCheckFrequency)
	}

	return c
}
```

这就导致了线上大量的`CLOSE_WAIT`状态的连接存在。


定位到原因之后，解决方法也就简单了，就是在初始化cluster  client时，设置`IdleTimeout`和`IdleCheckFrequency`两个参数。
