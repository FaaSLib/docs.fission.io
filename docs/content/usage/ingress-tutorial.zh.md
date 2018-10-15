---
title: "Exposing functions with Ingress"
draft: false
weight: 49
---

本片教程将会说明如何使用 ingress controller 暴露一个 funciton 。（更多 ingress 相关文档请参考 [here](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers)
我们将保证在云上环境正确的使用 fully qualified domain name

## 安装和准备工作

你将需要一个 k8s 和 fisison 。
本文将使用 cloud load balancer。如果使用 minikube ，你将需要另一篇文章 (https://github.com/kubernetes/minikube/issues/496)

后问将会使用 FQDN 来接触 function 。如果你想进行这一步，你需要设置域名。

### Setup an Ingerss Contoller

首先我们需要一个 ingerss controller。我们将用 nginx ingerss controller。 [one of the multiple ways to install Nginx ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/).也可以使用其他类型的 ingress controllers，但是没有经过测试。

让我们验证一下时候安装成功。

```
kubectl get all -n ingress-nginx
NAME                                           READY     STATUS    RESTARTS   AGE
po/default-http-backend-66b447d9cf-4q8f7       1/1       Running   0          19d
po/nginx-ingress-controller-58fcfdc6fd-2cwts   1/1       Running   0          19d

NAME                       CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
svc/default-http-backend   10.11.243.109   <none>           80/TCP                       19d
svc/ingress-nginx          10.11.245.254   35.200.150.175   80:31000/TCP,443:30666/TCP   19d

NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/default-http-backend       1         1         1            1           19d
deploy/nginx-ingress-controller   1         1         1            1           19d

NAME                                     DESIRED   CURRENT   READY     AGE
rs/default-http-backend-66b447d9cf       1         1         1         19d
rs/nginx-ingress-controller-58fcfdc6fd   1         1         1         19d

```

下面几点是检验是否安装成功的关键。

- ingress controller pod 正常运行
- ingress-nginx service 拥有 external IP 地址
- 如果访问 external IP，返回 nginx 默认页面。

## Deploying Function with ingress

一个 ingress 资源允许流量从集群外部接触集群内部的 service 。ingress 完全由 ingress controller 控制。后面几节我们将会创建 function 和启动外部流量到 function。

### Create a function

我们将会创建 environment，function 以及测试他们。

```
$ fission env create \
    --name nodejs \
    --image fission/node-env
environment 'nodejs' create

$ cat hello.js
module.exports = async function(context) {
    return {
        status: 200,
        body: "Hello, Fission!\n"
    };
}

$ fission fn create \
    --name hello \
    --env nodejs \
    --code hello.js
function 'hello' created

$ fission fn test \
    --name hello
Hello, Fission!
```

### Create a internal route

让我们通过 ingress controller 创建一个不对外暴露的路由。他只能在集群内部使用。

目前所有通过 Fission router 暴露的 function 都可以通过外部访问。将来路由也许不会把所有 function 暴露出去。

```
$ fission route create --url /ihello --function hello
trigger '249838c9-9ae3-492a-bba1-b0464ae65671' created
$ fission route list
NAME                                 METHOD HOST URL     INGRESS FUNCTION_NAME
249838c9-9ae3-492a-bba1-b0464ae65671 GET         /ihello false   hello
```

这个路由可以通过 `http://{$FISSOIN_ROUTER}/ihello` 访问。但是这个时候还不能通过 `http://{ingerss-controller-external-ip}/ihello` 来访问。因为我们还有没有为这个 route 创建 ingress。

### Create a external route

现在让我们创建一个将要用 ingerss controller 向外暴露的路由。
我们将会创建一个带有参数 `createingress` 参数的路由。

```
$ fission route create \
    --url /hello \
    --function hello \
    --createingress
trigger '301b3cb0-5ac1-4211-a1ed-2b0ad9143e34' created

$ fission route list
NAME                                 METHOD HOST URL     INGRESS FUNCTION_NAME
249838c9-9ae3-492a-bba1-b0464ae65671 GET         /ihello false   hello
301b3cb0-5ac1-4211-a1ed-2b0ad9143e34 GET         /hello  true    hello
```

如果你检查 ingress controller pod 的日志，你会注意到 ingress
 controller 重载了配置文件。

```
I0604 12:47:08.983567       5 controller.go:168] backend reload required
I0604 12:47:08.985535       5 event.go:218] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"fission", Name:"301b3cb0-5ac1-4211-a1ed-2b0ad9143e34", UID:"64bffe8c-67f5-11e8-98e8-42010aa00018", APIVersion:"extensions", ResourceVersion:"18017617", FieldPath:""}): type: 'Normal' reason: 'CREATE' Ingress fission/301b3cb0-5ac1-4211-a1ed-2b0ad9143e34
I0604 12:47:09.117629       5 controller.go:177] ingress backend successfully reloaded...
```

如果你现在通过 ingerss 的地址访问 function (`http://<INGRESS-CONTROLLER-EXTERNAL-IP>/hello`)。有时候 ingress 默认启动 ssl。

```
$ curl -k  https://35.200.150.175/hello
Hello, Fission!

```

### Create a FQDN route

这是一个可选步骤。如果 DNS 设置正确，你可以讲 FQDN 转到指定 funciton 。需要如下步骤。

- 将域名指向 cloud provider。具体环境，具体分析

- 在 cloud provider 中为 root domain 创建一个 zone

- 为子域名 sub-domain 创建一个 A record 指向 Ingress Controller load balancer 。

- 如果上述步骤都成功配置

```
$ curl -k  https://ing.fission.sh/hello
Hello, Fission!

```
