---
title: "cost optimization"
draft: false
weight: 35
---

### cost optimization

许多系统包含成本效益权衡。公有云无服务器服务让你为你所使用的付费，尽管随着使用率的提高，性价比会越来越差。就算是在私有云中，你还是要考虑资源使用率——资源按需被合理的分配，这样空闲资源可以供别的服务需要时使用。

### 冷启动

理想的状态，当服务没有被使用时应该没有资源占用。但是当有服务请求时，又希望服务能够快速响应。因此有个冷启动的问题在，我们如何保证较少的资源消耗，同时又能提供较低的延时？

### cold starts in fission 

fission提供内置冷启动优化：使用一个预热的容器池。容器池的大小可以配置，容器池的资源花费会被均摊到集群中所有的函数中。当函数被激活，函数会被动态的加载到预热的容器中。

函数也可以配置成不使用容器池，以高一点的延时来换取更低的资源花费。

### cost optimizations in fission

fission提供的资源花费优化：

* 函数执行器是可调的：选择一个成本效益权衡的点。
* 不同于lambda的计价模式，但是跟最便宜的VM实例一样便宜。
* 私有化可以节约成本，特别是当你已经有了基础设备。

通过配置来优化资源消耗，如：配置函数的cpu和memory资源使用限制，配置自动扩缩容参数：最大和最小扩缩容范围，触发扩缩容的cpu使用率