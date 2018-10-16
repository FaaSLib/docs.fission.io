---
title: "canary deployments"
draft: false
weight: 35
---

### 降低部署风险

* 尽管所有的测试都完成了，直接部署到现网上，还是有风险的
* 测试和展示环境，不可能和生产环境完全一致
* 一个好的策略是，在一个版本被完善的测试之后，进行增量部署
* 例如：10%的用户使用新的版本，如果没有问题，你在逐步的增加比例

### 自动金丝雀部署

* 金丝雀部署需要你监视金丝雀的成功率，然后决定是否要进一步增量部署
* 在Faas中，我们知道一个函数是否成功
* 因此我们可以自动化，滚动部署和回滚的流程

### fission

fission内置有自动金丝雀部署。提供如下配置：

* 为新版本分配的流量占比
* 定义多大的请求错误比例为部署失败
* 定义新版本部署的增长速率，当已部署反馈问成功
* 如果失败，函数可以在任何时间点回滚

```
[root@dev-86-208 hello]# fission function create --name hello-v1 --env nodejs --code hello-v1.js
function 'hello-v1' created

[root@dev-86-208 hello]# fission function create --name hello-v2 --env nodejs --code hello-v2.js
function 'hello-v2' created

[root@dev-86-208 hello]# fission route create --name route-canary --method GET --url /canary --function  hello-v1 --weight 0 --function hello-v2 --weight 100
Incorrect Usage: flag provided but not defined: -name

NAME:
   fission httptrigger create - Create HTTP trigger

USAGE:
   fission httptrigger create [command options] [arguments...]

OPTIONS:
   --method value                    HTTP Method: GET|POST|PUT|DELETE|HEAD (default: "GET")
   --url value                       URL pattern (See gorilla/mux supported patterns)
   --function value                  Function name
   --host value                      FQDN of the network host for route
   --createingress                   Creates ingress with same URL, defaults to false
   --fnNamespace value, --fns value  Namespace for function object (default: "default")
   --spec                            Save to the spec directory instead of creating on cluster
```

### canary deploy will coming in fission 0.11

