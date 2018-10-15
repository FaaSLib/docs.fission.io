---
title: "Compiling Fission"
date: 2017-09-07T20:10:05-07:00
draft: false
weight: 51
---

{{% notice info %}}
如果你需要更改 Fission，你才需要这篇教程。如果只是安装，请使用 fission.yaml 。
{{% /notice %}}

你需要一系列 go 的编译环境和工具。以及 glide 依赖管理工具。还需要 docker 环境来编译镜像。

服务端编译在一个二进制中 fission-bundle 。包含了 `controller`, `poolmgr`, `router`。具体调用哪个取决于启动命令。

