---
title: "[openresty实践]使用lua-zlib对收到的gzip压缩内容进行解压"
date: 2020-05-27T23:29:40+08:00
draft: false
tags: ["openresty"]
categories: [""]
---


## 背景

我们的系统需要对接各种媒体，接收媒体发过来的http请求，有的是protobuf格式，有的是json格式，但是最近遇到了一个媒体要求支持gzip压缩过的内容，解压后是json格式。

媒体的请求会在header中带着`Content-Encoding: gzip` ,我们根据这个选项对接收到的body进行解压缩。

openresty没有自带的zlib库，可以通过安装[lua-zlib](https://github.com/brimworks/lua-zlib)满足这个需求。

## 安装lua-zlib

1. 下载lua-zlib-1.2.tar.gz包，并解压。
2. cd lua-zlib-1.2/
3. cmake -DLUA_INCLUDE_DIR=/usr/local/openresty/luajit/include/luajit-2.1 -DLUA_LIBRARIES=/usr/local/openresty/luajit/lib -DUSE_LUAJIT=ON -DUSE_LUA=OFF
4. make 生成zlib.so文件
5. cp zlib.so /usr/local/openresty/lualib/

## lua代码

```lua
local zlib = require "zlib"
ngx.req.read_body()
local request_data = ngx.req.get_body_data()
local r = ""
local encoding = ngx.req.get_headers()["Content-Encoding"]
if encoding == "gzip" then
    local stream = zlib.inflate()
    r = stream(request_data)
else
    r = request_data
end
```

## 问题

经过上述过程，完成了在收到post body后，先对其进行解压再进行解析的需求。

但是在系统运行过程中却出现了另外的问题：

> _G write guard writing a global lua variable ('zlib') which may lead to race conditions between concurrent requests, so prefer the use of 'local' variables 
> stack traceback in function 'require'

一般情况下出现这类问题都是因为在代码中使用了全局变量，对变量使用local声明就能解决问题，但在这个case中很明显使用了local，却还是报出了问题。

经过搜索找到了这个[issue](https://github.com/brimworks/lua-zlib/issues/48)，根据下面的回复，修改lua_zlib.c文件，将其中的  *luaL_register(L, "zlib", zlib_functions);* 替换成:

```lua
lua_newtable(L);
luaL_setfuncs(L, zlib_functions, 0);
```

重新编译zlib.so，消除了global lua variable 的警告信息。

