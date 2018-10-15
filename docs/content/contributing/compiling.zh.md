---
title: "Compiling Fission"
date: 2017-09-07T20:10:05-07:00
draft: false
weight: 51
---

{{% notice info %}}
如果你需要更改 Fission，你才需要这篇教程。如果只是安装，请使用 fission.yaml 。
{{% /notice %}}

你需要一系列 go 的编译工具，glide 包管理。同事需要 docker 环境来构建镜像。

服务端编译在一个二进制包中(`fission-bundle`。它们包含 `controller`, `poolmgr`, `router`。根据启动参数的不同，分别启动不同的模块。

为了构建 fission-bundle。克隆该仓库 `$GOPATH/src/github.com/fission/fission` 
然后在顶层目录：

```
# Get dependencies
$ glide install

# Build fission server and an image
$ pushd fission-bundle
$ ./build.sh
```

你现在需要为 fission 构建 docker 镜像。你可以用 `push.sh` 讲镜像推到 docker hub 。但是更简单的是使用 minikube 和 docker daemon。

```
$ eval $(minikube docker-env)
$ docker build -t minikube/fission-bundle .
```

下一步，用心构建的镜像安装 fission


```
$ helm install --set "image=minikube/fission-bundle,pullPolicy=IfNotPresent,analytics=false" charts/fission-all
```

如果也改了命令行

```
# Build Fission CLI
$ cd fission && go install
```