---
title: "nodejs的json to csv 转换"
date: 2021-10-07T11:25:10+08:00
draft: false
tags: ["nodejs", "json", "csv"]
categories: ["技术", "文章迁移"]
---

> 原文链接：https://www.jianshu.com/p/1c9e1ac492ba


## 问题描述

node中有`json2csv`模块，但是当json数据的key未事先指定，并且有嵌套结构的时候，嵌套在内层的数据无法被识别并转换。

## 解决

参考一个在线的json to csv转换工具[Convert JSON to CSV](https://konklone.io/json/)，及其源码[parse_object](https://konklone.io/json/assets/site.js) 和 [csvkit](https://github.com/wireservice/csvkit/blob/61b9c208b7665c20e9a8e95ba6eee811d04705f0/csvkit/convert/js.py)
先对json数据进行递归遍历，将其拍平，然后再利用`json2csv`模块。

代码如下：
```node
var fs = require('fs');
var filename = 'data.txt';

function allItems(filename) {
    console.log("read file", filename)
    var contents = fs.readFileSync(filename).toString().split("\n")

    var arr = []

    contents.forEach(s => {
        try {
            arr.push(JSON.parse(s))
        } catch (e) {
            console.log("parse error", e)
            console.log("parse error", s)
        }
    })
    return arr
}

function parseObject(obj, path) {
    if (path == undefined)
        path = "";
    
    var type = obj.constructor;

    var scalar = (type == Number || type == String || type == Boolean || type == null);

    if (type == Array || type == Object) {
        var d = {};
        for (var i in obj) {

            var newD = parseObject(obj[i], path + i + ".");
            Object.assign(d, newD);
        }

        return d;
    }
    else if (scalar) {
        var d = {};
        var endPath = path.substr(0, path.length-1);
        d[endPath] = obj;
        return d;
    }
    else return {};
}

function csv() {
    var arr = allItems(filename);
    var arrNew = [];
    var json2csv = require('json2csv');
    for(var i in arr) {
        arrNew.push(parseObject(arr[i]));
    }

    try {
        var result = json2csv({ data: arrNew });
        fs.writeFile(filename+'.csv', result);

    } catch (err) {
        // Errors are thrown for bad options, or if the data is empty and no fields are provided.
        // Be sure to provide fields if it is possible that your data array will be empty.
        console.error("convert", err, err.stack);
    }
}

csv()
```
