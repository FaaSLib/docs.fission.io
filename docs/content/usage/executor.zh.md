---
title: "Controlling Function Execution"
draft: false
weight: 45
---

### Autoscaling

让我们创建一个可以动态自动伸缩的 function。我们创建一个简单的 NodeJS function 以输出 "Hello World"。把 CPU 的 request 和 limit 设置比较低的值。让 CPU 保持在 50% 左右。

```bash
$ fission fn create \
    --name hello \
    --env node \
    --code hello.js \
    --minmemory 64 \
    --maxmemory 128 \
    --minscale 1 \
    --maxscale 6 \
    --executortype newdeploy \
    --targetcpu 50
function 'hello' created
```

接下来让我们用 [hey](https://github.com/rakyll/hey) 来模拟 250 并发，共计 10000 个请求。 

```bash
$ hey -c 250 -n 10000 http://${FISSION_ROUTER}/hello
Summary:
  Total:	67.3535 secs
  Slowest:	4.6192 secs
  Fastest:	0.0177 secs
  Average:	1.6464 secs
  Requests/sec:	148.4704
  Total data:	160000 bytes
  Size/request:	16 bytes

Response time histogram:
  0.018 [1]	|
  0.478 [486]	|∎∎∎∎∎∎∎
  0.938 [971]	|∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  1.398 [2686]	|∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  1.858 [2326]	|∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  2.318 [1641]	|∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  2.779 [1157]	|∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  3.239 [574]	|∎∎∎∎∎∎∎∎∎
  3.699 [120]	|∎∎
  4.159 [0]	|
  4.619 [38]	|∎

Latency distribution:
  10% in 0.7037 secs
  25% in 1.1979 secs
  50% in 1.5038 secs
  75% in 2.1959 secs
  90% in 2.6670 secs
  95% in 2.8855 secs
  99% in 3.4102 secs

Details (average, fastest, slowest):
  DNS+dialup:	 0.0058 secs, 0.0000 secs, 1.0853 secs
  DNS-lookup:	 0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:	 0.0000 secs, 0.0000 secs, 0.0026 secs
  resp wait:	 1.6405 secs, 0.0176 secs, 3.6144 secs
  resp read:	 0.0001 secs, 0.0000 secs, 0.0056 secs

Status code distribution:
  [200]	10000 responses

```

当生成负载的时候，我们可以观察 k8s 的 HorizontalPodAutoscaler 以及他是如何随着时间伸缩的。正如你能注意到的，负载从 8 升到 103% 的时候pod 的数量从 1 变化到 3。当负载下降后， pod 数量也随之下降。

我们需要了解从负载达到阈值到增加 pod 服本书之间有个启动时间。所以需要根据非峰值使用情况设置合适的最小副本数。

scaling up and down has different behaviour. more information see [here](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-cooldowndelay)