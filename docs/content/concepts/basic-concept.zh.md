---
title: "Basic Concepts"
draft: false
weight: 20
---

Fisson 中有三个基本的概念/元素：

## Function

一个特定语言的代码片段，将会被 fission router 触发。

下面是一个例子。

```js
module.exports = async function(context) {
    return {
        status: 200,
        body: "Hello, world!\n"
    };
}
```

目前，fission 支持多种流行的语言。比如 Nodejs, Go, Python, Java 等等。使用范例见 [fission language examples](https://github.com/fission/fission/tree/master/examples)。

## Environment

运行用户 function 的容器会接受 HTTP 请求。当一个请求到达 fission router 的时候。env 容器会将 function 加载进来，然后执行并处理请求。

## Trigger

Trigger 是一个对象，用来定位请求到后端的 function。当一个 trigger 收到 requests/events 的时候，他将会通过发送 http 请求到 functon 所在的 pod 来调用目标 function。

目前，fission 支持一下类型的 trigger：

* HTTP Trigger
    * 在 router 上注册一个 url path，并把所有请求路由到用户的 func
* Time Trigger
    * 定时触发
* Message Queue Trigger
    * 触发器将会订阅和掌控所有到达 message 队列主题的消息。然后将 function 返回的相应或者错误发布到预定义的 topic 中。
* Kubernetes Watch Trigger
    * watcher 将会被创造出来监控 k8s 对象。如果有变化发生，就会调用对应的 function。