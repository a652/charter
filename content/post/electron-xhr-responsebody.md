---
title: "用electron得到请求页面时XHR的ResponseBody"
date: 2021-10-07T11:09:29+08:00
draft: false
tags: ["electron"]
categories: ["技术"]
---

> 原文链接：https://www.jianshu.com/p/e31228f17416


## 问题描述

利用electron访问一些页面时，想要获取到异步加载的内容。
如果利用webContents监听事件`did-get-response-details`，只能得到以下信息：
- `event` Event
- `status` Boolean
- `newURL` String
- `originalURL` String
- `httpResponseCode` Integer
- `requestMethod` String
- `referrer` String
- `headers` Object
- `resourceType` String

并没有想要的response body。

## 解决

经过请教与搜索之后，利用*Debugger*结合Chrome Debugging Protocol的*Network Domain*可以拿到response body。
> Network domain allows tracking network activities of the page. It exposes information about http, file, data and other requests and responses, their headers, bodies, timing, etc.

```
win.webContents.debugger.on('message', (event, method, params) => {

      if (method === 'Network.responseReceived') {
        if(params.response.url.indexOf('xxxxx') > 0) {
          console.log('Event: responseReceived ' + params.requestId + '-' + params.response.url)
          win.webContents.debugger.sendCommand('Network.getResponseBody', {"requestId": params.requestId}, (error, result) => {
            if(!error || JSON.stringify(error) == "{}") {
              console.log(`getResponseBody result: ${JSON.stringify(result)}`)
            } else {
              console.log(`getResponseBody error: ${JSON.stringify(error)}`)
            }
          })
        }
      }

      if(method === 'Network.webSocketFrameReceived'){
        console.log(params.response)
      }
  })
```

## Reference
- https://electron.atom.io/docs/api/debugger/
- https://chromedevtools.github.io/debugger-protocol-viewer/tot/Network/
