---
title: "JavaScript使用parseInt()的坑小结"
date: 2021-10-07T11:30:08+08:00
draft: false
tags: ["javascript"]
categories: ["技术", "文章迁移"]
---

> 原文链接：https://www.jianshu.com/p/46e3e720de62


## 问题描述

场景是根据包含几个变量的字符串算一个签名
例如
```
var data = {
            type: this.type,
            timestamp: this.timestamp,
            id: this.id,
        };
var s = 'hello';
var b = 'world'; 
var k = `${s}&${b}&${JSON.stringify(data)}`;
```
期待的变量`k`应该长这样
```
'hello&world&{"type":0,"timestamp":1488191490594,"id":9800000000085572}'
```
id是一个16位的数字串

本来`this.id`是一个string类型，但是经过JSON格式化后，会自动为string类型加上双引号，这样算出来的签名的值就会发生变化。

## 解决

一开始的解决方式比较粗暴，就是把id值转换成Number格式
```
id: parseInt(this.id)
```

但是后来发现，parseInt返回的结果相对于本来的值会发生改变
```
>> parseInt("9800000000085573")
output: 9800000000085572
```

查询[What is JavaScript's highest integer value that a Number can go to without losing precision?](http://stackoverflow.com/questions/307179/what-is-javascripts-highest-integer-value-that-a-number-can-go-to-without-losin)后发现js能安全处理的最大整数为：9007199254740991

最后采用字符串替换的方式解决：
```
var data = {
            type: this.type,
            timestamp: this.timestamp,
            id: "tmpid",
        };
var s = 'hello';
var b = 'world'; 
var k = `${s}&${b}&${JSON.stringify(data).replace('"tmpid"', this.id)}`;
```
