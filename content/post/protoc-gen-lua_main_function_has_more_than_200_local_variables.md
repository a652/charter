---
title: "protoc-gen-lua 生成的lua文件遇到error: main function has more than 200 local variables"
date: 2021-10-06T21:56:21+08:00
draft: false
tags: ["protobuf", "lua"]
categories: ["文章迁移"]
---


> 原文链接: https://www.jianshu.com/p/b78e342d857e


使用[protoc-gen-lua](https://github.com/sean-lin/protoc-gen-lua)生成的lua文件，由于源proto文件定义的字段过多，在运行生成的lua文件时遇到错误：`main function has more than 200 local variables`。

看GitHub上protoc-gen-lua这个项目已经好几年都没有更新了。。。

搜索发现了[protoc-gen-lua-bin](https://github.com/u0u0/protoc-gen-lua-bin)这个项目，修复了此问题。

## 操作步骤：

- 本机安装protobuf@2.6
- cd /path/to/protoc-gen-lua-bin
- cd python_mac && python setup.py install
- 为protoc文件和plugin/protoc-gen-lua文件添加可执行权限
- 将proto文件复制到./proto/路径下
- 运行bash buildproto.sh

lua文件生成在output文件夹下


## 针对protobuf3的支持

针对protobuf3格式的proto文件。可以使用库[lua-protobuf](https://github.com/starwing/lua-protobuf)

- 将此库的c代码编译成动态链接库
    ```
    gcc -O2 -shared -fPIC pb.c -o libpb.so
    ```
- 再将proto文件生成pb文件
- protoc -o xxx.pb xxx.proto
