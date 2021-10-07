---
title: "如何在docker中运行electron"
date: 2021-10-07T11:19:04+08:00
draft: false
tags: ["electron"]
categories: ["技术", "文章迁移"]
---

> 原文链接：https://www.jianshu.com/p/cc7a1398a8c7


> Electron 使用 web 页面作为它的 GUI，所以你能把它看作成一个被 JavaScript 控制的，精简版的 Chromium 浏览器。

在本文的应用场景中，将electron用于爬虫的一部分，去获取访问页面时的cookie等信息。

docker中的系统信息：
```
Distributor ID: Ubuntu
Description:    Ubuntu 14.04.1 LTS
Release:        14.04
Codename:       trusty
```

1. 安装nvm
2. 安装nodejs，版本为`v7.4.0`
3. 安装electron，版本：`v1.4.15`
4. 安装依赖包
```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y libgtk2.0-0 libgconf-2-4 libnotify-bin libasound2 libxtst6 libxss1 libnss3 xvfb
npm install segmentio/nightmare
```
5. Start xvfb server
```
export DISPLAY=':99.0'
Xvfb :99 -screen 0 1024x768x24 > /dev/null 2>&1 &
```

然后可以运行自己的app了。
运行到electron时，会打印error 信息：`Xlib:  extension "RANDR" missing on display ":99.0"`，但是仍然能够拿到需要的数据，就先将其忽略了...

## 参考

[Setup Electron on Ubuntu](https://fraserxu.me/2016/01/29/setup-electron-on-ubuntu/)
