---
title: "Nodejs Async Callbacks"
date: 2021-10-06T23:07:53+08:00
draft: false
tags: ["nodejs", "async", "callbacks"]
categories: ["文章迁移"]
---

> 原文链接：https://www.jianshu.com/p/736afcad50d6


One limitation of async functions is that await only affects the directly surrounding async function. Therefore, an async function can’t await in a callback (however, callbacks can be async functions themselves, as we’ll see later on). That makes callback-based utility functions and methods tricky to use. Examples include the Array methods map() and forEach().

Let’s start with the Array method map(). In the following code, we want to download the files pointed to by an Array of URLs and return them in an Array.
```javascript
async function downloadContent(urls) {
    return urls.map(url => {
        // Wrong syntax!
        const content = await httpGet(url);
        return content;
    });
}
```
This does not work, because await is syntactically illegal inside normal arrow functions. How about using an async arrow function, then?
```javascript
async function downloadContent(urls) {
    return urls.map(async (url) => {
        const content = await httpGet(url);
        return content;
    });
}
```
There are two issues with this code:

- The result is now an Array of Promises, not an Array of strings.
- The work performed by the callbacks isn’t finished once map() is finished, because await only pauses the surrounding arrow function and httpGet() is resolved asynchronously. That means you can’t use await to wait until downloadContent() is finished.

We can fix both issues via Promise.all(), which converts an Array of Promises to a Promise for an Array (with the values fulfilled by the Promises):
```javascript
async function downloadContent(urls) {
    const promiseArray = urls.map(async (url) => {
        const content = await httpGet(url);
        return content;
    });
    return await Promise.all(promiseArray);
}
```
The callback for map() doesn’t do much with the result of httpGet(), it only forwards it. Therefore, we don’t need an async arrow function here, a normal arrow function will do:
```javascript
async function downloadContent(urls) {
    const promiseArray = urls.map(
        url => httpGet(url));
    return await Promise.all(promiseArray);
}
```
There is one small improvement that we still can make: This async function is slightly inefficient – it first unwraps the result of Promise.all() via await, before wrapping it again via return. Given that return doesn’t wrap Promises, we can return the result of Promise.all() directly:
```javascript
async function downloadContent(urls) {
    const promiseArray = urls.map(
        url => httpGet(url));
    return Promise.all(promiseArray);
}
```

