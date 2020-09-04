---
title: "MacBook 设置通知横幅的显示时间"
date: 2020-09-04T10:49:55+08:00
draft: false
---


MacOS的通知横幅显示在屏幕的右上角，且位置不可改变。

在日常的工作、娱乐使用场景中，经常是打开很多浏览器选项卡，恰巧活跃的页面也会排列到右边。这样就导致一个问题，想要鼠标切换不同的选项卡时，被弹出的通知横幅挡住，尤其是碰到通知多，接二连三弹出的时候，令人崩溃。

一个可以缓解这个问题的方式就是缩短横幅弹出后的显示时间，让其弹出后尽快消失。

## 设置方法

需要在终端里通过命令行设置, 使用`defaults`命令

```shell
defaults write com.apple.notificationcenterui bannerTime -int {duration}
```

其中**duration**是你想设置的时间，比如1s。

执行完命令之后需要退出登录，然后重新登录。`cmd + shift + Q`


可以使用以下命令测试以下：

```shell
osascript -e 'display notification "test notification!"'
```

## 参考

> [How to Change the Duration of Notifications on MacOS](https://howchoo.com/mac/how-to-change-the-duration-of-notifications-on-macos)
