---
title: "[openresty实践]业务缓存架构的演进"
date: 2020-07-30T14:16:46+08:00
draft: false
tags: ["openresty"]
categories: [""]
---

## 背景

使用openresty提供对外API，接收json/protobuf格式的各种数据，并转换为系统内部统一的格式，然后转发给其他服务；

在openresty接收请求并返回的过程中需要一些业务上的配置数据，这些数据分成几个类别，一共能有几百兆的大小。

## 最初的方案

借助crontab任务同步数据，将数据落到openresty本地文件，然后触发nginx的reload，
在openresty的init阶段通过一个全局变量将数据加载到内存中以供使用。

#### 问题

nginx的reload方式并不会使对外的API服务不可用，但是因为openresty还依赖了其他内部服务，
各个worker都与内部服务保持了很多连接。nginx reload过程中会停掉旧的worker，启动
新的worker，这样就会导致同一时间新建很多连接。

在系统流量、并发量小的时候这个机制并不会给系统带来明显的影响，但随着流量、并发量
增大，每次reload都会使系统产生很多请求、连接超时。


## 思路

接下来就是思考如何针对上述情况做优化。

在openresty中做缓存，常用的组件有三个：shared dict；resty-lrucache；resty-mlcache。

其中shared dict缓存的数据只有一份，各个worker共享这份数据，好处就是内存空间占用小。
同时shared dict只能缓存字符串，如果有table等复杂数据结构的需求，还需要进行数据转换。

[lrucache](https://github.com/openresty/lua-resty-lrucache)可以缓存所有lua对象，但是只能在单个worker进程内访问，因此数据也是每个worker
内单独缓存，有多少个worker就有多少个副本。

[mlcache](https://github.com/thibaultcha/lua-resty-mlcache)是结合了shared dict和lrucache的一个多级缓存方案。


## 尝试

#### shared dict

首先从开发量最小的方式入手，将shared dict替换原始方案中的全局变量。每次服务启动时读取文件，
将数据写入shared dict，然后利用一个worker启动定时器，定时更新这部分数据。

由于shared dict使用的是共享内存，每次操作都需要全局锁，这样在高并发情况下很容易
引起worker之间的竞争。而实际上在实践中也确实出现了worker间竞争的问题，直接导致服务
卡死不可用。


#### lrucache

接着使用lrucache，同样是在init阶段读取文件中的数据加载到lrucache中，不同的是需要在
每个worker中都执行定时任务，以更新自身进程内缓存的数据。

这种方式避免了每次更新缓存都需要reload的问题，能够满足需求。但也存在它的问题。

一方面，nginx的每个worker都是单进程运行，加上磁盘IO操作又是阻塞的，因此在执行定时
任务读取文件的过程中，该worker中的请求就要等待缓存更新完才能被处理，这在一定程度上也
会影响系统。

那么有没有非阻塞的读取文件的方法呢？答案是有。有一个第三方的C模块lua-io-nginx-module，
它利用了 Nginx 的线程池，把磁盘 I/O 操作从主线程转移到另外一个线程中处理，这样，主线程就不会因为磁盘 I/O 操作而被阻塞。
但是要使用它需要重新编译nginx，而且它的GitHub主页上强调还是实验阶段并且已经2年没有更新了。
所以这不是一个好的方法。

另一方面我们的数据大小达到几百兆，在更新lrucache中缓存的过程中会创建、删除大量的table，
虽然luajit有自己的GC，但它并不会把回收的内存都释放给操作系统，这就导致worker进程
的内存占用量在开始会逐步增加并稳定在一个较高的水平，达到1.6G，这个占用量也太高了。

#### mlcache + redis

将数据更新落文件改为写入redis，并设置TTL。openresty内部只使用mlcache的get()方法。
当自身的缓存数据过期后，通过callback函数去访问redis获取。

```lua
local function callback(username)
    -- this only runs *once* until the key expires, so
    -- do expensive operations like connecting to a remote
    -- backend here. i.e: call a MySQL server in this callback
    return db:get_user(username) -- { name = "John Doe", email = "john@example.com" }
end

-- this call will try L1 and L2 before running the callback (L3)
-- the returned value will then be stored in L2 and L1
-- for the next request.
local user, err = cache:get("my_key", nil, callback, "John Doe")
```

这样既解决了读文件的阻塞问题，又通过被动更新的方式更新缓存数据，用多少存多少，
减少了worker进程的内存用量。

