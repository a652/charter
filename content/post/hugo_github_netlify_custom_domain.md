---
title: "Hugo+Github+Netlify搭建博客配置个人域名"
date: 2021-10-06T17:05:59+08:00
draft: false
tags: ["blog", "domain", "netlify"]
categories: ["其他"]
---


# 背景

之前基于hugo搭建了一个简单的个人博客，并push到github，部署到netlify。一直以来都是通过netlify分配的子域名进行访问，最近想着也配置一个个人域名试试。

# 域名购买

看到有人的文章里介绍到可以在[freenom](https://www.freenom.com)申请免费域名，于是尝试了一番，但最终没有成功。

freenom 提供`.TK / .ML / .GA / .CF / .GQ`这几种后缀的免费域名，而且也只是第一年免费。然而我在操作的过程中卡在了确认订单这一步，在点击“完成订单”按钮之后页面总是报错，于是放弃。


最终在[namecheap](https://www.namecheap.com)成功购买了`xyz`后缀的域名`charter652.xyz`。


# 配置DNS解析

我的博客部署在netlify上面，同时netlify提供了域名解析功能，于是在netlify上配置域名解析。登录后选择`Domains`然后点击自己的站点进入设置页面。

在设置时netlify会给出建议的DNS records和Name servers，可以采用它建议的DNS records， 并将Name servers添加到namecheap中的DNS配置中。然后等待几十分钟等DNS解析生效就可以通过自己的域名[charter652.xyz](https://charter652.xyz/)访问博客站点了。

### netlify设置页面

![](/images/dns_records.png)
![](/images/name_servers.png)

------

### namecheap截图
![namecheap截图](/images/namecheap_nameservers.png)
