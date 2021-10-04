---
title: "Python Parse Url Domain"
date: 2021-10-04T23:23:52+08:00
draft: false
tags: ["python"]
categories: ["文章迁移"]
---

> 原文链接：https://blog.csdn.net/a_652/article/details/49681673?spm=1001.2014.3001.5501

# python 提取url中的主域名

参考了[Ourren][1]的博客。

-------------------

## 需求

 给定url，比如  `http://blog.csdn.net` ，提取出`csdn.net`

## 方法
### urlparse

使用python的urlparse模块解析只能得到完整的域名，不能提取出主域名。
```python
import urlparse
url = 'http://blog.csdn.net'
o = urlparse.urlparse(url)
print o
```
```
>>>ParseResult(scheme='http', netloc='blog.csdn.net', path='', params='', query='', fragment='')
```

### 正则表达式
使用正则表达式不容易将所有情况考虑全面。

###tldextract


```python
import tldextract
url = 'http://blog.csdn.net'
o = tldextract.extract(url)
print o
>>> o = ExtractResult(subdomain='blog', domain='csdn', suffix='net')
```
```
domain = "{}.{}".format(o.domain, o.suffix)
```
---------

[1]: http://blog.ourren.com/2015/04/26/python-with-domain/
