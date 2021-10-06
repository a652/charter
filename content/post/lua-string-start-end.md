---
title: "Lua判断string是否以某个字符开头或结尾"
date: 2021-10-06T22:17:52+08:00
draft: false
tags: ["lua", "string"]
categories: ["文章迁移"]
---

> 原文链接: https://www.jianshu.com/p/90a443b693aa


代码如下：
```
function string.starts(String,Start)
   return string.sub(String,1,string.len(Start))==Start
end

function string.ends(String,End)
   return End=='' or string.sub(String,-string.len(End))==End
end
```



