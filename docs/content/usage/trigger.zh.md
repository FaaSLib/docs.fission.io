---
title: "Triggers"
draft: false
weight: 43
---

### 创建一个 HTTP 触发器

当有HTTP请求时，HTTP触发器会激活一个函数。你可以为一个触发器指定对应的URL和HTTP方法：

``` bash
$ fission httptrigger create --url /hello --method GET --function hello
trigger '94cd5163-30dd-4fb2-ab3c-794052f70841' created

$ curl http://$FISSION_ROUTER/hello
Hello World!
```

{{% notice tip %}} 
FISSION_ROUTER 是你的Fission router service的外部可见地址.  对于如何如设置
`FISSION_ROUTER`环境变量, 你可以看 [这里]({{< ref "/installation/env_vars" >}})
{{% /notice %}}****



如果你想用Kubernetes的Ingress作为HTTP触发器，你可提供`--createingress`标志和一个主机名。如果主机名没有提供，默认会是"*"，代表通配的主机。

```  bash
$ fission httptrigger create --url /hello --method GET --function hello --createingress --host acme.com
trigger '94cd5163-30dd-4fb2-ab3c-794052f70841' created

$ fission route list
NAME                                 METHOD HOST     URL      INGRESS FUNCTION_NAME
94cd5163-30dd-4fb2-ab3c-794052f70841 GET    acme.com /hello   true    hello
```



请注意，如果想要ingress顺利工作，你需要在kubernetes集群中部署一个ingress controller。kubernetes现在支持和维护这些ingress controllers:

- [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)
- [GCE Ingress Controller](https://github.com/kubernetes/ingress-gce)

还有一个ingress controller， 像 [F5 networks](http://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/v1.5/) 和 [Kong](https://konghq.com/blog/kubernetes-ingress-controller-for-kong/)。



### 创建一个 time 触发器

Time-based triggers 基于时间来触发函数.  它们可以运行一次，也可以重复运行.  他们通过 [cron string
specifications](https://en.wikipedia.org/wiki/Cron)格式来定义时间任务:

``` bash
$ fission tt create --name halfhourly --function hello --cron "*/30 * * * *"
trigger 'halfhourly' created
```

你甚至可以使用更友好的语法，如： "@every 1m" 或者 "@hourly":

``` bash
$ fission tt create --name minute --function hello --cron "@every 1m"
trigger 'minute' created
```

你可以通过列出time triggers，去查看他们关联的函数和cron字符串：

``` bash
$ fission tt list
NAME       CRON         FUNCTION_NAME
halfhourly 0 30 * * * * hello
minute     @every 1m    hello
```

你可以通过`showschedule`查看一个cron即将到来的时间事件列表。使用这个测试你的corn strings。注意这里使用的是服务器时间来触发函数，而不是你便携计算机的时间。

``` bash
$ fission tt showschedule --cron "0 30 * * * *" --round 5
Current Server Time: 	2018-06-12T05:07:41Z
Next 1 invocation: 	2018-06-12T05:30:00Z
Next 2 invocation: 	2018-06-12T06:30:00Z
Next 3 invocation: 	2018-06-12T07:30:00Z
Next 4 invocation: 	2018-06-12T08:30:00Z
Next 5 invocation: 	2018-06-12T09:30:00Z
```

### 创建一个 Message Queue 触发器

message queue trigger 基于消息队列中的消息来触发一个函数。 也就是说，它可以把一个函数的响应对应到一个消息队列上。

NATS 和Azure Storage Queue 是支持的队列：

``` bash
$ fission mqt create --name hellomsg --function hello --mqtype nats-streaming --topic newfile --resptopic newfileresponse 
trigger 'hellomsg' created
```

你可以通过 `fission mqt list`或者 `fission mqt update`来查看和更新message queue triggers。