---
title: "Fission Concepts"
chapter: false
draft: false
weight: 30
alwaysopen: true
---

Fission 有三个主要的概念：`Function`, `Environments` 和 `Triggers`。

## Functions

一个 Fission function 是 Fission 要执行的东西。它通常是一个单一入口的模块。该模块是一个入参界面固定的 function。目前支持很多语言，见后文。

## Environments

Environments 是跟语言相关的。一个 Env 包含了足够 Functino 构建和运行的软件环境。

Fission 里一个 env 的运行时(runtime)就是一个 HTTP server 容器。Fission 通过 HTTP 调用 Function。容器通常用一个动态 loader 来加载 function。有些 env 也包含构建容器。构建环境关心编译和集成依赖。

下面列出 Fission 默认提供的环境。

| Environment                          | Image                     |
| ------------------------------------ | ------------------------- |
| NodeJS (Alpine)                      | `fission/node-env`        |
| NodeJS (Debian)                      | `fission/node-env-debian` |
| Python 3                             | `fission/python-env`      |
| Go                                   | `fission/go-env`          |
| Ruby                                 | `fission/ruby-env`        |
| Binary (for executables or scripts)  | `fission/binary-env`      |
| .NET                                 | `fission/dotnet-env`      |
| .NET 2.0                             | `fission/dotnet20-env`    |
| Perl                                 | `fission/perl-env`        |
| PHP 7                                | `fission/php-env`         |

可以扩展其中的某一个来自定义环境。从新创建也可以。

## Triggers

Functions 会因事件的发生而被调用。_Trigger_ 就是用来调用 Function 的事件。

比如，一个 HTTP Trigger 会绑定某一个确定 path 上的 GET 请求到一个确定的 Function。

除了 HTTP Triggers 还有别的 Triggers：Timer Trigger 定时触发 func；Message queue triggers 用于 Kafka, NATS, Azure queues；K8s Watch triggers 用来根据集群内数据来触发 function 。

## 其他概念

这些概念也许在刚开始并没有什么用。但是了解他们会对后面高级使用有好处。

### Archives

一个 Archives 是一个包含源码或者二进制的压缩包。

包含可执行 functions 的 Archives 叫 _Deployment Archives_；包含源码的叫 _Source Archives_。

### Packages

一个 Package 是 Fission 中包含 Deployment Achive 和 Source Achive 的对象。一个 Package 也对应一个固定的 env。

当你用 Source Archive 创建一个 Package 的时候， Fission 自动用构建环境构建出对应的 Deployment Archive。

### Specification

Specification 是一些简单的 YAML 配置文件。他们包含了我们目前为止说到的所有对象，Functions，Environments， Triggers， Packages，Archives。

Specifications 只存在于客户端。一般用他们来给 Fission CLI 下达创建和更新的指令。他们也会被用来设置如何打包源代码或二进制  Archive。

Fission CLI 使用 Specifications 进行的部署操作是幂等的。