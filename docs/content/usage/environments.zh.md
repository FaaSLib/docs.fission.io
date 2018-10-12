---
title: "Environments"
draft: false
weight: 41
---

### Create an environment

你可以为特定语言用指定镜像创建一个 envronment 。

作为选项，你可以设置 CPU 和 memory 资源限制。你可以设置预热 pods 的数量，这杯称为 poolsize。

```
$ fission env create \
    --name node \
    --image fission/node-env \
    --mincpu 40 \
    --maxcpu 80 \
    --minmemory 64 \
    --maxmemory 128 \
    --poolsize 4
```

对于基于 pool 的 executor ，envronment 的资源设置会直接应用在 function 的 pod 上。对于基于 deployment 的 executor，envronment 的资源设置会被 function 上的资源设置覆盖。

### Using a builder

当你创建一个 environment 的时候，可以设置一个 builder image 和 builder cmd。他们可以用来构建源码。你可以在创建 function 的时候设置新的 builder cmd 来覆盖 env 的设置。

```bash
$ fission env create \
    --name python \
    --image fission/python-env:latest \
    --builder fission/python-builder:latest
```

### Viewing environment information

You can list the environments or view information of an individual environment:

```bash
$ fission env list
NAME UID                                  IMAGE                  POOLSIZE MINCPU MAXCPU MINMEMORY MAXMEMORY
node ac84d62e-001f-11e8-85c9-42010aa00010 fission/node-env:0.4.0 4        40m    80m    64Mi      128Mi

$ fission env get --name node
NAME UID                                  IMAGE
node ac84d62e-001f-11e8-85c9-42010aa00010 fission/node-env:0.4.0

$ kubectl get environment.fission.io -o yaml
# Full YAML of Fission environment object

```
