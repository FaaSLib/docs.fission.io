---
title: "Fission Workflows"
draft: false
weight: 70
---

### 预先条件

Fission Workflows 需要如下组件

- kubectl
- helm

Fission Workflows 部署在 k8s 集群之上。如果没有 k8s 集群，请自行安装一个。
同时也需要安装 Fission 。

注意：Fission Workflows 0.5.0 需要至少 Fission 0.4.1 ，且需要 NATS 组件。

### 安装 Fission Workflows

Fission Workflows 是 Fission 的一个插件。可以同过 Helm 安装他们。

假设你已经安装了所有先觉组件。

```bash
$ helm repo add fission-charts https://fission.github.io/fission-charts/

$ helm repo update

$ helm install --wait -n fission-workflows fission-charts/fission-workflows --version 0.5.0
```

### Creating your first workflow

```bash
# Fetch the required files, alternatively you could clone the fission-workflow repo
$ curl https://raw.githubusercontent.com/fission/fission-workflows/0.5.0/examples/whales/fortune.sh > fortune.sh

$ curl https://raw.githubusercontent.com/fission/fission-workflows/0.5.0/examples/whales/whalesay.sh > whalesay.sh

$ curl https://raw.githubusercontent.com/fission/fission-workflows/0.5.0/examples/whales/fortunewhale.wf.yaml > fortunewhale.wf.yaml

#
# Add binary environment and create two test functions on your Fission setup:
#
$ fission env create --name binary --image fission/binary-env

$ fission function create --name whalesay --env binary --deploy ./whalesay.sh

$ fission function create --name fortune --env binary --deploy ./fortune.sh



#
# Create a workflow that uses those two functions. A workflow is just
# a function that uses the "workflow" environment.
#
$ fission function create --name fortunewhale --env workflow --src ./fortunewhale.wf.yaml

#
# Map an HTTP GET to your new workflow function:
#
$ fission route create --method GET --url /fortunewhale --function fortunewhale
#
# Invoke the workflow with an HTTP request:
#
$ curl ${FISSION_ROUTER}/fortunewhale
``` 

This last command, the invocation of the workflow, should return a whale saying something wise 

```
 ______________________________________
/ Anthony's Law of Force:              \
|                                      |
\ Don't force it; get a larger hammer. /
 --------------------------------------
    \
     \
      \
                    ##         .
              ## ## ##        ==
           ## ## ## ## ##    ===
       /"""""""""""""""""\___/ ===
      {                       /  ===-
       \______ O           __/
         \    \         __/
          \____\_______/
```

所以这里都发生了些什么？
让我们看一看 workflow 的组成：

```yaml
# This whale shows off a basic workflow that combines both Fission Functions (fortune, whalesay) and internal functions (noop)
apiVersion: 1
output: WhaleWithFortune
tasks:
  InternalFuncShowoff:
    run: noop

  GenerateFortune:
    run: fortune
    requires:
    - InternalFuncShowoff

  WhaleWithFortune:
    run: whalesay
    inputs: "{$.Tasks.GenerateFortune.Output}"
    requires:
    - GenerateFortune
```

你所看到的是 YAML 格式的 workflow 定义，名字为 fortunewhale
一个 workflow 有多个任务组成。他们是许多需要完成的步骤。
每一个任务有独特的标志，例如 `GenerateFortune`。它可以在 `run` 字段中引用。
作为选项，可以定义 `inputs` 字段允许设置任务的输出。
`requires` 定义了哪些任务结束后才开始当前任务。
最后你会注意到顶部的字段 `output`。这里设置将某个 task 的输出作为整个 workflow 的输出。

在这个例子中 `fortunewhale` workflow 由 3 个任务组成。

`InternalFuncShowoff -> GenerateFortune -> WhaleWithFortune`

首先，任务 `InternalFuncShowoff` 运行 `noop`。这是一个内置于 workflow engine 的 internal function 。Internal functions 运行在 workflow engine，更快更高效。所以轻量级的 funciton ，比如逻辑或过程控制就是很好的使用场景。除了使用预设的内置 fucntion ，也可以自定义，跟普通的 function 一样。

当 `InternalFuncShowoff` 结束之后，`GenerateFortune` 的 `requires` 已经被满足。他将运行 `fortune` function 。他将会随机输出一段名言。

当 `GenerateFortune` 结束后，`WhaleWithFortune` 任务将会开始。这个任务在 `input` 中用 js 表达式来引用 `GenerateFortune` 的输出。
在任务的 inputs 中你可以引用 workflow 中的任意内容，比如 输出，输入，任务定义或者一个常量。
workflow engine 调用 `whalesay` function ，传入 inputs ，然后输出 outputs。

最后，当所有任务结束。workflow engine 用顶层 `output` 字段取出 `WhaleWithFortune` 的输出作为整个 workflow 的输出，返回给用户。

Workflow 可以作为 function 存在于另一个 workflow 中的 run 字段里。

### 下一步

为了了解更多，请参考 workflow 章节。

或者参考范例