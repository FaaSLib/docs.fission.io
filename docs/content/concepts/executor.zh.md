---
title: "Fission Function Executors"
draft: false
weight: 23
---

当创建一个 function 的时候，你可以为它指定一个 executor。一个 executor 控制了 function 的pod 如何创建以及有什么能力。

##  Pool-based executor

在创建 env 的同时，一个 pool based executor 会预先创建一组 env 定义的 pod。pool 的大小即预热容器的数量可有用户设置。这些预热的容器包含动态 loader，用来 load function。资源请求的设置在 env 这一层。所以 function 所在的 pod 自然继承这一设置。

一旦创建一个 function 然后调用他。pool 里的一个 pod 会被分化用来执行 function。这个 pod 会用来处理随后该 function 的请求。如果在一定空闲时间后没有更多的请求时，该 pod 将会被清理。如果一个新的请求再次来到，pool 里会再拿出一个 pod 用来执行 function。

Poolmgr executortype 适合低 latency 的请求。Poolmgr executortype 有一个限制，就是不能根据需求扩容。

## New-deployment executor

New-Deployment executor(Newdeploy) 在 execute function 时创建一个 k8s Deployment 以及 a service, a HorizontalPodAutoscaler。这将开启自动伸缩和负载均衡功能。将来还会增加挂载卷的功能。在 new-deploy executor 中，请求资源可以在 function 层面设置。这一设置将会覆盖  env  的设置。

Newdeploy executortype 可以用在对 low-latency 不敏感的请求上。比如异步调用时，把最低副本数设为 0。在这种情况下 k8s deployment 和其他对象会在第一次调用时被创建。如果没有请求，这些对象将会被清理。这一机制保证资源只在需要的时候才会被使用。这更适合异步请求。

### The latency vs. idle-cost tradeoff

executors 允许用户在延迟和空闲损耗之间做决定。根据需求选择。如下表。

| Executor Type | Min Scale| Latency | Idle cost |
|:---------|:---------:|:---------:|:---------|
|Newdeploy|0|High|Very low - pods get cleaned up after idlle time|
|Newdeploy|>0|Low|Medium, Min Scale number of pods are always up|
|Poolmgr|0|Low|Low, pool of pods are always up|

### 自动伸缩

New Deployment 类型的 executor 提供根据 CPU 使用的自动伸缩。将来也会支持其他测量指标。用户可以设置 CPU 使用的初始值和最大值。目标值会触发伸缩。自动伸缩用来处理间歇出现峰值的情况。