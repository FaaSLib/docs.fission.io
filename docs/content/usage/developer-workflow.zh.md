---
title: "Source Code Organization and Your Development Workflow"
date: 2017-12-01T18:01:57-08:00
weight: 48
---

你已经用自己喜欢的语言写了一个 function。然后跑在 Fission 中。下一步呢？

如何管理大量的 function 源码呢？如何自动部署，版本控制，自动测试等等？

这些问题的答案都追溯到同一个问题，如何定义一个应用。

## Declarative Specifications

为了替代调用 Fission 命令行。你可以用 YAML 文件定义你的 function。这比命令行脚本要好。因为命令行是用户交互界面，而不是程序交互界面。

你将会在版本控制系统里追踪这些 yaml 文件。Fission 提供命令行工具来生成这些 specification 文件，并 validating them, 应用 them。

apply specification 是什么意思？他意味着让 specification 生效，找出集群需要作出的修改，更新他们以保证和 specification 一致。

应用 fission spec 可以通过下面这些步骤。

* 资源只存在于 specification 中，并没有在集群中创建。
* 资源在 specification 和集群中的不同。
* 删除时之删除相关的资源

应用动作时 idempotent 幂等的。

## 用法总结

声明应用的 specification 共分三步：

1. 初始化一个目录: `fission spec init`
1. 生成一些 YAMLs 文件： `fission function create --spec ...`
1. 应用到集群中: `fission spec apply --wait`

持续观察部署过程 `fission spec apply --watch`

我们将会在随后的 tutorial 中看到所有这些命令的例子。

## Tutorial

这个教程将会假设你已经安装了 Fission ，以及保证基础功能都正常。

我们将会部署一个小的计算应用。他将会使用 python environment 和两个 function。他们都会用 Yaml 文件来声明。

### Make an empty directory

`mkdir spec-tutorial & cd spec-tutorial`

### 初始化 spec 目录

`fission spec init`

这将会创建 `specs/` 目录。你将会看到 `fission-config.yaml` 文件。这个文件将会有一个 unique ID 。所有在集群中创建的对象都将会在注释中携带这个 unique ID。

### 安装一个 a python environment

```shell
fission env create \
    --spec \
    --name python \
    --image fission/python-env:0.11.0 \
    --builder fission/python-builder:0.11.0
```

这个命令将会创建文件 `specs/env-python.yaml`。

### Code two functions

一个用来返回 web 表单。

```html
def main():
    return """
        <html>
            <body>
                <form action="/calculate" method="GET">
                    <input name="num_1"/>
                    <input name="num_2"/>
                    <input name="operator"/>
                    <button>Calculate</button>
                </form>
            </body>
        </html>
    """
```

该表单接受一个简单的数学表达式。当被触发时，他将会向第二个 func 发送请求，从而完成计算。

```python
def main():
    num_1 = int(request.form['num_1'])
    num_2 = int(request.form['num_2'])
    operator = request.form['operator']

    if operator == '+':
        result = num_1 + num_2
    elsif operator == '-':
        result = num_1 - num_2
        
    return "%s %s %s = %s" % (num_1, operator, num_2, result)
```

### Create specs for these functions

让我们为这些 function 创建 specification 。用它来定义 function name, where the code live, 与 env 的关系。

```shell
$ fission function create \
    --spec \
    --name calc-form \
    --env python \
    --src form.py \
    --entrypoint form.main

$ fission function create \
    --spec \
    --name calc-eval \
    --env python \
    --src calc.py \
    --entrypoint calc.main
```

然后就可以看到两个 YAML 文件。 `specs/function-calc-form.yaml`, `specs/function-calc-eval.yaml`.

### Create HTTP trigger specs

```shell
$ fission route create \
    --spec \
    --method GET \
    --url /form \
    --function calc-form 
    
$ fission route create \
    --spec \
    --method GET \
    --url /eval \
    --function calc-eval 
```

这将会创建两个 YAML 文件。文件里定义了发往 `/form`, `/eval` 的请求会调用 function `calc-form`, `calc-eval`。

### Validate your specs

检查重名。检查资源之间的关联。

`fission spec validate`

### Apply

简单的部署 environment, functions, HTTP triggers

`fission spec apply --wait`

### 测试

`fission function test --name calc-form`

更完整的测试请使用浏览器。如果不知道 Fission router 的地址请查询对应 k8s service 的地址。

`kubectl -n fission get service router`

### Modify the function and re-deploy it

让我们更改一下 function 

```python

    ...

    elsif operator == '*':
        result = num_1 * num_2
    
    ...
```

在 calc.py 中加入如上代码。

应用更改时只需要如下命令。

`fission spec apply --wait`

将会有如下输出

```
1 archive update: calc-eval-xyz
1 package update: calc-eval-xyz
1 function update: calc-eval
```

更新已经成功。可以测试新的操作符号 `*`。

### Add dependencies to the function

让我们增加一个 pip `requirements.txt` 来增加依赖库。这让你可以在 function 中增加 import

创建 `requirements.txt` 

修改 `specs/function-<name>.yaml` 中的 `ArchiveUploadSpec`。

再次应用。

`fission spec apply --wait`

这条命令发现有一个 function 发生变化。然后上传源代码到集群。等待集群应用新的源代码。

## 这些都是如何工作的

k8s 用资源的概念管理各种状态。`Deployment, pod, Services` 都是资源。他们描述一个状态。然后 k8s 尽可能保证达到这种状态。

k8s 可以用 CRD 来扩展资源类型。Fission 在 k8s 之上用自定义资源部署 functions, environments, trigger。用如下命令可以看到相关的资源类型。`kubectl get customeresourcedefinitions`, `kubectl get function.fission.io`

specs 目录里就是一组经过配置的资源。每个 YAML 文件包含了一个或多个资源。他们用 `---` 来分割。资源可以是 `functions`, `environments`, `triggers`。

有一个特殊的资源， `ArchiveUploadSpec`。事实上他不是种资源，只是在 YAML 中看起来像一种资源。他只是用来配置和命名将要被上传到集群中的文件。

`fission spec apply` 用 `ArchiveUploadSPec` 来在本地创建 Archive 然后上传他们。specs 里边用类似 `archive://` 一样的地址来引用这些 archive。这些都不是真正的 URL。在 archive 被上传之后，这些 URL 将会被替换成 http url。在集群里，Archives 将会用 checksums 来追踪。只有 checksum 改变时，cli 才会上传他们。

## Usage Reference

```
NAME:
   fission spec - Manage a declarative app specification

USAGE:
   fission spec command [command options] [arguments...]

COMMANDS:
     init      Create an initial declarative app specification
     validate  Validate Fission app specification
     apply     Create, update, or delete Fission resources from app specification
     destroy   Delete all Fission resources in the app specification
     helm      Create a helm chart from the app specification

OPTIONS:
   --help, -h  show help
   
```

### fission spec init

```
NAME:
   fission spec init - Create an initial declarative app specification

USAGE:
   fission spec init [command options] [arguments...]

OPTIONS:
   --specdir value  Directory to store specs, defaults to ./specs
   --name value     (optional) Name for the app, applied to resources as a Kubernetes annotation
   
```

### fission spec validate

```
NAME:
   fission spec validate - Validate Fission app specification

USAGE:
   fission spec validate [command options] [arguments...]

OPTIONS:
   --specdir value  Directory to store specs, defaults to ./specs
```

### fission spec apply

```
NAME:
   fission spec apply - Create, update, or delete Fission resources from app specification

USAGE:
   fission spec apply [command options] [arguments...]

OPTIONS:
   --specdir value  Directory to store specs, defaults to ./specs
   --delete         Allow apply to delete resources that no longer exist in the specification
   --wait           Wait for package builds
   --watch          Watch local files for change, and re-apply specs as necessary
```

### fission spec destroy

```
NAME:
   fission spec destroy - Delete all Fission resources in the app specification

USAGE:
   fission spec destroy [command options] [arguments...]

OPTIONS:
   --specdir value  Directory to store specs, defaults to ./specs

```
